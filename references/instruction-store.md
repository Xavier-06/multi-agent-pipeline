# Instruction Store (Hot-Loaded System Prompts)

> **Scope**: This file covers the **storage and loading** of sub-agent instructions — directory structure, index.json schema, caching mechanism, template variables.
> For how these instructions are **assembled into the final system prompt** and combined with connector IDs / built-in tools, see [subagent-capabilities.md](subagent-capabilities.md).

## Problem

Hardcoding sub-agent system prompts in Python means:
- Changing a prompt requires code deployment
- No way to A/B test prompts without touching runtime code
- Prompts are scattered across multiple handler functions
- No single place to review all sub-agent instructions

## Solution: File-Based Instruction Store

System prompts live in `instruction_store_{profile}/` as plain `.md` files. The pipeline loads them at runtime and injects them into sub-agent manifests.

## Directory Structure

```
instruction_store_myprofile/
├── index.json                  ← Role name → file mapping
├── role_A.md                   ← One file per role
├── role_B.md
├── role_C.md
├── synthesis.md                ← Synthesis role
├── plan_enrichment.md          ← Enrichment instruction (not a sub-agent role)
└── _common_tool_guide.md       ← Shared across all roles
```

### index.json

Maps role keys to their prompt files. This is the single registry the launcher reads:

```json
{
  "meta": {
    "version": 1,
    "description": "My pipeline instruction store",
    "updated_at": "2026-07-01"
  },
  "roles": [
    {
      "key": "role_A",
      "name": "Role A Analyst",
      "file": "role_A.md",
      "description": "Wave 1: ..."
    },
    {
      "key": "role_B",
      "name": "Role B Analyst",
      "file": "role_B.md",
      "description": "Wave 1: ..."
    },
    {
      "key": "synthesis",
      "name": "Synthesis Analyst",
      "file": "synthesis.md",
      "description": "Synthesis: assembles all wave outputs into final report"
    }
  ],
  "deprecated": []
}
```

**Key schema details**:
- `roles` is an **array** (not a dict), each entry has `key`, `name`, `file`, `description`
- `meta.version` is a number (not a string)
- `deprecated` tracks removed roles with reasons
- The launcher can filter by an active-roles set to only load current roles

## Loading Pattern

The instruction store uses a **module-level cache with mtime detection**, not a simple file-read-per-dispatch:

```python
# Module-level state
_INSTRUCTION_STORE_CACHE: dict[str, str] | None = None
_INSTRUCTION_STORE_MTIME: float = 0

def _load_instruction_store_prompts(force_reload: bool = False) -> dict[str, str]:
    """Load all active role prompts from index.json + .md files.

    Cache strategy:
    - index.json mtime unchanged → return cached dict (no disk I/O)
    - mtime changed OR force_reload=True → re-read all .md files
    - index.json missing/corrupt → return last cache (or empty dict)
    """
    global _INSTRUCTION_STORE_CACHE, _INSTRUCTION_STORE_MTIME

    index_path = INSTRUCTION_STORE / 'index.json'
    current_mtime = index_path.stat().st_mtime

    # Fast path: cache hit
    if not force_reload and _INSTRUCTION_STORE_CACHE is not None:
        if current_mtime == _INSTRUCTION_STORE_MTIME:
            return _INSTRUCTION_STORE_CACHE

    # Slow path: re-read index + all .md files
    index = json.loads(index_path.read_text())
    prompts = {}
    for role in index.get('roles', []):
        role_key = role.get('key', '')
        role_file = role.get('file', '')
        if role_key not in ACTIVE_ROLES:   # Filter: only active roles
            continue
        path = INSTRUCTION_STORE / role_file
        if path.exists():
            prompts[role_key] = path.read_text(encoding='utf-8')

    _INSTRUCTION_STORE_CACHE = prompts
    _INSTRUCTION_STORE_MTIME = current_mtime
    return prompts
```

**Key points**:
1. ALL active role prompts are loaded at module import time (first call)
2. Subsequent calls check `index.json` mtime — if unchanged, return cache instantly
3. `ACTIVE_ROLES` set filters which roles from index.json actually get loaded
4. Missing role → explicit error string: `"UNKNOWN ROLE: {key}. No instruction-store prompt is registered."`
5. `_common_tool_guide.md` is loaded **separately** at module level (see System Prompt Assembly below)

### How Prompts Flow into Manifests

```python
# In dispatch function:
system_prompt = ROLE_SYSTEM_PROMPTS.get(
    sub['role_name'],
    f"UNKNOWN ROLE: {sub['role_name']}. No instruction-store prompt is registered."
)

# Then more parts are APPENDED (not part of instruction store):
system_prompt = (
    system_prompt                    # Part 1: from instruction store
    + _CONCLUSION_APPENDIX           # Part 2: hardcoded conclusion rules
    + _TOOL_USAGE_GUIDE              # Part 3: from _common_tool_guide.md
    + _domain_appendix               # Part 4: domain-specific (optional)
)

manifest["system_prompt"] = system_prompt  # FULL assembled text
```

