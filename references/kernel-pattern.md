# OrchestratorKernel Pattern

## Base Classes

```python
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
from pathlib import Path

PhaseHandler = Callable[["JobContext"], dict[str, Any]]

@dataclass
class PipelineProfile:
    name: str
    job_type: str
    phase_handlers: dict[str, PhaseHandler] = field(default_factory=dict)

    def phases(self) -> list[str]:
        return list(self.phase_handlers.keys())

    def run_phase(self, phase_name: str, job_ctx: "JobContext") -> dict[str, Any]:
        handler = self.phase_handlers.get(phase_name)
        if handler is None:
            raise KeyError(f"Phase '{phase_name}' not registered for '{self.name}'")
        return handler(job_ctx)

    def phase_prerequisites(self) -> dict[str, list[str]]:
        """Declare {phase_name: [artifact_files_required_before_running]}.
        Used by kernel for dependency auto-backfill."""
        return {}

    def phase_outputs(self) -> dict[str, list[str]]:
        """Declare {phase_name: [artifact_files_produced]}.
        Kernel builds a reverse map (file → producer_phase) for precise backfill."""
        return {}

@dataclass
class JobContext:
    job_id: str
    entity: str = ""
    query: str = ""
    market: str = ""
    metadata: dict[str, Any] = field(default_factory=dict)
    workspace: Optional[Any] = None  # injected by kernel before phase execution
```

## Kernel Implementation

```python
@dataclass
class OrchestratorKernel:
    runtime_root: Path

    def prepare_job(self, job_ctx: JobContext) -> "JobWorkspace":
        workspace = build_job_workspace(self.runtime_root, job_ctx.job_id)
        job_ctx.workspace = workspace
        return workspace

    def run(self, profile, job_ctx: JobContext, start_phase: str | None = None) -> dict:
        workspace = self.prepare_job(job_ctx)

        all_phases = profile.phases()
        prerequisites = profile.phase_prerequisites()
        phase_outputs = profile.phase_outputs()

        # Build reverse map: file → which phase produces it
        file_to_producer: dict[str, str] = {}
        for pname, outputs in phase_outputs.items():
            for rel_path in outputs:
                file_to_producer[rel_path] = pname

        phases = all_phases
        if start_phase:
            idx = all_phases.index(start_phase)

            # ── DEPENDENCY AUTO-BACKFILL ──────────────────────────
            # Check if start_phase (or later phases) need artifacts that are missing.
            # If so, find WHICH phase produces them and backfill from that phase.
            # This is precise — not blind "go back one step".
            earliest_backfill = idx
            for check_idx in range(idx, len(all_phases)):
                phase_to_check = all_phases[check_idx]
                required_files = prerequisites.get(phase_to_check, [])
                for rel_path in required_files:
                    if not (workspace.root / rel_path).exists():
                        producer = file_to_producer.get(rel_path)
                        if producer is None:
                            continue  # nobody claims to produce it
                        state = self._read_phase_state(workspace, producer)
                        if state and state.get("status") == "completed":
                            continue  # completed but file gone — skip
                        producer_idx = all_phases.index(producer)
                        if 0 <= producer_idx < earliest_backfill:
                            earliest_backfill = producer_idx
                        break

            phases = all_phases[earliest_backfill:] if earliest_backfill < idx else all_phases[idx:]

        results = {"job_id": job_ctx.job_id, "profile": profile.name, "phases": []}

        for i, phase_name in enumerate(phases):
            # ── SKIP COMPLETED PHASES ──────────────────────────────
            # Phase state is persisted to disk. On resume, completed phases are skipped.
            phase_state = self._read_phase_state(workspace, phase_name)
            if phase_state and phase_state.get("status") == "completed":
                results["phases"].append({"phase": phase_name, "result": {"skipped": True}})
                continue

            # ── RUN PHASE ──────────────────────────────────────────
            phase_result = profile.run_phase(phase_name, job_ctx)
            phase_ok = phase_result.get("ok", True)
            results["phases"].append({"phase": phase_name, "result": phase_result})

            # Persist phase state (with attempt counter for audit trail)
            self._write_phase_state(workspace, phase_name, phase_result)

            # ── PAUSE: needs_dispatch ──────────────────────────────
            if phase_result.get("needs_dispatch"):
                dispatch_info = phase_result.get("dispatch_info") or phase_result.get("result", {})

                # CRITICAL: has_more handling
                # has_more=True  → next_phase = CURRENT phase (re-run to dispatch next role)
                # has_more=False → next_phase = NEXT phase (advance forward)
                has_more = dispatch_info.get("has_more")
                if has_more:
                    next_phase = phase_name        # same phase, dispatch next item
                else:
                    next_phase = phases[i + 1] if i + 1 < len(phases) else None

                return {
                    **results,
                    "ok": True,                    # NOT a failure — it's a planned pause
                    "status": "needs_dispatch",
                    "paused_after": phase_name,
                    "next_phase": next_phase,
                    "dispatch_info": dispatch_info,
                }

            # ── PAUSE: needs_poll (background process) ─────────────
            if phase_result.get("needs_poll"):
                return {
                    **results,
                    "ok": True,
                    "status": "needs_poll",
                    "paused_after": phase_name,
                    "next_phase": phases[i + 1] if i + 1 < len(phases) else None,
                    "poll_info": phase_result,
                }

            # ── FAIL: stop pipeline immediately ────────────────────
            if phase_ok is False:
                return {**results, "ok": False, "failed_phase": phase_name}

        return {**results, "ok": True}

    def _read_phase_state(self, workspace, phase_name: str) -> dict | None:
        state_file = workspace.state_dir / f"{phase_name}.json"
        if not state_file.exists():
            return None
        try:
            return json.loads(state_file.read_text())
        except Exception:
            return None

    def _write_phase_state(self, workspace, phase_name: str, phase_result: dict):
        """Persist phase state with attempt counter and timing metadata."""
        state_file = workspace.state_dir / f"{phase_name}.json"
        existing_attempt = 0
        if state_file.exists():
            try:
                existing = json.loads(state_file.read_text())
                existing_attempt = int(existing.get("attempt", 0))
            except Exception:
                existing_attempt = 1

        # Determine status
        if phase_result.get("needs_dispatch"):
            status = "needs_dispatch"
        elif phase_result.get("needs_poll"):
            status = "needs_poll"
        elif phase_result.get("ok") is False:
            status = "failed"
        else:
            status = "completed"

        payload = {
            "phase": phase_name,
            "status": status,
            "attempt": existing_attempt + 1,
            "started_at": ...,     # time.time()
            "finished_at": ...,    # time.time()
            "elapsed_seconds": ...,
            "result": phase_result,
        }
        state_file.write_text(json.dumps(payload, indent=2))
```

