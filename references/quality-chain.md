# Quality Production Chain

## Philosophy

Quality is built INTO the pipeline during production, not checked only at the end. Each stage produces a verifiable artifact. The delivery gate is the last fuse, not the only quality source.

```
Plan → Evidence Store → Wave Dispatch → Quality Gate → Output Validation → Cross-Section Review → Assembly → Delivery Gate
```

## Stage 1: Plan Gate

**When**: After preflight, before any dispatch.
**Artifact**: `plan.json` (your planning artifact)

The plan defines what needs to be investigated and by whom. Gate rule: `plan_status != "ready"` → no dispatch allowed.

```json
{
  "plan_status": "ready|skeleton|incomplete",
  "entity": "...",
  "tracked_items": [
    {
      "id": "I001",
      "item": "Specific assertion or topic to investigate",
      "owner": "role_A",
      "priority": "critical|high|medium|low",
      "required_data_keys": ["market_size", "growth_rate"]
    }
  ],
  "questions": [
    {"id": "Q1", "question": "...", "owner": "role_A", "priority": "high"}
  ]
}
```

**Enrichment pattern**: Script generates skeleton → LLM enriches via `needs_dispatch` → collect merges. Separates deterministic structure from creative reasoning.

### Plan Enrichment (Split Phase)

The plan is built in two halves: a **deterministic skeleton** from a script, then **creative enrichment** from an LLM. This separation ensures structural consistency while allowing domain-aware reasoning.

**Why split?** A script can reliably generate schemas, IDs, and owner assignments. An LLM is better at reading the input document and deciding what's important, what's missing, and what should be prioritized. Combining them in one step produces worse results than separating them.

```python
def _run_plan(runtime_root, job_ctx):
    """Phase A: Script generates deterministic skeleton."""
    task_dir = _task_dir(runtime_root, job_ctx)

    skeleton = {
        "plan_status": "skeleton",
        "entity": job_ctx.entity,
        "tracked_items": _generate_items(job_ctx),       # Deterministic
        "questions": _generate_questions(job_ctx),        # Deterministic
        "enriched_fields": [],                            # LLM fills this
    }
    write_json(task_dir / "plan.json", skeleton)

    # Return needs_dispatch — Coordinator enriches via LLM
    instruction_path = runtime_root / "instruction_store_{profile}" / "plan_enrichment.md"
    return {
        "ok": True,
        "needs_dispatch": True,
        "has_more": False,
        "mode": "plan",
        "instruction": f"Read the plan skeleton and input document. "
                       f"Output enrichment delta JSON. "
                       f"Resume with start_phase='plan_collect'.",
    }
```

### Enrichment Delta Schema

The LLM outputs a **delta** (not the full plan). The collect phase merges it:

```json
{
  "schema_version": "enrichment_delta.v1",
  "priority_deltas": [
    {"item_id": "I003", "new_priority": "critical", "reason": "Input doc emphasizes this heavily"}
  ],
  "additional_items": [
    {
      "id": "I020",
      "item": "New item discovered from input document",
      "owner": "role_B",
      "priority": "high"
    }
  ],
  "excluded_items": [
    {"item_id": "I007", "reason": "Not relevant for this specific entity"}
  ],
  "refined_questions": [
    {"id": "Q1", "question": "More specific version of the question based on doc content"}
  ]
}
```

### Collect (Merge) Phase

