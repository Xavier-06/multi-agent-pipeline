# Delivery: Multi-Format Output & Artifact Registration

## Problem

The final phase of a pipeline must deliver results in formats suitable for different consumers:
- **Markdown** — for downstream pipelines or agent consumption
- **DOCX/PDF** — for human stakeholders who expect formatted documents
- **Per-section exports** — for reviewers who want individual dimension reports
- **Artifact registration** — so the platform (WorkBuddy, CI/CD, etc.) knows what was produced

A naive delivery phase just writes one file. A production delivery phase handles multiple formats, graceful degradation (if DOCX generation fails, still deliver Markdown), and artifact tracking.

## Architecture

```
Synthesis Collect (PASS) → Final Assembly → Delivery Phase
                                              ↓
                                    ┌────────┼────────┐
                                    ↓        ↓        ↓
                              Markdown   DOCX/PDF  Per-Section
                                    ↓        ↓        ↓
                                    └────────┼────────┘
                                              ↓
                                    Artifact Registration
```

## Implementation

### Final Assembly Phase

Final assembly is typically a **script** (not a sub-agent) that reads the synthesis output and produces structured delivery artifacts:

```python
def _run_final_assembly(runtime_root, job_ctx):
    """Assemble synthesis output into delivery-ready artifacts.

    This is a SCRIPT phase, not a sub-agent dispatch.
    It reads synthesis.md and produces:
    - final_report.md (clean, formatted)
    - final_report.json (structured metadata)
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    outputs_dir = _outputs_dir(runtime_root, job_ctx)
    delivery_dir = task_dir / "delivery"
    delivery_dir.mkdir(exist_ok=True)

    synthesis_path = outputs_dir / "synthesis.md"
    if not synthesis_path.exists():
        return {"ok": False, "error": "synthesis.md not found"}

    text = synthesis_path.read_text(encoding="utf-8")

    # Clean and format
    clean_text = _clean_synthesis(text)

    # Write final markdown
    final_md = delivery_dir / "final_report.md"
    final_md.write_text(clean_text, encoding="utf-8")

    # Write structured metadata
    final_meta = delivery_dir / "final_report.json"
    final_meta.write_text(json.dumps({
        "entity": job_ctx.entity,
        "word_count": len(clean_text.split()),
        "section_count": _count_sections(clean_text),
        "assembled_at": timestamp_now(),
    }, indent=2), encoding="utf-8")

    return {
        "ok": True,
        "mode": "final_assembly",
        "result": {
            "delivery_dir": str(delivery_dir),
            "files": ["final_report.md", "final_report.json"],
        },
    }
```

### Delivery Phase (Multi-Format)

The delivery phase generates multiple formats with **graceful degradation**:

```python
def _run_delivery(runtime_root, job_ctx):
    """Generate delivery artifacts in multiple formats.

    Graceful degradation: if a format fails, continue with others.
    The pipeline never blocks on a non-critical format failure.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    delivery_dir = task_dir / "delivery"
    final_md = delivery_dir / "final_report.md"

    if not final_md.exists():
        return {"ok": False, "error": "final_report.md not found"}

    artifacts = {}
    deferred_fixes = []

    # Format 1: Markdown (always succeeds — it already exists)
    artifacts["markdown"] = {
        "path": str(final_md),
        "format": "md",
        "status": "ok",
    }

    # Format 2: DOCX (optional — may fail if python-docx unavailable)
    try:
        docx_path = delivery_dir / "final_report.docx"
        _generate_docx(final_md, docx_path)
        artifacts["docx"] = {
            "path": str(docx_path),
            "format": "docx",
            "status": "ok",
        }
    except Exception as e:
        deferred_fixes.append(f"DOCX generation failed: {e}")
        artifacts["docx"] = {"status": "failed", "reason": str(e)}

    # Format 3: Per-section exports (optional)
    try:
        section_exports = _export_sections(final_md, delivery_dir)
        for i, path in enumerate(section_exports):
            artifacts[f"section_{i}"] = {
                "path": str(path),
                "format": "md",
                "status": "ok",
            }
    except Exception as e:
        deferred_fixes.append(f"Section export failed: {e}")

    # Register artifacts
    _register_artifacts(task_dir, artifacts)

    return {
        "ok": True,
        "mode": "delivery",
        "result": {
            "artifacts": artifacts,
            "artifact_count": len([a for a in artifacts.values() if a["status"] == "ok"]),
            "deferred_fixes": deferred_fixes,
        },
    }
```

