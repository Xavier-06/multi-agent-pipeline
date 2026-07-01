---
name: "multi-agent-pipeline"
version: "3.0.0"
description: >
  Production-grade multi-agent pipeline orchestration framework.
  Patterns extracted from a battle-tested 33-phase production pipeline,
  generalized for any domain.
  Use when building any multi-step, multi-agent workflow that needs:
  (1) Phase-based execution with breakpoint resume and dependency auto-backfill,
  (2) Wave-based sequential sub-agent dispatch with file-lock concurrency safety,
  (3) Quality production chain with repair mechanisms and configurable thresholds,
  (4) Instruction store for hot-loaded system prompts (edit prompts without code changes),
  (5) Shared state hub for cross-wave context passing,
  (6) Structured output protocol (3-file contract: md + data sidecar + meta sidecar),
  (7) 4-layer output collection defense with JSON self-repair,
  (8) Sub-agent capability mapping (connector IDs + tool guide + prompt assembly).
  Triggers: "build a pipeline", "multi-agent workflow", "orchestration", "dispatch sub-agents",
  "wave dispatch", "quality production chain", "needs_dispatch pattern",
  "quality gate", "repair mechanism", "instruction store", "shared state",
  "sub-agent tools", "connector IDs", "tool mapping".
  NOT for: simple single-agent tasks, one-shot generation, trivial Q&A.
---

# Multi-Agent Pipeline Framework v3

Production-grade orchestration patterns, generalized from a battle-tested 33-phase production pipeline into domain-agnostic building blocks.

## Architecture: Shared Kernel + Profile

```
OrchestratorKernel          # Generic: runs phases, handles pause/resume, auto-backfill
  ├── ExampleProfile        # Reference implementation (e.g. due diligence, code review)
  └── YourNewProfile        # Your domain, same kernel
```

**Kernel responsibilities**: prepare workspace → run phases in order → handle `needs_dispatch` pause (with `has_more` for sequential dispatch) → handle `needs_poll` pause → dependency auto-backfill → skip completed phases → write phase state → fail fast on `ok:false`.

**Profile responsibilities**: define phase list (dict insertion order = execution order), implement each phase handler, declare `phase_prerequisites()` and `phase_outputs()` for precise dependency tracking, return structured results.

See [references/kernel-pattern.md](references/kernel-pattern.md) for the complete OrchestratorKernel implementation with dependency auto-backfill, phase state persistence, and `has_more` sequential dispatch logic.

## The Three Pause Signals

| Signal | Meaning | Kernel action |
|--------|---------|---------------|
| `{"ok": True}` | Phase done | Run next phase |
| `{"needs_dispatch": True, "has_more": bool}` | Sub-agents needed | **has_more=True** → `next_phase = current phase` (re-run, dispatch next role). **has_more=False** → `next_phase = next phase` (advance). |
| `{"needs_poll": True, "bg_pid": "..."}` | Background process | Coordinator polls, then resumes with `start_phase` |
| `{"ok": False}` | Fatal failure | **Stop immediately — no retry at kernel level** |

**Core insight**: `needs_dispatch` is NOT an error — it's a planned pause. The `has_more` field is what makes sequential dispatch work: it tells the kernel whether to re-run the same phase (dispatch next role) or advance forward.

## Wave-Based Sequential Dispatch

```
Wave 1: role_A, role_B, role_C   (sequential: one at a time)
         → Quality Gate (repair if FAIL, then degrade)
         → Evidence Merge
         → Shared State Refresh
Wave 2: role_D, role_E            (reads shared state from Wave 1)
         → Quality Gate → Shared State Refresh
Wave 3: role_F                    (depends on Wave 1+2)
         → Quality Chain → Delivery
```

**Why sequential, not parallel**: Parallel dispatch causes API 429 rate limits AND shared file concurrent write conflicts. Sequential dispatch with `has_more` solves both.

**4-layer output collection defense**:
1. **Soft constraint**: Dispatch instruction mandates 3-file output + sequential-only
2. **Hard constraint**: File existence + JSON validity + 8-second stability check
3. **Retry buffer**: Configurable retry count × interval = total timeout
4. **Semi-auto recovery**: Incomplete roles trigger re-dispatch

See [references/dispatch-protocol.md](references/dispatch-protocol.md) for the complete Coordinator dispatch loop, sequential protocol, 4-layer defense, manifest structure, and repair dispatch patterns.

## Quality Production Chain

Quality is built IN during production, not checked at the end. Each stage has a **repair mechanism** — gate failures don't kill the pipeline, they trigger targeted fixes.

