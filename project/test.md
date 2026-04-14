# Test Matrix

## 1. Test organization

Use these levels:

* **Unit**: pure functions and isolated business logic
* **Integration**: API + DB + grading + ledger behavior
* **Contract**: request/response shape and validation
* **UI**: annotation UX, state handling, rendering
* **E2E**: user journey from login to payout
* **Content QA**: generated job correctness and data validity

Use these priorities:

* **P0**: must pass before release
* **P1**: should pass before release
* **P2**: useful but can follow later

---

# 2. Scoring engine test matrix

## 2.1 Matching and span logic

| ID     | Priority | Area      | Scenario                                             | Input Shape                                                    | Expected Result                                                    |
| ------ | -------- | --------- | ---------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| SC-001 | P0       | Matching  | Exact span match                                     | 1 annotation exactly equals 1 error span                       | Annotation matches error                                           |
| SC-002 | P0       | Matching  | Slightly wider full-coverage span                    | Annotation contains whole error plus small adjacent text       | Match, span tier = Slightly Wide, span credit = 0.80               |
| SC-003 | P0       | Matching  | Very wide full-coverage span                         | Annotation contains whole error plus substantial extra text    | Match, span tier = Very Wide, span credit = 0.40                   |
| SC-004 | P0       | Matching  | Excessive span                                       | Annotation spans most of sentence while covering one error     | Match allowed, span tier = Excessive, span credit = 0.00           |
| SC-005 | P0       | Matching  | Partial coverage >= 50%                              | Annotation covers half or more of error span only              | Match allowed, span credit = 0.50                                  |
| SC-006 | P0       | Matching  | Partial coverage < 50%                               | Annotation overlaps only small part of error                   | No match                                                           |
| SC-007 | P0       | Matching  | No overlap                                           | Annotation span does not touch any error                       | False positive                                                     |
| SC-008 | P0       | Matching  | One annotation overlaps two nearby errors            | Single annotation spans text touching two errors               | Matches only best candidate; second error remains unmatched        |
| SC-009 | P0       | Matching  | Two annotations overlap same error                   | Two annotations both qualify for same error                    | Best-ranked one matches; other becomes FP unless matched elsewhere |
| SC-010 | P1       | Matching  | Same overlap, different type correctness             | Two candidate annotations same span quality, one correct type  | Correct-type annotation wins                                       |
| SC-011 | P1       | Matching  | Same span and type, different correction correctness | Two candidate annotations same span/type, one valid correction | Valid-correction annotation wins                                   |
| SC-012 | P1       | Matching  | Tie break by earlier annotation order                | Equivalent candidates                                          | Lower sequence index wins                                          |
| SC-013 | P1       | Matching  | Tie break by earlier error order                     | Equivalent candidates                                          | Lower error sequence wins                                          |
| SC-014 | P0       | Span math | Zero-length or invalid span                          | end <= start                                                   | Validation error before grading                                    |
| SC-015 | P1       | Span math | Span exactly one Chinese character                   | Character-level error                                          | Match computed correctly                                           |
| SC-016 | P1       | Span math | Multi-character phrase error                         | Phrase-level error                                             | Length and overlap computed correctly                              |

---

## 2.2 Error type scoring

| ID     | Priority | Area       | Scenario                          | Expected Result                        |
| ------ | -------- | ---------- | --------------------------------- | -------------------------------------- |
| SC-020 | P0       | Error type | Correct type selected             | error_type_credit = 1.00               |
| SC-021 | P0       | Error type | Wrong type selected               | error_type_credit = 0.00               |
| SC-022 | P1       | Error type | Correct correction but wrong type | Gets correction credit, no type credit |
| SC-023 | P1       | Error type | Correct type but wrong correction | Gets type credit, no correction credit |

---

## 2.3 Correction matching

