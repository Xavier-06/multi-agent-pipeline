# Synthesis as Sub-Agent (Final Report Assembly via Dispatch)

## Problem

Many pipelines treat final report assembly as a simple script that concatenates wave outputs. But for complex domains, assembly requires **reasoning**: cross-referencing findings, resolving contradictions, weaving a narrative, and adding citations. A script can't do this — it needs a sub-agent.

Treating synthesis as "just another sub-agent dispatch" means:
- It uses the same dispatch infrastructure (manifest, brief, instruction store, connector IDs)
- It benefits from the same quality gates (repair, retry, degradation)
- It produces the same 3-file contract (md + data + meta)
- It gets the same shared state context (all wave outputs + evidence store)

## Architecture

```
All Waves Complete → Coverage Check → Cross-Section Review
                                          ↓
                              Synthesis Prepare (needs_dispatch)
                                          ↓
                              Synthesis Sub-Agent (reads all wave outputs)
                                          ↓
                              Synthesis Collect (quality check + repair)
                                          ↓
                              Final Assembly → Delivery
```

## When to Use Sub-Agent Synthesis vs Script Assembly

| Scenario | Use sub-agent? | Reason |
|----------|---------------|--------|
| Simple concatenation of wave outputs | No — use script | No reasoning needed |
| Cross-reference findings across waves | **Yes** | Needs to resolve contradictions |
| Weave narrative from structured outputs | **Yes** | Creative reasoning required |
| Add citations/footnotes from evidence store | **Yes** | Needs to query evidence store |
| Generate executive summary with judgment | **Yes** | Needs synthesis reasoning |
| Output must be a single coherent document | **Yes** | Can't just concatenate |

## Implementation

### Synthesis Prepare Phase

The synthesis prepare phase is a **`needs_dispatch` phase** — same as wave dispatch, but with `has_more=False` (only one synthesis sub-agent):

```python
def _run_synthesis_prepare(runtime_root, job_ctx):
    """Dispatch a synthesis sub-agent to assemble the final report.

    This is a needs_dispatch phase, NOT a simple script.
    The synthesis sub-agent reads all wave outputs + evidence store
    and produces a coherent final report.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    outputs_dir = _outputs_dir(runtime_root, job_ctx)

    # Build synthesis brief — lists ALL inputs
    brief_path = _build_synthesis_brief(task_dir, outputs_dir)

    # Load instruction from instruction store
    instruction_store = runtime_root / f"instruction_store_{profile}"
    prompt_path = instruction_store / "synthesis.md"
    system_prompt = prompt_path.read_text(encoding="utf-8") if prompt_path.exists() else "ERROR: synthesis prompt not found"

    # Build manifest
    manifest = {
        "manifest_version": "1.0",
        "task_id": job_ctx.job_id,
        "role": "synthesis",
        "system_prompt": system_prompt,
        "brief_path": str(brief_path),
        "output_path": str(outputs_dir / "synthesis.md"),
        "connectorIds": _get_synthesis_connectors(),
        "created_at": timestamp_now(),
    }
    manifest_path = task_dir / "manifest_synthesis.json"
    manifest_path.write_text(json.dumps(manifest, indent=2))

    return {
        "ok": True,
        "needs_dispatch": True,
        "has_more": False,           # Only ONE synthesis sub-agent
        "mode": "synthesis_prepare",
        "dispatch_info": {
            "manifests": [str(manifest_path)],
            "roles": ["synthesis"],
        },
        "instruction": "MANDATORY: Read manifest. Spawn ONE synthesis sub-agent.",
    }
```

### Synthesis Brief Construction

The synthesis brief is special — it lists **ALL wave outputs** as input files:

```python
def _build_synthesis_brief(task_dir, outputs_dir):
    """Build brief for the synthesis sub-agent.

    The synthesis sub-agent needs ALL prior outputs to assemble the final report.
    This brief is typically larger than wave sub-agent briefs.
    """
    brief_path = task_dir / "brief_synthesis.md"
    lines = [
        "# Synthesis Brief",
        "",
        f"**Task ID:** {task_dir.name}",
        "",
        "## Input Files (read ALL completely)",
        "",
    ]

    # 1. Shared state (current state of all findings)
    shared_page = task_dir / "shared_state_page.md"
    if shared_page.exists():
        lines.append(f"- Shared State Page: `{shared_page}` ← START HERE")

    # 2. Evidence store (all collected data)
    evidence_store = task_dir / "evidence_store.json"
    if evidence_store.exists():
        lines.append(f"- Evidence Store: `{evidence_store}`")

    # 3. ALL wave outputs (not just the last wave)
    lines.append("")
    lines.append("## All Wave Outputs (read ALL)")
    lines.append("")
    for md_file in sorted(outputs_dir.glob("*.md")):
        if md_file.name != "synthesis.md":  # Skip self if resuming
            lines.append(f"- `{md_file}`")

    # 4. All data sidecars (for citation data)
    lines.append("")
    lines.append("## Data Sidecars (for citations)")
    lines.append("")
    for json_file in sorted(outputs_dir.glob("*-data.json")):
        lines.append(f"- `{json_file}`")

    # 5. Coverage / review results
    for gate_file in task_dir.glob("*_gate.json"):
        lines.append(f"- `{gate_file}`")

    lines.extend([
        "",
        "## Output Requirements",
        "",
        f"Write to: `{outputs_dir / 'synthesis.md'}`",
        "",
        "Your output MUST include 3 files:",
        "1. `synthesis.md` — Final assembled report",
        "2. `synthesis-data.json` — Citation index",
        "3. `synthesis-meta.json` — Assembly metadata",
        "",
        "## Assembly Rules",
        "",
        "- NEVER invent new data — only use what's in wave outputs and evidence store",
        "- Cross-reference findings across waves",
        "- Resolve contradictions explicitly",
        "- Add [^N] citations for all quantitative data",
        "- Generate executive summary with verdict",
    ])

    brief_path.write_text("\n".join(lines) + "\n", encoding="utf-8")
    return brief_path
```

