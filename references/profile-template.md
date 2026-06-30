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


# ── Shared State (cross-wave information hub) ──────────────────

def _run_shared_state_init(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Initialize shared state — a cross-wave information hub that all sub-agents read.

    Produces:
    - shared_state.json: machine-readable current state
    - shared_diligence_page.md: human/agent-readable summary
    - open_questions.json: unresolved data gaps
    - evidence_conflicts.json: counter-evidence queue
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    # Read research plan + claim matrix
    plan = json.loads((task_dir / "research_plan.json").read_text())
    # Build initial shared state
    state = {
        "schema_version": "shared_state.v1",
        "task_id": job_ctx.job_id,
        "entity": job_ctx.entity,
        "claim_status": {
            claim["claim_id"]: "planned"
            for claim in plan.get("claim_matrix", [])
        },
        "wave_progress": 0,
        "updated_at": time.strftime("%Y-%m-%dT%H:%M:%S"),
    }
    (task_dir / "shared_state.json").write_text(json.dumps(state, indent=2))
    # Build human-readable diligence page
    _write_diligence_page(task_dir, state, plan)
    return {"ok": True, "mode": "shared_state_init", "result": {"claim_count": len(state["claim_status"])}}


def _run_shared_state_refresh(runtime_root: Path, job_ctx: JobContext, after_wave: int) -> dict:
    """Refresh shared state after a wave completes.

    Sub-agents in Wave N+1 read the shared state to see what Wave N discovered.
    This is the cross-wave information passing mechanism.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    # Re-read all sub-agent outputs from completed waves
    # Update claim_status: planned → supported/partially_supported/contradicted/not_addressed
    # Update open_questions with new data gaps discovered
    # Update evidence_conflicts with counter-evidence found
    state = _rebuild_shared_state(task_dir, after_wave)
    (task_dir / "shared_state.json").write_text(json.dumps(state, indent=2))
    # Rebuild diligence page with latest info
    plan = json.loads((task_dir / "research_plan.json").read_text())
    _write_diligence_page(task_dir, state, plan)
    return {
        "ok": True, "mode": "shared_state_refresh",
        "result": {"after_wave": after_wave, "claim_count": len(state["claim_status"])},
    }


# ── Research Plan (LLM enrichment pattern) ─────────────────────

def _run_research_plan(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Script generates skeleton → LLM enriches → collect merges.

    This is a SPLIT phase:
    1. Script generates deterministic skeleton (questions + claims + fact_keys)
    2. Returns needs_dispatch → Coordinator calls LLM to enrich
    3. Coordinator resumes with start_phase=collect to merge enrichment
    """
    task_dir = _task_dir(runtime_root, job_ctx)

    # Step 1: Generate skeleton
    plan = {
        "plan_status": "skeleton",
        "entity": job_ctx.entity,
        "core_questions": [
            {"id": "Q1", "question": "...", "owner_section": "step_A", "priority": "high"},
        ],
        "strategic_questions": [],          # ← LLM will fill
        "claim_matrix": [
            {
                "claim_id": "C001",
                "claim": "...",
                "owner_section": "step_A",
                "priority": "high",
                "required_fact_keys": ["market_size", "growth_rate"],
            },
        ],
    }
    (task_dir / "research_plan.json").write_text(json.dumps(plan, indent=2))

    # Step 2: Return needs_dispatch — Coordinator enriches via LLM
    instruction_path = runtime_root / "instruction_store_{profile}" / "research_plan_enrichment.md"
    return {
        "ok": True,
        "needs_dispatch": True,
        "has_more": False,
        "mode": "research_plan",
        "dispatch_info": {},
        "instruction": f"Read {instruction_path}, the plan skeleton, and the input document. "
                       f"Output enrichment JSON with: strategic_questions, claim_priority_deltas, "
                       f"additional_claims, excluded_fact_keys. "
                       f"Then resume with start_phase='research_plan_collect'.",
    }


def _run_research_plan_collect(runtime_root: Path, job_ctx: JobContext) -> dict:
    """Merge LLM enrichment into the skeleton plan."""
    task_dir = _task_dir(runtime_root, job_ctx)
    plan = json.loads((task_dir / "research_plan.json").read_text())
    enrichment_path = task_dir / "enrichment_delta.json"
    if enrichment_path.exists():
        delta = json.loads(enrichment_path.read_text())
        plan["strategic_questions"] = delta.get("strategic_questions", [])
        # Apply priority deltas
        for d in delta.get("claim_priority_deltas", []):
            for claim in plan["claim_matrix"]:
                if claim["claim_id"] == d["claim_id"]:
                    claim["priority"] = d["new_priority"]
        # Add new claims
        plan["claim_matrix"].extend(delta.get("additional_claims", []))
        plan["plan_status"] = "ready"
        (task_dir / "research_plan.json").write_text(json.dumps(plan, indent=2))
    return {"ok": True, "mode": "research_plan_collect", "result": {"plan_status": plan["plan_status"]}}


# ── Wave Dispatch (prepare + collect split) ────────────────────

# Define your waves
WAVES = [
    {"name": "Wave 1", "roles": ["step_A", "step_B", "step_C", "step_D"], "depends_on": []},
    {"name": "Wave 2", "roles": ["step_E", "step_F"], "depends_on": ["Wave 1"]},
    {"name": "Wave 3", "roles": ["step_G"], "depends_on": ["Wave 1", "Wave 2"]},
]

def _run_dispatch_prepare(runtime_root: Path, job_ctx: JobContext, wave_index: int) -> dict:
    """Write manifest for ONE role in the current wave (sequential mode).

    Sequential: each call dispatches exactly one role. has_more signals if more remain.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    wave = WAVES[wave_index]
    roles = wave["roles"]

    # Find the first incomplete role
    for role_slug in roles:
        if not _role_outputs_complete(task_dir, role_slug):
            # Build manifest for this ONE role
            manifest = _build_manifest(runtime_root, job_ctx, role_slug, wave)
            manifest_path = task_dir / f"manifest_{role_slug}.json"
            manifest_path.write_text(json.dumps(manifest, indent=2))

            remaining_roles = [r for r in roles if r != role_slug and not _role_outputs_complete(task_dir, r)]
            has_more = len(remaining_roles) > 0

            return {
                "ok": True,
                "needs_dispatch": True,
                "has_more": has_more,           # ← KEY: drives sequential loop
                "mode": "dispatch_prepare",
                "dispatch_info": {
                    "manifests": [str(manifest_path)],   # ONE manifest
                    "remaining_manifests": [],
                    "roles": [role_slug],
                    "task_dir": str(task_dir),
                    "wave": wave_index + 1,
                },
                "instruction": f"MANDATORY: Read manifest at {manifest_path}. Use Agent tool with "
                               f"system_prompt from manifest. Do NOT summarize. "
                               f"{'Do NOT spawn multiple sub-agents.' if has_more else ''}",
            }

    # All roles complete
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
    completed = []
    incomplete = []

    for role_slug in wave["roles"]:
        if _role_outputs_complete(task_dir, role_slug):
            completed.append(role_slug)
        else:
            incomplete.append(role_slug)

    if incomplete:
        # Layer 4: Re-dispatch incomplete roles
        # (same prepare pattern, but with retry counter)
        return {
            "ok": True,
            "needs_dispatch": True,
            "has_more": False,
            "mode": "dispatch_collect",
            "dispatch_info": {"incomplete_roles": incomplete},
            "instruction": f"Re-dispatch roles: {incomplete}",
        }

    return {
        "ok": True,
        "mode": "dispatch_collect",
        "result": {"completed": len(completed), "wave": wave_index + 1},
    }


def _role_outputs_complete(task_dir: Path, role_slug: str) -> bool:
    """Check if a role has produced all 3 required files."""
    outputs_dir = task_dir / "outputs"
    md_path = outputs_dir / f"phase2_{role_slug}.md"
    facts_path = outputs_dir / f"phase2_{role_slug}-facts.json"
    section_path = outputs_dir / f"phase2_{role_slug}-section.json"

    if not md_path.exists() or md_path.stat().st_size < 100:
        return False
    if not facts_path.exists() or not _json_valid(facts_path):
        return False
    if not section_path.exists() or not _json_valid(section_path):
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


# ── Evidence Gate (with repair) ────────────────────────────────

_MAX_BLOCKING_RETRIES = 1

def _run_evidence_gate(runtime_root: Path, job_ctx: JobContext, wave: int) -> dict:
    """Gate: check if sub-agent claims are supported by facts.

    On FAIL: generate repair manifests → dispatch repair sub-agents → re-run gate.
    After max retries: degrade blocking claims to WARN and continue.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    gate_result = _evaluate_evidence_gate(task_dir, wave)

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
                "mode": "evidence_gate_repair",
                "dispatch_info": {
                    "manifests": [first],
                    "remaining_manifests": remaining,
                    "wave": wave,
                    "is_repair": True,
                },
                "instruction": f"Dispatch repair sub-agent. Sequential only.",
            }

    return {"ok": gate_result.get("ok", True), "mode": "evidence_gate", "result": gate_result}


