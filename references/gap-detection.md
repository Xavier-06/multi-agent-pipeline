# Gap Detection (Pre-Dispatch & Post-Wave)

## Problem

Even with a good plan, some tracked items will have weak or missing evidence. Without explicit gap detection:
- Sub-agents assume someone else covered it → items silently fall through the cracks
- The coverage check at the end catches gaps too late → expensive repair cycles
- Nobody knows which items need more research until the pipeline is almost done

## Solution: Two-Point Gap Detection

Run gap detection at **two points** in the pipeline:

```
Plan → Presearch → [Gap Detection #1] → Wave 1 → ... → All Waves Done → [Gap Detection #2] → Coverage Check
                     ↑ pre-dispatch                                    ↑ post-waves
                  catches gaps before                              catches gaps that
                  sub-agents start                                 waves didn't fill
```

### Pre-Dispatch Gap Detection

After presearch, check which tracked items have **zero or weak evidence**:

```python
def detect_gaps_pre_dispatch(task_dir):
    """Identify tracked items with insufficient evidence before Wave 1.

    Runs after presearch seeds the evidence store.
    Items with no matching evidence get flagged as gaps.
    """
    plan = load_json(task_dir / "plan.json")
    evidence_store = load_json(task_dir / "evidence_store.json")
    presearch = load_json(task_dir / "presearch_results.json")

    evidence_by_item = _index_evidence_by_item(evidence_store)
    gaps = []

    for item in plan.get("tracked_items", []):
        item_id = item["id"]
        matching = evidence_by_item.get(item_id, [])

        if not matching:
            gaps.append({
                "gap_id": f"G-{item_id}",
                "item_id": item_id,
                "item": item["item"],
                "owner": item.get("owner", "unassigned"),
                "priority": item.get("priority", "medium"),
                "gap_type": "no_evidence",
                "status": "unresolved",
                "suggested_action": "sub_agent_must_search",
            })
        elif all(e.get("confidence", "low") == "low" for e in matching):
            gaps.append({
                "gap_id": f"G-{item_id}",
                "item_id": item_id,
                "item": item["item"],
                "owner": item.get("owner", "unassigned"),
                "priority": item.get("priority", "medium"),
                "gap_type": "weak_evidence",
                "evidence_count": len(matching),
                "status": "unresolved",
                "suggested_action": "sub_agent_should_verify",
            })

    # Write gap report
    gap_report = {
        "schema_version": "gap_report.v1",
        "phase": "pre_dispatch",
        "total_gaps": len(gaps),
        "gaps": gaps,
        "critical_gaps": [g for g in gaps if g["priority"] in ("critical", "high")],
    }
    write_json(task_dir / "gap_report.json", gap_report)

    return gap_report
```

### Post-Wave Gap Detection

After all waves complete, check which gaps were filled and which remain:

```python
def detect_gaps_post_waves(task_dir, outputs_dir):
    """Check gap fill status after all waves complete.

    Compares pre-dispatch gaps against current evidence store
    and sub-agent outputs to determine what was resolved.
    """
    pre_gaps = load_json(task_dir / "gap_report.json")
    evidence_store = load_json(task_dir / "evidence_store.json")
    evidence_by_item = _index_evidence_by_item(evidence_store)

    updated_gaps = []
    for gap in pre_gaps.get("gaps", []):
        item_id = gap["item_id"]
        current_evidence = evidence_by_item.get(item_id, [])

        if current_evidence and gap["gap_type"] == "no_evidence":
            gap["status"] = "resolved"
            gap["resolved_by"] = "wave_sub_agent"
            gap["evidence_count"] = len(current_evidence)
        elif current_evidence and gap["gap_type"] == "weak_evidence":
            high_conf = [e for e in current_evidence if e.get("confidence") in ("high", "medium")]
            if high_conf:
                gap["status"] = "resolved"
                gap["resolved_by"] = "wave_sub_agent"
            else:
                gap["status"] = "partially_resolved"
        else:
            gap["status"] = "unresolved"

        updated_gaps.append(gap)

    # New gaps discovered during waves (from sub-agent data_gaps)
    new_gaps = _extract_gaps_from_outputs(outputs_dir)

    gap_report = {
        "schema_version": "gap_report.v1",
        "phase": "post_waves",
        "total_gaps": len(updated_gaps) + len(new_gaps),
        "resolved": len([g for g in updated_gaps if g["status"] == "resolved"]),
        "unresolved": len([g for g in updated_gaps if g["status"] == "unresolved"]),
        "new_gaps": len(new_gaps),
        "gaps": updated_gaps + new_gaps,
    }
    write_json(task_dir / "gap_report.json", gap_report)

    return gap_report
```

