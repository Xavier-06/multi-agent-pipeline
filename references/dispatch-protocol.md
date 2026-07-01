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

```python
def _file_stable(path: Path, interval: int = 8) -> bool:
    """Return True if file size hasn't changed in `interval` seconds."""
    if not path.exists():
        return False
    size1 = path.stat().st_size
    time.sleep(interval)
    size2 = path.stat().st_size
    return size1 == size2 and size1 > 0
```

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
