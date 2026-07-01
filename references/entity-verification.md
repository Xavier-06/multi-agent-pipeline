# Entity Verification (Pre-Flight Validation)

## Problem

Pipelines often assume the target entity exists and is correctly identified. Without verification:
- Sub-agents waste time searching for a non-existent entity
- Name typos or ambiguous entities produce garbage results
- The pipeline runs all 33 phases on a phantom target

## Solution: Entity Verification Phase

Run a **fast, cheap** verification step early in the pipeline (Phase 02) to confirm:
1. The entity exists in authoritative sources
2. The entity's basic attributes match user input
3. The entity is unique (not ambiguous)
4. Sufficient data exists to proceed

```
Intake → Entity Verification → Plan → Presearch → ...
              ↑
        If FAIL → stop pipeline early with clear error
```

## Architecture

### Input

The entity verification phase receives the user's input from the intake phase:

```json
{
  "entity_name": "Acme Corp",
  "entity_type": "company",
  "identifiers": {
    "registration_number": "91110000MA01XXXX",
    "ticker": "ACME",
    "website": "https://acme.example.com"
  },
  "market": "cn",
  "additional_context": "Series B startup in robotics"
}
```

### Output

```json
{
  "schema_version": "entity_verification.v1",
  "status": "verified | ambiguous | not_found | error",
  "verified_entity": {
    "name": "Acme Corp Ltd.",
    "official_name": "阿克米科技有限公司",
    "registration_number": "91110000MA01XXXX",
    "entity_type": "company",
    "status": "active",
    "founded": "2019-03-15",
    "source": "company_registry"
  },
  "confidence": "high | medium | low",
  "issues": [],
  "data_availability": {
    "registry_info": true,
    "financial_data": false,
    "news_coverage": true,
    "patent_data": true
  }
}
```

## Implementation

### Phase Handler

```python
def _run_entity_verify(runtime_root, job_ctx):
    """Verify the target entity exists and is uniquely identified.

    This is an EARLY STOP phase — if verification fails, the pipeline
    stops before wasting resources on plan generation and presearch.
    """
    task_dir = _task_dir(runtime_root, job_ctx)
    intake_data = load_json(task_dir / "intake_result.json")

    entity_name = intake_data.get("entity_name", job_ctx.entity)
    identifiers = intake_data.get("identifiers", {})

    # 1. Search authoritative sources
    verification = _verify_entity(entity_name, identifiers, intake_data.get("market", ""))

    # 2. Write verification result
    write_json(task_dir / "entity_verification.json", verification)

    # 3. Decide: proceed or stop
    status = verification.get("status", "error")

    if status == "verified":
        # Update job context with verified entity info
        job_ctx.entity = verification["verified_entity"]["official_name"] or entity_name
        return {
            "ok": True,
            "mode": "entity_verify",
            "result": {
                "status": "verified",
                "entity": job_ctx.entity,
                "confidence": verification.get("confidence", "high"),
                "data_availability": verification.get("data_availability", {}),
            },
        }

    elif status == "ambiguous":
        # Multiple matches — need user to disambiguate
        candidates = verification.get("candidates", [])
        return {
            "ok": False,
            "error": f"Ambiguous entity: {len(candidates)} matches found. "
                     f"Please specify: {[c.get('name') for c in candidates]}",
            "result": verification,
        }

    elif status == "not_found":
        return {
            "ok": False,
            "error": f"Entity '{entity_name}' not found in any authoritative source.",
            "result": verification,
        }

    else:  # error
        # Source unavailable — proceed with warning
        return {
            "ok": True,
            "mode": "entity_verify",
            "result": {
                "status": "unverified",
                "entity": entity_name,
                "warning": verification.get("error", "Verification source unavailable"),
            },
        }
```

### Verification Strategies (Domain-Specific)

```python
def _verify_entity(entity_name, identifiers, market):
    """Verify entity against authoritative sources.

    Customize the verification sources for your domain.
    """
    # Strategy 1: Use identifiers for exact match
    if identifiers.get("registration_number"):
        result = _lookup_by_registration(identifiers["registration_number"])
        if result:
            return _build_verified_result(result)

    # Strategy 2: Name-based search
    if identifiers.get("ticker"):
        result = _lookup_by_ticker(identifiers["ticker"], market)
        if result:
            return _build_verified_result(result)

    # Strategy 3: Fuzzy name search
    result = _search_by_name(entity_name, market)
    if result:
        if len(result) == 1:
            return _build_verified_result(result[0])
        elif len(result) > 1:
            return {
                "status": "ambiguous",
                "candidates": result[:5],
                "confidence": "low",
            }

    return {"status": "not_found", "confidence": "low"}
```

### Common Verification Sources by Domain

| Domain | Primary source | Secondary source | Fallback |
|--------|---------------|-----------------|----------|
| Company DD | Company registry API (QCC, SEC) | Stock exchange | Web search |
| Person | Social media (LinkedIn) | News search | — |
| Product | Product database | Company website | Web search |
| Legal case | Court records API | Legal database | Web search |
| Academic paper | DOI / ArXiv | University repository | Web search |
| Code project | GitHub API | Package registry | Web search |

### Data Availability Assessment

After verification, assess what data is available for the entity:

```python
def _assess_data_availability(verified_entity, market):
    """Check what data sources have information about this entity.

    This helps downstream phases know what to expect.
    """
    availability = {}

    # Check each potential data source (lightweight probes, not full queries)
    for source_name, probe_fn in DATA_SOURCE_PROBES.items():
        try:
            availability[source_name] = probe_fn(verified_entity)
        except Exception:
            availability[source_name] = False

    return availability


DATA_SOURCE_PROBES = {
    "registry_info": lambda e: True,  # Already verified
    "financial_data": lambda e: _probe_financial(e),
    "news_coverage": lambda e: _probe_news(e),
    "patent_data": lambda e: _probe_patents(e),
    "litigation_data": lambda e: _probe_litigation(e),
}
```

## Integration with Pipeline

The verification result influences downstream phases:

```python
# In the plan phase — use verified entity name
verified = load_json(task_dir / "entity_verification.json")
entity_name = verified["verified_entity"]["official_name"] or job_ctx.entity

# In presearch — use data_availability to skip empty sources
avail = verified.get("data_availability", {})
if not avail.get("patent_data"):
    # Skip patent-related presearch queries
    pass

# In stage classification — use entity type/size
entity_type = verified["verified_entity"].get("entity_type", "")
```

## When to Skip

| Scenario | Skip verification? |
|----------|-------------------|
| Entity provided by user with strong identifiers (reg number, ticker) | **Yes** — identifiers are sufficient |
| Entity is a concept/topic (not a named entity) | **Yes** — no authoritative source exists |
| Pipeline operates on user-provided documents only | **Yes** — entity existence is implied |
| Entity is ambiguous or poorly specified | **No** — verification is critical |
| Multiple entities could match | **No** — must disambiguate before proceeding |

## Design Checklist

- [ ] Entity verification runs early (Phase 02, before plan generation)
- [ ] FAIL on ambiguous entities — return clear error with candidates for user
- [ ] FAIL on not_found — stop pipeline before wasting resources
- [ ] WARN on source unavailable — proceed with caution
- [ ] Verification result includes data_availability assessment
- [ ] Downstream phases use verified entity name (not raw user input)
- [ ] Verification sources are configurable per domain
