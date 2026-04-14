Below is a formal v1 **Scoring Algorithm Specification** for the game. It is designed to be deterministic, reviewable, and implementable without LLMs in the grading path.

---

# Scoring Algorithm Specification

## 1. Goals

The scoring system must:

* reward correct error detection
* reward precise span selection
* reward correct error classification
* reward correct correction text
* penalize missed errors
* penalize false positives
* discourage overly broad highlighting
* produce a fair, explainable payout

This spec assumes:

* **1 annotation = 1 mistake**
* each annotation may match **at most one** ground-truth error
* each ground-truth error may match **at most one** annotation
* all grading is performed server-side

---

## 2. Terminology

### 2.1 Ground-truth error

A predefined error in a job.

Fields used in scoring:

* `error_id`
* `error_type_id`
* `span_start`
* `span_end`
* `accepted_corrections[]`

### 2.2 Player annotation

A submitted mark by the player.

Fields used in scoring:

* `annotation_id`
* `span_start`
* `span_end`
* `error_type_id`
* `correction_text`

### 2.3 Span convention

All spans use:

* zero-based indexing
* `start` inclusive
* `end` exclusive

So span length is:

`length = end - start`

---

## 3. High-level grading flow

For a submitted attempt:

1. validate annotations
2. normalize annotations
3. match annotations to ground-truth errors
4. compute component scores for each matched pair
5. mark unmatched annotations as false positives
6. mark unmatched errors as missed
7. aggregate component scores and penalties
8. compute final payout
9. persist detailed results

---

## 4. Validation rules

Before grading begins:

### 4.1 Annotation validity

Each annotation must satisfy:

* `span_start >= 0`
* `span_end > span_start`
* `span_end <= flawed_text.length`
* `error_type_id` present
* `correction_text` present, though it may be empty for some future error classes

For v1, reject invalid annotations at submit time with `VALIDATION_ERROR`.

### 4.2 Duplicate annotation handling

Two annotations are considered duplicate-like if they have:

* identical span
* identical error type
* identical correction text after normalization

Server behavior:

* keep both if submitted
* score independently
* duplicates will typically lead to one match and one false positive

This is simpler and discourages spam.

---

## 5. Normalization rules

Normalization is applied before correction matching.

## 5.1 Correction text normalization

Define:

`normalizeCorrection(s)`

Steps:

1. Unicode normalize to NFC
2. trim leading/trailing whitespace
3. collapse internal repeated ASCII spaces to one space
4. normalize common full-width punctuation to canonical form if desired
5. do not change character script
6. do not lowercase Chinese text

For v1, do **not** automatically convert:

* simplified ↔ traditional
* synonyms
* paraphrases

Those must be explicitly included in `accepted_corrections`.

---

## 6. Matching rules

## 6.1 Matching objective

Find the best one-to-one mapping between player annotations and ground-truth errors.

Constraints:

* one annotation can match at most one error
* one error can match at most one annotation

## 6.2 Candidate pair generation

An annotation is a candidate for an error only if their spans overlap.

Define overlap length:

`overlap = max(0, min(a.end, e.end) - max(a.start, e.start))`

If `overlap == 0`, the pair is not a candidate.

---

## 6.3 Pair ranking priority

If multiple candidate matches exist, select the best pair using this priority order:

1. highest span tier
2. exact error type match preferred
3. exact correction match preferred
4. greater overlap length
5. smaller extra coverage
6. lower annotation sequence index
7. lower error sequence index

This avoids unstable matches.

---

## 6.4 Recommended matching strategy

Use a deterministic greedy matcher over ranked candidate pairs.

Process:

1. generate all candidate pairs
2. compute preliminary pair features
3. sort descending by ranking priority
4. iterate through sorted pairs
5. match pair if both annotation and error are still unmatched

This is sufficient for v1 because jobs are small.

---

# 7. Span scoring

Span scoring should reward precise selection and penalize broad selection.

## 7.1 Definitions

For annotation span `A = [a1, a2)` and error span `E = [e1, e2)`:

* `lenA = a2 - a1`
* `lenE = e2 - e1`
* `overlap = max(0, min(a2, e2) - max(a1, e1))`

Define:

* `coverage_ratio = overlap / lenE`
* `precision_ratio = overlap / lenA`
* `extra_length = lenA - overlap`

Interpretation:

* `coverage_ratio` measures how much of the true error was captured
* `precision_ratio` measures how focused the selection was

---

## 7.2 Basic matching threshold

A matched pair requires:

* `overlap > 0`
* `coverage_ratio >= 0.5`

If `coverage_ratio < 0.5`, the annotation cannot match that error and is treated as unmatched for that error.

