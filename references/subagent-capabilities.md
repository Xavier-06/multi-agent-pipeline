# Sub-Agent Capabilities & Tool Mapping

> **Scope**: This file covers how sub-agents **acquire their capabilities** — the complete assembly chain from raw instruction files to a fully-equipped sub-agent. For the storage/loading details of instruction files, see [instruction-store.md](instruction-store.md).
>
> **Relationship**: instruction-store.md = WHAT is stored + HOW it's loaded → subagent-capabilities.md = HOW it's assembled + WHAT else is combined

## The Critical Insight

A sub-agent's capability is determined by **three things** working together:

1. **System prompt** — what it knows and how it thinks (assembled from instruction store, see [instruction-store.md](instruction-store.md))
2. **Connector IDs** — what external tools it can call (MCP servers)
3. **Built-in tools** — what platform tools it always has (Bash, Read, WebSearch, etc.)

Get ANY of these wrong and the sub-agent is crippled:
- Missing connector IDs → can't access enterprise data (QCC, GitHub, etc.)
- Wrong system prompt → searches wrong things, produces wrong format
- Missing built-in tool guidance → doesn't know how to use the tools it has

## 1. Connector IDs (MCP Server Access)

Sub-agents access external data through MCP (Model Context Protocol) connectors. The manifest's `connectorIds` field determines which MCP servers are available to the sub-agent.

### How It Works

```python
# In manifest construction:
manifest = {
    "connectorIds": ["qcc-company", "qcc-risk", "qcc-ipr"],  # MCP server IDs
    # ... other fields
}
```

The Coordinator spawns the sub-agent with these connector IDs:
```python
Agent(
    name=role_name,
    connectorIds=manifest["connectorIds"],  # ← passed to Agent tool
    # ... other params
)
```

### Common Connector Patterns

| Domain | Connector IDs | What they provide |
|--------|--------------|-------------------|
| Chinese enterprise DD | `qcc-company, qcc-executive, qcc-risk, qcc-ipr, qcc-operation, qcc-history` | Company registry, litigation, patents, qualifications |
| GitHub analysis | `github` | Repo access, PR reviews, issue search |
| Financial data (A/HK) | `neodata-financial-search` or custom | Structured financial data, stock quotes |
| Document platforms | `tencent-docs`, `kdocs` | Cloud document read/write |
| Communication | `feishu`, `dingtalk`, `wecom` | Messaging, calendar, approval flows |

### Role-to-Connector Mapping (BP Example)

```python
# BP pipeline: ALL roles get QCC connectors (enterprise DD)
_BP_QCC_CONNECTOR_IDS = [
    'qcc-company',      # Company info, shareholders, executives
    'qcc-executive',    # Executive positions, controlled companies
    'qcc-risk',         # Judicial docs, dishonesty, penalties
    'qcc-ipr',          # Patents, trademarks, copyrights
    'qcc-operation',    # Qualifications, bidding info
    'qcc-history',      # Historical shareholders, investments
]

# Applied uniformly to all roles:
manifest["connectorIds"] = _BP_QCC_CONNECTOR_IDS
```

### Customizing Per-Role

Some roles may need different connectors:

```python
ROLE_CONNECTOR_OVERRIDES = {
    # Valuation role needs financial data connectors
    "valuation_return": _BP_QCC_CONNECTOR_IDS + ["neodata-financial-search"],
    # Deal breaker role might need additional risk connectors
    "dealbreaker_risk": _BP_QCC_CONNECTOR_IDS + ["qcc-history"],
}

def get_connector_ids(role_slug: str) -> list[str]:
    return ROLE_CONNECTOR_OVERRIDES.get(role_slug, _BP_QCC_CONNECTOR_IDS)
```

## 2. System Prompt Assembly (3-Part Composition)

The final system prompt is assembled from 3+ sources at manifest build time:

```python
system_prompt = (
    instruction_store_prompt      # 1. Role-specific instructions (hot-loaded)
    + conclusion_appendix         # 2. Mandatory conclusion format rules
    + tool_usage_guide            # 3. Tool mapping and priority guide
    + stage_prompt_block          # 4. Stage-tier specific guidance (optional)
)
```

### Part 1: Instruction Store Prompt

Loaded from `instruction_store_{profile}/{role}.md` via module-level cache with mtime detection. Contains:
- Role identity and expertise area
- Analysis framework and methodology
- Section structure requirements
- Quality standards (claim-fact binding, counter-evidence)
- Output format specifications