| ID     | Priority | Area       | Scenario                                  | Input                                                | Expected Result                      |
| ------ | -------- | ---------- | ----------------------------------------- | ---------------------------------------------------- | ------------------------------------ |
| SC-030 | P0       | Correction | Exact accepted correction                 | Player correction exactly equals accepted correction | correction_credit = 1.00             |
| SC-031 | P0       | Correction | Accepted alternate correction             | Player uses another item in accepted_corrections     | correction_credit = 1.00             |
| SC-032 | P0       | Correction | Wrong correction                          | Not in accepted list                                 | correction_credit = 0.00             |
| SC-033 | P0       | Correction | Leading/trailing whitespace               | `" 你是哪国人？ "`                                         | Normalizes and matches               |
| SC-034 | P1       | Correction | Internal repeated spaces                  | String with multiple spaces                          | Normalizes per spec                  |
| SC-035 | P1       | Correction | Unicode normalization case                | Equivalent normalized forms                          | Match succeeds                       |
| SC-036 | P1       | Correction | Full-width punctuation variant if enabled | Variant punctuation form                             | Matches if normalization supports it |
| SC-037 | P0       | Correction | Blank correction                          | Empty string                                         | correction_credit = 0.00             |
| SC-038 | P1       | Correction | Traditional form not in accepted list     | Accepted only simplified                             | No match                             |
| SC-039 | P1       | Correction | Traditional form explicitly included      | Accepted list includes both                          | Match succeeds                       |

---

## 2.4 Per-error scoring weights

| ID     | Priority | Area      | Scenario                                                      | Expected Result                              |
| ------ | -------- | --------- | ------------------------------------------------------------- | -------------------------------------------- |
| SC-040 | P0       | Weighting | Fully correct match                                           | total_credit = 1.00                          |
| SC-041 | P0       | Weighting | Correct detection only, wrong type/correction, excessive span | total_credit reflects detection-only weight  |
| SC-042 | P0       | Weighting | Correct detection + type only                                 | total_credit = 0.35 + span component + 0.20  |
| SC-043 | P0       | Weighting | Correct detection + correction only                           | type = 0, correction = 1                     |
| SC-044 | P1       | Weighting | Partial coverage case                                         | Uses span credit = 0.50                      |
| SC-045 | P1       | Weighting | Float precision stability                                     | Repeated runs yield identical rounded totals |

---

## 2.5 Misses and false positives

| ID     | Priority | Area            | Scenario                                    | Expected Result                                  |
| ------ | -------- | --------------- | ------------------------------------------- | ------------------------------------------------ |
| SC-050 | P0       | Misses          | Unmatched error                             | Counted as missed                                |
| SC-051 | P0       | False positives | Unmatched annotation                        | Counted as false positive                        |
| SC-052 | P0       | Penalty         | One missed error                            | Miss penalty applied                             |
| SC-053 | P0       | Penalty         | One false positive                          | FP penalty applied                               |
| SC-054 | P0       | Penalty         | Missed error costs more than false positive | Penalty values reflect spec                      |
| SC-055 | P1       | Penalty         | No negative payout below zero               | final_payout clamped to 0                        |
| SC-056 | P1       | Penalty         | All wrong annotations, no true matches      | Payout = 0 or low per formula, never below floor |

---

## 2.6 Aggregate scoring and payout

| ID     | Priority | Area      | Scenario                               | Expected Result                                                                |
| ------ | -------- | --------- | -------------------------------------- | ------------------------------------------------------------------------------ |
| SC-060 | P0       | Aggregate | Mix of full, partial, missed, FP       | raw_performance, penalties, payout match expected fixture                      |
| SC-061 | P0       | Aggregate | Perfect run                            | accuracy_ratio = 1.0, full payout                                              |
| SC-062 | P0       | Aggregate | Zero-correct run                       | accuracy_ratio = 0, payout floor applies                                       |
| SC-063 | P0       | Aggregate | Single-error job                       | Penalty math stable, no division issues                                        |
| SC-064 | P1       | Aggregate | Large error count job                  | Consistent rounding and aggregation                                            |
| SC-065 | P1       | Aggregate | Rounding edge                          | base_fee and error_count produce non-integer penalty shares; rounded correctly |
| SC-066 | P1       | Aggregate | Idempotent re-grade of same submission | Same result each time                                                          |

---

# 3. Schema and persistence test matrix

## 3.1 Database constraints and integrity

