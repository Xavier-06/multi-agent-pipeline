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

@dataclass
class JobContext:
    job_id: str
    entity: str = ""
    query: str = ""
    market: str = ""
    metadata: dict[str, Any] = field(default_factory=dict)
    workspace: Optional[Any] = None  # injected by kernel
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
        phases = profile.phases()

        # Support breakpoint resume
        if start_phase:
            idx = phases.index(start_phase)
            phases = phases[idx:]

        results = {"job_id": job_ctx.job_id, "profile": profile.name, "phases": []}

        for i, phase_name in enumerate(phases):
            phase_result = profile.run_phase(phase_name, job_ctx)
            phase_ok = phase_result.get("ok", True)
            results["phases"].append({"phase": phase_name, "result": phase_result})
            self._write_phase_state(workspace, phase_name, phase_result)

            # PAUSE: sub-agents needed
            if phase_result.get("needs_dispatch"):
                return {
                    **results,
                    "status": "needs_dispatch",
                    "paused_after": phase_name,
                    "next_phase": phases[i+1] if i+1 < len(phases) else None,
                    "dispatch_info": phase_result.get("result", {}),
                    "ok": True,
                }

            # PAUSE: background process
            if phase_result.get("needs_poll"):
                return {
                    **results,
                    "status": "needs_poll",
                    "paused_after": phase_name,
                    "next_phase": phases[i+1] if i+1 < len(phases) else None,
                    "poll_info": phase_result,
                    "ok": True,
                }

            # FAIL: stop pipeline
            if phase_ok is False:
                return {**results, "ok": False, "failed_phase": phase_name}

        return {**results, "ok": True}
```

## Workspace Layout

```
runtime_root/
├── jobs/
│   └── {JOB_ID}/
│       ├── job_record.json          # Job lifecycle state
│       ├── state/
│       │   ├── phase0_preflight.json
│       │   ├── phase03_research_plan.json
│       │   └── ...                  # One file per completed phase
│       ├── outputs/
│       │   ├── step_sub01.md        # Sub-agent outputs
│       │   └── ...
│       └── delivery/
│           └── final_report.docx
├── scripts/                         # Pipeline-specific scripts
├── data/tasks/                      # Intermediate artifacts
└── jobs/{JOB_ID}/job_record.json    # Persisted JobRecord
```

## CLI Entry Pattern

```python
class PipelineOrchestrator:
    def submit(self, entity, market, query, input_file="") -> JobRecord:
        """Register a new job, return record with job_id."""

    def execute(self, job_id, start_phase=None) -> dict:
        """Run pipeline. Auto-resumes from last pause point if start_phase is None."""

    def recover(self) -> list[dict]:
        """Find all pending/running/paused jobs and resume them."""

    def status(self, job_id) -> dict:
        """Return job record + phase states + artifacts."""
```

## Phase Handler Contract

Each phase handler receives `JobContext` and returns a dict:

```python
def _run_my_phase(runtime_root: Path, job_ctx: JobContext) -> dict[str, Any]:
    task_dir = runtime_root / "jobs" / job_ctx.job_id

    # ... do work ...

    # Success (continue to next phase)
    return {"ok": True, "mode": "my_phase", "result": {...}}

    # Needs dispatch (pause, Coordinator spawns sub-agents)
    return {
        "ok": True,
        "needs_dispatch": True,
        "mode": "my_dispatch",
        "dispatch_info": {
            "manifests": [str(manifest_path)],
            "roles": ["step_A", "step_B"],
            "task_dir": str(task_dir),
        },
        "result": {"dispatched": 2},
        "instruction": "MANDATORY: Use Agent tool to spawn sub-agents. Do NOT skip.",
    }

    # Needs poll (background process)
    return {
        "ok": True,
        "needs_poll": True,
        "bg_pid": pid,
        "timeout": 900,
    }

    # Failure (stop pipeline)
    return {"ok": False, "error": "description"}
```
