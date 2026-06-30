# Instruction Store (Hot-Loaded System Prompts)

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

Maps role names to their prompt file. This is the single registry the launcher reads:

```json
{
  "version": "v8",
  "roles": {
    "market_analysis": {
      "file": "market_analysis.md",
      "description": "Market size, growth, segmentation analysis"
    },
    "tech_product": {
      "file": "tech_product.md",
      "description": "Product architecture, tech stack, IP/moat assessment"
    },
    "synthesis": {
      "file": "synthesis.md",
      "description": "Full report assembly from dimension analyses"
    }
  }
}
```

## Loading Pattern

### In Dispatch Prepare

```python
def _build_manifest(runtime_root, job_ctx, role_slug, wave):
    # 1. Hot load system prompt from instruction store
    instruction_store = runtime_root / "instruction_store_{profile}"
    prompt_path = instruction_store / f"{role_slug}.md"

    if prompt_path.exists():
        system_prompt = prompt_path.read_text(encoding="utf-8")
    else:
        # Fallback: use the launcher's built-in loader
        from scripts.subagent_launcher import load_instruction_store_prompts
        prompts = load_instruction_store_prompts()
        system_prompt = prompts.get(role_slug, f"ERROR: prompt not found for {role_slug}")

    # 2. Template variable substitution
    system_prompt = system_prompt.replace("{TASK_ID}", job_ctx.job_id)
    system_prompt = system_prompt.replace("{ENTITY}", job_ctx.entity)
    system_prompt = system_prompt.replace("{STAGE_TIER}", stage_tier or "unknown")
    system_prompt = system_prompt.replace("{OUTPUTS_DIR}", str(outputs_dir))

    # 3. Inject into manifest
    manifest = {
        "system_prompt": system_prompt,    # FULL text, not summarized
        "brief_path": str(brief_path),
        # ... other fields
    }
```

### Fallback Chain

If the instruction store file is missing, the system falls back through:

```
1. instruction_store_{profile}/{role_slug}.md     ← Primary (hot-loaded file)
2. launcher.load_instruction_store_prompts()       ← Secondary (Python loader)
3. "ERROR: prompt not found"                       ← Explicit failure marker
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

## Common Tool Guide

A shared document (`_common_tool_guide.md`) that all roles reference:

```markdown
# Common Tool Guide

## File Operations
- Use Read tool to read files
- Use Bash for file search (find, ls)
- Do NOT use Glob or Grep (not available in sub-agent context)

## Search Tools
- WebSearch for general web search
- Specific data source APIs as listed in your brief

## Writing Output
- Write main analysis as Markdown
- Write facts as JSON to sidecar file
- Write section package as JSON to sidecar file
- All 3 files must be complete before you are done

## When You Are Done
When all 3 output files are written, your task is complete.
Do NOT return to the Coordinator asking for more data.
```

## Template Variables

The instruction store supports these template variables (replaced at manifest build time):

| Variable | Source | Example |
|----------|--------|---------|
| `{TASK_ID}` | `job_ctx.job_id` | `job_20260630_001` |
| `{ENTITY}` | `job_ctx.entity` | `Acme Corp` |
| `{MARKET}` | `job_ctx.market` | `us` |
| `{STAGE_TIER}` | From profile metadata | `T1`, `T2`, `T3` |
| `{OUTPUTS_DIR}` | Workspace outputs path | `/path/to/outputs` |
| `{WAVE_NUMBER}` | Current wave index | `1`, `2`, `3` |

## Benefits

1. **Edit prompts without touching code** — change a `.md` file, next run picks it up
2. **Version control** — prompts are in the repo, diffable, reviewable
3. **Single source of truth** — one file per role, no duplicates
4. **A/B testing** — swap files to test different prompt strategies
5. **Separation of concerns** — prompt authors don't need to understand Python runtime

## Adding a New Role

1. Create `instruction_store_{profile}/{new_role}.md`
2. Add entry to `index.json`
3. Add the role to your wave definition in the profile
4. The dispatch prepare handler will automatically pick it up

No code changes needed in the runtime.