| ID     | Priority | Area        | Scenario                                      | Expected Result              |
| ------ | -------- | ----------- | --------------------------------------------- | ---------------------------- |
| DB-001 | P0       | users       | Create valid user                             | Row inserted                 |
| DB-002 | P0       | users       | Duplicate username                            | Unique constraint failure    |
| DB-003 | P0       | jobs        | Insert playable job with required fields      | Row inserted                 |
| DB-004 | P0       | job_errors  | Error references valid job and type           | FK enforced                  |
| DB-005 | P0       | attempts    | Attempt references valid user/job             | FK enforced                  |
| DB-006 | P0       | annotations | Annotation references valid attempt           | FK enforced                  |
| DB-007 | P0       | results     | Annotation result references valid annotation | FK enforced                  |
| DB-008 | P1       | ledger      | Ledger entry inserted with balance_after      | Stored correctly             |
| DB-009 | P1       | flags       | Flag references valid attempt and user        | FK enforced                  |
| DB-010 | P1       | jobs        | Retired job not served as playable            | Query filter respects status |

---

## 3.2 Transaction safety

| ID     | Priority | Area                           | Scenario                                           | Expected Result                               |
| ------ | -------- | ------------------------------ | -------------------------------------------------- | --------------------------------------------- |
| DB-020 | P0       | Submission transaction         | Grade attempt and ledger update in one transaction | Either both commit or neither                 |
| DB-021 | P0       | Double submit                  | Two near-simultaneous submits                      | Only one payout recorded                      |
| DB-022 | P0       | Retry after network timeout    | Client retries submit after server graded          | Existing result returned, no duplicate payout |
| DB-023 | P1       | Remediation payout             | Correct remediation grants refund once             | Single ledger entry only                      |
| DB-024 | P1       | Concurrent remediation submits | Same remediation submitted twice rapidly           | One refund only                               |

---

# 4. API contract test matrix

## 4.1 Auth and profile

| ID      | Priority | Endpoint           | Scenario                 | Expected Result              |
| ------- | -------- | ------------------ | ------------------------ | ---------------------------- |
| API-001 | P0       | POST /auth/signup  | Valid native signup      | 200/201 with user payload    |
| API-002 | P0       | POST /auth/signup  | Duplicate email/username | Validation or conflict error |
| API-003 | P0       | POST /auth/login   | Valid credentials        | Auth success                 |
| API-004 | P0       | POST /auth/login   | Invalid credentials      | Unauthorized                 |
| API-005 | P0       | GET /me            | Authenticated request    | Returns profile              |
| API-006 | P0       | GET /me            | Unauthenticated request  | Unauthorized                 |
| API-007 | P1       | PATCH /me/settings | Valid settings update    | Persisted                    |
| API-008 | P1       | PATCH /me/settings | Invalid setting enum     | Validation error             |

---

## 4.2 Job discovery and attempts

| ID      | Priority | Endpoint                 | Scenario                                | Expected Result                              |
| ------- | -------- | ------------------------ | --------------------------------------- | -------------------------------------------- |
| API-020 | P0       | GET /daily-contract      | Eligible user                           | Returns available jobs                       |
| API-021 | P0       | GET /daily-contract      | User below rank requirement for one job | Job returned as locked or omitted per design |
| API-022 | P1       | GET /jobs                | Pagination/cursor                       | Stable pagination                            |
| API-023 | P0       | POST /jobs/{id}/attempts | Playable job                            | Creates in-progress attempt                  |
| API-024 | P0       | POST /jobs/{id}/attempts | Locked job                              | Forbidden / JOB_LOCKED                       |
| API-025 | P0       | GET /attempts/{id}       | Owner fetches in-progress attempt       | Returns flawed_text and annotations          |
| API-026 | P0       | GET /attempts/{id}       | Non-owner fetches attempt               | Forbidden or not found                       |

---

## 4.3 Annotation save and submit

