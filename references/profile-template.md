# Profile Template

Use this template when building a new pipeline for a different domain.

## Minimal Profile Skeleton

```python
from __future__ import annotations
import json, time, shutil
from pathlib import Path
from typing import Any
from runtime.profiles.base import JobContext, PipelineProfile


def _task_dir(runtime_root: Path, job_ctx: JobContext) -> Path:
    ws = getattr(job_ctx, "workspace", None)
    if ws: return ws.root
    d = runtime_root / "tasks" / job_ctx.job_id
    d.mkdir(parents=True, exist_ok=True)
    return d


# ── Phase handlers ────────────────────────────────

def _run_preflight(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Environment check + task registration + token pre-check."""
    task_dir = _task_dir(runtime_root, job_ctx)
    # Check required tools/APIs are available
    # Write initial config files
    return {"ok": True, "mode": "preflight", "result": {"ready": True}}


def _run_research_plan(runtime_root: Path, job_ctx: JobContext) -> dict:
    """LLM generates research plan. Gate: plan_status must be 'ready'."""
    task_dir = _task_dir(runtime_root, job_ctx)
    plan_path = task_dir / f"{job_ctx.job_id}-research_plan.json"
    # LLM call to generate plan based on entity/query
    # Validate: plan_status == "ready"
    return {"ok": True, "mode": "research_plan", "result": {"plan_path": str(plan_path)}}


def _run_presearch(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Multi-source pre-search to gather raw data."""
    task_dir = _task_dir(runtime_root, job_ctx)
    # Call data source APIs, write raw_corpus.json
    return {"ok": True, "mode": "presearch", "result": {"corpus_size": 120}}


def _run_fact_store(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Bootstrap fact store from presearch results."""
    task_dir = _task_dir(runtime_root, job_ctx)
    # Extract facts, write fact_store.json
    return {"ok": True, "mode": "fact_store", "result": {"facts_count": 45}}


def _run_dispatch_prepare(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Write manifests + briefs for Wave 1 sub-agents. Return needs_dispatch."""
    task_dir = _task_dir(runtime_root, job_ctx)

    # Define sub-agent specs
    sub_specs = [
        {
            "role_name": "step_sub01",
            "description": "Sub-direction 1: ...",
            "output_file": str(task_dir / "step_sub01.md"),
            "key_inputs": {"plan_path": str(task_dir / "research_plan.json")},
        },
        # ... more steps
    ]

    # Write manifest for each sub-agent
    manifests = []
    for spec in sub_specs:
        manifest = {
            "task_id": job_ctx.job_id,
            "role": spec["role_name"],
            "output_path": spec["output_file"],
            "status": "pending",
        }
        mp = task_dir / f"manifest_{spec['role_name']}.json"
        mp.write_text(json.dumps(manifest, ensure_ascii=False, indent=2))
        manifests.append(str(mp))

    return {
        "ok": True,
        "needs_dispatch": True,
        "mode": "dispatch_prepare",
        "dispatch_info": {
            "manifests": manifests,
            "roles": [s["role_name"] for s in sub_specs],
            "task_dir": str(task_dir),
        },
        "instruction": "MANDATORY: Read manifests, use Agent tool to spawn sub-agents.",
    }


def _run_dispatch_collect(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Check sub-agent outputs. Validate Section Packages."""
    task_dir = _task_dir(runtime_root, job_ctx)
    completed = []
    missing = []

    for step in ["step_sub01", "step_sub02", "step_sub03"]:
        p = task_dir / f"{step}.md"
        if p.exists() and p.stat().st_size > 100:
            completed.append(step)
        else:
            missing.append(step)

    return {
        "ok": len(missing) == 0,
        "mode": "dispatch_collect",
        "result": {"completed": len(completed), "missing": missing},
    }


def _run_quality_chain(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Section Package validation → Debate Review → Final Assembly."""
    task_dir = _task_dir(runtime_root, job_ctx)
    # Run validation, debate review, assembly
    # Write final_report.md
    return {"ok": True, "mode": "quality_chain"}


def _run_delivery(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Adversarial verification + format generation + copy to desktop."""
    task_dir = _task_dir(runtime_root, job_ctx)
    # Run 6-layer verification
    # Generate DOCX/PDF/XLSX
    # Copy to ~/Desktop/
    return {"ok": True, "mode": "delivery", "deliver_to_user": True}


# ── Profile class ─────────────────────────────────

class MyNewProfile(PipelineProfile):
    def __init__(self, runtime_root: Path):
        super().__init__(
            name="my_new_pipeline",
            job_type="my_domain",
            phase_handlers={
                "phase0_preflight":           lambda ctx: _run_preflight(runtime_root, ctx),
                "phase03_research_plan":      lambda ctx: _run_research_plan(runtime_root, ctx),
                "phase04_presearch":          lambda ctx: _run_presearch(runtime_root, ctx),
                "phase2_fact_store":          lambda ctx: _run_fact_store(runtime_root, ctx),
                "phase4_dispatch_prepare":    lambda ctx: _run_dispatch_prepare(runtime_root, ctx),
                "phase4_dispatch_collect":    lambda ctx: _run_dispatch_collect(runtime_root, ctx),
                "phase45_quality_chain":      lambda ctx: _run_quality_chain(runtime_root, ctx),
                "phase5_delivery":            lambda ctx: _run_delivery(runtime_root, ctx),
            },
        )
        self.runtime_root = runtime_root
```

