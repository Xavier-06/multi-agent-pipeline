# Shared State (Cross-Wave Information Hub)

## Problem

In a multi-wave pipeline, sub-agents in Wave 2+ need to know what Wave 1 discovered:
- Which claims are already supported / contradicted / unverified?
- What facts have been collected and with what source quality?
- What data gaps remain?
- What counter-evidence was found?
- What did prior waves actually produce (their full outputs)?

Without shared state, each wave works in isolation → redundant searches, contradictory conclusions, and no cumulative progress.

## Architecture Overview

Shared state is implemented as **three layers**, each solving a different problem:

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: Shared State Snapshot                      │
│  Machine-readable hub (JSON) + Human-readable page   │
│  → Sub-agents read this to know current progress     │
├─────────────────────────────────────────────────────┤
│  Layer 2: Centralized Fact Store                     │
│  All facts from all waves, deduplicated              │
│  → Quality gates + final assembly read this          │
├─────────────────────────────────────────────────────┤
│  Layer 3: Prior Wave Outputs                         │
│  Full dimension files from completed waves           │
│  → Later-wave sub-agents read prior wave reports     │
└─────────────────────────────────────────────────────┘
```

## Layer 1: Shared State Snapshot

### Artifacts

| Artifact | Purpose | Consumers |
|----------|---------|-----------|
| `shared_state.json` | Machine-readable snapshot of claims, facts, risks, gaps | All sub-agents (via brief) |
| `shared_diligence_page.md` | Human/agent-readable dashboard | All sub-agents (first file in brief) |
| `claim_coverage.json` | Per-claim coverage detail | Quality chain phases |
| `open_questions.json` | Unresolved data gaps | Sub-agents + repair |
| `evidence_conflicts.json` | Counter-evidence queue | Debate review |

### Core Function: `build_shared_state()`

This is the heart of shared state. You implement this for your domain:

```python
def build_shared_state(task_dir: Path, after_wave: int = 0) -> dict:
    """Rebuild the full shared state snapshot from all available sources.

    Called by both init (before Wave 1) and refresh (after each wave).
    """
    # 1. Load the claim matrix from your research plan
    plan = load_json(task_dir / "research_plan.json")
    claim_rows = plan.get("claim_matrix", [])

    # 2. Collect ALL sub-agent outputs from waves 1..N
    #    Prefer a merged packages file; fallback to scanning sidecars on disk
    packages = _load_merged_packages(task_dir) or _scan_sidecars(task_dir)

    # 3. Build fact index — merge centralized fact store + all sidecar facts
    facts = _load_fact_store(task_dir) + _scan_fact_sidecars(task_dir)
    fact_index = deduplicate_by_id(facts)

    # 4. For each claim, determine its current status
    claim_status = {}
    for claim in claim_rows:
        claim_status[claim["claim_id"]] = _resolve_claim_status(claim, packages, fact_index)

    # 5. Extract open questions from sub-agent data_gaps
    open_questions = _extract_open_questions(packages)

    # 6. Extract risks from sub-agent counter_evidence
    risks = _extract_risks(packages)

    # 7. Derive a recommendation (optional — domain-specific)
    recommendation = _derive_recommendation(claim_status, risks)

    return {
        "schema_version": "shared_state.v1",
        "after_wave": after_wave,
        "claim_status": claim_status,
        "fact_index": fact_index,
        "open_questions": open_questions,
        "risks": risks,
        "current_recommendation": recommendation,
        "wave_history": _build_wave_history(task_dir, after_wave),
    }
```

### Claim Status Lifecycle

Define status values that make sense for your domain. A typical set:

```
planned → not_addressed       (initialized, no sub-agent output yet)
not_addressed → supported           (facts found, high-quality source)
not_addressed → partially_supported (some facts, lower source quality)
not_addressed → unverified          (no facts or low confidence)
not_addressed → contradicted        (counter-evidence found)
```

**Conservative merge rule**: when multiple sub-agents address the same claim, keep the worst status. This prevents false confidence when evidence is mixed.

```python
_STATUS_PRIORITY = {
    "contradicted": 0,       # worst
    "unverified": 1,
    "partially_supported": 2,
    "supported": 3,
    "not_addressed": 4,
    "planned": 5,
}