| ID      | Priority | Endpoint                       | Scenario                             | Expected Result                                      |
| ------- | -------- | ------------------------------ | ------------------------------------ | ---------------------------------------------------- |
| API-030 | P0       | PUT /attempts/{id}/annotations | Valid annotation list                | Replaces saved annotations                           |
| API-031 | P0       | PUT /attempts/{id}/annotations | Invalid span                         | Validation error                                     |
| API-032 | P0       | PUT /attempts/{id}/annotations | Missing error_type_id                | Validation error                                     |
| API-033 | P0       | POST /attempts/{id}/submit     | Valid final submission               | Returns graded result                                |
| API-034 | P0       | POST /attempts/{id}/submit     | Submit already graded attempt        | Idempotent existing result or ATTEMPT_ALREADY_GRADED |
| API-035 | P0       | POST /attempts/{id}/submit     | Submit attempt owned by another user | Forbidden                                            |
| API-036 | P1       | POST /attempts/{id}/submit     | Large annotation count abuse         | Validation/rate-limiting behavior                    |
| API-037 | P1       | PUT /attempts/{id}/annotations | Save on graded attempt               | ATTEMPT_NOT_EDITABLE                                 |

---

## 4.4 Results, remediation, flags

| ID      | Priority | Endpoint                                     | Scenario                        | Expected Result               |
| ------- | -------- | -------------------------------------------- | ------------------------------- | ----------------------------- |
| API-040 | P0       | GET /attempts/{id}/results                   | Graded attempt                  | Returns summary and details   |
| API-041 | P0       | GET /attempts/{id}/remediation               | Mistakes exist                  | Returns remediation items     |
| API-042 | P0       | POST /attempts/{id}/remediation/{rid}/submit | Correct answer                  | Refund granted                |
| API-043 | P0       | POST /attempts/{id}/remediation/{rid}/submit | Wrong answer                    | No refund                     |
| API-044 | P1       | POST /attempts/{id}/remediation/{rid}/submit | Re-submit completed remediation | Idempotent or rejected        |
| API-045 | P0       | POST /attempts/{id}/flag                     | Valid flag                      | Stored with pending status    |
| API-046 | P1       | POST /attempts/{id}/flag                     | Duplicate flags on same attempt | Allowed or deduped per policy |
| API-047 | P1       | GET /leaderboards/daily                      | Valid metric param              | Returns sorted entries        |
| API-048 | P1       | GET /leaderboards/daily                      | Invalid metric param            | Validation error              |

---

# 5. Frontend interaction test matrix

## 5.1 Text selection and annotation creation

| ID     | Priority | Area       | Scenario                           | Expected Result                                       |
| ------ | -------- | ---------- | ---------------------------------- | ----------------------------------------------------- |
| UI-001 | P0       | Selection  | Click-drag over Chinese characters | Correct span captured                                 |
| UI-002 | P0       | Selection  | Select exactly one character       | Exact offsets preserved                               |
| UI-003 | P0       | Selection  | Select phrase across punctuation   | Span preserved correctly                              |
| UI-004 | P0       | Annotation | Selection opens annotation panel   | Panel appears with type dropdown and correction input |
| UI-005 | P0       | Annotation | Save annotation                    | Annotation rendered and persisted                     |
| UI-006 | P0       | Annotation | Edit existing annotation           | Changes saved correctly                               |
| UI-007 | P0       | Annotation | Delete annotation                  | Removed locally and server-side on next save          |
| UI-008 | P1       | Selection  | Reverse-direction drag selection   | Same offsets as forward selection                     |
| UI-009 | P1       | Selection  | Double-click selection behavior    | No corrupted spans                                    |
| UI-010 | P1       | Selection  | Re-render after save               | Highlight positions remain stable                     |

---

## 5.2 Results UI

| ID     | Priority | Area    | Scenario                                   | Expected Result                                    |
| ------ | -------- | ------- | ------------------------------------------ | -------------------------------------------------- |
| UI-020 | P0       | Results | Submit attempt                             | Results screen shows summary, payout, explanations |
| UI-021 | P0       | Results | Missed errors present                      | Missed errors are shown clearly                    |
| UI-022 | P0       | Results | False positives present                    | FP feedback shown clearly                          |
| UI-023 | P1       | Results | Bilingual explanation mode                 | EN and ZH both shown                               |
| UI-024 | P1       | Results | Auto explanation mode at low rank          | English-forward display                            |
| UI-025 | P1       | Results | Auto explanation mode at HSK3–4 equivalent | Chinese-forward display                            |

