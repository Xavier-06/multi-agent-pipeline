# Coordinator Dispatch Protocol

## Overview

The Coordinator is the AI agent (main process) that drives the pipeline. When a phase returns `needs_dispatch`, the Coordinator enters the dispatch loop: spawn sub-agents → poll outputs → advance waves → repeat.

## Dispatch Loop (Pseudocode)

```python
# 1. Create team
team_create(team_name=f"{profile}-{task_id}")

# 2. Main dispatch loop
MAX_COLLECT_RETRIES = 40          # Number of poll cycles
COLLECT_RETRY_INTERVAL = 30       # Seconds between polls
# Total timeout: 40 × 30 = 1200s = 20 min

while True:
    result = kernel.execute(job_id, start_phase=next_phase)

    if not result.get("needs_dispatch"):
        break  # Pipeline advanced past dispatch phases or completed

    dispatch_info = result["dispatch_info"]
    manifests = dispatch_info["manifests"]        # ALWAYS a list of 1 (sequential mode)
    has_more = dispatch_info.get("has_more", False)
    is_repair = dispatch_info.get("is_repair", False)

    for manifest_path in manifests:
        manifest = json.loads(Path(manifest_path).read_text())
        system_prompt = manifest["system_prompt"]     # Loaded from instruction store
        brief_path = manifest.get("brief_path", "")
        output_path = manifest["output_path"]
        connector_ids = manifest.get("connectorIds", [])

        # Spawn sub-agent (async, via team)
        Agent(
            name=manifest["role"],
            team_name=f"{profile}-{task_id}",
            mode="bypassPermissions",
            description=manifest["role"],
            prompt=system_prompt,
            connectorIds=connector_ids,
        )

    # 3. Collect: poll for output files (4-layer defense)
    for manifest_path in manifests:
        manifest = json.loads(Path(manifest_path).read_text())
        output_path = manifest["output_path"]
        data_path = output_path.with_suffix("").with_name(output_path.stem + "-data.json")
        meta_path = output_path.with_suffix("").with_name(output_path.stem + "-meta.json")

        # Layer 1: File existence + size check
        # Layer 2: JSON validity check (data + meta sidecars)
        # Layer 3: File stability check (size not growing for 8s)
        elapsed = 0
        completed = False
        while elapsed < MAX_COLLECT_RETRIES * COLLECT_RETRY_INTERVAL:
            if (Path(output_path).exists() and Path(output_path).stat().st_size > 500
                and _json_valid(data_path)
                and _json_valid(meta_path)
                and _file_stable(output_path, interval=8)):
                completed = True
                break
            sleep(COLLECT_RETRY_INTERVAL)
            elapsed += COLLECT_RETRY_INTERVAL

        if not completed:
            # Layer 4: Re-dispatch same role (needs_dispatch with same phase)
            log(f"Role {manifest['role']} incomplete after {elapsed}s, re-dispatching")

    # 4. Advance: has_more determines next_phase
    if has_more:
        next_phase = result["paused_after"]   # Same phase → dispatch next role
    else:
        next_phase = result["next_phase"]     # Advance to collect phase

# 5. Cleanup
team_delete()
```

## Sequential Dispatch: Why and How

### Problem
Parallel dispatch causes:
- API 429 rate limits
- Shared file concurrent write conflicts → data loss

### Solution
Return **ONE manifest at a time** with `has_more` signaling:

```
prepare(first call):
  finds role_A not done → dispatch_info.manifests = [role_A]
  has_more = True  (roles B, C, D still pending)
  → kernel: next_phase = current phase (re-run)

prepare(second call):
  role_A done → finds role_B not done → manifests = [role_B]
  has_more = True  (roles C, D still pending)

prepare(last call):
  all prior done → manifests = [role_D]
  has_more = False  (last role)
  → kernel: next_phase = collect phase (advance)
```

## Per-Item Dispatch: When One Role Needs Splitting

### Problem
A single role may need to process a dataset too large for one sub-agent's context window. Example: deep_reader must read 284 papers across 7 sub-topics — even with batching, a single agent session accumulates ~80K tokens of notes + tool calls and blows up.

### Solution: Split into N sub-agents, dispatched sequentially via `has_more`

Instead of 1 agent processing all items, the prepare phase iterates through items (sub-topics, batches, etc.), dispatching one sub-agent per item:

