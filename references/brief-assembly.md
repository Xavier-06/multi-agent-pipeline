# Brief Assembly (Sub-Agent Mission Briefing)

## Problem

Each sub-agent needs a **mission briefing** — a document telling it WHAT to do, WHICH files to read, and WHAT to produce. The system prompt tells it HOW to think; the brief tells it WHAT to work on.

Without a well-structured brief, sub-agents:
- Don't know which input files to read
- Process the entire plan instead of their assigned slice
- Duplicate work that other roles are handling
- Miss prior wave outputs they should build on

## Three Components of a Brief

```
Brief = Assignment Slice + Search Work Order + File References
         (WHAT to analyze)  (HOW to search)     (WHERE to read/write)
```

## Component 1: Assignment Slice

The assignment slice is a **filtered view** of the plan, showing only the items assigned to this specific role. Without it, a sub-agent processing a 50-item plan would waste effort on items owned by other roles.

### Plan Structure (what gets sliced)

```json
{
  "tracked_items": [
    {
      "id": "I001",
      "item": "Revenue exceeds $100M",
      "owner": "financial_analyst",
      "priority": "critical",
      "required_data_keys": ["revenue", "growth_rate"]
    },
    {
      "id": "I002",
      "item": "Technology is differentiated",
      "owner": "tech_analyst",
      "priority": "high",
      "required_data_keys": ["patents", "benchmarks"]
    }
  ],
  "questions": [
    {"id": "Q1", "question": "What is the customer concentration?", "owner": "financial_analyst"},
    {"id": "Q2", "question": "Is the patent portfolio defensible?", "owner": "tech_analyst"}
  ]
}
```

### Slicing Logic

```python
def load_assignment_slice(plan: dict, role_name: str) -> dict:
    """Extract only the items and questions assigned to this role.

    This is the sub-agent's "scope of work" — it should NOT process
    items owned by other roles (write those to data_gaps instead).
    """
    items = [
        item for item in plan.get("tracked_items", [])
        if item.get("owner") == role_name
    ]
    questions = [
        q for q in plan.get("questions", [])
        if q.get("owner") == role_name
    ]
    return {
        "role": role_name,
        "items": items,
        "questions": questions,
        "item_count": len(items),
        "question_count": len(questions),
    }
```

### Injected into Brief

```markdown
## Your Assignment Slice

You are responsible for the following items and questions ONLY.
Do NOT investigate items owned by other roles — if you notice issues
with another role's items, write them to your `data_gaps` or
`counter_findings` section, not your main analysis.

```json
{
  "role": "financial_analyst",
  "items": [
    {"id": "I001", "item": "Revenue exceeds $100M", "priority": "critical"}
  ],
  "questions": [
    {"id": "Q1", "question": "What is the customer concentration?"}
  ]
}
```
```

### Why This Matters

In a 4-wave, 8-role pipeline with 50 tracked items, each role typically owns 5-8 items. Without slicing, each sub-agent processes all 50 → 7x wasted effort and contradictory conclusions across roles.

## Component 2: Search Work Order

The search work order tells the sub-agent **minimum search requirements** — how much external research it must do. Without explicit requirements, sub-agents tend to:
- Do one web search and call it done
- Only read search snippets instead of full pages
- Not cross-reference multiple sources

### Search Work Order Schema

```json
{
  "schema_version": "search_work_order.v1",
  "role": "financial_analyst",
  "search_tasks": [
    {
      "task_id": "SW001",
      "target_item_id": "I001",
      "query": "Acme Corp revenue 2025 annual report",
      "min_queries": 3,
      "min_fetched_urls": 2,
      "min_domains": 2,
      "search_type": "verification"
    }
  ],
  "aggregate_requirements": {
    "total_min_queries": 8,
    "total_min_fetched_urls": 3,
    "total_min_domains": 3,
    "search_rounds": ["broad_scan", "deep_verify", "counter_evidence"]
  }
}
```

### Generating the Work Order