**Important**: Role `.md` files should NOT include tool usage instructions — that's handled by Part 3 (`_common_tool_guide.md`) which is appended globally. Role prompts focus on methodology and output format only.

See [instruction-store.md](instruction-store.md) for the complete loading mechanism and index.json schema.

### Part 2: Conclusion Appendix

A mandatory block appended to ALL roles ensuring structured conclusions:

```markdown
## ⚠️ Mandatory Conclusion Rules (highest priority)
You MUST include a "Dimension Conclusion" section at the end:
1. One-line conclusion (≤50 chars, answer: yes/no/partial/unclear)
2. Confidence level: High (multi-source) / Medium (single credible) / Low (BP-only or inference)
3. Key evidence summary (1-2 sentences, cite most important facts)
4. If confidence is Medium or Low: list what's needed to upgrade
```

### Part 3: Tool Usage Guide (Hot-Loaded)

Loaded from `instruction_store_{profile}/_common_tool_guide.md`. This is the **tool mapping** that tells sub-agents:

```markdown
## Tool Priority Matrix

| What you need | Preferred tool | Fallback |
|--------------|---------------|----------|
| Stock quotes / financials | search_gateway (prefer=auto) | WebSearch |
| Company registry / litigation | QCC MCP tools (direct call) | WebSearch |
| Patents / trademarks | QCC MCP (qcc-ipr) | WebSearch |
| News / industry reports | search_gateway (prefer=multi) | WebSearch |
| Read a specific URL | WebFetch | — |
| Search + read in one step | search_gateway search_deep | — |

### Prohibited behaviors:
- Do NOT use only WebSearch for everything
- Do NOT use WebSearch when QCC can return structured data directly
- Do NOT use only search snippets — fetch full text for key claims
```

### Part 4: Stage-Tier Block (Optional)

For early-stage projects, a stage-specific guidance block:

```markdown
## Stage: T1 (Seed/Angel)
This is an early-stage company. Adjust your analysis:
- Revenue data may not exist — focus on team, technology, and market opportunity
- Customer validation is about pipeline, not actual revenue
- Valuation should use comparable transactions, not DCF
- Flag data gaps honestly — do not fabricate metrics
```

## 3. Built-in Tools Reference

Sub-agents always have access to these platform tools. The tool guide must reference them correctly:

### Always Available

| Tool | Usage | Notes |
|------|-------|-------|
| `Read` | Read any file by path | Primary file reading tool |
| `Bash` | Execute shell commands | For search_gateway, yfinance, etc. |
| `WebSearch` | General web search | Fallback for all search needs |
| `WebFetch` | Read URL content | Deep reading of search results |
| `Write` | Write files | For output files only |

### NOT Available in Sub-Agents

| Tool | Why not | Workaround |
|------|---------|------------|
| `Glob` | Sub-agent context limitation | Use `Bash: find -name` |
| `Grep` | Sub-agent context limitation | Use `Bash: grep -r` |
| `TaskCreate/Update` | Sub-agents don't manage tasks | Just do the work |

### Tool Guide Must Tell Sub-Agents:

```markdown
## File Operations
- Use Read tool to read files (NOT cat/head/tail)
- Use Bash for file search: `find /path -name "*.json"`
- Do NOT use Glob or Grep (not available in sub-agent context)

## Search Operations
- Use WebSearch for general web search
- Use Bash + search_gateway for structured/financial search
- Use QCC MCP tools directly (no Bash needed) for enterprise data
- Use WebFetch to read full text of important URLs

## Output Operations
- Use Write tool for your 3 output files
- Write all 3 files atomically (md → facts.json → section.json)
```

## 4. Brief Construction (Structured Input Document)

The brief is the sub-agent's "mission briefing" — it tells the sub-agent WHAT to do (the system prompt tells it HOW to do it).

### Brief Structure

