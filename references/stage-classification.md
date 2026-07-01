# Stage / Weight Classification System

## Problem

Not all pipeline runs are equal. A lightweight 3-role analysis of an early-stage startup needs different rigor than a 10-role deep dive into a public company. Without a classification system:

- Every run uses the same heavy gates → overkill for simple cases
- Early-stage projects get blocked by requirements they can't meet (e.g., audited financials)
- Risk severity doesn't adapt to context (a "no revenue" finding is critical for a mature company but irrelevant for a seed-stage startup)
- Sub-agents use the same analysis framework regardless of project maturity

## Solution: Stage-Aware Pipeline Behavior

Classify each pipeline run into a **stage tier**, then use that tier to adjust:
1. Which waves to run (skip unnecessary ones)
2. Gate severity thresholds (relax for lightweight projects)
3. Risk severity classification (context-dependent risk overrides)
4. Sub-agent analysis framework (stage-appropriate guidance in prompts)
5. Shared state recommendation logic (stage-adjusted verdicts)

## Architecture

```
Input Document → Stage Classifier → stage_tier (e.g., "T1", "lightweight", "full")
                                          ↓
                          ┌────────────────┼────────────────┐
                          ↓                ↓                ↓
                   Gate Thresholds   Risk Overrides   Prompt Blocks
                   (relax for T1)    (context-aware)  (injected into sub-agents)
```

## Stage Tiers (Example Set)

Define tiers that make sense for your domain. Here are two example schemas:

### Investment Due Diligence Tiers

| Tier | Label | Typical characteristics |
|------|-------|------------------------|
| T1 | Seed/Angel | Pre-revenue, team-focused, tech differentiation critical |
| T2 | Pre-A/A | Early revenue, PMF signals, no audit required |
| T3 | B+ round | Scaled revenue, audited financials, full DD |
| T4 | Pre-IPO | Public-ready, regulatory compliance, comprehensive |

### Generic Project Weight Tiers

| Tier | Label | Typical characteristics |
|------|-------|------------------------|
| Light | Quick scan | 1-2 roles, broad overview, minimal gates |
| Standard | Normal analysis | 3-5 roles, full gates, standard repair |
| Deep | Comprehensive | 6+ roles, strict gates, multi-round repair |

## Implementation

### Step 1: Stage Classifier

```python
def classify_stage(input_data: dict) -> str:
    """Classify the pipeline run into a stage tier.

    Customize the classification logic for your domain.
    Common signals:
    - Document metadata (financing stage, company age, revenue range)
    - User-specified tier (explicit override)
    - Heuristics from input document content
    """
    # Example: classify by financing stage keyword
    stage_text = input_data.get("financing_stage", "").lower()

    if any(kw in stage_text for kw in ["seed", "angel", "pre-seed"]):
        return "T1"
    elif any(kw in stage_text for kw in ["pre-a", "a round", "series a"]):
        return "T2"
    elif any(kw in stage_text for kw in ["b round", "series b", "growth"]):
        return "T3"
    else:
        return "T4"  # Default to full rigor
```

### Step 2: Stage Metadata

Each tier has a metadata dict that controls pipeline behavior:

```python
STAGE_META = {
    "T1": {
        "label": "Early Stage (Seed/Angel)",

        # Gate adjustments
        "gate_relaxation": {
            "quality_gate_max_retries": 0,     # No repair — just degrade
            "coverage_threshold": 0.3,          # Only 30% coverage needed (vs 60% for T3)
            "skip_waves": [2],                  # Skip Wave 2 (narrow validation)
        },

        # Risk severity overrides
        # Some risks are irrelevant for early-stage projects
        "risk_severity_override": {
            "no_revenue_data": "low",           # Expected for seed stage
            "no_audit_report": "low",           # Not required yet
            "team_unverified": "critical",       # Team IS critical for seed
            "tech_not_differentiated": "high",  # Tech matters most
        },

        # Sub-agent prompt guidance
        "analysis_focus": [
            "Team background verification",
            "Technology differentiation",
            "Early customer signals (if any)",
            "Market opportunity sizing",
        ],
        "de_emphasize": [
            "Revenue analysis (may not exist)",
            "Financial audit (not applicable)",
            "Regulatory compliance (minimal at this stage)",
        ],

        # Recommendation adjustment
        "verdict_adjustment": {
            "undecided_upgraded_to": "conditional",  # More lenient
        },
    },

    "T3": {
        "label": "Growth Stage (B+ Round)",

        "gate_relaxation": {},  # Full enforcement

        "risk_severity_override": {
            "no_revenue_data": "critical",      # Must have revenue at B round
            "no_audit_report": "high",          # Expected at this stage
        },

        "analysis_focus": [
            "Revenue growth trajectory",
            "Customer concentration risk",
            "Competitive positioning",
            "Unit economics",
        ],
        "de_emphasize": [],

        "verdict_adjustment": {},  # No adjustment
    },
}

def get_stage_meta(tier: str) -> dict:
    """Get metadata for a stage tier. Returns default if tier not found."""
    return STAGE_META.get(tier, STAGE_META.get("T3", {}))
```

