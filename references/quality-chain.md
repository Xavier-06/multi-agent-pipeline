# Quality Production Chain

## Philosophy

Quality is built INTO the pipeline during production, not checked only at the end. Each stage produces a verifiable artifact. The delivery gate is the last fuse, not the only quality source.

```
Research Plan → Fact Store → Wave Dispatch → Evidence Gate → Section Package → Debate → Assembly → Delivery Gate
```

## Stage 1: Research Plan Gate

**When**: After preflight, before any dispatch.
**Artifact**: `research_plan.json`

```json
{
  "plan_status": "ready|skeleton|incomplete",
  "entity": "...",
  "core_questions": [
    {"id": "Q1", "question": "...", "owner_section": "step_A", "priority": "high"}
  ],
  "strategic_questions": [
    {"id": "SQ1", "question": "...", "owner_section": "step_B", "evidence_needed": ["fact_type"]}
  ],
  "claim_matrix": [
    {
      "claim_id": "C001",
      "claim": "Specific assertion to verify",
      "owner_section": "step_A",
      "priority": "critical|high|medium|low",
      "required_fact_keys": ["market_size", "growth_rate"]
    }
  ]
}
```

**Gate rule**: `plan_status != "ready"` → no dispatch allowed. Coordinator must fix the plan first.

**Enrichment pattern**: Script generates skeleton → LLM enriches via `needs_dispatch` → collect merges. This separates deterministic structure from creative reasoning.

## Stage 2: Fact Store

**When**: After presearch/precompute, before dispatch.
**Artifact**: `fact_store.json`

```json
{
  "schema_version": "fact_store.v1",
  "facts": [
    {
      "fact_id": "F-0001",
      "claim": "Market size is $50B in 2025",
      "value": "$50B",
      "source": {"type": "report", "name": "McKinsey 2025", "url": "..."},
      "confidence": "high|medium|low",
      "source_quality": "official|institutional|reputable|auxiliary|unknown"
    }
  ],
  "conflicts": [
    {"fact_a": "F-0001", "fact_b": "F-0015", "nature": "contradictory values", "resolution": "kept F-0001 (more recent)"}
  ],
  "gaps": [
    {"topic": "competitor pricing", "owner_section": "step_C", "status": "unresolved"}
  ]
}
```

**Rules**:
- Sub-agents read Fact Store before searching (avoid redundant work)
- New facts discovered by sub-agents get appended (via `locked_read_modify_write`, not direct write)
- Conflicts between sources are recorded, not silently dropped
- `source_quality` hierarchy: `official` > `institutional` > `reputable` > `auxiliary` > `unknown`
- After each wave, merge sub-agent sidecar facts into central Fact Store

## Stage 3: Section Package

**When**: Each sub-agent step output.
**Artifact**: Three files per role (the "3-file contract"):

| File | Purpose | Required fields |
|------|---------|-----------------|
| `{role}.md` | Main analysis text | Must contain Section Package JSON block |
| `{role}-facts.json` | Discovered facts sidecar | `facts[]` with fact_id, confidence, source_quality |
| `{role}-section.json` | Structured section package | Full schema below |

```json
{
  "schema_version": "section_package.v2",
  "section_id": "step_sub01",
  "section_title": "...",
  "key_messages": ["Most important judgment from this section"],
  "claims": [
    {
      "claim_id": "C001",
      "claim": "Specific, verifiable assertion",
      "fact_ids": ["F-0001", "F-0003"],
      "reasoning": "How facts lead to this claim",
      "confidence": "high|medium|low",
      "source_quality": "official|institutional|reputable|auxiliary"
    }
  ],
  "facts_used": ["F-0001", "F-0003", "F-0012"],
  "counter_evidence": ["Opposing findings or uncertainties"],
  "data_gaps": ["What's still missing"],
  "markdown_draft": "Human-readable section text for final report",
  "answers": [
    {"question_id": "Q1", "answer": "...", "fact_ids": ["F-0001"], "confidence": "high", "limits": "..."}
  ],
  "claim_ids_covered": ["C001", "C002"],
  "narrative_blocks": [
    {"block_id": "NB01", "question_id": "Q1", "claim_ids": ["C001"], "fact_ids": ["F-0001"], "text": "..."}
  ],
  "search_audit": {"queries": 5, "unique_sources": 8, "sources_per_claim": 2.3}
}
```