### Synthesis Collect (with Repair)

Synthesis collect checks citation density and structural quality:

```python
def _run_synthesis_collect(runtime_root, job_ctx):
    """Validate synthesis output. Repair if citation density is too low."""
    task_dir = _task_dir(runtime_root, job_ctx)
    outputs_dir = _outputs_dir(runtime_root, job_ctx)
    synthesis_path = outputs_dir / "synthesis.md"

    if not synthesis_path.exists() or synthesis_path.stat().st_size < 500:
        return {
            "ok": True,
            "needs_dispatch": True,
            "has_more": False,
            "dispatch_info": {"incomplete_roles": ["synthesis"]},
            "instruction": "Re-dispatch synthesis sub-agent — output missing or too small.",
        }

    # Check citation density
    text = synthesis_path.read_text(encoding="utf-8")
    citation_count = len(re.findall(r"\[\^\d+\]", text))
    text_k = max(len(text) // 1000, 1)
    min_citations = max(3, (len(text) // 2000) * MIN_REFS_PER_2000)

    if citation_count < min_citations:
        # Trigger citation repair
        repair_manifest = _build_citation_repair_manifest(task_dir, synthesis_path)
        return {
            "ok": True,
            "needs_dispatch": True,
            "has_more": False,
            "dispatch_info": {
                "manifests": [str(repair_manifest)],
                "is_repair": True,
            },
            "instruction": "Dispatch citation repair sub-agent.",
        }

    return {
        "ok": True,
        "mode": "synthesis_collect",
        "result": {
            "word_count": len(text.split()),
            "citation_count": citation_count,
            "citation_density": citation_count / text_k,
        },
    }
```

### Citation Repair

The citation repair sub-agent reads the synthesis output + all data sidecars and inserts missing citations:

```python
def _build_citation_repair_manifest(task_dir, synthesis_path):
    """Build a repair manifest for citation density."""
    repair_prompt = f"""You are a citation repair agent. Read the synthesis report
and all data sidecars. Insert [^N] footnote markers after quantitative data
that lacks citations. Append footnote definitions at the end.

DO NOT change the report content — only add citations.
Use source_url from data sidecars for footnote URLs.
"""
    manifest = {
        "manifest_version": "1.0",
        "role": "synthesis_citation_repair",
        "system_prompt": repair_prompt,
        "output_path": str(synthesis_path),
        "is_repair": True,
    }
    manifest_path = task_dir / "manifest_synthesis_citation_repair.json"
    manifest_path.write_text(json.dumps(manifest, indent=2))
    return manifest_path
```

## Connector IDs for Synthesis

Synthesis sub-agents typically need the same connectors as wave sub-agents (to verify facts), plus optionally document generation connectors:

```python
def _get_synthesis_connectors():
    """Synthesis sub-agent connectors — typically same as wave + optional extras."""
    base = list(BASE_CONNECTOR_IDS)
    # Optional: add document platform connectors for writing
    # base.append("tencent-docs")
    return base
```

## Integration with Final Assembly

After synthesis collect passes, the final assembly phase converts the synthesis output into the delivery format:

```
Synthesis Collect (PASS) → Final Assembly → Delivery Gate
```

Final assembly is typically a **script** (not a sub-agent) that:
1. Reads `synthesis.md`
2. Converts to delivery format (DOCX, PDF, HTML)
3. Generates per-section exports (if applicable)
4. Registers artifacts

See [delivery-multi-format.md](delivery-multi-format.md) for the delivery pattern.

## Design Checklist

- [ ] Synthesis prepare returns `needs_dispatch` with `has_more=False` (single sub-agent)
- [ ] Synthesis brief lists ALL wave outputs + evidence store + gate results
- [ ] Synthesis instruction store prompt exists at `instruction_store_{profile}/synthesis.md`
- [ ] Synthesis collect checks citation density (configurable threshold)
- [ ] Citation repair manifest is generated when density is too low
- [ ] Max repair retries = 1 (citation repair rarely needs multiple rounds)
- [ ] Final assembly is a separate phase (script, not sub-agent)