### Step 3: Prompt Block Injection

Generate a text block that gets appended to sub-agent system prompts:

```python
def build_stage_prompt_block(tier: str, entity: str = "") -> str:
    """Generate a stage-aware text block for sub-agent prompts.

    This tells sub-agents how to adjust their analysis framework
    based on the project's stage/maturity level.
    """
    meta = get_stage_meta(tier)
    entity_prefix = f"For {entity}, " if entity else ""

    lines = [
        "## Stage-Aware Analysis Guidance",
        "",
        f"{entity_prefix}This project is classified as **{meta['label']}**.",
        "Adjust your analysis framework accordingly.",
        "",
    ]

    # What to focus on
    if meta.get("analysis_focus"):
        lines.append("### Priority Focus Areas")
        lines.append("")
        for focus in meta["analysis_focus"]:
            lines.append(f"- {focus}")
        lines.append("")

    # What to de-emphasize
    if meta.get("de_emphasize"):
        lines.append("### De-Emphasize (not applicable at this stage)")
        lines.append("")
        for item in meta["de_emphasize"]:
            lines.append(f"- {item} — flag as data gap but do NOT penalize")
        lines.append("")

    # Gate behavior
    relaxation = meta.get("gate_relaxation", {})
    if relaxation.get("skip_waves"):
        lines.append(f"**Note**: Some analysis waves are skipped for this stage.")
    if relaxation.get("coverage_threshold"):
        lines.append(f"**Coverage threshold**: {relaxation['coverage_threshold']*100:.0f}% (relaxed from standard)")

    return "\n".join(lines)
```

### Step 4: Gate Threshold Adjustment

Quality gates read the stage tier and adjust their behavior:

```python
def evaluate_quality_gate(task_dir, wave, outputs_dir):
    """Evaluate sub-agent output quality, adjusted for stage tier."""

    # Load stage tier
    stage_tier = _load_stage_tier(task_dir)
    meta = get_stage_meta(stage_tier)
    relaxation = meta.get("gate_relaxation", {})

    # Adjust thresholds
    coverage_threshold = relaxation.get("coverage_threshold", 0.6)  # Default 60%
    skip_repair = relaxation.get("quality_gate_max_retries", 1) == 0

    # ... evaluate gate ...

    # If stage says skip repair, degrade directly
    if gate_failed and skip_repair:
        return {
            "ok": True,  # Don't block
            "needs_repair": False,
            "verdict": "WARN",
            "stage_adjusted": True,
            "reason": f"Stage {stage_tier}: repair skipped, issues degraded to WARN",
        }
```

### Step 5: Risk Severity Override

When building the shared state, risk severity is adjusted by stage:

```python
def _extract_risks(packages, stage_meta):
    """Extract risks from sub-agent outputs, with stage-aware severity."""

    overrides = stage_meta.get("risk_severity_override", {})

    for risk in raw_risks:
        # Check if this risk type has a stage-specific override
        risk_type = risk.get("risk_type", "")
        if risk_type in overrides:
            risk["severity"] = overrides[risk_type]
            risk["severity_overridden_by"] = f"stage_{stage_tier}"

    return raw_risks
```

## Integration Points

| Where | How stage tier is used |
|-------|----------------------|
| **Profile init** | `classify_stage(input_data)` → stored in job context |
| **Wave prepare** | Skip waves listed in `gate_relaxation.skip_waves` |
| **Quality gates** | Adjust thresholds per `gate_relaxation` |
| **Sub-agent prompts** | Append `build_stage_prompt_block(tier)` |
| **Shared state** | Risk severity overridden by `risk_severity_override` |
| **Recommendation** | Verdict adjusted by `verdict_adjustment` |
| **Delivery gate** | Stage-specific deferred_fixes messaging |

## Persisting the Stage Tier

```python
# In job initialization or intake phase:
stage_tier = classify_stage(input_data)

# Store in a persistent file so all phases can read it:
stage_file = task_dir / "stage_classification.json"
stage_file.write_text(json.dumps({
    "stage_tier": stage_tier,
    "classification_reason": "...",
    "classified_at": timestamp_now(),
}, indent=2))

# In any phase handler:
def _load_stage_tier(task_dir) -> str:
    path = task_dir / "stage_classification.json"
    if path.exists():
        return json.loads(path.read_text()).get("stage_tier", "default")
    return "default"
```

## Design Checklist

- [ ] Define stage tiers for your domain (2-4 tiers is typical)
- [ ] Define `STAGE_META` with gate relaxation, risk overrides, analysis focus
- [ ] Implement `classify_stage()` — from input document or user override
- [ ] Implement `build_stage_prompt_block()` — text block for sub-agent prompts
- [ ] Integrate into quality gates — read stage tier, adjust thresholds
- [ ] Integrate into shared state — risk severity override
- [ ] Persist stage classification to disk for cross-phase access
- [ ] Allow user override of automatic classification