```
prepare(first call):
  finds item_0 not done → manifests = [{role: "deep_reader", slug: "deep_reader_sub1", ...}]
  has_more = True  (items 1-6 still pending)

prepare(second call):
  item_0 done → manifests = [{role: "deep_reader", slug: "deep_reader_sub2", ...}]
  has_more = True  (items 2-6 still pending)

...

prepare(last call):
  items 0-5 done → manifests = [{role: "deep_reader", slug: "deep_reader_sub7", ...}]
  has_more = True  (tech_strategist still pending)

prepare(final call):
  all deep_readers done → manifests = [{role: "tech_strategist", ...}]
  has_more = False  → advance to collect phase
```

### Key design rules

1. **Each sub-agent handles a bounded slice** — 1 sub-topic, 1 batch, 1 file set. Keep per-agent context under ~20K tokens of accumulated notes.
2. **Unique slug per instance** — `deep_reader_sub1`, `deep_reader_sub2`, etc. The slug is used for team member naming and output file prefix.
3. **Per-instance output files** — Each sub-agent writes to its own file: `sub_topic_1_reading_notes.json`, `sub_topic_2_reading_notes.json`, etc. Never share output files between instances.
4. **Collect merges results** — After all instances complete, a merge phase (or the collect phase itself) aggregates individual outputs into the unified result that downstream phases expect.
5. **has_more chains items + next role** — The prepare phase tracks both "are all items done?" and "is the next role done?" in a single loop.

### Prepare phase template

```python
def _run_wave2_dispatch_prepare(runtime_root, job_ctx):
    task = _task_dir(runtime_root, job_ctx)
    items = load_items(task)  # e.g., reading_tasks from shared_state

    # Find first incomplete item
    for idx, item in enumerate(items):
        output_file = task / f"item_{idx+1}_output.json"
        if not _output_complete(output_file):
            # Dispatch this item
            manifest = build_item_manifest(item, idx, ...)
            return {
                "ok": True,
                "needs_dispatch": True,
                "dispatch_info": {
                    "type": "per_item_dispatch",
                    "manifests": [str(manifest_path)],
                    "has_more": (idx < len(items) - 1) or next_role_pending,
                    "current_item": idx + 1,
                    "total_items": len(items),
                },
                "instruction": f"Dispatch sub-agent for item {idx+1}/{len(items)}...",
                "next_phase": "phase_collect",
            }

    # All items done → dispatch next role or complete
    if not next_role_done:
        return build_next_role_manifest(...)

    return {"ok": True, "needs_dispatch": True,
            "dispatch_info": {"type": "complete", "manifests": [], "has_more": False}}
```

### When to use per-item dispatch

| Condition | Use per-item? |
|-----------|--------------|
| Single role processes 50+ documents | **Yes** — split by document batch or sub-topic |
| Single role processes 5-10 items | No — single agent is fine |
| Context accumulation risk (>30K tokens of notes) | **Yes** — each sub-agent gets clean context |
| Items are independent (no cross-item dependency) | **Yes** — this is the ideal case |
| Items have ordering dependency | No — use sequential single agent |

### Role Completion Check

```python
def _role_outputs_complete(task_dir: Path, role_slug: str) -> bool:
    """Check if a role has produced all 3 required files.

    Customize file names for your domain's output contract.
    """
    outputs_dir = task_dir / "outputs"
    md_path = outputs_dir / f"{role_slug}.md"
    data_path = outputs_dir / f"{role_slug}-data.json"
    meta_path = outputs_dir / f"{role_slug}-meta.json"

    if not md_path.exists() or md_path.stat().st_size < 100:
        return False
    if not data_path.exists() or not _json_valid(data_path):
        return False
    if not meta_path.exists() or not _json_valid(meta_path):
        return False
    return True
```

## The 4-Layer Defense (Output Collection)

Sub-agents write 3 files: `.md` → `-data.json` → `-meta.json`. The collect phase validates through 4 layers:

| Layer | Mechanism | What it catches |
|-------|-----------|-----------------|
| **1. Soft constraint** | Dispatch instruction mandates 3-file output + sequential-only | Coordinator-level compliance |
| **2. Hard constraint** | `_role_outputs_complete` checks existence + size + JSON validity + `_file_stable` (8s no-growth) | Partial writes, corrupt JSON, in-progress writes |
| **2.5. Retry buffer** | Configurable retry count × interval = total timeout | Slow sub-agents, API delays |
| **3. Semi-auto** | Incomplete roles trigger `needs_dispatch` re-dispatch in the collect phase itself | Sub-agent crashes, timeout |