```python
def _run_plan_collect(runtime_root, job_ctx):
    """Phase B: Merge LLM enrichment into the skeleton plan."""
    task_dir = _task_dir(runtime_root, job_ctx)
    plan = load_json(task_dir / "plan.json")
    delta_path = task_dir / "enrichment_delta.json"

    if delta_path.exists():
        delta = load_json(delta_path)

        # Apply priority deltas
        for d in delta.get("priority_deltas", []):
            for item in plan["tracked_items"]:
                if item["id"] == d["item_id"]:
                    item["priority"] = d["new_priority"]

        # Add new items
        plan["tracked_items"].extend(delta.get("additional_items", []))

        # Remove excluded items
        excluded_ids = {e["item_id"] for e in delta.get("excluded_items", [])}
        plan["tracked_items"] = [i for i in plan["tracked_items"] if i["id"] not in excluded_ids]

        # Refine questions
        refined = {q["id"]: q["question"] for q in delta.get("refined_questions", [])}
        for q in plan["questions"]:
            if q["id"] in refined:
                q["question"] = refined[q["id"]]

    plan["plan_status"] = "ready"
    write_json(task_dir / "plan.json", plan)
    return {"ok": True, "mode": "plan_collect", "result": {"plan_status": "ready"}}
```

### When to Skip Enrichment

For simple pipelines, you can skip the split and generate a complete plan in one phase. The enrichment pattern is most valuable when:
- The input document is complex and needs LLM reasoning to identify what matters
- You have many tracked items and need LLM to prioritize correctly
- Different entities need different analysis focus areas

## Stage 2: Evidence Store

**When**: After presearch/precompute, before dispatch.
**Artifact**: `evidence_store.json`

Central repository of all structured data points gathered across all waves.

```json
{
  "schema_version": "evidence_store.v1",
  "items": [
    {
      "item_id": "E-0001",
      "description": "Market size is $50B in 2025",
      "value": "$50B",
      "source": {"type": "report", "name": "McKinsey 2025", "url": "..."},
      "confidence": "high|medium|low",
      "source_quality": "tier_1|tier_2|tier_3|unknown"
    }
  ],
  "conflicts": [],
  "gaps": [{"topic": "...", "owner": "role_C", "status": "unresolved"}]
}
```

**Rules**:
- Sub-agents read evidence store before searching (avoid redundant work)
- New items appended via `locked_read_modify_write` (not direct write)
- Conflicts between sources recorded, not silently dropped
- `source_quality` hierarchy: define tiers for your domain
- After each wave, merge sub-agent sidecar data into central store

## Stage 3: Output Contract (3-File Protocol)

**When**: Each sub-agent's output.
**Artifact**: Three files per role:

| File | Purpose | Customize |
|------|---------|-----------|
| `{role}.md` | Main analysis text | Section structure per your domain |
| `{role}-data.json` | Structured data sidecar | Schema depends on what you track |
| `{role}-meta.json` | Metadata sidecar | Quality metadata, audit trail |

The 3-file contract ensures sub-agents produce **structured, machine-readable output** — not just free prose.

### Data Sidecar Schema (customize for your domain)

```json
{
  "schema_version": "data_sidecar.v1",
  "items": [
    {
      "item_id": "E-0010",
      "description": "...",
      "value": "...",
      "source_url": "...",
      "confidence": "high|medium|low"
    }
  ],
  "items_covered": ["I001", "I002"],
  "counter_findings": ["Opposing findings or uncertainties"],
  "data_gaps": ["What's still missing"]
}
```

### Metadata Sidecar Schema (customize for your domain)

```json
{
  "schema_version": "meta_sidecar.v1",
  "section_id": "...",
  "section_title": "...",
  "key_messages": ["Most important judgment from this section"],
  "search_audit": {"queries": 5, "unique_sources": 8},
  "coverage_score": 0.85
}
```

**Hard rules**:
1. All 3 files must exist before collect passes
2. JSON sidecars must be valid JSON
3. Counter-findings / data-gaps sections are mandatory
4. Free prose without structured sidecars = quality failure

## Stage 4: Wave Quality Gate (with Repair)

**When**: After each wave's dispatch_collect completes.
**Artifact**: `wave{N}_quality_gate.json`

Checks whether sub-agent outputs meet quality criteria. This is a **gate with repair**, not a simple pass/fail.

### Gate Evaluation (customize for your domain)