### DOCX Generation Pattern

```python
def _generate_docx(md_path, docx_path):
    """Convert Markdown report to DOCX.

    This is a best-effort conversion. If python-docx is unavailable
    or conversion fails, the delivery phase continues with Markdown only.
    """
    try:
        from docx import Document
        from docx.shared import Pt, RGBColor
    except ImportError:
        raise RuntimeError("python-docx not installed — skipping DOCX generation")

    text = md_path.read_text(encoding="utf-8")
    doc = Document()

    # Parse markdown sections and build DOCX
    for section in _parse_md_sections(text):
        if section.get("heading"):
            doc.add_heading(section["heading"], level=section.get("level", 1))
        if section.get("content"):
            doc.add_paragraph(section["content"])
        if section.get("table"):
            _add_table(doc, section["table"])

    doc.save(str(docx_path))
```

### Per-Section Exports

When wave sub-agents produce dimension/section reports, the delivery phase can export each as a standalone document:

```python
def _export_sections(final_md, delivery_dir):
    """Export individual sections as standalone files.

    Useful when stakeholders want to review specific dimensions
    independently from the full report.
    """
    outputs_dir = delivery_dir.parent / "outputs"
    exported = []

    for md_file in sorted(outputs_dir.glob("*.md")):
        if md_file.name in ("synthesis.md",):
            continue
        export_path = delivery_dir / md_file.name
        # Copy or transform as needed
        import shutil
        shutil.copy2(md_file, export_path)
        exported.append(export_path)

    return exported
```

### Artifact Registration

Register produced artifacts so the platform knows what was delivered:

```python
def _register_artifacts(task_dir, artifacts):
    """Write artifact manifest for platform consumption.

    The manifest lists all produced files with their format, status,
    and metadata. WorkBuddy or other platforms read this to present
    results to the user.
    """
    manifest_path = task_dir / "delivery" / "artifacts.json"
    manifest_path.write_text(json.dumps({
        "schema_version": "delivery_artifacts.v1",
        "artifacts": artifacts,
        "total_ok": len([a for a in artifacts.values() if a.get("status") == "ok"]),
        "total_failed": len([a for a in artifacts.values() if a.get("status") == "failed"]),
        "generated_at": timestamp_now(),
    }, ensure_ascii=False, indent=2), encoding="utf-8")
```

### Delivery Gate (Final Quality Check)

The delivery gate runs AFTER delivery to verify no information leakage, placeholders, or contradictions:

```python
def _run_delivery_gate(runtime_root, job_ctx):
    """Final quality check on delivery artifacts.

    Multi-layer adversarial verification:
    - L1: Information leakage (internal paths, API keys)
    - L2: Placeholder residue (TODO, TBD, 待补充) — HARD BLOCK
    - L3: Internal contradictions
    - L4: Numeric verification against evidence store
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    delivery_dir = task_dir / "delivery"
    final_md = delivery_dir / "final_report.md"

    text = final_md.read_text(encoding="utf-8")
    issues = []

    # L1: Information leakage
    leaked = _check_leakage(text)
    if leaked:
        # Auto-redact
        text = _redact(text, leaked)
        final_md.write_text(text, encoding="utf-8")
        issues.append({"layer": "L1", "action": "auto-redacted", "count": len(leaked)})

    # L2: Placeholder residue — HARD BLOCK
    placeholders = re.findall(r"\[(?:TODO|TBD|待补充|PLACEHOLDER)[^\]]*\]", text, re.IGNORECASE)
    if placeholders:
        issues.append({
            "layer": "L2",
            "severity": "BLOCKING",
            "placeholders": placeholders,
        })
        return {
            "ok": False,
            "error": f"L2 BLOCKING: {len(placeholders)} placeholders found",
            "issues": issues,
        }

    # L3-L4: Soft checks
    # ... (domain-specific)

    verdict = "PASS" if not issues else "WARN"
    return {
        "ok": True,
        "mode": "delivery_gate",
        "result": {"verdict": verdict, "issues": issues},
    }
```