**Hard rules**:
1. Every claim MUST bind to `fact_ids` — no unsupported assertions
2. Free prose without Section Package = quality production failure
3. `counter_evidence` is mandatory — omitting it is a Debate Review failure
4. The 3 files must ALL exist and be valid JSON (for sidecars) before collect passes

## Stage 4: Wave Evidence Gate (with Repair)

**When**: After each wave's dispatch_collect completes.
**Artifact**: `wave{N}_evidence_gate.json`

Checks whether sub-agent claims are supported by facts. This is a **gate with repair**, not a simple pass/fail.

### Gate Evaluation

```python
def evaluate_evidence_gate(task_dir, wave, outputs_dir):
    # For each role in the wave:
    #   Load section package sidecar
    #   For each claim:
    #     Check fact_ids exist in fact store
    #     Check source_quality meets threshold
    #     Check claim is addressed (not just listed)
    # Classify issues:
    #   BLOCKING — extreme (empty output, 100% claims without facts, no packages at all)
    #   MEDIUM   — structural (some claims unbound, missing counter-evidence)
    #   LOW      — minor

    return {
        "ok": True/False,
        "needs_repair": True/False,    # True if BLOCKING or MEDIUM issues found
        "repair_tasks": [...],         # Which roles/claims need fixing
        "blocking_claims": [...],      # Claims that would hard-block
    }
```

### Repair Mechanism

```
Gate FAIL → build_repair_manifests()
  → Aggregate by ROLE (same role, multiple claims → ONE manifest)
  → Return needs_dispatch=True, has_more=True/False
  → Coordinator dispatches ONE repair sub-agent (sequential)
  → Repair sub-agent uses locked_read_modify_write() for shared files
  → Resume with start_phase=current_gate_phase
  → Re-evaluate gate
  → If still FAIL and retries exhausted → degrade to WARN, continue
```

### Repair Retry Limits and Degradation

| Gate type | Max repair retries | On exhaustion |
|-----------|-------------------|---------------|
| Wave evidence gate | 1 | Blocking claims degrade to WARN |
| Claim coverage (stage 5) | 2 | Degrade to PASS_WITH_DISCLOSURE |
| Synthesis quality (stage 6) | 1 | Degrade to WARN, add to deferred_fixes |

### Stage Tier Degradation

Early-stage projects (seed/angel round) get lighter gates automatically:
- **T1 (Seed/Angel)**: Wave 2 skipped entirely, blocking claims auto-degrade to WARN
- **T2 (Pre-A/A round)**: Repair triggered but with relaxed thresholds
- **T3+ (B round+)**: Full gate enforcement

### Severity Levels (Post-Relaxation)

| Severity | Action | Example |
|----------|--------|---------|
| **BLOCKING** | Hard stop, must repair | Empty dimension output, 100% claims without facts, no section packages at all |
| **MEDIUM** | Log as WARN, continue | Some claims unbound, missing counter-evidence, structural format issues |
| **LOW** | Informational only | Minor formatting, optional fields missing |

Previous HIGH severity checks have been downgraded to MEDIUM. Only extreme cases (completely empty output, 100% unbound claims) are BLOCKING.

## Stage 5: Claim Coverage Validation (with Repair)

**When**: After all waves complete.
**Artifact**: `claim_coverage_gate.json`

Checks if every claim from the research plan has been addressed by at least one sub-agent.