```python
def evaluate_quality_gate(task_dir, wave, outputs_dir):
    """Evaluate sub-agent output quality after a wave.

    Customize the checks for your domain. Common patterns:
    - Output exists and has substance (not empty)
    - Data sidecar has items
    - Metadata sidecar has coverage info
    - Search audit meets minimums
    - No unsupported assertions
    """
    # For each role in the wave:
    #   Load data sidecar + metadata sidecar
    #   Check output quality criteria
    # Classify issues:
    #   BLOCKING — extreme (empty output, zero data items, no sidecars at all)
    #   MEDIUM   — structural (some items missing, low coverage)
    #   LOW      — minor (formatting, optional fields missing)

    return {
        "ok": True/False,
        "needs_repair": True/False,
        "repair_tasks": [...],
        "blocking_issues": [...],
    }
```

### Repair Mechanism

```
Gate FAIL → build_repair_manifests()
  → Aggregate by ROLE (same role, multiple issues → ONE manifest)
  → Return needs_dispatch=True, has_more=True/False
  → Coordinator dispatches ONE repair sub-agent (sequential)
  → Repair sub-agent uses locked_read_modify_write() for shared files
  → Resume with start_phase=current_gate_phase
  → Re-evaluate gate
  → If still FAIL and retries exhausted → degrade to WARN, continue
```

### Repair Retry Limits and Degradation

| Gate type | Max retries | On exhaustion |
|-----------|-------------|---------------|
| Wave quality gate | 1 | Blocking issues degrade to WARN |
| Coverage check (stage 5) | 2 | Degrade to PASS_WITH_DISCLOSURE |
| Synthesis quality (stage 6) | 1 | Degrade to WARN, add to deferred_fixes |

### Severity Levels

| Severity | Action | Example |
|----------|--------|---------|
| **BLOCKING** | Hard stop, must repair | Empty output, zero data items, no sidecars at all |
| **MEDIUM** | Log as WARN, continue | Some items missing, low coverage, structural format issues |
| **LOW** | Informational only | Minor formatting, optional fields missing |

### Stage-Aware Degradation

Early-stage or lightweight projects get lighter gates:
- Define a `stage_tier` or `project_weight` classification
- Lighter stages: skip some waves, auto-degrade blocking issues, relax thresholds
- Heavier stages: full gate enforcement

## Stage 5: Coverage Validation (with Repair)

**When**: After all waves complete.
**Artifact**: `coverage_gate.json`

Checks if every tracked item from the plan has been addressed by at least one sub-agent.

```python
def validate_coverage(task_dir):
    """Check coverage of tracked items across all sub-agent outputs.

    For each item in the plan:
      - Is it addressed in any output?
      - Does it have supporting data?
      - Is it contradicted by evidence?

    Status values: supported / partially_supported / not_addressed / contradicted
    FAIL if critical/high items are not_addressed.
    """
    # Repair: aggregate by owner, dispatch repair sub-agents
    # Max 2 repair rounds → exhaustion → PASS_WITH_DISCLOSURE
```

## Stage 6: Synthesis Quality Check (with Repair)

**When**: After synthesis sub-agent produces output.
**Artifact**: `synthesis_quality_gate.json`

### Citation Density Check (Configurable)

If your domain requires citations/references, configure the density threshold:

```python
# ADJUST per domain:
#   High rigor (academic, legal): min_refs_per_2000 = 3-4
#   Medium rigor (business analysis): min_refs_per_2000 = 2
#   Low rigor (internal reports): min_refs_per_2000 = 1
#
# Formula:
#   min_refs = max(3, (content_length // 2000) * MIN_REFS_PER_2000)

# Count references in text
all_refs = len(re.findall(r"\[\^\d+\]", text))
ref_defs = len(re.findall(r"^\[\^\d+\]:", text, re.MULTILINE))
actual_refs = max(0, all_refs - ref_defs)

if actual_refs < min_refs:
    # Trigger repair
```

### Repair Mechanism