This prevents tiny accidental overlap from counting.

---

## 7.3 Span tiers

For matched pairs, assign one of four span tiers.

### Tier 1: Perfect

Conditions:

* `coverage_ratio = 1.0`
* `precision_ratio >= 0.8`

Span credit:

* `1.00`

Interpretation:

* player covered the full error
* selection is exact or tightly focused

---

### Tier 2: Slightly Wide

Conditions:

* `coverage_ratio = 1.0`
* `0.5 <= precision_ratio < 0.8`

Span credit:

* `0.80`

Interpretation:

* player selected the whole error plus some nearby text
* still acceptable but imprecise

---

### Tier 3: Very Wide

Conditions:

* `coverage_ratio = 1.0`
* `0.25 <= precision_ratio < 0.5`

Span credit:

* `0.40`

Interpretation:

* player found the right area but highlighted far too much

---

### Tier 4: Excessive

Conditions:

* `coverage_ratio = 1.0`
* `precision_ratio < 0.25`

Span credit:

* `0.00`

Interpretation:

* selection is so broad that it should not receive span credit

---

## 7.4 Partial coverage tier

If:

* `0.5 <= coverage_ratio < 1.0`

Then:

* assign `span_credit = 0.50`

This covers cases where the player selected part of the real error but not all of it.

For v1, partial coverage should still allow a match if overlap is meaningful.

---

## 7.5 No-match span case

If:

* `coverage_ratio < 0.5`

Then:

* pair is invalid
* annotation does not match that error

---

# 8. Error type scoring

Once an annotation is matched to an error:

* if `annotation.error_type_id == error.error_type_id`

  * `error_type_credit = 1.00`
* else

  * `error_type_credit = 0.00`

For v1, use strict type matching.
No partial similarity between categories.

---

# 9. Correction scoring

## 9.1 Accepted corrections

A correction is correct if the normalized annotation correction exactly equals any normalized accepted correction for the matched error.

Define:

`normalized_player_correction in normalized_accepted_corrections`

If true:

* `correction_credit = 1.00`

Else:

* `correction_credit = 0.00`

---

## 9.2 Empty correction handling

If player leaves correction blank:

* `correction_credit = 0.00`

Even if they found the right error.

---

## 9.3 Future extensibility

Later versions may add:

* structured replacement operations
* synonym sets
* script normalization
* partial correction credit

But not in v1.

---

# 10. Detection scoring

Detection is binary per ground-truth error.

If an error is matched by an annotation:

* `detection_credit = 1.00`

If not matched:

* `detection_credit = 0.00`

---

# 11. Component weights

Each matched error produces a weighted score.

Recommended v1 weights:

* Detection: `0.35`
* Span: `0.15`
* Error type: `0.20`
* Correction: `0.30`

Weights sum to `1.00`.

## 11.1 Per-error raw score

For a matched error:

`error_raw_score =`
`0.35 * detection_credit +`
`0.15 * span_credit +`
`0.20 * error_type_credit +`
`0.30 * correction_credit`

Since matched errors always have `detection_credit = 1.00`, players get meaningful credit for finding the issue even if they classify or correct it poorly.

---

## 11.2 Missed error raw score

For an unmatched error:

* `detection_credit = 0`
* `span_credit = 0`
* `error_type_credit = 0`
* `correction_credit = 0`

So:

`error_raw_score = 0`

---

# 12. False positives

A false positive is any annotation that is not matched to a ground-truth error.

Each false positive incurs a penalty.

Recommended v1 false-positive penalty:

* `false_positive_penalty_minor = 0.15 * base_fee_minor / error_count`

Round using standard currency rounding rules.

To keep it stable, compute:

`unit_penalty_minor = ceil(base_fee_minor / max(1, error_count) * 0.15)`

Each false positive subtracts `unit_penalty_minor`.

---

# 13. Missed error penalties

Each unmatched ground-truth error incurs a penalty.

Recommended v1 missed-error penalty:

* `missed_error_penalty_minor = 0.25 * base_fee_minor / error_count`

Compute:

`missed_unit_penalty_minor = ceil(base_fee_minor / max(1, error_count) * 0.25)`

Each missed error subtracts `missed_unit_penalty_minor`.

Missed errors should hurt more than false positives.

---

# 14. Incorrect matched annotation penalties

For matched annotations, do not apply separate penalties for:

* wrong type
* wrong correction
* broad span

Instead, handle these through reduced credit.

This keeps the system easier to explain.

So v1 penalties are only:

* missed errors
* false positives

And quality differences are expressed through reduced earned score.

---

# 15. Converting score to payout

## 15.1 Aggregate raw performance