# ── Instruction Store (hot-loaded system prompts) ──────────────

def _build_manifest(runtime_root: Path, job_ctx: JobContext, role_slug: str, wave: dict) -> dict:
    """Build sub-agent manifest with system_prompt from instruction store."""

    # HOT LOAD: system prompt from file, not hardcoded
    instruction_store = runtime_root / "instruction_store_{profile}"
    prompt_path = instruction_store / f"{role_slug}.md"
    if prompt_path.exists():
        system_prompt = prompt_path.read_text(encoding="utf-8")
    else:
        system_prompt = f"ERROR: prompt not found at {prompt_path}"

    # Template variable substitution
    system_prompt = system_prompt.replace("{TASK_ID}", job_ctx.job_id)
    system_prompt = system_prompt.replace("{ENTITY}", job_ctx.entity)

    # Generate brief (structured input document)
    brief_path = _build_brief(runtime_root, job_ctx, role_slug, wave)

    return {
        "manifest_version": "1.0",
        "task_id": job_ctx.job_id,
        "role": role_slug,
        "slug": role_slug,
        "label": f"{job_ctx.job_id}-{role_slug}",
        "system_prompt": system_prompt,
        "brief_path": str(brief_path),
        "brief_content_preview": brief_path.read_text()[:2000],
        "output_path": str(_outputs_dir(runtime_root, job_ctx) / f"phase2_{role_slug}.md"),
        "timeout": 900,
        "thinking": "high",
        "dispatch_mode": "team_async",
        "mode": "bypassPermissions",
        "subagent_type": "general-purpose",
        "team_name_template": f"{runtime_root.name}-{{task_id}}",
        "connectorIds": [],    # Fill with your MCP connector IDs
        "created_at": time.strftime("%Y-%m-%dT%H:%M:%S"),
        "status": "pending",
    }