```python
def compile_search_work_order(plan: dict, role_name: str) -> dict:
    """Generate search tasks for a specific role based on its assignment slice.

    Each tracked item / question generates one or more search tasks.
    The sub-agent must complete all search tasks and record results
    in the search_audit section of its metadata sidecar.
    """
    items = [i for i in plan.get("tracked_items", []) if i.get("owner") == role_name]
    questions = [q for q in plan.get("questions", []) if q.get("owner") == role_name]

    search_tasks = []
    for item in items:
        search_tasks.append({
            "task_id": f"SW-{item['id']}",
            "target_item_id": item["id"],
            "query": _generate_verification_query(item),
            "min_queries": 2,
            "min_fetched_urls": 1,
            "min_domains": 1,
            "search_type": "verification",
        })
    for q in questions:
        search_tasks.append({
            "task_id": f"SW-{q['id']}",
            "target_question_id": q["id"],
            "query": q["question"],
            "min_queries": 3,
            "min_fetched_urls": 2,
            "min_domains": 2,
            "search_type": "discovery",
        })

    return {
        "schema_version": "search_work_order.v1",
        "role": role_name,
        "search_tasks": search_tasks,
        "aggregate_requirements": {
            "total_min_queries": max(8, len(search_tasks) * 2),
            "total_min_fetched_urls": 3,
            "total_min_domains": 3,
            "search_rounds": ["broad_scan", "deep_verify", "counter_evidence"],
        },
    }
```

### Injected into Brief

```markdown
## Your Search Work Order

You must complete the following search tasks. Record all queries,
fetched URLs, and source domains in your metadata sidecar's search_audit.

Minimum requirements:
- ≥ 8 independent search queries
- ≥ 3 URLs deep-read (WebFetch)
- ≥ 3 independent source domains
- 3 rounds: broad scan → deep verify → counter-evidence

| Task ID | Target | Query | Min Queries | Min Fetches |
|---------|--------|-------|-------------|-------------|
| SW-I001 | I001 | "Acme Corp revenue 2025" | 2 | 1 |
| SW-Q1   | Q1   | "Customer concentration risk" | 3 | 2 |
```

## Component 3: File References

The brief must list ALL input files the sub-agent should read, in priority order. This is the **file discovery** mechanism — sub-agents don't glob directories, they read the brief to find file paths.

### File Categories

```python
def _build_file_references(task_dir, outputs_dir, role_name, wave, completed_waves):
    """Build the ordered file reference list for a sub-agent brief."""

    files = []

    # Category 1: Shared state (ALWAYS first — sub-agent reads this for context)
    shared_page = task_dir / "shared_state_page.md"
    if shared_page.exists():
        files.append({"path": shared_page, "category": "shared_state", "read_first": True})
    shared_json = task_dir / "shared_state.json"
    if shared_json.exists():
        files.append({"path": shared_json, "category": "shared_state"})
    evidence_store = task_dir / "evidence_store.json"
    if evidence_store.exists():
        files.append({"path": evidence_store, "category": "shared_state"})

    # Category 2: Plan and presearch
    plan = task_dir / "plan.json"
    if plan.exists():
        files.append({"path": plan, "category": "plan"})
    presearch = task_dir / "presearch_results.json"
    if presearch.exists():
        files.append({"path": presearch, "category": "plan"})

    # Category 3: Source documents (domain-specific)
    source_docs = sorted(task_dir.glob("source_*"))
    for doc in source_docs:
        files.append({"path": doc, "category": "source"})

    # Category 4: Prior wave outputs (Layer 3 from shared-state.md)
    #    CUMULATIVE — Wave 4 sees Wave 1 + 2 + 3 outputs
    for cw in completed_waves:
        for cw_role, cw_path in cw.items():
            if Path(cw_path).exists():
                files.append({
                    "path": cw_path,
                    "category": "prior_wave",
                    "wave_role": cw_role,
                })

    return files
```

### Injected into Brief

