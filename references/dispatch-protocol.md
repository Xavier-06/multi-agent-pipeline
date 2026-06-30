# Coordinator Dispatch Protocol

## Overview

The Coordinator is the AI agent (WorkBuddy main process) that drives the pipeline. When a phase returns `needs_dispatch`, the Coordinator enters the dispatch loop: spawn sub-agents → poll outputs → advance waves → repeat.

## Dispatch Loop (Pseudocode)

```python
# 1. Create team
team_create(team_name=f"{profile}-{task_id}")

# 2. Main dispatch loop
MAX_RETRIES = 2
TOOL_LIMITS = """⚠️ Tool restrictions: No Glob/Grep. Use Bash (find/ls) for file search,
Read for file reading, Bash (grep) for content search.
NeoData queries: cd {runtime_root} && python3 -c "from scripts.search_gateway import neodata_search; ..."
"""

while True:
    result = launch_next_wave(task_id, entity, query, market, sequential=True)

    if result['all_done']:
        break

    if not result['task_tool_instructions']:
        # All steps in current wave blocked, wait and retry
        sleep(60)
        continue

    for instruction in result['task_tool_instructions']:
        step = instruction['step']
        output_path = instruction['output_path']

        # Spawn sub-agent (async, via team)
        Agent(
            name=step,                              # unique name within team
            team_name=f"{profile}-{task_id}",
            mode='bypassPermissions',                # sub-agent needs file write access
            description=step,
            prompt=TOOL_LIMITS + "\n" + instruction['prompt'],
            # Do NOT set run_in_background explicitly — use default
        )

    # 3. Poll for output file
    # Single step polling:
    elapsed = 0
    while elapsed < 1200:  # 20 min timeout
        if test -s {output_path}:
            break
        sleep(30)
        elapsed += 30

    if elapsed >= 1200:
        # Timeout → retry (max MAX_RETRIES)
        retry_count += 1
        if retry_count <= MAX_RETRIES:
            # Re-spawn same step
            Agent(name=step, team_name=..., ...)
        else:
            # Record failure, skip step, continue
            log(f"Step {step} failed after {MAX_RETRIES} retries")

    # has_more=True → next iteration dispatches next step in same wave
    # has_more=False → wave complete, next call returns next wave's first step
    # all_done=True → all waves complete

# 4. Cleanup
team_delete()

# 5. Continue pipeline (quality chain phases)
execute --start-phase {next_phase}
```

## `launch_next_wave` API

```python
def launch_next_wave(task_id, entity, query, market, sequential=True) -> dict:
    """
    Returns:
    {
        "all_done": False,
        "wave_index": 1,           # current wave (0-indexed)
        "has_more": True,          # more steps in this wave?
        "task_tool_instructions": [
            {
                "step": "step_sub01",
                "output_path": "/path/to/outputs/step_sub01.md",
                "prompt": "Full prompt including brief_path, output_path, instructions...",
            }
        ]
    }
    """
```

**Key behaviors:**
- `sequential=True`: returns at most 1 step per call (avoids API 429)
- `has_more=True`: same wave has more steps → call again to get next step
- `has_more=False`: wave exhausted → next call returns first step of next wave
- `all_done=True`: all waves complete

## Sub-Agent Prompt Structure

Each sub-agent prompt (built by the launcher script) includes:

```
{TOOL_LIMITS}

## Your Task
{step-specific instructions}

## Input Files
- Brief: {brief_path}
- Research Plan: {research_plan_path}
- Fact Store: {fact_store_path}
- Prior step outputs: {prior_output_paths}

## Output Requirements
Write your output to: {output_path}

Your output file MUST contain a Section Package JSON block:
{section_package_schema}

## Rules
- You have full read/write access to workspace
- Search for missing data yourself (NeoData/WebSearch/etc)
- Do NOT return to Coordinator asking for more data
- When output file is written, you are DONE
```

## Quality Check on Sub-Agent Output

```python
def _quality_check(output_path: Path) -> dict:
    text = output_path.read_text(encoding="utf-8")
    score = 0

    # Length scoring
    if len(text) >= 6000: score = 5
    elif len(text) >= 3000: score = 3
    elif len(text) >= 1000: score = 2

    # Source diversity check
    unique_domains = extract_unique_domains(text)
    if unique_domains < 3: score -= 1

    # Unverified claims penalty
    unverified_count = text.count("未经搜索验证")
    if unverified_count > 2: score -= 1

    # Section structure
    sections = text.count("## ")
    if sections < 3: score -= 1

    return {"score": score, "verdict": "pass" if score >= 3 else "fail"}
```

## Team Lifecycle Rules

1. **Create team before dispatch**: `team_create(team_name=...)`
2. **All sub-agents use same team**: `Agent(team_name=..., mode='bypassPermissions')`
3. **Sequential dispatch**: one step at a time, wait for completion before next
4. **Poll output files actively**: `test -s {path}` every 30s, don't rely on messages
5. **Retry on timeout**: max 2 retries per step
6. **Skip on persistent failure**: record, continue, flag in quality chain
7. **Delete team after all waves**: `team_delete()`
8. **Shutdown cleanup**: remove exited members from team config before continuing

## Common Dispatch Patterns

### Pattern A: Single Wave (all steps independent)
```
Wave 1: step_A, step_B, step_C, step_D  (sequential dispatch)
→ Final Assembly
```

### Pattern B: Multi-Wave (dependency chains)
```
Wave 1: step_data                  (foundation, no dependencies)
Wave 2: step_A, step_B, step_C    (depend on step_data)
Wave 3: step_synthesis             (depends on Wave 2)
Wave 4: step_review, step_risk     (depend on Wave 2 + 3)
Wave 5: step_master                 (depends on all)
→ Quality Chain → Delivery
```

### Pattern C: Prepare + Collect (split dispatch)
```
phase_dispatch_prepare  → returns needs_dispatch
[Coordinator spawns sub-agents]
phase_dispatch_collect  → checks outputs, validates
```
This pattern splits "launch sub-agents" and "verify outputs" into two phases, allowing the kernel to track state precisely.