```
Plan → Evidence Store → Wave Dispatch → Quality Gate → Coverage Check → Synthesis → Cross-Section Review → Assembly → Delivery Gate
```

| Stage | Artifact | Repair mechanism | Max retries | On exhaustion |
|-------|----------|-----------------|-------------|---------------|
| Plan | `plan.json` | LLM enrichment via needs_dispatch | N/A | N/A |
| Evidence Store | `evidence_store.json` | locked_read_modify_write for concurrent writes | N/A | N/A |
| Quality Gate | `wave{N}_gate.json` | Repair sub-agent (sequential, per role) | 1 | Degrade blocking→WARN |
| Coverage Check | `coverage_gate.json` | Repair sub-agent (per owner) | 2 | PASS_WITH_DISCLOSURE |
| Synthesis | `synthesis.md` | Citation density repair sub-agent | 1 | WARN + deferred_fixes |
| Cross-Section Review | `review.json` | Targeted rewrite (flagged sections only) | 1 | WARN + deferred_fixes |
| Delivery Gate | Final artifact | Auto-redact (L1), hard-block (L2 only) | 0 | L2 blocks, others deferred |

**Severity levels**:
- **BLOCKING**: Hard stop (empty output, zero data items, no sidecars at all)
- **MEDIUM**: Log as WARN, continue (structural issues, low coverage)
- **LOW**: Informational only

**Citation density is configurable**: Default is 3 refs per 2000 chars, but adjust per domain — competitive analysis can use 2, academic review can use 4.

See [references/quality-chain.md](references/quality-chain.md) for the complete quality chain with all repair mechanisms, severity levels, status lifecycle, and configurable thresholds.

## Concurrency Safety

Multi-agent pipelines have sub-agents writing to shared files. Two-pronged protection:

1. **Sequential dispatch** — dispatch ONE role at a time, eliminates 90% of conflicts
2. **File locking** — `locked_read_modify_write()` with POSIX `fcntl.flock` for shared files (evidence_store, shared sidecars)
3. **Atomic writes** — `atomic_write()` (write-to-temp + `os.replace`) prevents partial reads

See [references/concurrency-safety.md](references/concurrency-safety.md) for the complete file lock module, the 4-layer defense diagram, and JSON self-repair.

## Instruction Store (Hot-Loaded Prompts)

Sub-agent system prompts live in `instruction_store_{profile}/` as `.md` files, loaded at runtime:

```
instruction_store_{profile}/
├── index.json          ← Role name → file mapping
├── role_A.md           ← Complete system prompt per role
├── role_B.md
├── synthesis.md
└── _common_tool_guide.md   ← Shared across all roles
```

**Benefits**: Edit prompts without code changes, version-controlled, A/B testable, single source of truth per role.

**Loading chain**: `.md` file → module-level cache with mtime detection → explicit error marker for missing roles.

See [references/instruction-store.md](references/instruction-store.md) for directory structure, loading pattern, template variables, and prompt file structure.

## Sub-Agent Capabilities & Tool Mapping

A sub-agent's capability is determined by **three things** working together:

1. **System prompt** — what it knows (instruction store + conclusion appendix + tool guide + domain appendix)
2. **Connector IDs** — what external tools it can call (MCP servers)
3. **Built-in tools** — what platform tools it always has (Read, Bash, WebSearch, WebFetch, Write)

**The system prompt is a 3+ part composition** assembled at manifest build time:

```python
system_prompt = (
    instruction_store_prompt      # Role-specific instructions (hot-loaded from .md)
    + conclusion_appendix         # Mandatory conclusion format rules (all roles)
    + tool_usage_guide            # Tool priority matrix (hot-loaded from _common_tool_guide.md)
    + domain_appendix             # Domain/stage-specific guidance (optional)
)
```

**Connector IDs** are passed in the manifest and determine MCP server access. Configure a base set + optional per-role overrides:

```python
BASE_CONNECTOR_IDS = ["your-data-source", "your-search-tool"]

ROLE_CONNECTOR_OVERRIDES = {
    "financial_analyst": BASE_CONNECTOR_IDS + ["neodata-financial-search"],
    "legal_analyst": BASE_CONNECTOR_IDS + ["pkulaw"],
}
```

**Tool guide** (hot-loaded from `_common_tool_guide.md`) tells sub-agents which tool to use for each scenario. Customize for your domain's data sources.

**Sub-agents do NOT have**: Glob, Grep, TaskCreate/Update. The tool guide must tell them to use `Bash: find/grep` as workarounds.

