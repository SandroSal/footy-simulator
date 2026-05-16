# Stage-1 Classifier Evaluation Methodology

## What this eval validates

Per §3 of the GDD, the classifier's reliability is the load-bearing prototype risk: if a cheap-tier model can't reliably route player utterances, the §10.5 cost architecture (cheap routing for the common case, expensive pipeline for novel input) collapses.

This eval measures two coupled properties:

1. **Routing accuracy** — given a labeled test set, does the classifier produce the correct `type`, `intent_or_domain`, and `officer`?
2. **Same-instruction consistency** — given the same utterance issued multiple times, does the classifier produce the same classification?

## Eval procedure

### Inputs

- `prompt_stage1.md` — the classifier prompt being evaluated.
- `test_set.csv` — labeled test utterances.
- Model: the cheap-tier model being validated (Anthropic Haiku-class, OpenAI small models, or equivalent). Pick one and stick with it for a given eval run.

### Procedure

1. For each row in `test_set.csv`, format the prompt by substituting `{officers}`, `{active_events}`, and `{utterance}`. For the prototype eval, use a fixed context:
   - Officers: `DoF: Alex Kim`, `CD: Sam Reyes`
   - Active events: `(none)` for most rows. For `crisis_response` rows specifically, include a placeholder crisis event so the routing has a valid target.
2. Call the model with the formatted prompt.
3. Parse the JSON response.
4. Compare against the row's expected fields.

### Scoring

Per-row outcomes:

- **Correct** — all three of `type`, `intent_or_domain`, `officer` match.
- **Type-correct, sub-incorrect** — `type` matches but `intent_or_domain` or `officer` is wrong. (This separates routing-tier failures from within-tier failures.)
- **Type-incorrect** — `type` is wrong. (Most serious failure — wrong system gets invoked.)
- **Parse-failure** — JSON couldn't be parsed, or fields missing.

### Aggregate metrics

Per category (recognized / novel / out_of_scope):

- Overall accuracy (correct / total)
- Type accuracy (type-correct / total)
- Sub-accuracy among type-correct (correct / type-correct)

Per intent and per domain (finer-grained):

- Accuracy
- Confusion: when this category is wrong, what does it get classified as?

Confusion matrix:

- Rows = ground-truth `intent_or_domain`
- Columns = predicted `intent_or_domain`
- Cells = counts

Same-instruction consistency:

- For consistency probes (`c###` rows): run each probe N times (N=5 recommended for prototype).
- Score: % of runs where the classification matches the modal classification for that probe.

## Passing the prototype-validation bar

The eval informs the §3 §11 "Classifier reliability and stability" open question. Specifically:

**Floor for the prototype to be worth building**: ≥80% overall accuracy, ≥90% type accuracy on recognized intents (the cheap path must reliably handle the cheap path; misrouting a recognized intent to novel doubles the cost on that interaction).

**Healthy signal**: ≥90% overall accuracy, ≥95% type accuracy on recognized.

**Investigate**: type accuracy on out_of_scope below 75% (means the classifier is over-eager to route into a domain), or novel-domain accuracy below 70% (means the domain taxonomy isn't crisp enough).

These thresholds are first-pass targets, not commitments. Adjust after the first eval run produces baseline data.

## When the eval fails

Misclassifications fall into categories. Each suggests a different fix:

- **Recognized → novel** (false negative on recognized): the recognized-intent list isn't covering phrasing variety. Fix in the prompt by expanding intent descriptions or examples.
- **Novel → recognized** (false positive on recognized): the recognized-intent descriptions are too broad. Tighten them.
- **Anything → out_of_scope** (under-routing): prompt is too conservative about claiming a domain. Loosen the domain descriptions or add a soft default.
- **Out_of_scope → anything** (over-routing): the domain descriptions are too broad and absorbing things that shouldn't fit. Tighten domain boundaries.
- **Cross-domain confusion** (e.g., brand_and_marketing → fan_engagement): the two domains are too close. Either disambiguate in the prompt or accept that they're a single domain.

## Iterating

The expected flow is:

1. Run eval against the initial prompt.
2. Identify the worst-performing categories and confusion patterns.
3. Adjust the prompt (or, less often, the test set if labels were wrong).
4. Re-run eval.
5. Repeat until thresholds are met or it becomes clear they can't be at the chosen model tier.

If thresholds can't be met at cheap-tier, that's a finding worth having — it informs whether the §10.5 architecture needs revisiting (e.g., cascade to mid-tier on low-confidence; route everything to mid-tier; abandon two-stage for single-stage).

## Out of scope for this eval

- Parameter extraction quality (stage 2).
- Officer dialogue quality (downstream of intent classification).
- End-to-end task resolution (§2 outcome math validation is a separate eval).
- Cost (measured separately via §10.5 telemetry; this eval cares about correctness, not token spend).