## Phase Naming Convention

Use the IR/BP naming for consistency:

| Phase | Pattern | Purpose |
|-------|---------|---------|
| `phase0_*` | Preflight | Environment check, task registration |
| `phase02_*` | Entity verify | Validate subject exists, gather basic info |
| `phase03_*` | Research plan | LLM generates plan, gate check |
| `phase04_*` | Presearch | Multi-source data gathering |
| `phase12_*` | Precompute | Derived data (metrics, networks, etc.) |
| `phase15_*` | Extract | Full content extraction |
| `phase2_*` | Fact store | Initialize fact store |
| `phase4_*` | Dispatch | Sub-agent wave dispatch + collect |
| `phase45_*` | Validation | Section Package validation |
| `phase46_*` | Debate | Cross-section critique |
| `phase47_*` | Assembly | Final assembly from validated packages |
| `phase5_*` | Delivery | Verification + format + ship |

## Wave Definition Pattern

```python
# Define in a separate config or in the research plan
WAVES = [
    {"name": "Wave 1", "steps": ["step_data"], "depends_on": []},
    {"name": "Wave 2", "steps": ["step_A", "step_B", "step_C"], "depends_on": ["Wave 1"]},
    {"name": "Wave 3", "steps": ["step_synthesis"], "depends_on": ["Wave 2"]},
    {"name": "Wave 4", "steps": ["step_review", "step_risk"], "depends_on": ["Wave 2", "Wave 3"]},
    {"name": "Wave 5", "steps": ["step_master"], "depends_on": ["Wave 1", "Wave 2", "Wave 3", "Wave 4"]},
]
```

## Sub-Agent Brief Template

Each sub-agent gets a brief file that includes:

```markdown
# Brief: {step_name}

## Your Task
{what this specific step should accomplish}

## Entity
{entity name and context}

## Input Files
- Research Plan: {path}
- Fact Store: {path}
- Presearch Results: {paths}
- Prior Step Outputs: {paths if any}

## Output File
Write to: {output_path}

## Output Format
Your file MUST contain this Section Package JSON block at the end:

```json
{
  "section_id": "{step_name}",
  "section_title": "...",
  "key_messages": [...],
  "claims": [{"claim": "...", "fact_ids": [...], "confidence": "..."}],
  "counter_evidence": [...],
  "data_gaps": [...],
  "markdown_draft": "..."
}
```

## Search Tools
{list available data sources and how to call them}

## Rules
1. Every factual claim must cite a fact_id from Fact Store or add new facts
2. Search for missing data yourself — don't return empty-handed
3. Counter-evidence is mandatory for every major claim
4. When output file is written, you are DONE
```

## Adapting to Your Domain

Key decisions when creating a new profile:

1. **How many steps/waves?** → Depends on how many independent research dimensions
2. **What data sources?** → Determines presearch phase complexity
3. **What goes in Section Package?** → Domain-specific fields (e.g., `trl_signal` for tech assessment, `financial_metrics` for finance)
4. **What does Debate Review check?** → Domain-specific quality dimensions
5. **What format for delivery?** → DOCX, PDF, HTML, XLSX, etc.
6. **Does it need document intake?** → If input is a file (PDF/PPTX), add OCR phase