## Per-Section Independent Export (Multi-Artifact Delivery)

For complex reports, individual sections/dimensions may need to be exported as **standalone deliverables** alongside the consolidated report. This is common when different stakeholders consume different sections independently, or when each section is a self-contained analysis product.

### Pattern

```python
SECTION_REGISTRY = {
    "section_a": {"title": "Section A Title", "role_slug": "section_a_analyst", "order": 1},
    "section_b": {"title": "Section B Title", "role_slug": "section_b_analyst", "order": 2},
    # ... customize per pipeline
}

def export_per_section(task_dir, runtime_root, job_ctx):
    """Export each section as an independent deliverable.

    Use when:
    - Different stakeholders need different sections
    - Sections are self-contained enough to stand alone
    - User requests individual section deliverables
    """
    outputs_dir = _outputs_dir(runtime_root, job_ctx)
    delivery_dir = task_dir / "delivery"
    delivery_dir.mkdir(parents=True, exist_ok=True)

    section_exports = []
    for section_id, section_meta in SECTION_REGISTRY.items():
        md_path = outputs_dir / f"{section_meta['role_slug']}.md"
        if not md_path.exists() or md_path.stat().st_size < 100:
            continue  # Skip empty/missing sections

        content = md_path.read_text(encoding="utf-8")

        # Per-section DOCX — reuses consolidated report styling/rendering
        section_docx_path = delivery_dir / f"{section_meta['title']}.docx"
        try:
            _build_section_docx(section_meta, content, section_docx_path)
            section_exports.append({
                "section_id": section_id,
                "title": section_meta["title"],
                "docx_path": str(section_docx_path),
                "status": "ok",
            })
        except Exception as e:
            # Per-section DOCX failure is non-blocking
            section_exports.append({
                "section_id": section_id,
                "title": section_meta["title"],
                "status": "failed",
                "reason": str(e),
            })

    return section_exports
```

### Design Decisions

| Decision | Guidance |
|----------|----------|
| Output directory | **Flat** in `delivery/` root — not nested subdirs, for easy access |
| Naming | Use section title as filename (human-readable) |
| Styling reuse | Per-section DOCX should share styling/rendering with consolidated report |
| Conditional export | Only export sections whose MD exists and has substance (>100 chars) |
| Artifact registration | Register each section export as a separate artifact in `artifacts.json` |
| Graceful degradation | Per-section DOCX failure is non-blocking — MD copy still available |

### When to Use Per-Section Export

| Scenario | Use it? |
|----------|---------|
| Report has 3+ self-contained sections | Yes |
| Different readers for different sections | Yes |
| Single-section report or tightly coupled sections | No — consolidated is sufficient |
| User explicitly requests per-section output | Yes |

## Graceful Degradation Summary

| Format | Required? | On failure |
|--------|-----------|------------|
| Markdown | **Yes** | Pipeline fails — this is the minimum |
| DOCX/PDF | No | Logged as deferred_fix, Markdown still delivered |
| Per-section exports | No | Logged as deferred_fix |
| Artifact registration | **Yes** | Pipeline fails — platform can't find outputs |

## Design Checklist

- [ ] Final assembly is a script (not sub-agent dispatch)
- [ ] Delivery phase generates multiple formats with graceful degradation
- [ ] DOCX/PDF failure does NOT block pipeline — Markdown is always delivered
- [ ] Per-section exports are optional
- [ ] Artifact manifest (`artifacts.json`) is always written
- [ ] Delivery gate checks L1 (leakage) + L2 (placeholders) at minimum
- [ ] L2 placeholder residue is the ONLY hard-block layer
