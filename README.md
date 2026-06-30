# Multi-Agent Pipeline Framework

Production-grade multi-agent orchestration patterns, extracted from a battle-tested 33-phase BP investment research pipeline and a 7-phase IC industry analysis pipeline.

## What This Is

A **WorkBuddy skill** (not a standalone library) that teaches an AI agent how to design and build multi-step, multi-agent pipelines from scratch. Install it in WorkBuddy, ask "build me a competitive analysis pipeline" or "design a multi-agent workflow for literature review," and the skill provides the architectural blueprints.

## Architecture

```
OrchestratorKernel          # Generic: runs phases, handles pause/resume, auto-backfill
  ├── BPProfile             # 33-phase BP due diligence pipeline
  ├── ICProfile             # 7-phase industry coverage pipeline
  └── YourNewProfile        # Your domain, same kernel
```

**Shared Kernel** handles the generic execution loop: phase sequencing, `needs_dispatch` pause (with `has_more` for sequential dispatch), `needs_poll` pause, dependency auto-backfill, completed-phase skip, phase state persistence.

**Profile** defines domain-specific phases: what phases exist, what each phase does, what dependencies exist between phases.

## Key Concepts

| Concept | What it solves | Reference |
|---------|---------------|-----------|
| **Three Pause Signals** | How phases communicate with the kernel | [kernel-pattern.md](references/kernel-pattern.md) |
| **Wave-Based Sequential Dispatch** | Rate-limit-safe sub-agent spawning | [dispatch-protocol.md](references/dispatch-protocol.md) |
| **Quality Production Chain** | 7-stage quality with 3 repair mechanisms | [quality-chain.md](references/quality-chain.md) |
| **Concurrency Safety** | File locks + atomic writes for shared state | [concurrency-safety.md](references/concurrency-safety.md) |
| **Instruction Store** | Hot-loaded system prompts (edit without code changes) | [instruction-store.md](references/instruction-store.md) |
| **Shared State** | Cross-wave information passing | [shared-state.md](references/shared-state.md) |
| **Sub-Agent Capabilities** | Connector IDs + tool mapping + prompt assembly | [subagent-capabilities.md](references/subagent-capabilities.md) |

## Project Structure

```
multi-agent-pipeline/
├── SKILL.md                          # Skill entry point (loaded by WorkBuddy)
├── README.md                         # This file
└── references/
    ├── kernel-pattern.md             # OrchestratorKernel implementation
    ├── dispatch-protocol.md          # Coordinator dispatch loop + 4-layer defense
    ├── profile-template.md           # Complete profile template with all patterns
    ├── quality-chain.md              # Quality chain + repair mechanisms + thresholds
    ├── concurrency-safety.md         # File lock + atomic write + JSON self-repair
    ├── instruction-store.md          # Hot-loaded prompt system
    ├── shared-state.md               # Cross-wave information hub
    └── subagent-capabilities.md      # Sub-agent capability chain + tool mapping
```

## How It Works (30-Second Overview)

1. **Profile** defines phases in execution order (dict insertion order)
2. **Kernel** runs phases sequentially, writing phase state to disk after each
3. When a phase needs sub-agents, it returns `needs_dispatch: True` with a manifest
4. **Coordinator** (the main AI) reads the manifest, spawns sub-agents via `Agent` tool
5. `has_more` controls whether to re-run the same phase (next role) or advance
6. After dispatch, **collect** validates outputs with 4-layer defense (existence → JSON validity → file stability → re-dispatch)
7. **Quality gates** check results — failures trigger repair sub-agents, not pipeline death
8. After max repair retries, gates degrade to WARN and pipeline continues

## Using This Skill

### In WorkBuddy

1. Install this repo as a WorkBuddy skill
2. Ask: "Design a multi-agent pipeline for [your domain]"
3. The skill guides you through: define phases → define waves → define roles → define gates → configure repair

### Creating a New Pipeline

See [profile-template.md](references/profile-template.md) for the complete template. The short version:

1. Define your waves (independent research dimensions)
2. For each wave, define roles (one sub-agent per role)
3. Create `instruction_store_{profile}/` with one `.md` per role
4. Add `phase_prerequisites()` and `phase_outputs()` for dependency auto-backfill
5. Add evidence gates after each wave, claim coverage after all waves

## Design Principles

- **Quality is built IN, not checked at the end** — each stage produces verifiable artifacts
- **Gate failures trigger repair, not death** — targeted fixes with retry limits and graceful degradation
- **Sequential dispatch, not parallel** — avoids API rate limits AND file write conflicts
- **Prompts are files, not code** — edit `.md` files to change sub-agent behavior without touching runtime
- **Dependency backfill is precise** — kernel knows which phase produces which file, backfills to the right one

## Production Pipeline Stats

| Pipeline | Phases | Waves | Roles | Quality Gates | Repair Types |
|----------|--------|-------|-------|---------------|--------------|
| BP Due Diligence | 33 | 4 | 8 + synthesis | 7 | 3 (wave gate, claim coverage, synthesis) |
| IC Industry Analysis | 7 | 1 | varies | 3 | 1 (wave gate) |

## Version

**v2.1.0** — See [SKILL.md](SKILL.md) for full description and triggers.

## License

MIT