def _build_brief(runtime_root: Path, job_ctx: JobContext, role_slug: str, wave: dict) -> Path:
    """Generate a structured brief for the sub-agent."""
    task_dir = _task_dir(runtime_root, job_ctx)
    brief_path = task_dir / f"brief_{role_slug}.md"
    lines = [
        f"# Brief: {role_slug}",
        f"",
        f"**Task ID:** {job_ctx.job_id}",
        f"**Entity:** {job_ctx.entity}",
        f"",
        f"## Input Files",
        f"- Research Plan: `{task_dir / 'research_plan.json'}`",
        f"- Fact Store: `{task_dir / 'fact_store.json'}`",
        f"- Shared State: `{task_dir / 'shared_state.json'}`",
    ]
    # Include prior wave outputs for dependency chains
    if wave["depends_on"]:
        lines.append(f"", f"## Prior Wave Outputs")
        for dep_wave_name in wave["depends_on"]:
            dep_wave = next(w for w in WAVES if w["name"] == dep_wave_name)
            for dep_role in dep_wave["roles"]:
                dep_path = _outputs_dir(runtime_root, job_ctx) / f"phase2_{dep_role}.md"
                if dep_path.exists():
                    lines.append(f"- {dep_role}: `{dep_path}`")
    lines.extend([
        f"",
        f"## Output File",
        f"Write to: `{_outputs_dir(runtime_root, job_ctx) / f'phase2_{role_slug}.md'}`",
        f"",
        f"## Output Requirements",
        f"Your output MUST include 3 files:",
        f"1. `phase2_{role_slug}.md` — Main analysis with Section Package",
        f"2. `phase2_{role_slug}-facts.json` — Discovered facts",
        f"3. `phase2_{role_slug}-section.json` — Structured section package",
    ])
    brief_path.write_text("\n".join(lines) + "\n")
    return brief_path


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
                # Phase 03: Research plan (split: skeleton → LLM enrich → collect)
                "phase03_research_plan":      lambda ctx: _run_research_plan(runtime_root, ctx),
                "phase03_research_plan_collect": lambda ctx: _run_research_plan_collect(runtime_root, ctx),
                # Phase 04: Presearch
                "phase04_presearch":          lambda ctx: _run_presearch(runtime_root, ctx),
                # Phase 05: Shared state init
                "phase05_shared_state_init":  lambda ctx: _run_shared_state_init(runtime_root, ctx),
                # Phase 06-07: Fact store
                "phase06_fact_store":         lambda ctx: _run_fact_store(runtime_root, ctx),
                # ── Wave 1: Foundation (4 roles, sequential) ──
                "phase08_dispatch_prepare":   lambda ctx: _run_dispatch_prepare(runtime_root, ctx, wave_index=0),
                "phase09_dispatch_collect":   lambda ctx: _run_dispatch_collect(runtime_root, ctx, wave_index=0),
                "phase10_wave1_evidence_gate": lambda ctx: _run_evidence_gate(runtime_root, ctx, wave=1),
                "phase11_fact_store_merge":   lambda ctx: _run_fact_store_merge(runtime_root, ctx),
                "phase12_shared_state_refresh": lambda ctx: _run_shared_state_refresh(runtime_root, ctx, after_wave=1),
                # ── Wave 2: Deep analysis ──
                "phase13_wave2_prepare":      lambda ctx: _run_dispatch_prepare(runtime_root, ctx, wave_index=1),
                "phase14_wave2_collect":      lambda ctx: _run_dispatch_collect(runtime_root, ctx, wave_index=1),
                "phase15_wave2_evidence_gate": lambda ctx: _run_evidence_gate(runtime_root, ctx, wave=2),
                "phase16_shared_state_refresh": lambda ctx: _run_shared_state_refresh(runtime_root, ctx, after_wave=2),
                # ── Wave 3: Synthesis ──
                "phase17_wave3_prepare":      lambda ctx: _run_dispatch_prepare(runtime_root, ctx, wave_index=2),
                "phase18_wave3_collect":      lambda ctx: _run_dispatch_collect(runtime_root, ctx, wave_index=2),
                # ── Quality chain ──
                "phase19_claim_coverage":     lambda ctx: _run_claim_coverage(runtime_root, ctx),
                "phase20_section_package":    lambda ctx: _run_section_package_validation(runtime_root, ctx),
                "phase21_synthesis_prepare":  lambda ctx: _run_synthesis_prepare(runtime_root, ctx),
                "phase22_synthesis_collect":  lambda ctx: _run_synthesis_collect(runtime_root, ctx),
                # ── Delivery ──
                "phase23_debate_review":      lambda ctx: _run_debate_review(runtime_root, ctx),
                "phase24_final_assembly":     lambda ctx: _run_final_assembly(runtime_root, ctx),
                "phase25_delivery":           lambda ctx: _run_delivery(runtime_root, ctx),
            },
        )
        self.runtime_root = runtime_root

    def phase_prerequisites(self) -> dict[str, list[str]]:
        return {
            "phase06_fact_store": ["research_plan.json"],
            "phase08_dispatch_prepare": ["research_plan.json", "fact_store.json"],
            "phase11_fact_store_merge": ["research_plan.json"],
            "phase19_claim_coverage": ["research_plan.json"],
            "phase23_debate_review": ["research_plan.json"],
        }

    def phase_outputs(self) -> dict[str, list[str]]:
        return {
            "phase03_research_plan": ["research_plan.json"],
            "phase04_presearch": ["presearch_results.json"],
            "phase06_fact_store": ["fact_store.json"],
            "phase05_shared_state_init": ["shared_state.json"],
        }
