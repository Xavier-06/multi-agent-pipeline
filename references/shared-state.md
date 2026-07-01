# Shared State (Cross-Wave Context Passing)

## Problem

In a multi-wave pipeline, sub-agents in Wave 2+ need context from Wave 1's results. Without it, each wave works in isolation → redundant work, contradictory conclusions, no cumulative progress.

Three things need to cross wave boundaries:

1. **Progress snapshot** — what's been verified, what's still open, what failed
2. **Collected evidence** — all data points gathered so far, deduplicated
3. **Prior wave outputs** — the actual reports/files produced by earlier sub-agents

## Architecture: Three Layers

```
┌────────────────────────────────────────────────────┐
│  Layer 1: Progress Snapshot (domain-specific hub)  │
│  Machine-readable JSON + human-readable page       │
│  → Sub-agents read to know what's done/open        │
├────────────────────────────────────────────────────┤
│  Layer 2: Centralized Evidence Store               │
│  All sidecar data merged, deduplicated             │
│  → Quality gates + final assembly read this        │
├────────────────────────────────────────────────────┤
│  Layer 3: Prior Wave Outputs                       │
│  Full output files from completed waves            │
│  → Later-wave sub-agents read prior reports        │
└────────────────────────────────────────────────────┘
```

Each layer is independent. Your pipeline might need all three, or just Layers 1+3.

## Layer 1: Progress Snapshot

This is the **core pattern**. After each wave completes, rebuild a snapshot from ALL available outputs.

### Skeleton

```python
def build_shared_state(task_dir: Path, after_wave: int = 0) -> dict:
    """Rebuild progress snapshot from all available sources.

    Called by init phase (before Wave 1) and refresh phase (after each wave).
    You MUST implement this for your domain — the skeleton below shows
    the general shape, but the fields depend on what your pipeline tracks.
    """
    # 1. Load your planning artifact (whatever defines "what needs to be done")
    plan = load_json(task_dir / "your_plan.json")

    # 2. Collect ALL sub-agent outputs from completed waves
    #    Prefer a merged summary; fallback to scanning sidecars on disk
    outputs = _load_merged_outputs(task_dir) or _scan_sidecars(task_dir)

    # 3. For each tracked item, determine current status
    item_status = _resolve_all_statuses(plan, outputs)

    # 4. Extract open items from sub-agent reports
    open_items = _extract_open_items(outputs)

    # 5. (Optional) Derive a recommendation — domain-specific
    verdict = _derive_verdict(item_status)

    return {
        "schema_version": "your_shared_state.v1",
        "after_wave": after_wave,
        "item_status": item_status,
        "open_items": open_items,
        "verdict": verdict,
        "wave_history": _build_wave_history(task_dir, after_wave),
    }
```

### Key Design Decisions

**Status lifecycle**: define values for your domain. Examples:

| Domain | Possible statuses |
|--------|------------------|
| Due diligence | `planned → not_addressed → supported / partially_supported / contradicted` |
| Code review | `pending → reviewed → approved / issues_found / needs_rework` |
| Legal analysis | `open → researched → confirmed / disputed / unresolvable` |
| Generic | `pending → done / partial / failed` |

**Conservative merge**: when multiple sub-agents touch the same item, keep the *worst* status. Prevents false confidence when evidence is mixed.

```python
def _conservative_merge(status_a: str, status_b: str, priority: dict) -> str:
    """Keep the worse (more conservative) status."""
    return status_a if priority[status_a] <= priority[status_b] else status_b
```

### Refresh Placement

**Always refresh AFTER the quality gate, not before.** This ensures the snapshot only reflects verified outputs.

```
init phase:     shared_state_init        (before Wave 1, skeleton from plan)
Wave 1:         dispatch → collect → quality_gate
                sidecar_merge            (collect sidecars → central store)
refresh phase:  shared_state_refresh     (full rebuild from disk)  ← HERE
Wave 2:         dispatch → collect → quality_gate                  ← reads refreshed state
refresh phase:  shared_state_refresh
...
```