```markdown
# Research Brief — {role_name}

## Output File
`{output_path}`

## Key Input Files (read ALL of these completely)
- Shared Diligence Page: `shared_diligence_page.md`
- Research Plan: `research_plan.json`
- Fact Store: `fact_store.json`
- Shared State: `shared_state.json`
- OCR text: `ocr_text.txt`
- Company profile: `step0_profile.json`
{prior wave outputs if applicable}

## Your Research Plan Slice
```json
{questions and claims assigned to THIS role only}
```

## Your Search Work Order
```json
{claim-level search tasks with minimum query/fetch/domain requirements}
```

## Cross-Dimension Awareness
{other roles being investigated in parallel — for cross-referencing only}

## Self-Closing Rules
1. Data gap → search yourself (use tools per priority guide)
2. Insufficient sources → search more, add to output
3. Contradictory data → judge reliability, note contradiction
4. Completion = output files written (3 files)

## Search Depth Requirements
- ≥ 8 independent search queries
- ≥ 3 URLs deep-read (WebFetch or search_deep)
- ≥ 3 independent source domains
- Multi-round: broad scan → deep verify → cross-verify/counter-evidence

## Delivery Protocol (highest priority)
3 files required:
1. Markdown analysis: `{role}.md`
2. Facts sidecar: `{role}-facts.json`
3. Section package: `{role}-section.json`
```

### Brief Template Variables

| Variable | Source | Example |
|----------|--------|---------|
| `{role_name}` | manifest role | `market_analysis` |
| `{output_path}` | manifest output_path | `/path/to/outputs/phase2_market.md` |
| `{entity}` | job_ctx.entity | `Acme Corp` |
| `{task_id}` | job_ctx.job_id | `job_20260630_001` |

## 5. Quality Validation (Per-Role Output Check)

After sub-agent completes, validate output quality BEFORE advancing:

```python
def check_role_quality(role_name, task_dir, output_path) -> dict:
    """Validate sub-agent output against section package schema."""

    checks = {
        "md_exists": output_path.exists() and output_path.stat().st_size > 200,
        "facts_sidecar": _json_valid(facts_path) and len(facts) > 0,
        "section_sidecar": _json_valid(section_path)
                          and schema_version == "section_package.v2",
        "search_audit": queries >= 8 and fetched_urls >= 3
                       and unique_domains >= 3,
        "claim_coverage": len(claim_ids_covered) >= 1,
        "fact_id_binding": all fact_ids in answers/claims/narrative_blocks
                          exist in facts sidecar,
    }

    return {
        "passed": all(checks.values()),
        "score": sum(checks.values()) / len(checks),
        "errors": [k for k, v in checks.items() if not v],
    }
```

## 6. Complete Capability Chain

```
┌─────────────────────────────────────────────────────────────────┐
│ Pipeline Profile defines:                                       │
│   - Which roles exist (wave definitions)                        │
│   - Which connectors each role needs                            │
│   - Which instruction store to use                              │
├─────────────────────────────────────────────────────────────────┤
│ Manifest Builder assembles:                                     │
│   system_prompt = instruction_store + conclusion + tool_guide   │
│   + stage_block                                                 │
│   brief = structured input document with file paths             │
│   connectorIds = MCP server access list                         │
│   output_path = where to write 3 files                          │
├─────────────────────────────────────────────────────────────────┤
│ Coordinator spawns sub-agent:                                   │
│   Agent(                                                        │
│     name=role,                                                  │
│     prompt=manifest.system_prompt + brief_content,              │
│     connectorIds=manifest.connectorIds,                         │
│     mode="bypassPermissions",                                   │
│     team_name=...,                                              │
│   )                                                             │
├─────────────────────────────────────────────────────────────────┤
│ Sub-agent executes with:                                        │
│   - Platform tools: Read, Bash, WebSearch, WebFetch, Write      │
│   - MCP tools: qcc-*, github, neodata, etc.                     │
│   - Knowledge: system prompt + brief + input files              │
│   - Output: 3 files (md + facts + section)                      │
├─────────────────────────────────────────────────────────────────┤
│ Collect validates:                                              │
│   - 3 files exist with correct schema                           │
│   - Search audit meets minimums                                 │
│   - Fact IDs bind correctly                                     │
│   - Quality score passes threshold                              │
└─────────────────────────────────────────────────────────────────┘
```

## Design Checklist for New Pipelines

When creating a new pipeline, for each role define:

- [ ] **Role name and slug** — unique identifier
- [ ] **Instruction store file** — `instruction_store_{profile}/{role}.md`
- [ ] **Connector IDs** — which MCP servers this role needs
- [ ] **Wave assignment** — which wave this role belongs to
- [ ] **Research plan slice** — which questions/claims this role owns
- [ ] **Search work order** — minimum search requirements
- [ ] **Prior wave inputs** — which previous wave outputs to reference
- [ ] **Output path** — where to write the 3 output files
- [ ] **Quality thresholds** — minimum search audit, claim coverage
- [ ] **Stage-tier behavior** — how to adjust for early-stage projects