See [references/subagent-capabilities.md](references/subagent-capabilities.md) for the complete capability chain: connector ID mapping, system prompt assembly, brief construction, built-in tool reference, quality validation per role, and design checklist for new pipelines.

## Shared State (Cross-Wave Context Passing)

After each wave, a dedicated phase rebuilds a shared state snapshot that subsequent waves read:

```
Wave 1 completes → Quality Gate → Evidence Merge → Shared State Refresh
                                                           ↓
Wave 2 starts → sub-agents read shared_state.json → knows what Wave 1 found
```

**Three-layer architecture**:
1. **Progress snapshot** — `shared_state.json` (machine-readable) + `shared_state_page.md` (human-readable dashboard)
2. **Centralized evidence store** — all sidecar data merged, deduplicated
3. **Prior wave outputs** — full output files from completed waves, listed in briefs

**Status lifecycle** (customize for your domain): `pending → done / partial / failed`

See [references/shared-state.md](references/shared-state.md) for the three-layer architecture, skeleton code, refresh placement rules, and injection into sub-agent briefs.

## Creating a New Pipeline

To build a pipeline for a new domain:

1. **Define phases**: Map your workflow to the prepare/collect split pattern
2. **Define waves**: Group roles by dependency, with shared state refresh between waves
3. **Define roles**: Each role = one sub-agent with an instruction store prompt file
4. **Define gates**: Quality gate per wave + coverage check + synthesis quality check
5. **Configure repair**: Set max retries and degradation policy per gate type
6. **Configure thresholds**: Citation density, output validation rules, review severity
7. **Declare dependencies**: `phase_prerequisites()` + `phase_outputs()` for precise auto-backfill

See [references/profile-template.md](references/profile-template.md) for a complete profile template with all production patterns: shared state init/refresh, plan enrichment, wave dispatch with sequential has_more, quality gate with repair, and instruction store integration.

## Error Recovery Patterns

| Pattern | When to use | Mechanism |
|---------|-------------|-----------|
| Breakpoint resume | Pipeline interrupted | `start_phase` + kernel auto-backfills missing dependencies |
| Phase skip | Phase already completed | Kernel reads phase state, skips `status=completed` |
| Gate repair | Gate FAIL | Repair manifest → sequential dispatch → re-run gate → degrade on exhaustion |
| Collect retry | Sub-agent slow/incomplete | Configurable poll cycles with 4-layer validation |
| Re-dispatch | Role incomplete after collect | Collect returns needs_dispatch for incomplete roles |
| Stage-aware degradation | Lightweight project | Auto-degrade blocking issues, skip unnecessary waves |
| JSON self-repair | Malformed sub-agent JSON | Auto-fix unescaped quotes, trailing commas before failing |

## Common Pitfalls

- **Never use synchronous task()**: Always use `Agent(name=..., team_name=..., mode='bypassPermissions')`. Sync task returns code=10003.
- **Always poll output files**: Don't rely on sub-agent messages. Check existence + JSON validity + file stability.
- **Sequential mode is mandatory**: Parallel dispatch triggers API 429 rate limits AND file write conflicts.
- **Prompt must include tool restrictions**: Sub-agents don't have Glob/Grep. Tell them to use Bash + Read.
- **Sub-agents must be self-closing**: If they find data gaps, they search themselves. Only return when output file is written.
- **Never hardcode system prompts**: Use instruction store `.md` files. Hardcoded prompts are unmaintainable at scale.
- **Always declare phase_prerequisites and phase_outputs**: Without them, dependency auto-backfill is blind.
- **Shared state refresh after every significant wave**: Without it, Wave 2+ sub-agents work in isolation.

## Reference Files

| File | What it covers |
|------|---------------|
| [kernel-pattern.md](references/kernel-pattern.md) | OrchestratorKernel implementation, base classes, workspace layout, phase handler contract |
| [dispatch-protocol.md](references/dispatch-protocol.md) | Coordinator dispatch loop, sequential protocol, 4-layer defense, manifest structure |
| [profile-template.md](references/profile-template.md) | Complete profile template with all production patterns |
| [quality-chain.md](references/quality-chain.md) | Quality chain stages, repair mechanisms, severity levels, configurable thresholds |
| [concurrency-safety.md](references/concurrency-safety.md) | File lock module, 4-layer defense, JSON self-repair |
| [instruction-store.md](references/instruction-store.md) | Directory structure, loading pattern, template variables |
| [shared-state.md](references/shared-state.md) | Three-layer architecture, skeleton code, refresh placement |
| [subagent-capabilities.md](references/subagent-capabilities.md) | Connector mapping, prompt assembly, brief construction, tool reference |