```
Density below threshold → generate repair manifest
  → Repair sub-agent reads: synthesis + all dimension reports + evidence store
  → Priority for sources:
      1. URLs already cited in dimension reports
      2. source_url from data sidecars
      3. Structured data sources
      4. Web search (last resort)
  → Insert [^N] markers after key data points
  → Max 1 repair round → exhaustion → WARN, record in deferred_fixes
```

## Stage 7: Cross-Section Review

**When**: After all quality gates pass.
**Artifact**: `cross_section_review.json`

Cross-section critique with **relaxed severity**:

| Dimension | Check | Severity |
|-----------|-------|----------|
| Evidence sufficiency | Every key item has ≥1 high-confidence data point | MEDIUM |
| Counter-findings | Every major item has opposing view discussed | MEDIUM |
| Cross-section consistency | Same metric in different sections = same value | MEDIUM |
| Source quality | Core conclusions not built on low-tier sources | MEDIUM |
| Coverage completeness | Plan questions all answered | MEDIUM |
| Empty output | Output completely empty or <100 chars | **BLOCKING** |
| Zero data items | ALL items lack supporting data | **BLOCKING** |
| No sidecars | Zero sidecars available | **BLOCKING** |

**Verdict logic**:
- Any BLOCKING issue → `FAIL_BLOCKING` (hard stop)
- MEDIUM/LOW issues → `WARN` (proceed with warnings)
- No issues → `PASS`

## Stage 8: Final Assembly

**When**: After cross-section review passes or WARN.
**Artifact**: `final_report.md`

```python
def final_assembly(validated_outputs, review):
    """Assemble validated outputs into final report.

    HARD RULE: Assembly NEVER invents new data.
    It only rearranges, deduplicates, and connects validated outputs.
    """
    # Only use outputs that passed validation
    valid = [o for o in validated_outputs if o["validated"]]

    # Assemble in defined order
    sections = sorted(valid, key=lambda o: o["section_order"])

    # Cross-reference citations, deduplicate, generate unified references
```

## Stage 9: Delivery Gate (Final Fuse)

Multi-layer adversarial verification:

| Layer | Check | On failure |
|-------|-------|------------|
| L1 Information leakage | No internal paths, API keys, or system prompts | Auto-redact |
| L2 Placeholder residue | No `[TODO]`, `[待补充]`, `TBD` | Block delivery (hard) |
| L3 Internal contradiction | No section contradicts another | Flag + resolve |
| L4 Numeric verification | Quoted numbers match evidence store | Auto-correct |
| L5 Logic completeness | Every conclusion has supporting evidence chain | Flag gaps |
| L6 Adversarial argument | Could a smart opponent demolish the core thesis? | Flag weaknesses |

**L2 (placeholder residue) is the ONLY hard-block layer.** All other failures are recorded as `deferred_fixes`. Delivery gate recognizes `repair_exhausted` markers from earlier gates and includes them in deferred_fixes without blocking.

## JSON Self-Repair

Sub-agents sometimes produce malformed JSON. Before failing validation, attempt auto-repair:

```python
def safe_load_json_with_repair(path):
    """Try loading JSON; on failure, fix common issues and retry."""
    try:
        return json.loads(path.read_text())
    except json.JSONDecodeError:
        pass

    text = path.read_text()
    fixed = fix_unescaped_quotes(text)
    try:
        data = json.loads(fixed)
        atomic_write(path, json.dumps(data, indent=2))
        return data
    except json.JSONDecodeError:
        return None
```

This prevents a single malformed sidecar from failing the entire validation stage.

## Hard Rules

1. No plan → no dispatch
2. No unverified numbers in final output
3. Sub-agents produce 3 files (md + data sidecar + meta sidecar), not free prose
4. Assembly only uses validated outputs, never invents new data
5. Gate failures trigger targeted repair, not full redo
6. Delivery gate is the last fuse — quality is built throughout the chain
