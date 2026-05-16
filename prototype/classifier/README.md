# Classifier — Stage-1 Routing

The first artifact of the football sim prototype. Validates whether a cheap-tier LLM can reliably route player input into the three buckets defined in GDD §3:

- recognized (11 known intents)
- novel (8 supported domains)
- out_of_scope (everything else)

## Why this matters

Per GDD §3 §11 ("Classifier reliability and stability"), this is the load-bearing prototype risk. If the cheap-tier classifier doesn't work, the §10.5 cost architecture collapses and the whole novel-input-with-bounded-output system has to be re-architected.

This is the first thing the prototype validates, before any game state, UI, or end-to-end flow.

## Architecture

Two-stage classification (decision logged in GDD §12, dated 2026-05-XX):

- **Stage 1** (this directory): routing only. Outputs `type`, `intent_or_domain`, `officer`. Cheap-tier.
- **Stage 2** (not yet built): parameter extraction and target resolution for recognized intents. Cheap-tier.

Novel intents bypass stage 2 and go to the mid-tier pipeline defined in GDD §3.

Stage 2 is built after stage 1 hits the eval thresholds defined in `eval_methodology.md`.

## Files

- `prompt_stage1.md` — the stage-1 classifier prompt. Source of truth for what the cheap tier is asked to do.
- `test_set.csv` — 80 labeled utterances covering recognized intents, novel domains, out-of-scope inputs, and consistency probes.
- `eval_methodology.md` — how to run the eval, what to measure, what passes.
- `README.md` — this file.

## Running the eval

The eval script doesn't exist yet. It needs to:

1. Read `test_set.csv`.
2. For each row, format the prompt from `prompt_stage1.md` with substituted context.
3. Call the cheap-tier model.
4. Parse the JSON response.
5. Score against expected labels per `eval_methodology.md`.
6. Output per-category accuracy, a confusion matrix, and consistency-probe agreement.

Recommended stack: Python with the provider's SDK (anthropic, openai, etc.). ~100 lines.

## State

- [x] Prompt drafted
- [x] Test set scaffolded (15 of 80 rows)
- [ ] Test set complete (65 more rows to author)
- [ ] Eval script written
- [ ] First eval run
- [ ] Iteration on prompt based on confusion patterns
- [ ] Validation thresholds met (or finding: architecture revision needed)

## Cross-references to the GDD

- §3 — interaction model, recognized intents list, supported domains list, out-of-scope handling
- §3 §11 — classifier reliability open question (this eval feeds it)
- §10.5 — cost architecture (this validates the cheap-routing assumption)
- §2 — officer system (the officer field in classifier output)