```python
def validate_claim_coverage(task_dir):
    plan = load research_plan.json
    section_packages = load all section packages

    for claim in plan.claim_matrix:
        status = determine_status(claim, section_packages)
        # supported: claim has fact_ids and is addressed
        # partially_supported: some facts but incomplete
        # not_addressed: no sub-agent touched this claim
        # contradicted: evidence contradicts the claim

    # FAIL if critical/high claims are not_addressed
    # Repair: aggregate by owner_section, dispatch repair sub-agents
```

### Repair Mechanism

```
FAIL → build_claim_repair_manifests()
  → Aggregate by owner_section (same section, multiple claims → ONE manifest)
  → Repair sub-agent: external search → append to fact store → update sidecar
  → Uses locked_read_modify_write() for shared file writes
  → Max 2 repair rounds
  → Exhaustion → PASS_WITH_DISCLOSURE + record in delivery deferred_fixes
```

## Stage 6: Synthesis Quality Check (with Repair)

**When**: After synthesis sub-agent produces output.
**Artifact**: `synthesis_quality_gate.json`

### Footnote Density Check (Configurable)

The synthesis output must cite sources with inline footnotes. The threshold is **dynamic and configurable**:

```python
# DEFAULT: Every 2000 characters needs at least 3 footnote references
# This is a starting point — adjust based on your domain:
#
# Investment research (high rigor):
#   min_refs_per_2000 = 3    (default)
#
# Competitive analysis (medium rigor):
#   min_refs_per_2000 = 2
#
# Literature review (high rigor):
#   min_refs_per_2000 = 4
#
# Product teardown (low rigor):
#   min_refs_per_2000 = 1
#
# Formula:
#   min_footnote_refs = max(3, (content_length // 2000) * min_refs_per_2000)

# Count footnote references (not definitions)
all_fn_matches = len(re.findall(r"\[\^\d+\]", text))
footnote_defs = len(re.findall(r"^\[\^\d+\]:", text, re.MULTILINE))
footnote_refs = max(0, all_fn_matches - footnote_defs)

# Dynamic threshold
content_k = max(content_length // 1000, 1)
min_footnote_refs = max(3, (content_length // 2000) * MIN_REFS_PER_2000)

if footnote_refs < min_footnote_refs:
    footnote_fail = True  # Trigger repair
elif footnote_refs < min_footnote_refs * 1.5:
    # Warning only, no repair triggered
```

### Repair Mechanism

```
Footnote density below threshold → generate repair manifest
  → Repair sub-agent reads: synthesis output + dimension reports + facts JSON
  → Priority for footnote sources:
      1. URLs already cited in dimension reports
      2. source_url from facts JSON sidecars
      3. Structured data sources (e.g., company registry)
      4. Web search (last resort only)
  → Insert [^N] markers after quantitative data
  → Append footnote definitions at report end
  → Max 1 repair round
  → Exhaustion → WARN, record in deferred_fixes
```

## Stage 7: Debate Review

**When**: After all quality gates pass.
**Artifact**: `debate_review.json`

Cross-section critique with **relaxed severity**:

| Dimension | Check | Severity on failure |
|-----------|-------|---------------------|
| Evidence sufficiency | Every key claim has ≥1 high-confidence fact | MEDIUM (was HIGH) |
| Counter-evidence | Every major claim has opposing view discussed | MEDIUM |
| Cross-section consistency | Same metric in different sections = same value | MEDIUM |
| Source quality | Core conclusions not built on auxiliary/unknown sources | MEDIUM |
| Coverage completeness | Research Plan questions all answered | MEDIUM |
| Novelty check | Output contains original analysis, not just restated facts | LOW |
| Empty dimension | Output completely empty or <100 chars | **BLOCKING** |
| 100% unbound claims | ALL claims lack fact_ids | **BLOCKING** |
| No section packages | Zero packages available | **BLOCKING** |