### Fuzzy Matching for Cross-Wave Item Resolution

When Wave 1 sub-agents tag their outputs with item keys (e.g., `sub_topic`), the keys may not exactly match the plan's item names. The refresh phase must handle this:

```python
def resolve_item_key(agent_output_key: str, plan_items: list[str]) -> str | None:
    """Match agent output to plan item with fuzzy fallback.

    Priority:
    1. Exact match (case-insensitive)
    2. Substring match (plan item contained in agent key or vice versa)
    3. Keyword overlap (tokenize both, check ≥50% keyword overlap)
    4. None (unmatched — record as "general" bucket)
    """
    key_lower = agent_output_key.lower().strip()

    # Exact match
    for item in plan_items:
        if key_lower == item.lower():
            return item

    # Substring match
    for item in plan_items:
        if item.lower() in key_lower or key_lower in item.lower():
            return item

    # Keyword overlap
    key_tokens = set(key_lower.replace("-", " ").replace("_", " ").split())
    key_tokens = {t for t in key_tokens if len(t) >= 2}
    best_match, best_ratio = None, 0
    for item in plan_items:
        item_tokens = set(item.lower().replace("-", " ").replace("_", " ").split())
        item_tokens = {t for t in item_tokens if len(t) >= 2}
        if not item_tokens:
            continue
        overlap = len(key_tokens & item_tokens) / max(len(item_tokens), 1)
        if overlap > best_ratio and overlap >= 0.5:
            best_match, best_ratio = item, overlap

    return best_match  # May be None — unmatched items go to "general"
```

**Why this matters**: Academic search agents tag papers with `sub_topic` fields derived from search results. These may differ from the plan's canonical names (e.g., agent writes "solid state electrolyte materials" but plan says "硫化物固态电解质"). Without fuzzy matching, papers get orphaned into empty buckets and downstream phases see `total_docs=0` for items that actually have data.

**Which waves need a refresh?**

| Condition | Refresh? | Reason |
|-----------|----------|--------|
| Wave with 3+ roles | Yes | Lots of new information |
| Wave with 1-2 narrow roles | Optional | May not justify the cost |
| Last wave before quality chain | No | Quality chain reads sidecars directly |

### Human-Readable Page

Alongside the JSON, generate a Markdown dashboard for sub-agents to scan quickly. Typical sections:

1. Current verdict / status summary
2. Tracked items status table
3. Confirmed evidence table
4. Open items / gaps table
5. Wave handoff instructions (what to reuse, what to avoid, what to verify next)

```python
def render_shared_page(state: dict) -> str:
    """Render state JSON as Markdown dashboard. Customize for your domain."""
    ...
```

## Layer 2: Centralized Evidence Store

If your sub-agents produce structured data (not just reports), merge their sidecars into one store after each wave.

```python
def merge_sidecars(outputs_dir: Path, task_dir: Path) -> dict:
    """Scan outputs_dir for sidecar files, deduplicate, write central store.

    Design rules:
    - Malformed sidecars: SKIP, don't block (one bad file ≠ pipeline death)
    - Dedup by item ID across sidecars
    - ADDITIVE: each call rebuilds from all sidecars on disk
    """
    items = []
    seen_ids = set()
    malformed = []

    for path in outputs_dir.glob("*-data.json"):
        try:
            data = json.loads(path.read_text())
            for item in data.get("items", []):
                if item["id"] not in seen_ids:
                    seen_ids.add(item["id"])
                    items.append(item)
        except (json.JSONDecodeError, KeyError):
            malformed.append(str(path))

    write_json(task_dir / "evidence_store.json", {
        "items": items,
        "source_files": len(list(outputs_dir.glob("*-data.json"))),
        "malformed": malformed,
    })
```

**Why additive?** Each merge rebuilds from scratch. New wave sidecars are on disk → automatically included. No inter-merge state to track.

**Do you need this layer?** If your sub-agents only produce reports (Markdown), skip it. You only need Layer 2 when sub-agents produce structured data that quality gates or final assembly need to query.