The Coordinator's dispatch instruction tells it: "Do NOT write your own simplified prompt — the manifest system_prompt contains the complete instructions."

## Prompt File Structure

Each `.md` file in the instruction store is the COMPLETE system prompt for that role:

```markdown
# Role: {Role Name}

## Identity
You are a specialist in {domain area}.

## Your Task
Analyze {aspect} based on the plan and available data.

## Input Files
- Plan: read from brief
- Evidence Store: read from brief
- Shared State: read from brief (shows what other roles have found)

## Output Requirements
Write your analysis to the output file specified in the brief.

Your output MUST include 3 files:
1. `{role_slug}.md` — Main analysis
2. `{role_slug}-data.json` — Structured data sidecar
3. `{role_slug}-meta.json` — Metadata sidecar

## Evidence Store Protocol
Before searching externally, check the Evidence Store for existing data.
If you discover new data, append to your data sidecar file.
Use locked_read_modify_write when modifying shared files.

## Quality Standards
- Every key assertion must cite data sources
- Counter-findings are mandatory for major conclusions
- Source quality hierarchy: define tiers for your domain

## Stage-Specific Guidance
Current stage tier: {STAGE_TIER}
(Adjust depth based on stage)
```

## Common Tool Guide (Separate File, Appended at Runtime)

`_common_tool_guide.md` is **NOT embedded inside role prompts**. It's loaded separately at module level and **appended** to every role's system prompt during manifest assembly:

```python
# Module-level load
_TOOL_USAGE_GUIDE = _load_tool_usage_guide()   # reads _common_tool_guide.md once

# In dispatch function: appended to ALL roles after the role prompt
system_prompt = system_prompt + _CONCLUSION_APPENDIX + _TOOL_USAGE_GUIDE + _domain_appendix
```

This means role `.md` files should **NOT** include tool usage instructions — the tool guide handles that globally. Role prompts focus on methodology and output format.

**What the tool guide typically covers**:
1. Tool priority matrix (which tool for which data type)
2. Prohibited behaviors (don't use only WebSearch for everything)
3. File operation rules (use Read, not cat/head/tail)
4. Search operation rules (MCP tools for domain data, WebSearch for general)

See [subagent-capabilities.md](subagent-capabilities.md) § "System Prompt Assembly" for the complete multi-part assembly chain.

## Template Variables

Two approaches for getting runtime values into sub-agent context:

| Approach | How | When to use |
|----------|-----|-------------|
| **Brief injection** (recommended) | Brief contains actual file paths, entity name, role-specific data. Prompt references "read from brief". | Most cases — keeps prompts generic |
| **Python `.replace()`** | Launcher does `prompt.replace("{ENTITY}", entity)` before writing manifest | Only if prompt text itself must contain the value |

**Variables commonly available via brief** (not prompt substitution):

| Variable | Where it appears | Source |
|----------|-----------------|--------|
| Entity name | Brief header | `job_ctx.entity` |
| Task ID | Brief header | `job_ctx.job_id` |
| Output path | Brief "Output File" section | manifest `output_path` |
| Stage tier | Appended as domain appendix | Profile metadata |
| Prior wave outputs | Brief "Prior Wave" section | Wave dependency graph |

## Benefits

1. **Edit prompts without touching code** — change a `.md` file, next run picks it up
2. **Version control** — prompts are in the repo, diffable, reviewable
3. **Single source of truth** — one file per role, no duplicates
4. **A/B testing** — swap files to test different prompt strategies
5. **Separation of concerns** — prompt authors don't need to understand Python runtime

## Adding a New Role (3-Step Process)

| Step | Where | What | Why |
|------|-------|------|-----|
| 1 | `instruction_store_{profile}/{new_role}.md` | Create the prompt file | Role-specific instructions |
| 2 | `instruction_store_{profile}/index.json` | Add to `roles[]` array | Registry entry |
| 3 | Launcher Python (`ACTIVE_ROLES` set) | Add the role key string | **Runtime filter** — without this, the role is invisible |

Then add the role to your wave definition in the profile class.

### Why Step 3 Exists

The launcher uses a hardcoded `ACTIVE_ROLES` set as a safety net:

```python
ACTIVE_ROLES = {
    'role_A',
    'role_B',
    # ... N active roles
}
```

`_load_instruction_store_prompts()` only loads prompts for roles in this set. This is intentional — if `index.json` gets corrupted, the hardcoded set ensures known-good roles still load.

**Tradeoff**: Adding a role requires a code change, but the system degrades gracefully when config is broken.