---

## 5.3 Failure and recovery UX

| ID     | Priority | Area     | Scenario                          | Expected Result                                 |
| ------ | -------- | -------- | --------------------------------- | ----------------------------------------------- |
| UI-030 | P0       | Network  | Annotation save fails             | User sees recoverable error                     |
| UI-031 | P0       | Network  | Submit fails before grading       | Retry possible without data loss                |
| UI-032 | P0       | Network  | Submit succeeds but response lost | Reload shows graded result, no duplicate payout |
| UI-033 | P1       | Recovery | Start remediation after result    | Flow works from result screen                   |
| UI-034 | P1       | Flagging | Flag grading for review           | Confirmation shown                              |

---

# 6. End-to-end journey matrix

| ID      | Priority | Journey                                                      | Expected Result                                  |
| ------- | -------- | ------------------------------------------------------------ | ------------------------------------------------ |
| E2E-001 | P0       | Sign up → get daily contract → start job → annotate → submit | Full happy path works                            |
| E2E-002 | P0       | Existing user → endless job → perfect score                  | Full payout, leaderboard-affecting stats updated |
| E2E-003 | P0       | Existing user → partial score → remediation → refund         | Refund ledger and balance updated                |
| E2E-004 | P0       | Existing user → submit poor score → flag for review          | Flag stored                                      |
| E2E-005 | P1       | User with low rank sees locked higher-tier job               | Lock handling works                              |
| E2E-006 | P1       | User changes explanation preference and plays next job       | Preference reflected                             |
| E2E-007 | P1       | Revisit past attempt results                                 | Results load consistently                        |

---

# 7. Content QA matrix

This is especially important because jobs are template-driven and pre-generated.

## 7.1 Job validity

| ID     | Priority | Area        | Scenario                                                                    | Expected Result                                            |
| ------ | -------- | ----------- | --------------------------------------------------------------------------- | ---------------------------------------------------------- |
| CQ-001 | P0       | Job content | `flawed_text` differs from `clean_text` for each declared error             | Difference aligns with error objects                       |
| CQ-002 | P0       | Job content | Every `job_error` span points to actual `incorrect_text` in flawed_text     | Exact match                                                |
| CQ-003 | P0       | Job content | Replacing each error span with canonical correction reconstructs clean text | Clean text reproducible or consistent per generation model |
| CQ-004 | P0       | Job content | Accepted corrections are non-empty                                          | Present for each error                                     |
| CQ-005 | P0       | Job content | Error type consistent with explanation                                      | No mislabeled taxonomy                                     |
| CQ-006 | P1       | Job content | Word count within template range                                            | Valid                                                      |
| CQ-007 | P1       | Job content | Estimated duration plausible for text length/error count                    | Within expected band                                       |

---

## 7.2 Linguistic quality

| ID     | Priority | Area     | Scenario                                              | Expected Result              |
| ------ | -------- | -------- | ----------------------------------------------------- | ---------------------------- |
| CQ-020 | P0       | Language | Clean text is valid, natural Chinese for target level | Passes linguistic review     |
| CQ-021 | P0       | Language | Flawed text has only intended errors                  | No accidental extra errors   |
| CQ-022 | P0       | Language | Explanations are accurate in EN and ZH                | Correct and understandable   |
| CQ-023 | P1       | Language | Beginner jobs use accessible explanation language     | Suitable for target audience |
| CQ-024 | P1       | Language | Scenario context matches passage content              | Coherent                     |
| CQ-025 | P1       | Tone     | Satirical bureaucratic framing consistent             | Fits narrative style         |

---

# 8. Security and abuse matrix

