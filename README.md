# Multi-Agent Pipeline Framework

Production-grade multi-agent orchestration patterns, generalized from a battle-tested 33-phase production pipeline into domain-agnostic building blocks.

## What This Is

A **WorkBuddy skill** (not a standalone library) that teaches an AI agent how to design and build multi-step, multi-agent pipelines for any domain. Install it in WorkBuddy, ask "build me a competitive analysis pipeline" or "design a multi-agent workflow for literature review," and the skill provides the architectural blueprints.

## Architecture

```
OrchestratorKernel          # Generic: runs phases, handles pause/resume, auto-backfill
  ├── ExampleProfile        # Reference implementation (e.g. due diligence, code review)
  └── YourNewProfile        # Your domain, same kernel
```

**Shared Kernel** handles the generic execution loop: phase sequencing, `needs_dispatch` pause (with `has_more` for sequential dispatch), `needs_poll` pause, dependency auto-backfill, completed-phase skip, phase state persistence.

**Profile** defines domain-specific phases: what phases exist, what each phase does, what dependencies exist between phases.

## Key Concepts

| Concept | What it solves | Reference |
|---------|---------------|-----------|
| **Three Pause Signals** | How phases communicate with the kernel | [kernel-pattern.md](references/kernel-pattern.md) |
| **Wave-Based Sequential Dispatch** | Rate-limit-safe sub-agent spawning | [dispatch-protocol.md](references/dispatch-protocol.md) |
| **Quality Production Chain** | Multi-stage quality with repair mechanisms | [quality-chain.md](references/quality-chain.md) |
| **Concurrency Safety** | File locks + atomic writes for shared state | [concurrency-safety.md](references/concurrency-safety.md) |
| **Instruction Store** | Hot-loaded system prompts (edit without code changes) | [instruction-store.md](references/instruction-store.md) |
| **Shared State** | Cross-wave context passing | [shared-state.md](references/shared-state.md) |
| **Sub-Agent Capabilities** | Connector IDs + tool mapping + prompt assembly | [subagent-capabilities.md](references/subagent-capabilities.md) |
| **Presearch** | Pre-dispatch intelligence gathering | [presearch-pattern.md](references/presearch-pattern.md) |
| **Brief Assembly** | Assignment slice + search work order + file refs | [brief-assembly.md](references/brief-assembly.md) |
| **Stage Classification** | Gate relaxation + risk overrides per tier | [stage-classification.md](references/stage-classification.md) |
| **Synthesis** | Final report assembly via sub-agent dispatch | [synthesis-pattern.md](references/synthesis-pattern.md) |
| **Delivery** | Multi-format output + artifact registration | [delivery-multi-format.md](references/delivery-multi-format.md) |
| **Gap Detection** | Pre-dispatch + post-wave gap tracking | [gap-detection.md](references/gap-detection.md) |
| **Entity Verification** | Pre-flight validation + early-stop pattern | [entity-verification.md](references/entity-verification.md) |

## Project Structure

```
multi-agent-pipeline/
├── SKILL.md                          # Skill entry point (loaded by WorkBuddy)
├── README.md                         # This file
└── references/
    ├── kernel-pattern.md             # OrchestratorKernel implementation
    ├── dispatch-protocol.md          # Coordinator dispatch loop + 4-layer defense
    ├── profile-template.md           # Complete profile template with all patterns
    ├── quality-chain.md              # Quality chain + repair + plan enrichment
    ├── concurrency-safety.md         # File lock + atomic write + JSON self-repair
    ├── instruction-store.md          # Hot-loaded prompt system
    ├── shared-state.md               # Cross-wave context passing (3-layer architecture)
    ├── subagent-capabilities.md      # Sub-agent capability chain + tool mapping
    ├── presearch-pattern.md          # Pre-dispatch intelligence gathering
    ├── brief-assembly.md             # Assignment slice + search work order + file refs
    ├── stage-classification.md       # Stage/weight tiers + gate relaxation + risk overrides
    ├── synthesis-pattern.md          # Synthesis as sub-agent dispatch + citation repair
    ├── delivery-multi-format.md      # Multi-format output + artifact registration
    ├── gap-detection.md              # Pre-dispatch + post-wave gap tracking
    └── entity-verification.md        # Pre-flight validation + early-stop pattern
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

1. Define your waves (independent analysis dimensions)
2. For each wave, define roles (one sub-agent per role)
3. Create `instruction_store_{profile}/` with one `.md` per role
4. Add `phase_prerequisites()` and `phase_outputs()` for dependency auto-backfill
5. Add quality gates after each wave, coverage check after all waves

## Design Principles

- **Quality is built IN, not checked at the end** — each stage produces verifiable artifacts
- **Gate failures trigger repair, not death** — targeted fixes with retry limits and graceful degradation
- **Sequential dispatch, not parallel** — avoids API rate limits AND file write conflicts
- **Prompts are files, not code** — edit `.md` files to change sub-agent behavior without touching runtime
- **Dependency backfill is precise** — kernel knows which phase produces which file, backfills to the right one

## Production Reference

This skill was extracted from a 33-phase, 4-wave, 8-role production pipeline. All patterns have been battle-tested and generalized.

## Version

**v3.1.0** — 15 reference files covering the complete pipeline lifecycle: kernel, dispatch, quality, shared state, instruction store, concurrency, presearch, brief assembly, stage classification, synthesis, delivery, gap detection, entity verification. See [SKILL.md](SKILL.md) for full description and triggers.

## License

MIT