def _conservative_merge(status_a: str, status_b: str) -> str:
    """Keep the worse (more conservative) status."""
    return status_a if _STATUS_PRIORITY[status_a] <= _STATUS_PRIORITY[status_b] else status_b
```

### Status Resolution Logic

```python
def _resolve_claim_status(claim, packages, fact_index):
    """Determine a claim's current status from section packages + facts.

    Customize this for your domain's quality standards.
    """
    cid = claim["claim_id"]
    relevant_packages = [p for p in packages if cid in p.get("claim_ids_covered", [])]

    if not relevant_packages:
        return "not_addressed"

    bound_facts = [
        fact_index[fid]
        for pkg in relevant_packages
        for fid in pkg.get("facts_used", [])
        if fid in fact_index
    ]

    if not bound_facts:
        return "unverified"

    # Check for counter-evidence
    counter_evidence = [f for f in bound_facts if f.get("fact_type") == "counter_evidence"]
    if counter_evidence:
        return "contradicted"

    # Check source quality — adjust tiers for your domain
    best_tier = min(bound_facts, key=lambda f: _TIER_RANK.get(f.get("source_tier", "low"), 99))
    tier = best_tier.get("source_tier", "low")

    if tier in ("official", "regulatory", "database"):
        return "supported"
    elif tier in ("industry_report", "news"):
        return "partially_supported"
    else:
        return "unverified"
```

### Refresh Placement in Pipeline

**Always refresh AFTER the evidence gate, not before.** This ensures the shared state only reflects verified outputs.

```
Phase init:   shared_state_init     (before Wave 1, skeleton from plan)
Phase W1:     wave1_dispatch → collect → evidence_gate
Phase merge:  fact_store_merge      (sidecars → central store)
Phase refresh: shared_state_refresh  (after Wave 1, full rebuild)  ← HERE
Phase W2:     wave2_dispatch → collect → evidence_gate              ← reads refreshed state
Phase refresh: shared_state_refresh  (after Wave 2)
...
```

Which waves need a refresh?

| Wave type | Refresh needed? | Reason |
|-----------|----------------|--------|
| Wave with 3+ roles | Yes | Significant new information |
| Wave with 1-2 narrow roles | Optional | May not produce enough new info |
| Last wave before quality chain | No | Quality chain reads packages directly |
| Early-stage single-role wave | No | Limited data, overhead > value |

### Human-Readable Page

Generate a Markdown dashboard alongside the JSON for sub-agents to quickly scan:

```python
def render_shared_page(state: dict) -> str:
    """Render shared_state.json as a human-readable Markdown dashboard.

    Typical sections (customize for your domain):
    1. Current verdict / recommendation snapshot
    2. Claim verification status table
    3. Confirmed facts table
    4. Risks and counter-evidence table
    5. Open data gaps table
    6. Cross-dimension conflicts
    7. Wave handoff instructions (what to reuse, what to avoid, what to verify next)
    """
    lines = [f"# Shared State — {state['entity']}", ""]
    # ... render each section as a Markdown table ...
    return "\n".join(lines)
```

## Layer 2: Centralized Fact Store

After each wave, merge all fact sidecars into a single authoritative store:

```python
def _run_fact_store_merge(task_dir, outputs_dir):
    """Collect all *-facts.json sidecars into one fact store.

    Key design decisions:
    - Malformed sidecars are SKIPPED, not blocking (one bad file ≠ pipeline death)
    - Deduplication by fact_id across sidecars
    - The merge is ADDITIVE: each call rebuilds from all sidecars on disk
    """
    facts = []
    seen_ids = set()
    malformed = []

    for sidecar_path in outputs_dir.glob("*-facts.json"):
        try:
            data = json.loads(sidecar_path.read_text())
            for fact in data.get("facts", []):
                if fact["fact_id"] not in seen_ids:
                    seen_ids.add(fact["fact_id"])
                    facts.append(fact)
        except (json.JSONDecodeError, KeyError):
            malformed.append({"path": str(sidecar_path)})
            continue  # skip, don't block

    write_json(task_dir / "fact_store.json", {
        "facts": facts,
        "source_files": [str(p) for p in outputs_dir.glob("*-facts.json")],
        "malformed_source_files": malformed,
    })