### Extracting Gaps from Sub-Agent Outputs

Sub-agents report data gaps in their data sidecars. Collect these:

```python
def _extract_gaps_from_outputs(outputs_dir):
    """Extract data_gaps reported by sub-agents in their data sidecars."""
    new_gaps = []
    for sidecar_path in sorted(outputs_dir.glob("*-data.json")):
        try:
            sidecar = json.loads(sidecar_path.read_text(encoding="utf-8"))
        except (json.JSONDecodeError, ValueError):
            continue

        role = sidecar_path.stem.replace("-data", "")
        for gap_text in sidecar.get("data_gaps", []):
            new_gaps.append({
                "gap_id": f"G-NEW-{len(new_gaps)+1:03d}",
                "item_id": None,  # Not tied to a tracked item
                "item": gap_text,
                "owner": role,
                "priority": "medium",
                "gap_type": "discovered_during_analysis",
                "status": "unresolved",
                "discovered_by": role,
            })

    return new_gaps
```

## Integration with Coverage Check

Gap detection feeds into the coverage check (Stage 5 in quality-chain.md):

```python
def validate_coverage(task_dir):
    """Check if every tracked item has been addressed.

    Uses gap_report.json to identify unresolved items.
    """
    gap_report = load_json(task_dir / "gap_report.json")
    plan = load_json(task_dir / "plan.json")

    unresolved = [g for g in gap_report.get("gaps", []) if g["status"] == "unresolved"]
    unresolved_item_ids = {g["item_id"] for g in unresolved if g.get("item_id")}

    coverage = {
        "total_items": len(plan.get("tracked_items", [])),
        "resolved": gap_report.get("resolved", 0),
        "unresolved": len(unresolved),
        "coverage_rate": 0,
    }
    if coverage["total_items"] > 0:
        coverage["coverage_rate"] = coverage["resolved"] / coverage["total_items"]

    # FAIL if critical/high items are unresolved
    critical_unresolved = [
        g for g in unresolved
        if g.get("priority") in ("critical", "high") and g.get("item_id")
    ]

    if critical_unresolved:
        return {
            "ok": True,
            "needs_repair": True,
            "coverage": coverage,
            "repair_items": critical_unresolved,
        }

    return {"ok": True, "coverage": coverage}
```

## When to Use

| Scenario | Pre-dispatch gap detection? | Post-wave gap detection? |
|----------|---------------------------|------------------------|
| Large pipeline (5+ roles, 20+ items) | **Yes** | **Yes** |
| Small pipeline (2-3 roles, <10 items) | Optional | **Yes** (always useful) |
| Presearch already covers all items | Skip (redundant) | **Yes** |
| Evidence store pre-populated from prior run | Skip | **Yes** |

## Design Checklist

- [ ] Pre-dispatch gap detection runs after presearch, before Wave 1
- [ ] Post-wave gap detection runs after all waves, before coverage check
- [ ] Gap report tracks resolution status (unresolved → partially_resolved → resolved)
- [ ] Sub-agent data_gaps are collected as new gaps
- [ ] Gap report feeds into coverage check for FAIL/repair decisions
- [ ] Critical/high priority unresolved gaps trigger repair