## Key Kernel Behaviors Explained

### 1. has_more — The Sequential Dispatch Driver

This is the mechanism that makes sequential dispatch work:

```
phase_prepare returns:
  needs_dispatch=True, has_more=True, dispatch_info.manifests=[role_A only]

Coordinator spawns role_A sub-agent, waits for output.

Coordinator calls execute(start_phase="phase_prepare")   ← SAME phase!
  → Kernel re-runs phase_prepare
  → phase_prepare finds role_A done, dispatches role_B, has_more=True

... repeat for role_B, role_C ...

phase_prepare returns:
  needs_dispatch=True, has_more=False, dispatch_info.manifests=[role_Z]
  → Kernel next_phase = phase_collect (advances forward)
```

**Why this matters**: Without `has_more`, you'd need to return ALL manifests at once, which leads to the Coordinator dispatching all sub-agents in parallel → API 429 rate limits and file write conflicts.

### 2. Dependency Auto-Backfill

When resuming from `start_phase`, the kernel doesn't just start from that phase. It checks ALL upcoming phases for missing prerequisites:

```
start_phase = phase08_dispatch_prepare
  → phase08 needs bp_research_plan.json
  → bp_research_plan.json doesn't exist
  → file_to_producer says phase03_research_plan produces it
  → kernel backfills from phase03, not phase08
```

This prevents the "resume from wrong phase" problem where you skip a prerequisite.

### 3. Phase State Persistence

Every phase writes a state JSON on completion. This enables:
- **Breakpoint resume**: After interruption, kernel skips completed phases
- **Audit trail**: Each attempt is numbered (`attempt: 3` = third time this phase ran)
- **Precise backfill**: Kernel checks if producer phase was already completed before backfilling

### 4. Completed Phase Skip

On resume, phases with `status=completed` are skipped entirely. This means:
- You can call `execute(start_phase="phase24")` even if phases 1-23 are done
- The kernel automatically skips 1-23 and starts at 24
- But if phase 24 depends on a file from phase 7 that's missing, it backfills

## Workspace Layout

```
runtime_root/
├── jobs/
│   └── {JOB_ID}/
│       ├── job_record.json          # Job lifecycle state
│       ├── state/                   # Phase state persistence
│       │   ├── phase01.json         # {status, attempt, elapsed_seconds, result}
│       │   ├── phase02.json
│       │   └── ...
│       ├── outputs/                 # Sub-agent output files
│       │   ├── bp_phase2_market.md
│       │   ├── bp_phase2_market-facts.json      # sidecar: discovered facts
│       │   ├── bp_phase2_market-section.json    # sidecar: section package
│       │   └── ...
│       ├── fact_store.json          # Central fact repository
│       ├── research_plan.json       # Research plan (questions + claims)
│       └── delivery/
│           └── final_report.docx
├── scripts/                         # Pipeline-specific scripts
├── instruction_store_{profile}/     # Sub-agent system prompts (hot-loaded)
│   ├── index.json
│   ├── role_A.md
│   └── role_B.md
└── references/                      # Knowledge base docs
```

## Phase Handler Contract

Each phase handler receives `JobContext` and returns a dict:

```python
def _run_my_phase(runtime_root: Path, job_ctx: JobContext) -> dict[str, Any]:
    task_dir = runtime_root / "jobs" / job_ctx.job_id

    # Success (continue to next phase)
    return {"ok": True, "mode": "my_phase", "result": {...}}

    # Needs dispatch — pause, Coordinator spawns sub-agents
    return {
        "ok": True,
        "needs_dispatch": True,
        "has_more": True,           # ← more items to dispatch in same phase
        "mode": "my_dispatch",
        "dispatch_info": {
            "manifests": [str(manifest_path)],   # ONE manifest per call (sequential)
            "remaining_manifests": [...],
            "roles": ["role_A"],
            "task_dir": str(task_dir),
        },
        "instruction": "MANDATORY: Use Agent tool to spawn sub-agents.",
    }

    # Needs poll (background process)
    return {
        "ok": True,
        "needs_poll": True,
        "bg_pid": pid,
        "timeout": 900,
    }

    # Failure (stop pipeline — no retry at kernel level)
    return {"ok": False, "error": "description"}
```

## CLI Entry Pattern

```python
class PipelineOrchestrator:
    def submit(self, entity, market, query, input_file="") -> JobRecord:
        """Register a new job, return record with job_id."""

    def execute(self, job_id, start_phase=None) -> dict:
        """Run pipeline. If start_phase given, auto-backfills missing dependencies."""

    def recover(self) -> list[dict]:
        """Find all pending/paused jobs and resume from their last pause point."""

    def status(self, job_id) -> dict:
        """Return job record + phase states + artifacts."""
```
