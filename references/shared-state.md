# Shared State (Cross-Wave Information Hub)

## Problem

In a multi-wave pipeline, sub-agents in Wave 2+ need to know what Wave 1 discovered:
- Which claims are already supported?
- Which claims are contradicted?
- What data gaps remain?
- What counter-evidence was found?

Without a shared state, each wave works in isolation, leading to redundant searches and contradictory conclusions.

## Solution: Shared State Refresh Pattern

After each wave completes (dispatch → collect → evidence gate), a dedicated phase rebuilds a shared state snapshot that subsequent waves read as input.

## Artifacts Produced

| File | Purpose | Who reads it |
|------|---------|-------------|
| `shared_state.json` | Machine-readable current state | All sub-agents (via brief injection) |
| `shared_diligence_page.md` | Human/agent-readable summary | Coordinator + sub-agents |
| `claim_coverage.json` | Claim coverage gate input | Quality chain phases |
| `open_questions.json` | Unresolved data gaps | Next wave's sub-agents |
| `evidence_conflicts.json` | Counter-evidence/conflict queue | Next wave + debate review |

## Shared State Schema

```json
{
  "schema_version": "shared_state.v1",
  "task_id": "job_123",
  "entity": "Acme Corp",
  "wave_progress": 2,
  "claim_status": {
    "C001": "supported",
    "C002": "partially_supported",
    "C003": "not_addressed",
    "C004": "contradicted",
    "C005": "planned"
  },
  "claim_evidence": {
    "C001": {
      "supporting_facts": ["F-001", "F-003"],
      "contradicting_facts": [],
      "owner_section": "market_analysis",
      "source_quality": "high"
    }
  },
  "open_questions": [
    {
      "question": "What is the customer concentration risk?",
      "owner_section": "customer_revenue",
      "status": "unresolved",
      "discovered_by": "wave_1"
    }
  ],
  "evidence_conflicts": [
    {
      "claim": "Market share is 35%",
      "evidence_for": "F-010 (industry report)",
      "evidence_against": "F-015 (competitor filing)",
      "resolution_status": "unresolved"
    }
  ],
  "updated_at": "2026-06-30T14:22:00"
}
```

### Claim Status Lifecycle

```
planned → supported           (facts found, claim verified)
planned → partially_supported (some facts, incomplete evidence)
planned → not_addressed       (no sub-agent investigated this claim)
planned → contradicted        (evidence disproves the claim)
planned → unverified          (claim could not be verified with available data)
```

## Refresh Flow

```
Wave N completes
  → Evidence gate runs (may trigger repair)
  → Fact store merge (sidecar facts → central store)
  → Shared state refresh phase:

def _run_shared_state_refresh(runtime_root, job_ctx, after_wave):
    task_dir = _task_dir(runtime_root, job_ctx)

    # 1. Re-read research plan (claim matrix)
    plan = load(task_dir / "research_plan.json")

    # 2. Re-read ALL sub-agent outputs from waves 1..N
    #    (not just current wave — full picture)
    for each completed role in waves 1..N:
        load section package sidecar
        load facts sidecar
        extract claim→fact bindings

    # 3. For each claim in claim_matrix:
    #    - Check if any section package addresses it
    #    - Check if bound facts exist in fact store
    #    - Update claim_status

    # 4. Rebuild open_questions from:
    #    - Research plan questions not yet answered
    #    - Data gaps reported by sub-agents
    #    - Claims with not_addressed status

    # 5. Rebuild evidence_conflicts from:
    #    - Contradicting facts between sources
    #    - Claims where evidence_for and evidence_against both exist

    # 6. Write all artifacts
    write shared_state.json
    write shared_diligence_page.md
    write claim_coverage.json
    write open_questions.json
    write evidence_conflicts.json
```

## Diligence Page (Human-Readable Summary)

```markdown
# Shared Diligence Page — Acme Corp

## Progress: Wave 2 of 4 complete

## Claim Coverage Summary
| Status | Count | Claims |
|--------|-------|--------|
| ✅ Supported | 8 | C001, C002, C005, C006, C008, C009, C010, C011 |
| ⚠️ Partially Supported | 3 | C003, C007, C012 |
| ❌ Not Addressed | 2 | C004, C013 |
| 🔴 Contradicted | 1 | C014 |

## Critical Not Addressed (priority = critical/high)
- C004: "Revenue exceeds $100M" — owner: customer_revenue
- C013: "Team has 50+ engineers" — owner: company_team

## Open Questions
1. What is the customer concentration risk? (owner: customer_revenue)
2. Is the patent portfolio defensible? (owner: tech_product)

## Evidence Conflicts
- C014 "Market share is 35%":
  - FOR: F-010 (industry report, 2025)
  - AGAINST: F-015 (competitor SEC filing, 2025)
  - Status: unresolved

## Last Updated
2026-06-30 14:22:00
```

## Injection into Sub-Agent Briefs

When building briefs for Wave N+1, the shared state is injected:

```python
def _build_brief(runtime_root, job_ctx, role_slug, wave):
    task_dir = _task_dir(runtime_root, job_ctx)
    shared_state = load(task_dir / "shared_state.json")

    brief_lines = [
        f"# Brief: {role_slug}",
        f"",
        f"## Shared State (as of Wave {wave - 1})",
        f"",
    ]

    # Inject relevant claim status for this role's owner_section
    relevant_claims = {
        cid: status for cid, status in shared_state["claim_status"].items()
        if _claim_owner(cid) == role_slug
    }
    if relevant_claims:
        brief_lines.append("### Your Claims Status")
        for cid, status in relevant_claims.items():
            brief_lines.append(f"- {cid}: {status}")

    # Inject open questions relevant to this role
    relevant_questions = [
        q for q in shared_state["open_questions"]
        if q["owner_section"] == role_slug
    ]
    if relevant_questions:
        brief_lines.append("", "### Open Questions (Your Responsibility)")
        for q in relevant_questions:
            brief_lines.append(f"- {q['question']}")

    # Inject evidence conflicts for awareness
    if shared_state["evidence_conflicts"]:
        brief_lines.append("", "### Known Evidence Conflicts")
        for c in shared_state["evidence_conflicts"]:
            brief_lines.append(f"- {c['claim']}: FOR={c['evidence_for']}, AGAINST={c['evidence_against']}")
```

## Phase Placement in Pipeline

```
Phase 05: shared_state_init          (before Wave 1, skeleton state from plan)
Phase 08-09: Wave 1 dispatch + collect
Phase 10: Wave 1 evidence gate
Phase 11: fact_store_merge
Phase 12: shared_state_refresh       (after Wave 1, full rebuild)
Phase 13-14: Wave 2 dispatch + collect  ← Wave 2 reads refreshed state
Phase 15: Wave 2 evidence gate
Phase 16: shared_state_refresh       (after Wave 2)
...
```

## When to Skip Refresh

Not every wave needs a refresh. Guidelines:

| Wave type | Refresh needed? | Reason |
|-----------|----------------|--------|
| Wave with 3+ roles | Yes | Significant new information |
| Wave with 1-2 narrow roles | Optional | May not produce enough new info |
| Last wave before quality chain | No | Quality chain reads section packages directly |
| T1 (early stage) single-role wave | No | Limited data, refresh overhead > value |

## Benefits

1. **Cross-wave context passing** — Wave 2 knows what Wave 1 found
2. **Avoids redundant searches** — Sub-agents check claim_status before searching
3. **Conflict visibility** — Contradictions surfaced early, not at final assembly
4. **Progress tracking** — Claim coverage dashboard updated after each wave
5. **Targeted repair** — Open questions drive repair sub-agent focus