Let:

* `N = number of ground-truth errors`

Compute:

`raw_performance_score = sum(error_raw_score for all errors) / N`

This yields a score from `0.0` to `1.0`.

---

## 15.2 Gross earned amount

Compute:

`gross_earned_minor = round(base_fee_minor * raw_performance_score)`

This means good performance earns more of the fee.

---

## 15.3 Total penalties

Compute:

`total_penalties_minor =`
`(missed_errors * missed_unit_penalty_minor) +`
`(false_positives * false_positive_unit_penalty_minor)`

---

## 15.4 Final payout

Compute:

`final_payout_minor = gross_earned_minor - total_penalties_minor`

Clamp to a minimum floor.

Recommended v1 floor:

* `min_payout_minor = 0`

So:

`final_payout_minor = max(0, final_payout_minor)`

This avoids negative money from a single job unless you explicitly want debt mechanics later.

---

# 16. Accuracy metric

Leaderboards need a simple accuracy metric.

Recommended v1:

`accuracy_ratio = raw_performance_score`

This keeps it easy and aligned with error mastery.

Do not subtract false-positive penalties from `accuracy_ratio`.
Those should affect payout more than mastery display.

If you want a stricter metric later, add a separate `review_precision_ratio`.

---

# 17. Result classifications

Each ground-truth error should receive one of these statuses:

* `corrected`
* `partially_corrected`
* `missed`

Recommended rule:

### corrected

If matched and:

* `error_type_credit = 1`
* `correction_credit = 1`
* `span_credit >= 0.8`

### partially_corrected

If matched but not `corrected`

### missed

If unmatched

---

# 18. Worked example

Assume:

* `base_fee_minor = 200`
* `error_count = 4`

Then:

* `missed_unit_penalty_minor = ceil(200 / 4 * 0.25) = ceil(12.5) = 13`
* `false_positive_unit_penalty_minor = ceil(200 / 4 * 0.15) = ceil(7.5) = 8`

Player result:

* 2 errors fully corrected
* 1 error matched but wide span and wrong correction
* 1 error missed
* 1 false positive

## Error scores

### Error 1: fully correct

* detection = 1
* span = 1
* type = 1
* correction = 1

Score:
`0.35 + 0.15 + 0.20 + 0.30 = 1.00`

### Error 2: fully correct

Score:
`1.00`

### Error 3: matched, wide, wrong correction, right type

* detection = 1
* span = 0.4
* type = 1
* correction = 0

Score:
`0.35 + 0.06 + 0.20 + 0 = 0.61`

### Error 4: missed

Score:
`0`

## Aggregate

`raw_performance_score = (1 + 1 + 0.61 + 0) / 4 = 0.6525`

`gross_earned_minor = round(200 * 0.6525) = 131`

Penalties:

* missed errors = `1 * 13 = 13`
* false positives = `1 * 8 = 8`

`total_penalties = 21`

`final_payout = 131 - 21 = 110`

`accuracy_ratio = 0.6525`

---

# 19. Matching edge cases

## 19.1 Two annotations overlap the same error

Only one annotation may match that error.

Preferred match:

* higher span tier
* then correct type
* then correct correction
* then more overlap
* then smaller extra coverage
* then earlier annotation

The losing annotation becomes a false positive unless it validly matches another error.

---

## 19.2 One annotation overlaps two nearby errors

For v1, one annotation may match only one error.

Choose the best valid candidate using ranking priority.

The other error remains missed unless separately annotated.

This enforces the “1 annotation = 1 mistake” rule.

---

## 19.3 Annotation covers entire sentence

Usually:

* coverage ratio may be 1
* precision ratio will be very low
* span tier becomes `Excessive`
* span credit = 0

It may still match and get some score if it clearly captures one error, but with poor payout value.

If this becomes too forgiving in testing, add a hard guard:

* if `lenA > 4 * lenE` and `lenA >= 8`, disallow match

I would keep this as optional, not default, for v1.

---

## 19.4 Wrong type but correct correction

Allowed.

Score reflects:

* credit for finding and fixing
* no type credit

This is desirable pedagogically.

---

## 19.5 Correct type but wrong correction

Allowed.

Score reflects:

* credit for diagnosis
* no correction credit

---

## 19.6 Partial span with correct replacement

Allowed if `coverage_ratio >= 0.5`.

Player receives:

* detection credit
* reduced span credit
* type/correction credit as applicable

---

# 20. Suggested server-side output fields

For each matched annotation result, store:

* `matched_error_id`
* `span_overlap`
* `coverage_ratio`
* `precision_ratio`
* `span_tier`
* `detection_credit`
* `span_credit`
* `error_type_credit`
* `correction_credit`
* `total_credit`