| ID      | Priority | Area         | Scenario                                            | Expected Result                             |
| ------- | -------- | ------------ | --------------------------------------------------- | ------------------------------------------- |
| SEC-001 | P0       | AuthZ        | User requests another user’s attempt                | Forbidden                                   |
| SEC-002 | P0       | AuthZ        | User submits annotations to another user’s attempt  | Forbidden                                   |
| SEC-003 | P0       | Integrity    | Client tampers with payout fields in request        | Ignored; server authoritative               |
| SEC-004 | P0       | Integrity    | Client submits impossible spans beyond text length  | Validation error                            |
| SEC-005 | P1       | Abuse        | Spam many annotations to brute-force matching       | Request limited or still scored predictably |
| SEC-006 | P1       | Abuse        | Repeated remediation submissions for refund farming | One refund per item                         |
| SEC-007 | P1       | Abuse        | Automated repeated flag submissions                 | Rate limiting / moderation policy           |
| SEC-008 | P1       | Input safety | Large correction text payload                       | Validation or size limit enforced           |

---

# 9. Performance matrix

| ID       | Priority | Area         | Scenario                                          | Expected Result                     |
| -------- | -------- | ------------ | ------------------------------------------------- | ----------------------------------- |
| PERF-001 | P1       | Grading      | Typical job with 5–10 errors and 5–15 annotations | Grading within acceptable latency   |
| PERF-002 | P1       | Grading      | Worst-case annotation volume allowed by API       | No timeout, no duplicate processing |
| PERF-003 | P1       | Leaderboards | Daily leaderboard query under realistic load      | Acceptable latency                  |
| PERF-004 | P2       | Content      | Job start endpoint under load                     | Playable job returned quickly       |
| PERF-005 | P2       | DB           | Concurrent submits on same job by many users      | Stable performance                  |

---

# 10. Regression fixture set

I strongly recommend building a **small canonical fixture library** for the grader. These become gold-standard tests.

## Suggested fixture groups

### Fixture A: Single-character wrong-character error

* One exact answer
* One alternate accepted correction
* Tests character-level span precision

### Fixture B: Multi-character phrase error

* Tests phrase spans and wide selection penalties

### Fixture C: Two adjacent errors

* Tests one-to-one matching and ambiguity resolution

### Fixture D: One error, many noisy annotations

* Tests false positives and submit behavior

### Fixture E: Mixed-result job

* perfect + partial + missed + false positive
* used for payout math verification

### Fixture F: Simplified/traditional accepted variants

* tests normalization policy

### Fixture G: Remediation-enabled job

* tests refund flow and ledger integrity

---

# 11. Minimal release gate

If you want a practical pre-launch bar, I’d define this as the minimum:

## Must-pass P0 groups

* scoring engine core: SC-001 to SC-007, SC-020, SC-030, SC-040, SC-050, SC-060 to SC-063
* DB integrity: DB-001 to DB-007, DB-020 to DB-022
* core APIs: API-001, API-003, API-005, API-020, API-023, API-025, API-030 to API-035, API-040 to API-045
* key UI: UI-001 to UI-007, UI-020 to UI-022, UI-030 to UI-032
* E2E: E2E-001 to E2E-004
* content QA: CQ-001 to CQ-005, CQ-020 to CQ-022
* security: SEC-001 to SEC-004

That gives you confidence in:

* fairness
* correctness
* payout safety
* basic UX trust

---

# 12. Recommended automation split

## Automate first

* all scoring engine tests
* payout math
* API validation and authz
* DB transaction/idempotency tests
* core E2E happy path
* content structural QA checks

## Manual QA first

* feel of text selection
* annotation UX quality
* clarity of result explanations
* satirical tone and content feel
* subjective fairness of broad span penalties

---

# 13. Suggested test case format

For implementation, use a standard template per case:

* `id`
* `title`
* `priority`
* `preconditions`
* `input`
* `steps`
* `expected_result`
* `automation_level` (`unit`, `integration`, `e2e`, `manual`)
* `notes`

Example:

```text
ID: SC-003
Title: Very wide full-coverage span receives reduced span credit
Priority: P0
Preconditions: Job has one error spanning indices [10,12)
Input: Annotation span [8,16), correct type, correct correction
Steps:
1. Submit attempt with single annotation
2. Grade attempt
Expected Result:
- Annotation matches target error
- span_tier = VERY_WIDE
- span_credit = 0.40
- total_credit reflects weighting
Automation Level: unit
```

---

