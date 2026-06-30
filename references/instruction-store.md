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
instruction_store_bp/
├── index.json                  ← Role name → file mapping
├── market_analysis.md          ← One file per role
├── tech_product.md
├── financial_model.md
├── competition_positioning.md
├── company_team_compliance.md
├── customer_revenue.md
├── valuation.md
├── deal_breaker.md
├── synthesis.md                ← Synthesis role (different naming allowed)
├── research_plan_enrichment.md ← Enrichment instruction (not a sub-agent role)
└── _common_tool_guide.md       ← Shared across all roles
```

### index.json

Maps role keys to their prompt files. This is the single registry the launcher reads:

```json
{
  "meta": {
    "version": 8,
    "description": "BP 8-role instruction store",
    "updated_at": "2026-06-13"
  },
  "roles": [
    {
      "key": "bp_company_team_compliance",
      "name": "Company, Team & Compliance Analyst",
      "file": "bp_company_team_compliance.md",
      "description": "Wave 1: corporate structure, team, governance, equity, compliance"
    },
    {
      "key": "bp_product_commercial",
      "name": "Product Commercialization Analyst",
      "file": "bp_product_commercial.md",
      "description": "Wave 1: product matrix, customers, orders, revenue signals"
    },
    {
      "key": "bp_tech_ip_moat",
      "name": "Tech, IP & Moat Analyst",
      "file": "bp_tech_ip_moat.md",
      "description": "Wave 1: tech roadmap, patents, certifications, moat validation"
    },
    {
      "key": "bp_统稿",
      "name": "Synthesis Analyst",
      "file": "bp_统稿.md",
      "description": "Synthesis: assembles Wave 1-4 outputs into final report"
    }
  ],
  "pipeline_bindings": {
    "bp": {
      "company_team_compliance": "bp_company_team_compliance",
      "product_commercial": "bp_product_commercial",
      "synthesis": "bp_统稿"
    }
  },
  "deprecated": [
    {
      "key": "bp_moat_anchor",
      "file": "bp_moat_anchor.md.deprecated",
      "reason": "Merged into tech_ip_moat dimension"
    }
  ]
}
```

**Key schema details** (from actual BP pipeline):
- `roles` is an **array** (not a dict), each entry has `key`, `name`, `file`, `description`
- `meta.version` is a number (not a string)
- `pipeline_bindings` maps short slugs to full role keys (for legacy compatibility)
- `deprecated` tracks removed roles with reasons
- The launcher filters by `CURRENT_BP_ROLES` set to only load active roles

## Loading Pattern (from BP pipeline actual code)

The instruction store uses a **module-level cache with mtime detection**, not a simple file-read-per-dispatch:

```python
# Module-level state (bp_subagent_launcher_wb.py)
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
        if role_key not in CURRENT_BP_ROLES:   # Filter: only active roles
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
3. `CURRENT_BP_ROLES` set filters which roles from index.json actually get loaded
4. Missing role → explicit error string: `"UNKNOWN BP ROLE: {key}. No instruction-store prompt is registered."`
5. `_common_tool_guide.md` is loaded **separately** at module level (see System Prompt Assembly below)

### How Prompts Flow into Manifests

```python
# In _spawn_one() — the actual dispatch function:
system_prompt = ROLE_SYSTEM_PROMPTS.get(
    sub['role_name'],
    f"UNKNOWN BP ROLE: {sub['role_name']}. No instruction-store prompt is registered."
)

# Then 3 more parts are APPENDED (not part of instruction store):
system_prompt = (
    system_prompt                    # Part 1: from instruction store
    + _CONCLUSION_APPENDIX           # Part 2: hardcoded conclusion rules
    + _TOOL_USAGE_GUIDE              # Part 3: from _common_tool_guide.md
    + _stage_block                   # Part 4: from bp_stage_utils (optional)
)

manifest["system_prompt"] = system_prompt  # FULL assembled text
```

The Coordinator's dispatch instruction tells it: "Do NOT write your own simplified prompt — the manifest system_prompt contains the complete instructions."

## Prompt File Structure

Each `.md` file in the instruction store is the COMPLETE system prompt for that role:

```markdown
# Role: Market Analysis Sub-Agent

## Identity
You are a market analysis specialist for investment due diligence.

## Your Task
Analyze the market opportunity for {ENTITY} based on the research plan and available data.

## Input Files
- Research Plan: read from brief
- Fact Store: read from brief
- Shared State: read from brief (shows what other dimensions have found)

## Output Requirements
Write your analysis to the output file specified in the brief.

Your output MUST include 3 files:
1. `{role_slug}.md` — Main analysis
2. `{role_slug}-facts.json` — New facts discovered
3. `{role_slug}-section.json` — Structured section package

## Fact Store Protocol
Before searching externally, check the Fact Store for existing facts.
If you discover new facts, append them to your facts sidecar file.
Use locked_read_modify_write when modifying shared files.

## Search Protocol
{common_tool_guide content or reference}

## Quality Standards
- Every claim must cite fact_ids
- Counter-evidence is mandatory for major claims
- Source quality hierarchy: official > institutional > reputable > auxiliary

## Stage-Specific Guidance
Current stage tier: {STAGE_TIER}
(Adjust depth based on stage — early-stage needs less granular data)
```

## Common Tool Guide (Separate File, Appended at Runtime)

`_common_tool_guide.md` is **NOT embedded inside role prompts**. It's loaded separately at module level and **appended** to every role's system prompt during manifest assembly:

```python
# Module-level load (bp_subagent_launcher_wb.py)
_TOOL_USAGE_GUIDE = _load_tool_usage_guide()   # reads _common_tool_guide.md once

# In _spawn_one(): appended to ALL roles after the role prompt
system_prompt = system_prompt + _CONCLUSION_APPENDIX + _TOOL_USAGE_GUIDE + _stage_block
```

This means role `.md` files should **NOT** include tool usage instructions — the tool guide handles that globally. Role prompts focus on methodology and output format.

**What the tool guide covers** (from actual `_common_tool_guide.md`):
1. **脚注引用规范** — footnote format (`[^N]`), source priority, mandatory for all quantitative data
2. **搜索与数据工具** — tool priority matrix per data type:
   - A/HK stocks → NeoData (`search_gateway prefer=auto`) or yfinance
   - Enterprise DD → QCC MCP (direct tool calls, not Bash)
   - News/reports → `search_gateway prefer=multi` or WebSearch
   - URL deep reading → WebFetch or `search_deep`
3. **禁止行为** — don't use only WebSearch, don't skip fetching full text

See [subagent-capabilities.md](subagent-capabilities.md) § "System Prompt Assembly" for the complete 4-part assembly chain.

## Template Variables

**Important**: In the BP pipeline, the launcher does **NOT** do `.replace()` substitution on prompt text. Variables like `{ENTITY}` and `{TASK_ID}` in the prompt files are **documentation placeholders** — the actual values come from the **brief** (which IS constructed dynamically per-dispatch).

If your pipeline needs runtime substitution, you have two options:

| Approach | How | When to use |
|----------|-----|-------------|
| **Brief injection** (BP pattern) | Brief contains actual file paths, entity name, role-specific data. Prompt references "read from brief". | Most cases — keeps prompts generic |
| **Python `.replace()`** | Launcher does `prompt.replace("{ENTITY}", entity)` before writing manifest | Only if prompt text itself must contain the value |

**Variables commonly available via brief** (not prompt substitution):

| Variable | Where it appears | Source |
|----------|-----------------|--------|
| Entity name | Brief header, role description | `job_ctx.entity` |
| Task ID | Brief header | `job_ctx.job_id` |
| Output path | Brief "Output File" section | manifest `output_path` |
| Stage tier | Appended as `_stage_block` (Part 4) | `bp_step0_profile.json` |
| Prior wave outputs | Brief "Prior Wave" section | Wave dependency graph |

## Benefits

1. **Edit prompts without touching code** — change a `.md` file, next run picks it up
2. **Version control** — prompts are in the repo, diffable, reviewable
3. **Single source of truth** — one file per role, no duplicates
4. **A/B testing** — swap files to test different prompt strategies
5. **Separation of concerns** — prompt authors don't need to understand Python runtime

## Adding a New Role

1. Create `instruction_store_{profile}/{new_role}.md`
2. Add entry to `index.json` (with `key`, `name`, `file`, `description`)
3. Add the role key to `CURRENT_BP_ROLES` set in the launcher (Python — this IS a code change)
4. Add the role to your wave definition in the profile
5. The dispatch prepare handler will automatically pick it up on next mtime change

**Important**: Adding a role requires BOTH a `.md` file AND a code change (adding to `CURRENT_BP_ROLES`). The "no code changes" claim is only true if you use a dynamic `CURRENT_BP_ROLES` that reads all roles from index.json without filtering.