For each error result, store:

* `status`
* `matched_annotation_id`
* same component credits
* explanation payload

This makes review and tuning much easier.

---

# 21. Tuning knobs

These are the safest values to adjust after playtesting:

## 21.1 Weight tuning

Default:

* detection 0.35
* span 0.15
* type 0.20
* correction 0.30

Possible future adjustments:

* increase correction weight for advanced ranks
* decrease type weight if taxonomy proves too hard early on

## 21.2 Span tiers

Default thresholds:

* perfect: precision >= 0.8
* slightly wide: 0.5 to <0.8
* very wide: 0.25 to <0.5
* excessive: <0.25
* partial coverage allowed at coverage >= 0.5

## 21.3 Penalty rates

Default:

* missed error: 25% of per-error fee share
* false positive: 15% of per-error fee share

These should be tuned based on whether the game feels too punitive.

---

# 22. Recommended v1 defaults

I recommend shipping v1 with exactly these defaults:

### Matching

* overlap required
* coverage ratio must be at least 0.5
* one-to-one greedy ranked matching

### Span tiers

* perfect: full coverage and precision >= 0.8 → 1.00
* slightly wide: full coverage and precision >= 0.5 → 0.80
* very wide: full coverage and precision >= 0.25 → 0.40
* excessive: full coverage and precision < 0.25 → 0.00
* partial coverage: coverage >= 0.5 but < 1.0 → 0.50

### Weights

* detection 0.35
* span 0.15
* error type 0.20
* correction 0.30

### Penalties

* missed error: 25% of per-error fee share
* false positive: 15% of per-error fee share
* final payout floor: 0

---

# 23. Pseudocode

```text
gradeAttempt(attempt, job):
    errors = job.errors
    annotations = normalize(attempt.annotations)

    candidatePairs = []

    for ann in annotations:
        for err in errors:
            overlap = computeOverlap(ann.span, err.span)
            if overlap == 0:
                continue

            coverage = overlap / err.length
            if coverage < 0.5:
                continue

            precision = overlap / ann.length
            spanTier, spanCredit = classifySpan(coverage, precision)
            typeMatch = (ann.error_type_id == err.error_type_id)
            correctionMatch = correctionMatches(ann.correction_text, err.accepted_corrections)

            candidatePairs.append({
                ann_id,
                err_id,
                spanTier,
                spanCredit,
                typeMatch,
                correctionMatch,
                overlap,
                extraLength = ann.length - overlap
            })

    sort candidatePairs by:
        spanTier descending
        typeMatch descending
        correctionMatch descending
        overlap descending
        extraLength ascending
        annotationSequence ascending
        errorSequence ascending

    matchedAnnotations = set()
    matchedErrors = set()
    matches = []

    for pair in candidatePairs:
        if pair.ann_id in matchedAnnotations:
            continue
        if pair.err_id in matchedErrors:
            continue
        matches.append(pair)
        matchedAnnotations.add(pair.ann_id)
        matchedErrors.add(pair.err_id)

    annotationResults = []
    errorResults = []
    rawScoreSum = 0

    for err in errors:
        pair = findMatchForError(err.id)

        if no pair:
            errorResults.add(missed result)
            continue

        detection = 1.0
        span = pair.spanCredit
        typeCredit = 1.0 if pair.typeMatch else 0.0
        correctionCredit = 1.0 if pair.correctionMatch else 0.0

        totalCredit =
            0.35 * detection +
            0.15 * span +
            0.20 * typeCredit +
            0.30 * correctionCredit

        rawScoreSum += totalCredit
        errorResults.add(...)
        annotationResults.add(...)

    falsePositives = annotations not in matchedAnnotations
    missedErrors = errors not in matchedErrors

    rawPerformance = rawScoreSum / len(errors)
    grossEarned = round(job.base_fee_minor * rawPerformance)

    missedPenalty = ceil(job.base_fee_minor / len(errors) * 0.25) * count(missedErrors)
    falsePositivePenalty = ceil(job.base_fee_minor / len(errors) * 0.15) * count(falsePositives)

    finalPayout = max(0, grossEarned - missedPenalty - falsePositivePenalty)

    return graded result
```

---

# 24. One implementation note worth considering

A very practical refinement for early-user fairness is to log, but not expose initially, these values for every match:

* overlap
* coverage ratio
* precision ratio
* extra length
* selected text
* expected text
* accepted corrections

That will make it much easier to debug player complaints and tune thresholds after launch.

The next logical step is a compact **test-case matrix** for engineering and QA, so the team can verify the grader against representative edge cases before implementation.
