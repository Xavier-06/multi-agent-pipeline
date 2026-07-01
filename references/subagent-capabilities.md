# Sub-Agent Capabilities & Tool Mapping

> **Scope**: This file covers how sub-agents **acquire their capabilities** — the complete assembly chain from raw instruction files to a fully-equipped sub-agent. For the storage/loading details of instruction files, see [instruction-store.md](instruction-store.md).
>
> **Relationship**: instruction-store.md = WHAT is stored + HOW it's loaded → subagent-capabilities.md = HOW it's assembled + WHAT else is combined

## The Critical Insight

A sub-agent's capability is determined by **three things** working together:

1. **System prompt** — what it knows and how it thinks (assembled from instruction store)
2. **Connector IDs** — what external tools it can call (MCP servers)
3. **Built-in tools** — what platform tools it always has (Bash, Read, WebSearch, etc.)

Get ANY of these wrong and the sub-agent is crippled:
- Missing connector IDs → can't access domain-specific data sources
- Wrong system prompt → searches wrong things, produces wrong format
- Missing built-in tool guidance → doesn't know how to use the tools it has

## 1. Connector IDs (MCP Server Access)

Sub-agents access external data through MCP (Model Context Protocol) connectors. The manifest's `connectorIds` field determines which MCP servers are available to the sub-agent.

### How It Works

```python
# In manifest construction:
manifest = {
    "connectorIds": ["github", "your-data-source"],  # MCP server IDs
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

| Domain | Example Connector IDs | What they provide |
|--------|----------------------|-------------------|
| Enterprise data | `qcc-company, qcc-risk, qcc-ipr` | Company registry, litigation, patents |
| Code analysis | `github` | Repo access, PR reviews, issue search |
| Financial data | `neodata-financial-search` | Structured financial data, stock quotes |
| Document platforms | `tencent-docs`, `kdocs` | Cloud document read/write |
| Communication | `feishu`, `dingtalk`, `wecom` | Messaging, calendar, approval flows |

### Configuring Per-Role

Define a base set of connectors for your pipeline, with optional per-role overrides:

```python
# Base connectors — all roles get these
BASE_CONNECTOR_IDS = ["your-data-source", "your-search-tool"]

# Per-role overrides (optional)
ROLE_CONNECTOR_OVERRIDES = {
    "financial_analyst": BASE_CONNECTOR_IDS + ["neodata-financial-search"],
    "legal_analyst": BASE_CONNECTOR_IDS + ["pkulaw"],
}

def get_connector_ids(role_slug: str) -> list[str]:
    return ROLE_CONNECTOR_OVERRIDES.get(role_slug, BASE_CONNECTOR_IDS)
```

## 2. System Prompt Assembly (Multi-Part Composition)

The final system prompt is assembled from 3+ sources at manifest build time:

```python
system_prompt = (
    instruction_store_prompt      # 1. Role-specific instructions (hot-loaded)
    + conclusion_appendix         # 2. Mandatory conclusion format rules
    + tool_usage_guide            # 3. Tool mapping and priority guide
    + domain_appendix             # 4. Domain-specific guidance (optional)
)
```

### Part 1: Instruction Store Prompt

Loaded from `instruction_store_{profile}/{role}.md` via module-level cache with mtime detection. Contains:
- Role identity and expertise area
- Analysis framework and methodology
- Section structure requirements
- Quality standards (data binding, counter-findings)
- Output format specifications

**Important**: Role `.md` files should NOT include tool usage instructions — that's handled by Part 3 (common tool guide) which is appended globally. Role prompts focus on methodology and output format only.

See [instruction-store.md](instruction-store.md) for the complete loading mechanism.

### Part 2: Conclusion Appendix

A mandatory block appended to ALL roles ensuring structured conclusions. Customize for your domain:

```markdown
## Mandatory Conclusion Rules (highest priority)
You MUST include a conclusion section at the end:
1. One-line conclusion (answer: yes/no/partial/unclear)
2. Confidence level: High (multi-source) / Medium (single credible) / Low (inference)
3. Key evidence summary (cite most important data points)
4. If confidence is Medium or Low: list what's needed to upgrade
```

### Part 3: Tool Usage Guide (Hot-Loaded)

Loaded from `instruction_store_{profile}/_common_tool_guide.md`. Tells sub-agents which tools to use for which task:

```markdown
## Tool Priority Matrix