```

## Phase Naming Convention

Use descriptive names with wave numbers for clarity:

| Pattern | Example | Purpose |
|---------|---------|---------|
| `phaseNN_*` | `phase01_intake`, `phase03_research_plan` | Pre-dispatch phases |
| `phaseNN_waveN_*` | `phase10_wave1_evidence_gate` | Wave-specific gates |
| `phaseNN_shared_state_*` | `phase12_shared_state_refresh` | Cross-wave state updates |
| `phaseNN_dispatch_*` | `phase08_dispatch_prepare`, `phase09_dispatch_collect` | Prepare/collect pairs |
| `phaseNN_*_repair` | `phase24_claim_coverage` (with repair mode) | Gate phases with repair |

## Key Design Decisions

When creating a new profile, decide:

1. **How many waves?** → Independent research dimensions = more waves
2. **Roles per wave?** → Each role = one sub-agent responsibility
3. **Shared state refresh?** → After which waves? (recommended: after every wave with >2 roles)
4. **Evidence gate per wave?** → Yes for every wave that produces claims
5. **Repair strategy?** → Max retries + degradation policy per gate type
6. **Footnote/citation density?** → Configure threshold based on domain needs
7. **Stage tier degradation?** → Early-stage projects get lighter gates
8. **Instruction store location?** → `instruction_store_{profile}/` directory with one `.md` per role
