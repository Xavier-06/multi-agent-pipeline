# Presearch Pattern (Pre-Dispatch Intelligence Gathering)

## Problem

Sub-agents dispatched into a wave need context to do their work. If they start from zero, they waste time on basic fact-finding that could have been done once, upfront. Without a presearch phase:

- Each sub-agent independently searches for the same baseline information → duplicated effort
- The evidence store starts empty, so early-wave agents have nothing to cross-reference
- Claims from the plan have no prior validation — agents may pursue dead ends

## Solution: Presearch Phase

Run a **fast, broad, cheap** search pass between planning and the first wave dispatch. The goal is not deep analysis — it's to seed the evidence store and give Wave 1 sub-agents a head start.

```
Plan → Presearch → Evidence Store Bootstrap → Wave 1 Dispatch
                        ↑
                  This phase seeds the store
```

## When to Use

| Scenario | Presearch needed? |
|----------|-------------------|
| Domain requires external data (company info, market data, regulatory filings) | **Yes** |
| All data comes from user-provided documents | No — skip and let sub-agents read the docs directly |
| Evidence store already populated from a previous run | No — skip or run in "refresh" mode |
| Very small pipeline (1-2 roles, simple topic) | Optional — overhead may not justify value |

## Architecture

### Input

The presearch phase receives:
- The **plan** (with its tracked items / claims / questions)
- The **source document** (if any — e.g., uploaded PDF, user brief)
- A list of **search dimensions** derived from the plan

### Output

```json
{
  "schema_version": "presearch_results.v1",
  "searches": [
    {
      "query": "What was searched",
      "source": "web_search | database_api | document_extraction",
      "results": [
        {
          "title": "Result title",
          "url": "https://...",
          "snippet": "Relevant excerpt",
          "relevance": "high | medium | low",
          "extracted_facts": [
            {
              "fact_id": "F-PRE-001",
              "description": "Extracted fact text",
              "value": "42%",
              "source_url": "https://...",
              "confidence": "medium",
              "source_tier": "industry_report"
            }
          ]
        }
      ]
    }
  ],
  "facts_extracted": 15,
  "gaps_identified": ["No data found for X", "Conflicting data on Y"],
  "duration_seconds": 45
}
```

### Phase Handler Pattern

```python
def _run_presearch(runtime_root, job_ctx):
    """Fast broad search to seed the evidence store before Wave 1."""
    task_dir = _task_dir(runtime_root, job_ctx)

    # 1. Load the plan
    plan = load_json(task_dir / "plan.json")

    # 2. Generate search queries from plan
    #    - One query per tracked item / claim / core question
    #    - Deduplicate overlapping queries
    search_queries = _generate_search_queries(plan)

    # 3. Execute searches (parallel where possible)
    results = []
    for query in search_queries:
        result = _execute_search(query, job_ctx)  # YOUR IMPLEMENTATION
        results.append(result)

    # 4. Extract facts from search results
    facts = _extract_facts(results)  # YOUR IMPLEMENTATION

    # 5. Write presearch results
    presearch_output = {
        "schema_version": "presearch_results.v1",
        "searches": results,
        "facts_extracted": len(facts),
        "gaps_identified": _identify_gaps(plan, facts),
    }
    write_json(task_dir / "presearch_results.json", presearch_output)

    # 6. Seed the evidence store
    _seed_evidence_store(task_dir, facts)

    return {
        "ok": True,
        "mode": "presearch",
        "phase": "phase04_presearch",
        "result": {
            "queries_executed": len(results),
            "facts_extracted": len(facts),
            "gaps": presearch_output["gaps_identified"],
        },
    }
```

## Search Query Generation Strategies

| Strategy | When to use | Example |
|----------|-------------|---------|
| **Claim-driven** | Plan has specific claims to verify | "Acme Corp revenue 2025" |
| **Question-driven** | Plan has open questions | "What is the market size for X?" |
| **Entity-driven** | Plan identifies key entities | "Acme Corp", "Product X", "CEO Y" |
| **Dimension-driven** | Plan defines analysis dimensions | "Competitive landscape of X market" |

A good presearch combines all four: start with entities for broad context, then target specific claims and questions.

## Search Execution Patterns

### Parallel vs Sequential

```python
# Parallel (preferred for independent queries)
import asyncio

async def _execute_searches_parallel(queries):
    tasks = [_execute_search_async(q) for q in queries]
    return await asyncio.gather(*tasks)

# Sequential (when rate-limited or queries depend on prior results)
def _execute_searches_sequential(queries):
    results = []
    for query in queries:
        results.append(_execute_search(query))
    return results
```

### Search Source Priority

Configure based on your domain:

```python
SEARCH_SOURCE_PRIORITY = {
    "financial_data": ["financial_api", "regulatory_filings", "web_search"],
    "company_info": ["company_registry_api", "web_search"],
    "market_data": ["industry_reports", "news_search", "web_search"],
    "technical": ["patent_database", "academic_search", "web_search"],
    "general": ["web_search"],
}
```

## Evidence Store Seeding

After presearch, extracted facts are written to the evidence store so Wave 1 sub-agents can reference them:

```python
def _seed_evidence_store(task_dir, presearch_facts):
    """Seed the evidence store with presearch findings."""
    store_path = task_dir / "evidence_store.json"

    existing = {}
    if store_path.exists():
        existing = load_json(store_path)

    existing_facts = existing.get("items", [])
    existing_ids = {f["item_id"] for f in existing_facts}

    new_facts = [f for f in presearch_facts if f["item_id"] not in existing_ids]
    existing_facts.extend(new_facts)

    existing["items"] = existing_facts
    existing["last_seeded"] = timestamp_now()
    write_json(store_path, existing)
```

## Integration with Plan Enrichment

Presearch results can feed back into the plan:

```
Plan (skeleton) → Presearch → Results → Plan Enrichment (LLM) → Enriched Plan → Wave 1
```

The LLM enrichment step can use presearch results to:
- Adjust claim priorities (confirmed claims → lower priority, contradicted → higher)
- Refine questions (narrow broad questions based on what was found)
- Add new claims discovered during presearch
- Remove claims that presearch proved irrelevant

## Cost Control

Presearch can be expensive if unbounded. Set limits:

```python
PRESEARCH_CONFIG = {
    "max_queries": 20,           # Cap total search queries
    "max_results_per_query": 5,  # Cap results per query
    "max_depth": 1,              # Don't follow links recursively
    "timeout_seconds": 120,      # Hard timeout for entire presearch
    "max_tokens": 50000,         # Token budget for extraction
}
```

## Common Pitfalls

- **Don't over-search.** Presearch is a quick scan, not deep research. 20 queries max is usually enough.
- **Don't analyze.** Presearch extracts facts; it doesn't draw conclusions. Leave analysis to wave sub-agents.
- **Don't block on failures.** If a search source is unavailable, skip it and note the gap. Don't let one failed API call kill the presearch.
- **Do deduplicate.** Multiple queries may return the same fact. Deduplicate by content similarity, not just exact match.
- **Do preserve source URLs.** Sub-agents need to cite sources; presearch must preserve the provenance chain.