```markdown
## Key Input Files (read in order)

**Read these FIRST — they contain the current state of the investigation:**
- `shared_state_page.md` ← START HERE
- `shared_state.json`
- `evidence_store.json`

**Plan and presearch results:**
- `plan.json`
- `presearch_results.json`

**Source documents:**
- `source_document.pdf`

**Prior wave outputs (build on these, don't repeat):**
- `financial_analyst.md` (from Wave 1)
- `tech_analyst.md` (from Wave 1)
```

## Complete Brief Assembly

Putting it all together:

```python
def build_brief(task_dir, outputs_dir, role_spec, wave, completed_waves, plan):
    """Build the complete brief for a sub-agent.

    This is called during dispatch prepare, before writing the manifest.
    The brief path is included in the manifest's brief_path field.
    """
    role_name = role_spec["role_name"]

    # Build the 3 components
    assignment_slice = load_assignment_slice(plan, role_name)
    search_work_order = compile_search_work_order(plan, role_name)
    file_references = _build_file_references(
        task_dir, outputs_dir, role_name, wave, completed_waves
    )

    # Assemble brief
    lines = [
        f"# Brief — {role_name}",
        "",
        f"**Task ID:** {role_spec.get('task_id', '')}",
        f"**Entity:** {role_spec.get('entity', '')}",
        "",
        "## Key Input Files (read in order)",
        "",
    ]

    # File references (Category 1 first — shared state)
    for f in file_references:
        prefix = "🔵 " if f.get("read_first") else "- "
        lines.append(f"{prefix}`{f['path']}`")

    lines.extend([
        "",
        "## Your Assignment Slice",
        "",
        "You are responsible for the following items and questions ONLY.",
        "Do NOT investigate items owned by other roles.",
        "",
        "```json",
        json.dumps(assignment_slice, ensure_ascii=False, indent=2),
        "```",
        "",
        "## Your Search Work Order",
        "",
        "Complete all search tasks below. Record results in search_audit.",
        "",
        "```json",
        json.dumps(search_work_order, ensure_ascii=False, indent=2),
        "```",
        "",
        "## Output Requirements",
        "",
        f"Write to: `{outputs_dir / role_name}.md`",
        "",
        "Your output MUST include 3 files:",
        f"1. `{role_name}.md` — Main analysis",
        f"2. `{role_name}-data.json` — Structured data sidecar",
        f"3. `{role_name}-meta.json` — Metadata sidecar (with search_audit)",
        "",
        "## Completion Protocol",
        "",
        "You are done when ALL 3 files are written. If you find data gaps,",
        "search for them yourself before writing. Do NOT return without",
        "writing all 3 files.",
    ])

    brief_path = task_dir / f"brief_{role_name}.md"
    brief_path.write_text("\n".join(lines) + "\n", encoding="utf-8")
    return brief_path
```

## Cross-Wave File Passing (Layer 3)

Later-wave sub-agents need the actual outputs from prior waves. The brief assembly handles this via `completed_waves`:

```python
# In the wave prepare function:
completed_waves = []
for prior_wave_index in range(wave_index):
    wave_roles = WAVES[prior_wave_index]["roles"]
    wave_outputs = {}
    for role_slug in wave_roles:
        path = outputs_dir / f"{role_slug}.md"
        if path.exists():
            wave_outputs[role_slug] = str(path)
    completed_waves.append(wave_outputs)

# Pass to brief builder
brief_path = build_brief(task_dir, outputs_dir, role_spec, wave, completed_waves, plan)
```

The brief then lists prior wave outputs in the "Key Input Files" section, with a note: "Build on these, don't repeat their work."

## Design Checklist

- [ ] Assignment slice filters by `owner` field — sub-agent only sees its own items
- [ ] Search work order has minimum thresholds (queries, fetches, domains)
- [ ] File references are ordered: shared state first, then plan, then prior outputs
- [ ] Prior wave outputs are CUMULATIVE (Wave N sees all waves 1..N-1)
- [ ] Brief includes explicit "do NOT process other roles' items" instruction
- [ ] Completion protocol requires all 3 files written before sub-agent stops