```json
{
  "verdict": "PASS|WARN|FAIL_BLOCKING",
  "issues": [
    {
      "severity": "MEDIUM|BLOCKING|LOW",
      "dimension": "evidence_sufficiency",
      "section_id": "step_B",
      "description": "3 claims without fact binding",
      "action": "REWRITE|SUPPLEMENT|DEFER"
    }
  ],
  "deferred_fixes": ["issues recorded but not blocking delivery"]
}
```

**Verdict logic**:
- Any BLOCKING issue → `FAIL_BLOCKING` (hard stop)
- MEDIUM/LOW issues → `WARN` (proceed with warnings recorded)
- No issues → `PASS`

## Stage 8: Final Assembly

**When**: After Debate Review passes or WARN.
**Artifact**: `final_assembly.json` + `final_report.md`

```python
def final_assembly(section_packages, debate_review):
    # Only use packages that passed validation
    valid_packages = [p for p in section_packages if p["validated"]]

    # Assemble in defined order
    sections = sorted(valid_packages, key=lambda p: p["section_order"])

    # Cross-reference citations
    # Deduplicate facts across sections
    # Generate unified reference list

    # HARD RULE: Assembly NEVER invents new facts, numbers, or claims
    # It only rearranges, deduplicates, and connects validated packages
```

## Stage 9: Delivery Gate (Final Fuse)

6-layer adversarial verification:

| Layer | Check | On failure |
|-------|-------|------------|
| L1 Information leakage | No internal paths, API keys, or system prompts in output | Auto-redact |
| L2 Placeholder residue | No `[TODO]`, `[待补充]`, `TBD` in output | Block delivery (hard) |
| L3 Internal contradiction | No section contradicts another section | Flag + resolve |
| L4 Numeric verification | Quoted numbers match Fact Store values | Auto-correct |
| L5 Logic completeness | Every conclusion has supporting evidence chain | Flag gaps |
| L6 Adversarial argument | Could a smart opponent demolish the core thesis? | Flag weaknesses |

```json
{
  "verdict": "PASS|FAIL|PARTIAL",
  "layers": {
    "L1": {"status": "PASS", "findings": 0},
    "L2": {"status": "PASS", "findings": 0},
    "L3": {"status": "FAIL", "findings": ["Section 2 says X, Section 5 says not-X"]},
    "L4": {"status": "PASS", "findings": 0},
    "L5": {"status": "PASS", "findings": 0},
    "L6": {"status": "PARTIAL", "findings": ["No discussion of regulatory risk"]}
  },
  "deferred_fixes": ["issues from debate review + quality gates that were degraded"],
  "action": "proceed|block"
}
```

**L2 (placeholder residue) is the ONLY hard-block layer.** All other failures are recorded as deferred_fixes. Delivery gate recognizes `repair_exhausted` and `blocking_claims_degraded` markers from earlier gates and includes them in deferred_fixes without blocking.

## JSON Self-Repair

Sub-agents sometimes produce malformed JSON (unescaped quotes, trailing commas). Before failing validation, attempt auto-repair:

```python
def safe_load_json_with_repair(path):
    """Try loading JSON; on failure, attempt to fix common issues."""
    try:
        return json.loads(path.read_text())
    except json.JSONDecodeError:
        pass

    # Attempt repair: fix unescaped quotes inside JSON strings
    text = path.read_text()
    fixed = fix_unescaped_quotes(text)
    try:
        data = json.loads(fixed)
        # Write back the fixed version (atomic write)
        atomic_write(path, json.dumps(data, indent=2))
        return data
    except json.JSONDecodeError:
        return None  # Repair failed
```

This prevents a single malformed sidecar from failing the entire validation stage.

## 6 Hard Rules

1. No research plan → no dispatch
2. No unverified numbers in final output
3. Sub-agents produce 3 files (md + facts sidecar + section sidecar), not free prose
4. Assembly only uses validated packages, never invents new facts
5. Gate failures trigger targeted repair, not full redo
6. Delivery gate is the last fuse — quality is built throughout the chain
