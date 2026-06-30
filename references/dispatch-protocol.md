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
    remaining = dispatch_info.get("remaining_manifests", [])
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
        section_path = output_path.replace(".md", "-section.json")
        facts_path = output_path.replace(".md", "-facts.json")

        # Layer 1: File existence + size check
        # Layer 2: JSON validity check (section + facts sidecars)
        # Layer 3: File stability check (size not growing for 8s)
        elapsed = 0
        completed = False
        while elapsed < MAX_COLLECT_RETRIES * COLLECT_RETRY_INTERVAL:
            if (Path(output_path).exists() and Path(output_path).stat().st_size > 500
                and _json_valid(section_path)
                and _json_valid(facts_path)
                and _file_stable(output_path, interval=8)):
                completed = True
                break
            sleep(COLLECT_RETRY_INTERVAL)
            elapsed += COLLECT_RETRY_INTERVAL

        if not completed:
            # Layer 4: Re-dispatch same role (needs_dispatch with same phase)
            log(f"Role {manifest['role']} incomplete after {elapsed}s, re-dispatching")
            # ... re-spawn sub-agent with same manifest

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
- `fact_store.json` / `sidecar.json` concurrent write conflicts → data loss

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

prepare(third call):
  role_A, B done → finds role_C not done → manifests = [role_C]
  has_more = True  (role D still pending)

prepare(fourth call):
  role_A, B, C done → manifests = [role_D]
  has_more = False  (last role)
  → kernel: next_phase = collect phase (advance)
```

### Role Completion Check

```python
def _role_outputs_complete(task_dir: Path, role_slug: str) -> bool:
    """Check if a role has produced all 3 required files."""
    outputs_dir = task_dir / "outputs"
    md_path = outputs_dir / f"bp_phase2_{role_slug}.md"
    facts_path = outputs_dir / f"bp_phase2_{role_slug}-facts.json"
    section_path = outputs_dir / f"bp_phase2_{role_slug}-section.json"

    if not md_path.exists() or md_path.stat().st_size < 100:
        return False
    if not facts_path.exists() or not _json_valid(facts_path):
        return False
    if not section_path.exists() or not _json_valid(section_path):
        return False
    return True
```

## The 4-Layer Defense (Output Collection)

Sub-agents must write 3 files: `.md` → `-facts.json` → `-section.json`. The collect phase validates all 3 through 4 layers:

| Layer | Mechanism | What it catches |
|-------|-----------|-----------------|
| **1. Soft constraint** | Dispatch instruction mandates 3-file output + sequential-only | Coordinator-level compliance |
| **2. Hard constraint** | `_role_outputs_complete` checks existence + size + JSON validity + `_file_stable` (8s no-growth) | Partial writes, corrupt JSON, in-progress writes |
| **2.5. Retry buffer** | `COLLECT_RETRY_COUNT=40` × `COLLECT_RETRY_INTERVAL=30s` = **20 min total timeout** | Slow sub-agents, API delays |
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

Each sub-agent prompt is assembled from 3 sources:

```
{system_prompt from instruction store}     ← Hot-loaded from instruction_store_{profile}/role.md
+
{brief content}                            ← Generated per-wave, includes input file paths
+
{common tool guide}                        ← Shared instructions for all sub-agents
```

### Manifest Structure

```json
{
    "manifest_version": "1.0",
    "task_id": "job_123",
    "role": "market_analysis",
    "slug": "market",
    "label": "job_123-bp-phase2-market",
    "system_prompt": "...(full text from instruction store)...",
    "brief_path": "/path/to/brief_market.md",
    "brief_content_preview": "...(first 2000 chars of brief)...",
    "output_path": "/path/to/outputs/bp_phase2_market.md",
    "timeout": 900,
    "thinking": "high",
    "dispatch_mode": "team_async",
    "mode": "bypassPermissions",
    "subagent_type": "general-purpose",
    "team_name_template": "bp-{task_id}",
    "connectorIds": ["qcc-company", "qcc-risk"],
    "created_at": "2026-06-30T13:20:00",
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
| `manifests` | Full wave roles | Only roles with failed claims |
| `system_prompt` | Research/analysis | Targeted fix instruction |
| `has_more` | Per sequential | Per remaining repair manifest |
| Max retries | N/A (determined by gate) | Wave gate: 1, Claim coverage: 2, Synthesis: 1 |
| On exhausted | N/A | Degrade to WARN and continue |

## Dispatch Pattern: Multi-Wave with Shared State Refresh

This is the **only** dispatch pattern in this skill, extracted from the production BP pipeline (33 phases, 4 waves). Do not use single-wave or non-refresh patterns.

```
Phase 01-07: Intake → Plan → Presearch → Fact Store → Shared State Init
Phase 08-09: Wave 1 Prepare → Collect (4 foundation roles, sequential)
Phase 10:    Wave 1 Evidence Gate (repair if FAIL, max 1 retry)
Phase 11-12: Fact Store Merge → Shared State Refresh (Wave 2 reads this)
Phase 13-14: Wave 2 Prepare → Collect (revenue validation, sequential)
Phase 15:    Wave 2 Evidence Gate
Phase 16-17: Wave 3 Prepare → Collect (competition + valuation, sequential)
Phase 18-19: Wave 3 Evidence Gate → Shared State Refresh
Phase 20-21: Wave 4 Prepare → Collect (deal breaker, sequential)
Phase 22-23: Wave 4 Evidence Gate → Shared State Refresh
Phase 24-26: Claim Coverage → Cross-dimension → Section Package
Phase 27-28: Synthesis Prepare → Collect
Phase 29-33: Debate → Assembly → Readability → Judgment → Delivery
```

### Mandatory Structural Rules

1. **Prepare + Collect must be separate phases** — never combine dispatch and collect into one phase. Prepare returns `needs_dispatch` (pause), Collect validates outputs (resume).
2. **Every wave has: Prepare → Collect → Evidence Gate** — gate checks claims against facts, triggers repair if needed.
3. **Shared State Refresh after each significant wave** — Wave N+1 sub-agents read what Wave N discovered. See [shared-state.md](shared-state.md).
4. **Fact Store Merge before Shared State Refresh** — sidecar facts → central store, then refresh shared state.
5. **Sequential dispatch within each wave** — `has_more` drives the one-at-a-time loop.

## Team Lifecycle Rules

1. **Create team before first dispatch**: `team_create(team_name=...)`
2. **All sub-agents use same team**: `Agent(team_name=..., mode='bypassPermissions')`
3. **Sequential dispatch only**: ONE sub-agent at a time, wait for completion
4. **Poll output files actively**: Check existence + size + JSON validity + stability
5. **Retry on timeout**: Collect has built-in retry (40 × 30s = 20 min)
6. **Repair on gate failure**: Gate returns repair manifests, dispatch repair sub-agents
7. **Skip on persistent failure**: After max repair retries, degrade to WARN and continue
8. **Delete team after delivery**: `team_delete()`
9. **Shutdown cleanup**: Remove exited members from team config before continuing