## Layer 3: Prior Wave Outputs

The simplest layer — just file paths. Later-wave sub-agents read the actual reports from earlier waves.

```python
def _prior_wave_outputs(completed_waves: list[dict], outputs_dir: Path) -> dict[str, str]:
    """Collect output file paths from all completed waves.

    CUMULATIVE: Wave 4 sees Wave 1 + 2 + 3 outputs, not just Wave 3.
    """
    result = {}
    for wave in completed_waves:
        for role, slug in wave["roles"].items():
            path = outputs_dir / f"{slug}.md"
            if path.exists():
                result[role] = str(path)
    return result
```

### Injection into Briefs

Combine all three layers into each sub-agent's brief:

```python
def _build_brief_with_shared_state(sub, task_dir, outputs_dir, completed_waves):
    shared = {
        "progress_page": str(task_dir / "shared_state_page.md"),   # Layer 1
        "progress_json": str(task_dir / "shared_state.json"),
        "evidence_store": str(task_dir / "evidence_store.json"),   # Layer 2 (if applicable)
    }
    prior_outputs = _prior_wave_outputs(completed_waves, outputs_dir)  # Layer 3

    # Merge into brief — progress page listed FIRST so sub-agent reads it first
    sub["shared_inputs"] = {**shared, **prior_outputs}

    # Brief builder adds these to the "Input Files" section:
    # - shared_state_page.md    ← listed first, read first
    # - shared_state.json
    # - evidence_store.json
    # - wave1_role_a.md         ← prior wave output
    # - wave1_role_b.md
```

The brief should explicitly instruct: "Read the shared state page FIRST, then prior wave outputs, then raw materials."

## Wiring Into Profile Phase Handlers

```python
def _run_shared_state_init(runtime_root, job_ctx):
    """Before Wave 1: build skeleton state from plan."""
    task_dir = _task_dir(runtime_root, job_ctx)
    state = build_shared_state(task_dir, after_wave=0)
    _write_state_artifacts(task_dir, state)  # JSON + page + open items
    return {"ok": True, "mode": "shared_state_init"}

def _run_shared_state_refresh(runtime_root, job_ctx, after_wave):
    """After Wave N: full rebuild from all outputs on disk."""
    task_dir = _task_dir(runtime_root, job_ctx)
    state = build_shared_state(task_dir, after_wave=after_wave)
    _write_state_artifacts(task_dir, state)
    return {"ok": True, "mode": "shared_state_refresh", "result": {"after_wave": after_wave}}
```

Register these in your profile's `phase_handlers`:

```python
phase_handlers = {
    ...
    "phase05_shared_state_init": lambda jc: _run_shared_state_init(runtime_root, jc),
    "phase12_shared_state_refresh_w1": lambda jc: _run_shared_state_refresh(runtime_root, jc, after_wave=1),
    "phase19_shared_state_refresh_w3": lambda jc: _run_shared_state_refresh(runtime_root, jc, after_wave=3),
    ...
}
```

## Implementation Checklist

- [ ] Define your tracked items and status values (what does your pipeline track?)
- [ ] Implement `build_shared_state()` for your domain
- [ ] Implement `render_shared_page()` — Markdown dashboard
- [ ] (If needed) Implement `merge_sidecars()` — sidecar scanning + dedup
- [ ] Insert refresh phases after quality gates (always post-gate)
- [ ] Inject shared state into briefs (progress page listed first)
- [ ] Add conservative merge rule for conflicting results
- [ ] Add cumulative prior wave output injection (Layer 3)

## Why Not Just Pass the Prior Wave Output?

You might think: "Why not just tell Wave 2 to read Wave 1's output?" Three reasons:

1. **Wave 2 doesn't know what to look for.** Without a structured snapshot, the sub-agent has to parse the full report to find what's relevant to its dimension.
2. **No progress tracking.** Without explicit status per item, you can't tell what's still open.
3. **No conflict detection.** Without a centralized view, contradictory findings across waves stay hidden until final assembly — too late to fix.
