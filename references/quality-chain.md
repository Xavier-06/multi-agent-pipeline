# Quality Production Chain

## Philosophy

Quality is built INTO the pipeline during production, not checked only at the end. Each stage produces a verifiable artifact. The delivery gate is the last fuse, not the only quality source.

```
Research Plan → Fact Store → Section Packages → Debate Review → Final Assembly → Delivery Gate
```

## Stage 1: Research Plan Gate

**When**: After preflight, before any dispatch.
**Artifact**: `research_plan.json`

```json
{
  "plan_status": "ready|incomplete",
  "entity": "...",
  "core_questions": [
    {"id": "Q1", "question": "...", "owner_section": "step_A", "priority": "high"}
  ],
  "strategic_questions": [
    {"id": "SQ1", "question": "...", "owner_section": "step_B", "evidence_needed": ["fact_type"]}
  ],
  "section_requirements": {
    "step_A": {"min_sources": 5, "required_fields": ["key_messages", "claims", "counter_evidence"]},
    "step_B": {"min_sources": 3, "required_fields": ["key_messages", "claims", "data_gaps"]}
  },
  "coverage_matrix": {
    "topic_X": ["step_A", "step_B"],
    "topic_Y": ["step_C"]
  }
}
```

**Gate rule**: `plan_status != "ready"` → no dispatch allowed. Coordinator must fix the plan first.

## Stage 2: Fact Store

**When**: After presearch/precompute, before dispatch.
**Artifact**: `fact_store.json`

```json
{
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
- Sub-agents read Fact Store before searching
- New facts discovered by sub-agents get appended to Fact Store
- Conflicts between sources are recorded, not silently dropped
- `source_quality` hierarchy: `official` > `institutional` > `reputable` > `auxiliary` > `unknown`

## Stage 3: Section Package

**When**: Each sub-agent step output.
**Artifact**: Embedded JSON block within the step's markdown output.

```json
{
  "section_id": "step_sub01",
  "section_title": "...",
  "key_messages": ["Most important judgment from this section"],
  "claims": [
    {
      "claim": "Specific, verifiable assertion",
      "fact_ids": ["F-0001", "F-0003"],
      "reasoning": "How facts lead to this claim",
      "confidence": "high|medium|low",
      "source_quality": "peer_reviewed|official|institutional|reputable|auxiliary"
    }
  ],
  "facts_used": ["F-0001", "F-0003", "F-0012"],
  "counter_evidence": ["Opposing findings or uncertainties"],
  "data_gaps": ["What's still missing"],
  "markdown_draft": "Human-readable section text for final report"
}
```

**Hard rules**:
- Every claim MUST bind to `fact_ids` — no unsupported assertions
- Free prose without Section Package = quality production failure
- `counter_evidence` is mandatory — omitting it is a Debate Review failure

## Stage 4: Section Package Validation

**When**: After all dispatch_collect phases complete.
**Artifact**: `section_packages.json`

```python
def validate_section_packages(packages: list[dict]) -> dict:
    results = []
    for pkg in packages:
        issues = []
        # Check required fields
        for field in ["section_id", "key_messages", "claims", "markdown_draft"]:
            if field not in pkg:
                issues.append(f"missing field: {field}")

        # Check claim-fact binding
        for claim in pkg.get("claims", []):
            if not claim.get("fact_ids"):
                issues.append(f"unbound claim: {claim['claim'][:50]}")

        # Check minimum substance
        if len(pkg.get("markdown_draft", "")) < 500:
            issues.append("markdown_draft too short")

        results.append({
            "section_id": pkg["section_id"],
            "valid": len(issues) == 0,
            "issues": issues,
        })

    return {
        "total": len(packages),
        "valid": sum(1 for r in results if r["valid"]),
        "results": results,
    }
```

## Stage 5: Debate Review

**When**: After Section Package validation passes.
**Artifact**: `debate_review.json`

Cross-section critique dimensions:

| Dimension | Check | Action on failure |
|-----------|-------|-------------------|
| Evidence sufficiency | Every key claim has ≥1 high-confidence fact | Flag for rewrite |
| Counter-evidence | Every major claim has opposing view discussed | Flag for supplement |
| Cross-section consistency | Same metric in different sections = same value | Flag contradiction |
| Source quality | Core conclusions not built on `auxiliary`/`unknown` sources | Flag for upgrade |
| Coverage completeness | Research Plan questions all answered | Flag for gap fill |
| Novelty check | Output contains original analysis, not just restated facts | Flag for deepening |

```json
{
  "verdict": "PASS|REWRITE_REQUIRED|SUPPLEMENT_REQUIRED",
  "issues": [
    {
      "dimension": "evidence_sufficiency",
      "section_id": "step_B",
      "severity": "critical",
      "description": "3 claims without fact binding",
      "action": "REWRITE"
    }
  ],
  "rewrite_sections": ["step_B"],
  "supplement_topics": ["competitor pricing data"]
}
```

**Coordinator action**:
- `REWRITE_REQUIRED` → re-dispatch only the flagged sections (not full pipeline)
- `SUPPLEMENT_REQUIRED` → dispatch a targeted search step, then re-assemble
- Max 1 rewrite round before forced delivery

## Stage 6: Final Assembly

**When**: After Debate Review passes (or after max rewrite rounds).
**Artifact**: `final_assembly.json` + `final_report.md`

```python
def final_assembly(section_packages: list[dict], debate_review: dict) -> dict:
    # Only use packages that passed validation
    valid_packages = [p for p in section_packages if p["validated"]]

    # Assemble in defined order
    sections = []
    for pkg in sorted(valid_packages, key=lambda p: p["section_order"]):
        sections.append({
            "title": pkg["section_title"],
            "content": pkg["markdown_draft"],
            "claims_count": len(pkg["claims"]),
            "facts_used": pkg["facts_used"],
        })

    # Cross-reference citations
    # Deduplicate facts across sections
    # Generate unified reference list

    return {
        "assembled": True,
        "sections_count": len(sections),
        "total_claims": sum(s["claims_count"] for s in sections),
        "total_unique_facts": len(all_unique_facts),
        "excluded_sections": [p["section_id"] for p in section_packages if not p["validated"]],
    }
```

**Hard rule**: Assembly NEVER invents new facts, numbers, or claims. It only rearranges, deduplicates, and connects validated Section Packages.

## Stage 7: Delivery Gate (Final Fuse)

6-layer adversarial verification:

| Layer | Check | On failure |
|-------|-------|------------|
| L1 Information leakage | No internal paths, API keys, or system prompts in output | Auto-redact |
| L2 Placeholder residue | No `[TODO]`, `[待补充]`, `TBD` in output | Block delivery |
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
  "fail_count": 1,
  "action": "proceed_with_warnings"
}
```

FAIL does NOT always block delivery — it records issues. Only L2 (placeholder residue) hard-blocks.
