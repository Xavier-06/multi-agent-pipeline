---
name: "multi-agent-pipeline"
description: >
  Generic multi-agent pipeline orchestration framework. Extracted from a production BP/IR
  investment research pipeline (OrchestratorKernel + Profile pattern). Use when building any
  multi-step, multi-agent workflow that needs: (1) Phase-based execution with breakpoint resume,
  (2) Wave-based sequential sub-agent dispatch, (3) Quality production chain (not just end-stage QA),
  (4) Structured output protocol (Section Package), (5) Error recovery and retry.
  Triggers: "build a pipeline", "multi-agent workflow", "orchestration", "dispatch sub-agents",
  "wave dispatch", "quality production chain", "section package", "needs_dispatch pattern".
  NOT for: simple single-agent tasks, one-shot generation, trivial Q&A.
---

# Multi-Agent Pipeline Framework

Battle-tested orchestration patterns from production investment research pipelines.

## Architecture: Shared Kernel + Profile

```
OrchestratorKernel          # Generic: runs phases sequentially, handles pause/resume
  ├── ProfileA              # Domain-specific: defines phases + handlers
  ├── ProfileB              # Different domain, same kernel
  └── ProfileC              # ...
```

**Kernel responsibilities**: prepare workspace → run phases in order → handle `needs_dispatch` pause → handle `needs_poll` pause → write phase state → error/fail fast.

**Profile responsibilities**: define phase list, implement each phase handler, return structured results.

See [references/kernel-pattern.md](references/kernel-pattern.md) for the complete OrchestratorKernel contract and Profile base class.

## The Three Pause Signals

A phase handler returns one of:

| Signal | Meaning | Kernel action |
|--------|---------|---------------|
| `{"ok": True}` | Phase done, continue | Run next phase |
| `{"needs_dispatch": True, "result": {...}}` | Sub-agents needed, pause | Return to Coordinator, Coordinator dispatches agents, then resumes with `start_phase` |
| `{"needs_poll": True, "bg_pid": "...", "timeout": 900}` | Heavy background job | Coordinator polls process, then resumes with `start_phase` |
| `{"ok": False, "error": "..."}` | Fatal failure | Stop pipeline |

**This is the core insight**: `needs_dispatch` is NOT an error — it's a planned pause. The kernel returns control to the Coordinator (the AI agent), which spawns sub-agents, waits for output, then calls `execute --start-phase <next_phase>`.

## Wave-Based Sequential Dispatch

```
Wave 1: step_A                    (serial, one at a time)
Wave 2: step_B, step_C, step_D   (serial within wave, avoid API 429)
Wave 3: step_E                    (depends on Wave 2 output)
Phase N: finalize                 (assembly + delivery)
```

**Why sequential, not parallel**: WorkBuddy's Agent tool has API rate limits. `launch_next_wave(sequential=True)` returns one step at a time. The Coordinator dispatches it → waits for completion → calls again for next step.

See [references/dispatch-protocol.md](references/dispatch-protocol.md) for the complete Coordinator dispatch loop, including team management, polling, retry, and the `launch_next_wave` API pattern.

## Quality Production Chain

Quality is built IN during production, not checked at the end:

```
Research Plan  →  Fact Store  →  Section Package  →  Debate Review  →  Final Assembly  →  Delivery Gate
```

| Stage | Artifact | Purpose |
|-------|----------|---------|
| Research Plan | `research_plan.json` | Define questions, evidence needs, step owners BEFORE any work |
| Fact Store | `fact_store.json` | Centralized, traceable facts with confidence + source quality |
| Section Package | Each step's output JSON | Structured claims bound to fact_ids, counter-evidence, data gaps |
| Debate Review | `debate_review.json` | Cross-step critique: weak evidence, missing counter-evidence |
| Final Assembly | `final_assembly.json` | Only assemble validated Section Packages, never invent new facts |
| Delivery Gate | Final artifact | Adversarial verification (6-layer) + format generation |

**6 hard rules:**
1. No research plan → no dispatch
2. No unverified numbers in final output
3. Sub-agents produce Section Packages, not free prose
4. Assembly only uses validated packages
5. Debate Review failures trigger targeted rewrite, not full redo
6. Delivery gate is the last fuse, not the only quality source

See [references/quality-chain.md](references/quality-chain.md) for the complete quality chain protocol, Section Package schema, and Debate Review dimensions.

## Creating a New Pipeline Profile

To build a pipeline for a new domain (e.g., literature review, competitive analysis, product teardown):

1. **Define phases**: Map your workflow to phase names. Follow the pattern: `preflight → plan → presearch → dispatch_prepare → dispatch_collect → quality_stages → delivery`
2. **Define waves**: Group steps by dependency. Steps in the same wave can theoretically run in parallel but use sequential mode for rate limiting.
3. **Define steps**: Each step = one sub-agent responsibility. Write clear briefs.
4. **Define Section Package fields**: What structured fields does each step need to output?
5. **Implement phase handlers**: Each handler takes `JobContext`, returns a result dict.
6. **Register with kernel**: `Profile(phase_handlers={...})`

See [references/profile-template.md](references/profile-template.md) for a complete template with all common phase patterns.

## Error Recovery Patterns

| Pattern | When to use |
|---------|-------------|
| Breakpoint resume | Pipeline interrupted (context window, timeout) → `get_pipeline_status` + `start_phase` |
| Step retry | Sub-agent timeout (>20min) or bad output → re-dispatch same step, max 2 retries |
| Step skip | Non-critical step fails → record failure, continue, flag in Debate Review |
| Source fallback | Primary data source unavailable → try secondary source, record degradation |
| Assembly fallback | Final DOCX fails → deliver Markdown instead |

## Common Pitfalls

- **Never use synchronous task()**: Always use `Agent(name=..., team_name=..., mode='bypassPermissions')`. Sync task returns code=10003.
- **Always poll output files**: Don't rely on sub-agent messages. Use `test -s {output_path}` every 30s.
- **Sequential mode is mandatory**: Parallel dispatch triggers API 429 rate limits.
- **Prompt must include tool restrictions**: Sub-agents don't have Glob/Grep. Tell them to use Bash + Read.
- **Sub-agents must be self-closing**: If they find data gaps, they search themselves. Only return to Coordinator when output file is written.
