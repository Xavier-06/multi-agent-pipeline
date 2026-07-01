# Profile Template

Use this template when building a new pipeline for a different domain.

## Complete Profile with All Production Patterns

```python
from __future__ import annotations
import json, time, shutil, re
from pathlib import Path
from typing import Any
from runtime.profiles.base import JobContext, PipelineProfile


def _task_dir(runtime_root: Path, job_ctx: JobContext) -> Path:
    ws = getattr(job_ctx, "workspace", None)
    if ws: return ws.root
    d = runtime_root / "tasks" / job_ctx.job_id
    d.mkdir(parents=True, exist_ok=True)
    return d


def _outputs_dir(runtime_root: Path, job_ctx: JobContext) -> Path:
    ws = getattr(job_ctx, "workspace", None)
    if ws: return ws.outputs_dir
    return _task_dir(runtime_root, job_ctx)


# ── Shared State (cross-wave context passing) ─────────────────

def _run_shared_state_init(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Initialize shared state before Wave 1.

    Builds a skeleton snapshot from your planning artifact.
    Sub-agents in Wave 1+ read this to know what needs to be done.

    Customize: replace 'your_plan.json' and state fields with your domain's concepts.
    See shared-state.md for the three-layer architecture pattern.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    plan = json.loads((task_dir / "your_plan.json").read_text())

    # Build initial state — adapt fields to your domain
    state = {
        "schema_version": "shared_state.v1",
        "task_id": job_ctx.job_id,
        "entity": job_ctx.entity,
        "item_status": {
            item["id"]: "pending"
            for item in plan.get("tracked_items", [])
        },
        "wave_progress": 0,
        "updated_at": time.strftime("%Y-%m-%dT%H:%M:%S"),
    }
    (task_dir / "shared_state.json").write_text(json.dumps(state, indent=2))
    (task_dir / "shared_state_page.md").write_text(_render_shared_page(state))
    return {"ok": True, "mode": "shared_state_init"}


def _run_shared_state_refresh(runtime_root: Path, job_ctx: JobContext, after_wave: int) -> dict:
    """Rebuild shared state after a wave completes (always post-gate).

    Scans ALL sub-agent outputs from completed waves, resolves statuses,
    extracts open items. Wave N+1 sub-agents read this for context.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    state = _rebuild_shared_state(task_dir, after_wave)  # YOUR IMPLEMENTATION
    (task_dir / "shared_state.json").write_text(json.dumps(state, indent=2))
    (task_dir / "shared_state_page.md").write_text(_render_shared_page(state))
    return {
        "ok": True, "mode": "shared_state_refresh",
        "result": {"after_wave": after_wave},
    }


# ── Planning Phase (script skeleton → LLM enrichment → collect) ───

def _run_plan(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Script generates skeleton → LLM enriches → collect merges.

    This is a SPLIT phase:
    1. Script generates deterministic structure
    2. Returns needs_dispatch → Coordinator calls LLM to enrich
    3. Coordinator resumes with start_phase=collect to merge enrichment

    Customize: the plan schema is entirely domain-specific.
    """
    task_dir = _task_dir(runtime_root, job_ctx)

    plan = {
        "plan_status": "skeleton",
        "entity": job_ctx.entity,
        "tracked_items": [                  # ← your domain's "what needs investigating"
            {"id": "I001", "item": "...", "owner": "role_A", "priority": "high"},
        ],
        "enriched_fields": [],             # ← LLM will fill
    }
    (task_dir / "your_plan.json").write_text(json.dumps(plan, indent=2))

    instruction_path = runtime_root / "instruction_store_yourprofile" / "plan_enrichment.md"
    return {
        "ok": True,
        "needs_dispatch": True,
        "has_more": False,
        "mode": "plan",
        "instruction": f"Read {instruction_path}, the plan skeleton, and the input. "
                       f"Output enrichment JSON. Resume with start_phase='plan_collect'.",
    }


def _run_plan_collect(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Merge LLM enrichment into the skeleton plan."""
    task_dir = _task_dir(runtime_root, job_ctx)
    plan = json.loads((task_dir / "your_plan.json").read_text())
    enrichment_path = task_dir / "enrichment_delta.json"
    if enrichment_path.exists():
        delta = json.loads(enrichment_path.read_text())
        # Apply enrichment — adapt to your schema
        plan["enriched_fields"] = delta.get("enriched_fields", [])
        plan["plan_status"] = "ready"
        (task_dir / "your_plan.json").write_text(json.dumps(plan, indent=2))
    return {"ok": True, "mode": "plan_collect", "result": {"plan_status": plan["plan_status"]}}


# ── Wave Dispatch (prepare + collect split) ────────────────────

# Define your waves — group roles by dependency
WAVES = [
    {"name": "Wave 1", "roles": ["role_A", "role_B", "role_C"], "depends_on": []},
    {"name": "Wave 2", "roles": ["role_D", "role_E"], "depends_on": ["Wave 1"]},
    {"name": "Wave 3", "roles": ["role_F"], "depends_on": ["Wave 1", "Wave 2"]},
]

def _run_dispatch_prepare(runtime_root: Path, job_ctx: JobContext, wave_index: int) -> dict:
    """Write manifest for ONE role in the current wave (sequential mode).

    Sequential: each call dispatches exactly one role. has_more signals if more remain.
    This prevents concurrent writes to shared files.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    wave = WAVES[wave_index]
    roles = wave["roles"]

    # Find the first incomplete role
    for role_slug in roles:
        if not _role_outputs_complete(task_dir, role_slug):
            manifest = _build_manifest(runtime_root, job_ctx, role_slug, wave)
            manifest_path = task_dir / f"manifest_{role_slug}.json"
            manifest_path.write_text(json.dumps(manifest, indent=2))

            remaining = [r for r in roles if r != role_slug and not _role_outputs_complete(task_dir, r)]
            has_more = len(remaining) > 0

            return {
                "ok": True,
                "needs_dispatch": True,
                "has_more": has_more,           # ← drives sequential loop
                "mode": "dispatch_prepare",
                "dispatch_info": {
                    "manifests": [str(manifest_path)],   # ONE manifest
                    "roles": [role_slug],
                    "wave": wave_index + 1,
                },
                "instruction": f"MANDATORY: Read manifest at {manifest_path}. Use Agent tool with "
                               f"system_prompt from manifest. Do NOT spawn multiple sub-agents."
                               + (" Resume with same phase to dispatch next role." if has_more else ""),
            }

    return {"ok": True, "mode": "dispatch_prepare", "result": {"all_roles_complete": True}}


def _run_dispatch_collect(runtime_root: Path, job_ctx: JobContext, wave_index: int) -> dict:
    """Validate all sub-agent outputs with 4-layer defense.

    Layer 1: File existence + size
    Layer 2: JSON validity (sidecars)
    Layer 3: File stability (8s no-growth)
    Layer 4: Re-dispatch incomplete roles
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    wave = WAVES[wave_index]
    incomplete = [r for r in wave["roles"] if not _role_outputs_complete(task_dir, r)]

    if incomplete:
        return {
            "ok": True, "needs_dispatch": True, "has_more": False,
            "mode": "dispatch_collect",
            "dispatch_info": {"incomplete_roles": incomplete},
            "instruction": f"Re-dispatch roles: {incomplete}",
        }

    completed = [r for r in wave["roles"] if _role_outputs_complete(task_dir, r)]
    return {"ok": True, "mode": "dispatch_collect",
            "result": {"completed": len(completed), "wave": wave_index + 1}}


def _role_outputs_complete(task_dir: Path, role_slug: str) -> bool:
    """Check if a role has produced all required files.

    Customize the file list for your domain. The 3-file pattern
    (report.md + data sidecar + metadata sidecar) is a common default.
    """
    outputs_dir = task_dir / "outputs"
    md_path = outputs_dir / f"{role_slug}.md"
    data_path = outputs_dir / f"{role_slug}-data.json"
    meta_path = outputs_dir / f"{role_slug}-meta.json"

    if not md_path.exists() or md_path.stat().st_size < 100:
        return False
    if not data_path.exists() or not _json_valid(data_path):
        return False
    if not meta_path.exists() or not _json_valid(meta_path):
        return False
    return True


def _json_valid(path: Path) -> bool:
    if not path.exists():
        return False
    try:
        json.loads(path.read_text(encoding="utf-8"))
        return True
    except (json.JSONDecodeError, ValueError):
        return False


# ── Quality Gate (with repair) ─────────────────────────────────

_MAX_REPAIR_RETRIES = 1

def _run_quality_gate(runtime_root: Path, job_ctx: JobContext, wave: int) -> dict:
    """Gate: check if sub-agent outputs meet quality criteria.

    On FAIL: generate repair manifests → dispatch repair sub-agents → re-run.
    After max retries: degrade to WARN and continue.

    Customize: the gate evaluation logic is entirely domain-specific.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    gate_result = _evaluate_quality_gate(task_dir, wave)  # YOUR IMPLEMENTATION

    if gate_result.get("needs_repair"):
        repair_manifests = _build_repair_manifests(task_dir, wave, gate_result)
        if repair_manifests:
            first = repair_manifests[0]
            remaining = repair_manifests[1:]
            has_more = len(remaining) > 0
            return {
                "ok": True,
                "needs_dispatch": True,
                "has_more": has_more,
                "mode": "quality_gate_repair",
                "dispatch_info": {
                    "manifests": [first],
                    "remaining_manifests": remaining,
                    "wave": wave,
                    "is_repair": True,
                },
                "instruction": "Dispatch repair sub-agent. Sequential only.",
            }

    return {"ok": gate_result.get("ok", True), "mode": "quality_gate", "result": gate_result}


# ── Instruction Store (hot-loaded system prompts) ──────────────

def _build_manifest(runtime_root: Path, job_ctx: JobContext, role_slug: str, wave: dict) -> dict:
    """Build sub-agent manifest with system_prompt from instruction store."""

    # HOT LOAD: system prompt from file, not hardcoded
    instruction_store = runtime_root / "instruction_store_yourprofile"
    prompt_path = instruction_store / f"{role_slug}.md"
    system_prompt = prompt_path.read_text(encoding="utf-8") if prompt_path.exists() else f"ERROR: prompt not found"

    # Template variable substitution
    system_prompt = system_prompt.replace("{TASK_ID}", job_ctx.job_id)
    system_prompt = system_prompt.replace("{ENTITY}", job_ctx.entity)

    brief_path = _build_brief(runtime_root, job_ctx, role_slug, wave)

    return {
        "manifest_version": "1.0",
        "task_id": job_ctx.job_id,
        "role": role_slug,
        "system_prompt": system_prompt,
        "brief_path": str(brief_path),
        "output_path": str(_outputs_dir(runtime_root, job_ctx) / f"{role_slug}.md"),
        "timeout": 900,
        "connectorIds": [],    # Fill with your MCP connector IDs
        "created_at": time.strftime("%Y-%m-%dT%H:%M:%S"),
        "status": "pending",
    }


def _build_brief(runtime_root: Path, job_ctx: JobContext, role_slug: str, wave: dict) -> Path:
    """Generate a structured brief for the sub-agent.

    The brief lists input files the sub-agent should read, including:
    - Planning artifact
    - Shared state (progress snapshot)
    - Prior wave outputs (Layer 3 cross-wave context)
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    brief_path = task_dir / f"brief_{role_slug}.md"
    lines = [
        f"# Brief: {role_slug}",
        f"",
        f"**Task ID:** {job_ctx.job_id}",
        f"**Entity:** {job_ctx.entity}",
        f"",
        f"## Input Files (read in order)",
        f"- Shared State Page: `{task_dir / 'shared_state_page.md'}` ← read FIRST",
        f"- Shared State JSON: `{task_dir / 'shared_state.json'}`",
        f"- Plan: `{task_dir / 'your_plan.json'}`",
    ]
    # Include prior wave outputs for dependency chains (Layer 3)
    if wave["depends_on"]:
        lines.append(f"", f"## Prior Wave Outputs")
        for dep_wave_name in wave["depends_on"]:
            dep_wave = next(w for w in WAVES if w["name"] == dep_wave_name)
            for dep_role in dep_wave["roles"]:
                dep_path = _outputs_dir(runtime_root, job_ctx) / f"{dep_role}.md"
                if dep_path.exists():
                    lines.append(f"- {dep_role}: `{dep_path}`")
    lines.extend([
        f"",
        f"## Output Files",
        f"Your output MUST include 3 files:",
        f"1. `{role_slug}.md` — Main analysis",
        f"2. `{role_slug}-data.json` — Structured data sidecar",
        f"3. `{role_slug}-meta.json` — Metadata sidecar",
        f"",
        f"Write to: `{_outputs_dir(runtime_root, job_ctx)}`",
    ])
    brief_path.write_text("\n".join(lines) + "\n")
    return brief_path


# ── Sidecar Merge (centralized evidence store) ─────────────────

def _run_sidecar_merge(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Merge all data sidecars into one centralized store.

    See shared-state.md Layer 2 for the pattern.
    Malformed sidecars are skipped, not blocking.
    """
    outputs_dir = _outputs_dir(runtime_root, job_ctx)
    task_dir = _task_dir(runtime_root, job_ctx)

    items = []
    seen_ids = set()
    malformed = []
    for path in outputs_dir.glob("*-data.json"):
        try:
            data = json.loads(path.read_text())
            for item in data.get("items", []):
                if item.get("id") and item["id"] not in seen_ids:
                    seen_ids.add(item["id"])
                    items.append(item)
        except (json.JSONDecodeError, KeyError):
            malformed.append(str(path))

    store_path = task_dir / "evidence_store.json"
    store_path.write_text(json.dumps({"items": items, "malformed": malformed}, indent=2))
    return {"ok": True, "mode": "sidecar_merge",
            "result": {"total_items": len(items), "malformed": len(malformed)}}


# ── Profile class ──────────────────────────────────────────────

class MyNewProfile(PipelineProfile):
    def __init__(self, runtime_root: Path):
        super().__init__(
            name="my_pipeline",
            job_type="my_domain",
            phase_handlers={
                # Phase 01: Intake
                "phase01_intake":             lambda ctx: _run_intake(runtime_root, ctx),
                # Phase 02: Entity verify
                "phase02_entity_verify":      lambda ctx: _run_entity_verify(runtime_root, ctx),
                # Phase 03: Plan (split: skeleton → LLM enrich → collect)
                "phase03_plan":               lambda ctx: _run_plan(runtime_root, ctx),
                "phase03_plan_collect":       lambda ctx: _run_plan_collect(runtime_root, ctx),
                # Phase 04: Presearch / precompute
                "phase04_presearch":          lambda ctx: _run_presearch(runtime_root, ctx),
                # Phase 05: Shared state init (before Wave 1)
                "phase05_shared_state_init":  lambda ctx: _run_shared_state_init(runtime_root, ctx),
                # ── Wave 1 ──
                "phase06_dispatch_prepare":   lambda ctx: _run_dispatch_prepare(runtime_root, ctx, wave_index=0),
                "phase07_dispatch_collect":   lambda ctx: _run_dispatch_collect(runtime_root, ctx, wave_index=0),
                "phase08_wave1_quality_gate": lambda ctx: _run_quality_gate(runtime_root, ctx, wave=1),
                "phase09_sidecar_merge":      lambda ctx: _run_sidecar_merge(runtime_root, ctx),
                "phase10_shared_state_refresh": lambda ctx: _run_shared_state_refresh(runtime_root, ctx, after_wave=1),
                # ── Wave 2 ──
                "phase11_wave2_prepare":      lambda ctx: _run_dispatch_prepare(runtime_root, ctx, wave_index=1),
                "phase12_wave2_collect":      lambda ctx: _run_dispatch_collect(runtime_root, ctx, wave_index=1),
                "phase13_wave2_quality_gate": lambda ctx: _run_quality_gate(runtime_root, ctx, wave=2),
                "phase14_shared_state_refresh": lambda ctx: _run_shared_state_refresh(runtime_root, ctx, after_wave=2),
                # ── Wave 3 ──
                "phase15_wave3_prepare":      lambda ctx: _run_dispatch_prepare(runtime_root, ctx, wave_index=2),
                "phase16_wave3_collect":      lambda ctx: _run_dispatch_collect(runtime_root, ctx, wave_index=2),
                # ── Quality chain + delivery ──
                "phase17_final_assembly":     lambda ctx: _run_final_assembly(runtime_root, ctx),
                "phase18_delivery":           lambda ctx: _run_delivery(runtime_root, ctx),
            },
        )
        self.runtime_root = runtime_root

    def phase_prerequisites(self) -> dict[str, list[str]]:
        return {
            "phase06_dispatch_prepare": ["your_plan.json"],
            "phase09_sidecar_merge": ["your_plan.json"],
        }

    def phase_outputs(self) -> dict[str, list[str]]:
        return {
            "phase03_plan": ["your_plan.json"],
            "phase04_presearch": ["presearch_results.json"],
            "phase05_shared_state_init": ["shared_state.json", "shared_state_page.md"],
            "phase09_sidecar_merge": ["evidence_store.json"],
        }
```

## Phase Naming Convention

Use descriptive names with wave numbers for clarity:

| Pattern | Example | Purpose |
|---------|---------|---------|
| `phaseNN_*` | `phase01_intake`, `phase03_plan` | Pre-dispatch phases |
| `phaseNN_waveN_*` | `phase08_wave1_quality_gate` | Wave-specific gates |
| `phaseNN_shared_state_*` | `phase10_shared_state_refresh` | Cross-wave context updates |
| `phaseNN_dispatch_*` | `phase06_dispatch_prepare`, `phase07_dispatch_collect` | Prepare/collect pairs |

## Key Design Decisions

When creating a new profile, decide:

1. **How many waves?** → Independent dimensions = more waves
2. **Roles per wave?** → Each role = one sub-agent responsibility
3. **Shared state refresh?** → After which waves? (recommended: after waves with 3+ roles)
4. **Quality gate per wave?** → Yes for every wave that produces verifiable output
5. **Repair strategy?** → Max retries + degradation policy per gate type
6. **Output file convention?** → 3-file pattern (report + data sidecar + meta sidecar) or simpler
7. **Stage-aware behavior?** → Early-stage projects get lighter gates
8. **Instruction store location?** → `instruction_store_{profile}/` directory with one `.md` per role