```

**Why additive?** The fact store is rebuilt from scratch each time. New wave sidecars are automatically picked up because they're now on disk. No state needs to be tracked between merges.

## Layer 3: Prior Wave Outputs

Later-wave sub-agents need to read the actual reports from prior waves. This is the simplest layer — just file paths.

```python
def _shared_inputs(task_dir):
    """Shared state artifact paths (Layer 1)."""
    return {
        "shared_page": str(task_dir / "shared_diligence_page.md"),
        "shared_state": str(task_dir / "shared_state.json"),
        "fact_store": str(task_dir / "fact_store.json"),
    }

def _prior_wave_outputs(waves_completed, task_dir, outputs_dir):
    """Full dimension output files from all completed waves (Layer 3).

    This is CUMULATIVE — Wave 4 sees Wave 1 + 2 + 3 outputs.
    """
    result = {}
    for role, slug in waves_completed:
        path = outputs_dir / f"phase2_{slug}.md"
        if path.exists():
            result[role] = str(path)
    return result
```

### Injection into Sub-Agent Briefs

Combine all three layers into the brief:

```python
def _build_wave_brief(sub, task_dir, outputs_dir, waves_completed):
    shared = _shared_inputs(task_dir)
    prior_outputs = _prior_wave_outputs(waves_completed, task_dir, outputs_dir)

    # Merge into wave_inputs — brief builder will list these as input files
    sub["wave_inputs"] = {**shared, **prior_outputs}

    # The brief builder adds these to the "Input Files" section:
    # - shared_diligence_page.md  ← listed FIRST, sub-agent reads first
    # - shared_state.json
    # - fact_store.json
    # - wave1_role_a.md           ← prior wave output
    # - wave1_role_b.md
    # - wave2_role_c.md
```

The brief should explicitly instruct: "Read the shared state page FIRST, then read prior wave outputs, then read raw materials."

## Implementation Checklist

When implementing shared state for your pipeline:

- [ ] **Define your claim matrix schema** — what fields does each claim have? (claim_id, claim text, owner_section, priority, ...)
- [ ] **Define your fact schema** — what fields does each fact have? (fact_id, value, source_url, source_tier, confidence, ...)
- [ ] **Define your claim status values** — what statuses make sense for your domain?
- [ ] **Implement `build_shared_state()`** — the core function that reads all sources and produces the snapshot
- [ ] **Implement `render_shared_page()`** — Markdown dashboard for sub-agents
- [ ] **Implement `fact_store_merge()`** — sidecar scanning + dedup + central store writing
- [ ] **Insert refresh phases after evidence gates** — always post-gate
- [ ] **Inject shared state into briefs** — list shared files + prior wave outputs
- [ ] **Define conservative merge rule** — how do you handle conflicting evidence?
- [ ] **Add stage-aware behavior** — early-stage projects should have relaxed thresholds

## Adaptation Examples

### For a Market Research Pipeline
- Claims = market size assertions, competitive claims, customer quotes
- Facts = data points from reports, surveys, interviews
- Shared state = "Market Opportunity Dashboard"
- Refresh after each research wave

### For a Code Review Pipeline
- Claims = "module X has no tests", "function Y has a bug"
- Facts = code snippets, test results, static analysis output
- Shared state = "Code Quality Dashboard"
- Refresh after each reviewer completes

### For a Legal Due Diligence Pipeline
- Claims = "contract clause X is standard", "regulation Y requires Z"
- Facts = contract excerpts, regulation text, case law citations
- Shared state = "Legal Risk Dashboard"
- Refresh after each legal dimension is reviewed