### File Stability Check

Two implementations for different call sites:

**Polling variant** (used by Coordinator's collect loop — blocks intentionally):
```python
def _file_stable(path: Path, interval: int = 8) -> bool:
    """Return True if file size hasn't changed in `interval` seconds.

    Used in the Coordinator's collect retry loop where we're actively
    polling. The sleep is intentional — we're waiting for the sub-agent
    to finish writing.
    """
    if not path.exists():
        return False
    size1 = path.stat().st_size
    time.sleep(interval)
    size2 = path.stat().st_size
    return size1 == size2 and size1 > 0
```

**Non-blocking variant** (used by `_role_outputs_complete` in prepare phase):
```python
_STABLE_THRESHOLD_SEC = 8

def _file_stable_check(path: Path, threshold: int = _STABLE_THRESHOLD_SEC) -> bool:
    """Return True if file hasn't been modified in `threshold` seconds.

    Used in prepare phase to check if a role's outputs are complete.
    Non-blocking — just reads mtime, no sleep.
    """
    if not path.exists():
        return False
    return time.time() - path.stat().st_mtime >= threshold
```

The polling variant is correct for collect loops (we're already sleeping between polls). The non-blocking variant avoids an 8-second stall when checking role completion status.

### JSON Validity Check

```python
def _json_valid(path: Path) -> bool:
    """Return True if file exists and contains valid JSON."""
    if not path.exists():
        return False
    try:
        json.loads(path.read_text(encoding="utf-8"))
        return True
    except (json.JSONDecodeError, ValueError):
        return False
```

### Collect Retry Loop (Complete Implementation)

The collect phase wraps the 4-layer defense in a retry loop with progress logging:

```python
def _collect_with_retry(wave_name, check_fn, job_id, outputs_dir,
                        retry_count=40, retry_interval=30):
    """Poll for sub-agent outputs with retry buffer and progress detection.

    Args:
        wave_name: Label for logging (e.g., "wave1_collect")
        check_fn: Callable that returns (completed: list, incomplete: list)
        retry_count: Max poll cycles (default 40)
        retry_interval: Seconds between polls (default 30)
        Total timeout: retry_count × retry_interval = 1200s = 20 min
    """
    for attempt in range(1, retry_count + 1):
        completed, incomplete = check_fn()

        # Progress logging — helps debug stuck sub-agents
        log(f"[{wave_name}] attempt {attempt}/{retry_count}: "
            f"{len(completed)} complete, {len(incomplete)} pending")
        for role in incomplete:
            log(f"  ⏳ {role} — still waiting")

        if not incomplete:
            return {
                "ok": True,
                "mode": f"{wave_name}",
                "result": {
                    "completed": len(completed),
                    "attempts_used": attempt,
                },
            }

        if attempt < retry_count:
            sleep(retry_interval)

    # Exhausted all retries — Layer 3: return needs_dispatch for incomplete roles
    log(f"[{wave_name}] TIMEOUT after {retry_count} attempts. "
        f"Incomplete: {incomplete}")
    return {
        "ok": True,
        "needs_dispatch": True,
        "has_more": False,
        "mode": f"{wave_name}",
        "dispatch_info": {
            "incomplete_roles": incomplete,
            "reason": "collect_timeout",
            "attempts_exhausted": retry_count,
        },
        "instruction": f"Re-dispatch incomplete roles: {incomplete}. "
                       f"These sub-agents failed to produce valid output within "
                       f"{retry_count * retry_interval}s.",
    }
```

**Key design decisions**:
- **Progress detection**: Each cycle logs which roles are done vs pending. If a role stays "pending" for 10+ cycles, you know it's stuck.
- **Graceful timeout**: When retries exhaust, return `needs_dispatch` for incomplete roles rather than `ok: False`. This triggers Layer 3 re-dispatch instead of killing the pipeline.
- **Attempt tracking**: The result includes `attempts_used` so you can tell if a wave collected quickly (2 attempts) or barely made it (39 attempts).

## Sub-Agent Prompt Structure

Each sub-agent prompt is assembled from 3+ sources:

```
{system_prompt from instruction store}     ← Hot-loaded from instruction_store_{profile}/role.md
+
{brief content}                            ← Generated per-wave, includes input file paths
+
{common tool guide}                        ← Shared instructions for all sub-agents
+
{domain-specific appendix}                 ← Optional: conclusion rules, stage guidance
```

### Manifest Structure

```json
{
    "manifest_version": "1.0",
    "task_id": "job_123",
    "role": "market_analysis",
    "slug": "market",
    "system_prompt": "...(full text from instruction store)...",
    "brief_path": "/path/to/brief_market.md",
    "brief_content_preview": "...(first 2000 chars of brief)...",
    "output_path": "/path/to/outputs/market.md",
    "timeout": 900,
    "connectorIds": ["github", "your-data-source"],
    "created_at": "2026-07-01T06:20:00",
    "status": "pending"
}
```

**Key**: The `system_prompt` field contains the COMPLETE instruction text — the Coordinator must use it as-is, not summarize or rewrite it.

## Dispatch Instruction Template

The phase handler returns an `instruction` field that tells the Coordinator exactly how to spawn the sub-agent:

```
MANDATORY: Read the manifest JSON file at '{manifest_path}'. Use the Agent tool with these EXACT parameters:
1. prompt = manifest's 'system_prompt' field (the FULL text, do NOT summarize)
2. name = '{role_name}'
3. team_name = '{team_name}'
4. mode = 'bypassPermissions'
5. connectorIds = manifest's 'connectorIds' field
Also pass the manifest's 'brief_content_preview' as context.
Do NOT write your own simplified prompt — the manifest system_prompt contains the complete instructions.
```

For sequential dispatch, append:
```
CRITICAL: Do NOT spawn multiple sub-agents. Only dispatch the ONE sub-agent described above.
Wait for it to complete before calling execute again.
```

## Repair Dispatch

When a gate FAIL triggers repair, the dispatch loop is identical but with these differences:

| Aspect | Normal Dispatch | Repair Dispatch |
|--------|----------------|-----------------|
| `is_repair` | `false` | `true` |
| `manifests` | Full wave roles | Only roles with quality issues |
| `system_prompt` | Analysis/production | Targeted fix instruction |
| `has_more` | Per sequential | Per remaining repair manifest |
| Max retries | N/A (determined by gate) | Configure per gate type |
| On exhausted | N/A | Degrade to WARN and continue |

## Dispatch Pattern: Multi-Wave with Shared State Refresh

The standard pattern for multi-wave pipelines:

```
Phase 01-05: Intake → Plan → Precompute → Evidence Store → Shared State Init
Phase 06-07: Wave 1 Prepare → Collect (N roles, sequential)
Phase 08:    Wave 1 Quality Gate (repair if FAIL)
Phase 09-10: Evidence Merge → Shared State Refresh (Wave 2 reads this)
Phase 11-12: Wave 2 Prepare → Collect
Phase 13:    Wave 2 Quality Gate
Phase 14:    Shared State Refresh
...
Phase N:     Coverage → Cross-Section Review → Assembly → Delivery
```

### Mandatory Structural Rules

1. **Prepare + Collect must be separate phases** — never combine dispatch and collect. Prepare returns `needs_dispatch` (pause), Collect validates outputs (resume).
2. **Every wave has: Prepare → Collect → Quality Gate** — gate checks output quality, triggers repair if needed.
3. **Shared State Refresh after each significant wave** — Wave N+1 sub-agents read what Wave N discovered. See [shared-state.md](shared-state.md).
4. **Evidence Merge before Shared State Refresh** — sidecar data → central store, then refresh shared state.
5. **Sequential dispatch within each wave** — `has_more` drives the one-at-a-time loop.

## Team Lifecycle Rules

1. **Create team before first dispatch**: `team_create(team_name=...)`
2. **All sub-agents use same team**: `Agent(team_name=..., mode='bypassPermissions')`
3. **Sequential dispatch only**: ONE sub-agent at a time, wait for completion
4. **Poll output files actively**: Check existence + size + JSON validity + stability
5. **Retry on timeout**: Collect has built-in retry buffer
6. **Repair on gate failure**: Gate returns repair manifests, dispatch repair sub-agents
7. **Skip on persistent failure**: After max repair retries, degrade to WARN and continue
8. **Delete team after delivery**: `team_delete()`
9. **Shutdown cleanup**: Remove exited members from team config before continuing
