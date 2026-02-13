# Code Review: Auto + Manual Stratification Tools

## Scope
This review covers:
- `Auto Stratifcation`
- `Manual Stratification`

Focus areas:
1. Computational robustness for small, medium, and large project areas.
2. Statistical/method best-practice alignment for practitioner use.
3. Plain-language flow and usability.

## Executive summary
Both tools have a strong foundation (clear workflow, reproducible seed strategy, defensible UNFCCC/IPCC framing). The main technical risk was allocation stability when the number of strata is high relative to total target plots. That risk has now been addressed with bounded and minimum-feasible allocation logic in both tools.

For practitioners, the method flow is mostly sound, but can still be improved with simpler wording in some UI messages and stronger guardrails around interpretability (e.g., warning users when defaults are doing most of the work).

## What was improved in this pass

### 1) Robust allocation under many-strata/small-sample conditions
In both tools, proportional allocation now:
- clamps the total sample size to configured min/max bounds,
- guarantees feasibility by enforcing at least `MIN_SAMPLES_PER_STRATUM * number_of_strata`,
- applies minimum-per-stratum first,
- then reconciles the total using a stable largest-remainder approach that never indexes out of range.

Why it matters:
- **Small areas / many strata:** prevents under-allocation and unstable balancing.
- **Large areas:** prevents excessive plot recommendations beyond configured cap.

### 2) Seed consistency in auto stratification ML path
The auto tool referenced `CONFIG.RANDOM_SEED` in several ML sampling/training calls, but the configuration defines `DEFAULT_SEED`. This now consistently uses `DEFAULT_SEED`.

Why it matters:
- avoids accidental non-determinism,
- keeps reproducibility aligned with documented behavior.

## Detailed review findings

## A. Computational robustness by area size

### Small areas (few hectares to low hundreds)
Strengths:
- Minimum sample constraints are present.
- AOI buffering and minimum geometry checks reduce invalid point generation.

Risks and recommendations:
- If users choose very small custom plot sizes, the implied population size can be unrealistically high relative to operational capacity. Add a plain-language warning: “This setup implies many possible plots; consider a larger plot size or larger error tolerance.”
- For tiny strata, random point placement can repeatedly fail after buffering. Consider adaptive buffer reduction (e.g., retry with 50 m, then 25 m, then 10 m).

### Medium areas (typical project scale)
Strengths:
- Current defaults are appropriate for most operational use.
- Tier defaults + iterative t-based estimation is practical.

Risks and recommendations:
- Add a short “Assumptions summary” panel after calculation with three bullets:
  - confidence level used,
  - margin of error used,
  - whether Tier 1 defaults or user priors drove variance.

### Large areas (regional scale)
Strengths:
- `maxPixels` and `tileScale` are used in key reducers.
- Capped total samples avoid runaway recommendations.

Risks and recommendations:
- In auto covariate clustering, fixed `TRAINING_PIXELS` may underrepresent very large heterogeneity. Consider adaptive training size by AOI area (with upper bound).
- Add user-facing runtime hint: “Large AOIs may take several minutes, especially ML stratification.”

## B. Statistical and methodological best-practice alignment

What is good:
- Stratified formula and iterative t-based refinement are correctly structured.
- Bayesian blending approach is transparent and documented.
- Reproducible seed strategy is present.

Recommended upgrades:
1. **Expose uncertainty provenance:** show how much of mean/std-dev came from measured values vs defaults (blend weight already exists; present it in UI).
2. **Sensitivity quick-check:** add optional “low/medium/high variance scenario” preview so practitioners can see how sample size changes.
3. **Guard against over-confidence defaults:** if user inputs very high confidence + low error + small plot size, show a gentle feasibility warning before generating points.

## C. Practitioner clarity and jargon minimization

Current status:
- Structure is step-by-step and understandable.
- Some terms remain technical (e.g., “t-distribution,” “covariates,” “largest remainder”).

Suggested wording simplifications:
- “Covariates” → “satellite predictor layers”.
- “t-value (df=...)” → “statistical correction factor”.
- “Converged” → “calculation stabilized”.

Add one-line tooltips for:
- Confidence (%): “How certain you want to be in the estimate.”
- Margin of error (%): “How close you want sample-based estimates to the true mean.”
- Plot size: “Area represented by one field sample.”

## Priority roadmap
1. **High priority (done in this pass):** robust sample allocation edge-case handling + seed consistency.
2. **High priority (next):** adaptive sampling/training settings by AOI size class.
3. **Medium priority:** UI phrasing simplification and assumptions summary.
4. **Medium priority:** scenario/sensitivity panel for practitioner decision support.

## Validation checks to run in GEE manually
1. Tiny AOI with 4+ strata and strict precision settings.
2. AOI with many small fragmented strata.
3. Large AOI (regional extent) with ML stratification and max cluster count.
4. Re-run same setup twice to verify identical output point set with same seed.