| What you need | Preferred tool | Fallback |
|--------------|---------------|----------|
| Domain data source X | MCP tool (direct call) | WebSearch |
| Web search | WebSearch | — |
| Read a specific URL | WebFetch | — |

### Prohibited behaviors:
- Do NOT use only WebSearch for everything
- Do NOT skip fetching full text for key sources
```

### Part 4: Domain-Specific Appendix (Optional)

For projects with stage/weight classifications:

```markdown
## Stage: Early-Stage
This is an early-stage project. Adjust your analysis:
- Some data may not exist — focus on what's available
- Flag data gaps honestly — do not fabricate metrics
```

## 3. Built-in Tools Reference

Sub-agents always have access to these platform tools:

### Always Available

| Tool | Usage | Notes |
|------|-------|-------|
| `Read` | Read any file by path | Primary file reading tool |
| `Bash` | Execute shell commands | For scripts, CLI tools |
| `WebSearch` | General web search | Fallback for all search needs |
| `WebFetch` | Read URL content | Deep reading of search results |
| `Write` | Write files | For output files |

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
- Use MCP tools directly for domain-specific data
- Use WebSearch for general web search
- Use WebFetch to read full text of important URLs

## Output Operations
- Use Write tool for your 3 output files
- Write all 3 files (md + data sidecar + meta sidecar)
```

## 4. Brief Construction (Structured Input Document)

The brief is the sub-agent's "mission briefing" — it tells the sub-agent WHAT to do (the system prompt tells it HOW to do it).

### Brief Structure

```markdown
# Brief — {role_name}

## Output File
`{output_path}`

## Key Input Files (read ALL of these completely)
- Shared State Page: `shared_state_page.md`
- Plan: `plan.json`
- Evidence Store: `evidence_store.json`
{prior wave outputs if applicable}

## Your Assignment Slice
```json
{items and questions assigned to THIS role only}
```

## Output Requirements
3 files required:
1. `{role}.md` — Main analysis
2. `{role}-data.json` — Structured data sidecar
3. `{role}-meta.json` — Metadata sidecar
```

### Brief Template Variables

| Variable | Source | Example |
|----------|--------|---------|
| `{role_name}` | manifest role | `market_analysis` |
| `{output_path}` | manifest output_path | `/path/to/outputs/market.md` |
| `{entity}` | job_ctx.entity | `Acme Corp` |
| `{task_id}` | job_ctx.job_id | `job_20260701_001` |

## 5. Quality Validation (Per-Role Output Check)

After sub-agent completes, validate output quality BEFORE advancing:

```python
def check_role_quality(role_name, task_dir, output_path) -> dict:
    """Validate sub-agent output against your domain's contract.

    Customize these checks for your pipeline.
    """
    data_path = output_path.with_name(f"{role_name}-data.json")
    meta_path = output_path.with_name(f"{role_name}-meta.json")

    checks = {
        "md_exists": output_path.exists() and output_path.stat().st_size > 200,
        "data_sidecar": _json_valid(data_path),
        "meta_sidecar": _json_valid(meta_path),
        # Add your domain-specific checks here
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
│   + domain_appendix                                             │
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
│   - MCP tools: per connectorIds                                 │
│   - Knowledge: system prompt + brief + input files              │
│   - Output: 3 files (md + data + meta)                          │
├─────────────────────────────────────────────────────────────────┤
│ Collect validates:                                              │
│   - 3 files exist with correct schema                           │
│   - Quality score passes threshold                              │
└─────────────────────────────────────────────────────────────────┘
```

## Design Checklist for New Pipelines

When creating a new pipeline, for each role define:

- [ ] **Role name and slug** — unique identifier
- [ ] **Instruction store file** — `instruction_store_{profile}/{role}.md`
- [ ] **Connector IDs** — which MCP servers this role needs
- [ ] **Wave assignment** — which wave this role belongs to
- [ ] **Assignment slice** — which items/questions this role owns
- [ ] **Prior wave inputs** — which previous wave outputs to reference
- [ ] **Output path** — where to write the 3 output files
- [ ] **Quality thresholds** — minimum output requirements
- [ ] **Stage-aware behavior** — how to adjust for lightweight projects
