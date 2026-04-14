Below is a v1-oriented **data schema and API contract** designed for:

* template-driven content
* server-side scoring
* desktop SPA frontend
* short jobs with post-submission grading
* future expansion into progression, cosmetics, moderation, and more languages

I’ll keep it implementation-neutral, but concrete enough to hand to engineering.

---

# Data Schema

## 1. Design principles

The schema should support these constraints:

* **Jobs are pre-generated and stored**
* **Ground truth is structured**
* **Player submissions are stored separately from grading results**
* **Scoring is reproducible**
* **Jobs can be reused in daily contracts or endless play**
* **Explanations and remediation are attached to individual errors**
* **Player progression is persistent**

For v1, I would separate the model into these groups:

* identity and progression
* content and templates
* assignments and gameplay
* scoring and review
* leaderboards and analytics

---

## 2. Core entities

## 2.1 users

Represents a player account.

**Fields**

* `id` UUID
* `email` string nullable
* `oauth_provider` enum nullable (`google`, `apple`, `discord`, etc.)
* `oauth_subject` string nullable
* `username` string unique
* `display_name` string
* `created_at` timestamp
* `updated_at` timestamp
* `last_seen_at` timestamp
* `is_active` boolean
* `preferred_ui_language` enum (`en`, `zh`)
* `preferred_explanation_language_mode` enum (`auto`, `en`, `zh`, `bilingual`)
* `preferred_script` enum (`simplified`, `traditional`)
* `daily_contract_streak` integer default 0
* `rank_level` integer default 1
* `money_balance` integer default 0
* `lifetime_earnings` integer default 0
* `lifetime_accuracy_score` numeric nullable
* `flags_count` integer default 0

**Notes**

* `money_balance` should use integer minor units to avoid floating-point issues.
* `preferred_explanation_language_mode=auto` supports the “default to Chinese later” rule.

---

## 2.2 user_profiles

Optional split if you want to keep gameplay preferences out of `users`.

**Fields**

* `user_id` UUID PK/FK
* `current_department_id` UUID nullable
* `current_rank_title` string
* `tutorial_completed` boolean
* `onboarding_state` jsonb
* `accessibility_settings` jsonb
* `cosmetic_loadout` jsonb

For a small v1, this can be folded into `users`.

---

## 2.3 departments

Represents the satirical bureaucracy progression tiers.

**Fields**

* `id` UUID
* `code` string unique
* `name_en` string
* `name_zh` string
* `description_en` string
* `description_zh` string
* `sort_order` integer
* `min_rank_level` integer
* `is_active` boolean

**Example**

* `ELEMENTARY_REVIEW`
* `LOCAL_CORRESPONDENCE`
* `NEWS_BUREAU`
* `CIVIL_DOCUMENTATION`
* `ACADEMIC_REVIEW`

---

## 2.4 error_types

Canonical list for the dropdown and scoring logic.

**Fields**

* `id` UUID
* `code` string unique
* `name_en` string
* `name_zh` string
* `description_en` string
* `description_zh` string
* `sort_order` integer
* `is_active` boolean

**Suggested codes**

* `WRONG_CHARACTER`
* `WRONG_WORD_CHOICE`
* `WORD_ORDER`
* `MISSING_WORD`
* `EXTRA_WORD`
* `GRAMMAR_PATTERN`
* `MEASURE_WORD`

---

## 2.5 scenario_templates

The high-level narrative shell for a job.

**Fields**

* `id` UUID
* `code` string unique
* `department_id` UUID FK
* `name` string
* `context_template_en` text
* `context_template_zh` text
* `tone_tags` text[]
* `difficulty_tier` integer
* `is_active` boolean
* `created_at` timestamp

**Purpose**
This defines things like:

* “A student wrote an essay about their family”
* “A local reporter is covering a small event”
* “A clerk submitted a memo”

---

## 2.6 job_templates

Defines reusable passage-generation scaffolds.

**Fields**

* `id` UUID
* `code` string unique
* `department_id` UUID FK
* `scenario_template_id` UUID FK nullable
* `title` string
* `description` text
* `target_language` string default `zh-CN`
* `script` enum (`simplified`, `traditional`)
* `difficulty_tier` integer
* `hsk_band_min` integer nullable
* `hsk_band_max` integer nullable
* `min_word_count` integer
* `max_word_count` integer
* `target_error_count_min` integer
* `target_error_count_max` integer
* `allowed_error_type_codes` text[]
* `content_constraints` jsonb
* `is_active` boolean
* `created_at` timestamp

**Notes**
`content_constraints` could include:

* topic families
* required grammar points
* banned grammar points
* voice/style instructions
* sentence length range

---

## 2.7 jobs

A concrete pre-generated playable unit.

This is the most important content table.

**Fields**

* `id` UUID
* `job_template_id` UUID FK
* `scenario_template_id` UUID FK nullable
* `department_id` UUID FK
* `difficulty_tier` integer
* `rank_band_min` integer nullable
* `rank_band_max` integer nullable
* `target_language` string
* `script` enum
* `title_en` string
* `title_zh` string
* `context_en` text
* `context_zh` text
* `clean_text` text
* `flawed_text` text
* `word_count` integer
* `estimated_duration_seconds` integer
* `base_fee_minor` integer
* `refund_value_minor` integer
* `status` enum (`draft`, `validated`, `playable`, `retired`)
* `generation_version` string
* `generated_by_model` string nullable
* `validated_by` string nullable
* `created_at` timestamp
* `updated_at` timestamp
* `play_count` integer default 0
* `flag_count` integer default 0

**Notes**

* `clean_text` is the corrected reference passage.
* `flawed_text` is what the player sees.
* `status` supports moderation and content QA.

---

## 2.8 job_errors

Ground truth for each error inside a job.

**Fields**

* `id` UUID
* `job_id` UUID FK
* `sequence_index` integer
* `error_type_id` UUID FK
* `span_start` integer
* `span_end` integer
* `incorrect_text` text
* `canonical_correction` text
* `accepted_corrections` jsonb
* `span_tolerance_mode` enum default `tiered`
* `explanation_en` text
* `explanation_zh` text
* `grammar_tags` text[]
* `difficulty_weight` numeric
* `created_at` timestamp

**Notes**

* `accepted_corrections` should be a JSON array of strings for v1.
* If you later need richer matching, convert to structured variants.

**Example**

```json
[
  "你是哪国人？",
  "你是哪國人？"
]
```

For most v1 cases, accepted corrections should match the selected span replacement text, not the entire sentence.

---

## 2.9 remediation_items

Optional post-job practice prompts associated with a specific error.

**Fields**

* `id` UUID
* `job_error_id` UUID FK
* `exercise_type` enum (`multiple_choice`, `fill_blank`)
* `prompt_en` text
* `prompt_zh` text
* `payload` jsonb
* `correct_answer` jsonb
* `refund_minor` integer nullable
* `created_at` timestamp

**Examples**
For multiple choice:

```json
{
  "choices": [
    "你是哪国人？",
    "泥是哪国人？",
    "你是哪里人？"
  ]
}
```

For fill-in-the-blank:

```json
{
  "sentence": "__是哪国人？"
}
```

---

## 2.10 daily_contracts

Represents the featured daily set.

**Fields**

* `id` UUID
* `contract_date` date
* `title_en` string
* `title_zh` string
* `description_en` text
* `description_zh` text
* `department_id` UUID FK nullable
* `min_rank_level` integer default 1
* `max_rank_level` integer nullable
* `is_active` boolean
* `created_at` timestamp

---

## 2.11 daily_contract_jobs

Join table mapping jobs into a daily contract.

**Fields**

* `id` UUID
* `daily_contract_id` UUID FK
* `job_id` UUID FK
* `sequence_index` integer

---

## 2.12 job_attempts

A player’s playthrough of a single job.

**Fields**

* `id` UUID
* `user_id` UUID FK
* `job_id` UUID FK
* `source_type` enum (`daily_contract`, `endless`, `practice`, `admin_test`)
* `source_id` UUID nullable
* `started_at` timestamp
* `submitted_at` timestamp nullable
* `completed_at` timestamp nullable
* `time_spent_seconds` integer default 0
* `status` enum (`in_progress`, `submitted`, `graded`, `abandoned`)
* `client_version` string nullable
* `score_summary` jsonb nullable
* `earnings_minor` integer default 0
* `refund_earned_minor` integer default 0
* `accuracy_ratio` numeric nullable
* `flagged_for_review` boolean default false
* `review_status` enum nullable (`pending`, `reviewed_no_change`, `reviewed_adjusted`)
* `created_at` timestamp
* `updated_at` timestamp

**Notes**

* `score_summary` is denormalized for fast reads.
* Keep the per-annotation and per-error details in separate tables.

---

## 2.13 attempt_annotations

Raw annotations submitted by the player.

**Fields**

* `id` UUID
* `attempt_id` UUID FK
* `sequence_index` integer
* `span_start` integer
* `span_end` integer
* `selected_text` text
* `error_type_id` UUID FK nullable
* `correction_text` text
* `created_at` timestamp

---

## 2.14 attempt_annotation_results

Stores grading outcomes per submitted annotation.

**Fields**

* `id` UUID
* `attempt_annotation_id` UUID FK
* `matched_job_error_id` UUID FK nullable
* `is_false_positive` boolean
* `detection_credit` numeric
* `span_credit` numeric
* `error_type_credit` numeric
* `correction_credit` numeric
* `total_credit` numeric
* `penalty_minor` integer default 0
* `feedback_en` text nullable
* `feedback_zh` text nullable
* `created_at` timestamp

**Purpose**
This makes the grading transparent and reviewable.

---

## 2.15 attempt_error_results

Stores grading outcomes from the error-centric perspective.

**Fields**

* `id` UUID
* `attempt_id` UUID FK
* `job_error_id` UUID FK
* `matched_annotation_id` UUID FK nullable
* `status` enum (`corrected`, `partially_corrected`, `missed`)
* `detection_credit` numeric
* `span_credit` numeric
* `error_type_credit` numeric
* `correction_credit` numeric
* `total_credit` numeric
* `penalty_minor` integer default 0
* `created_at` timestamp

**Why both annotation results and error results?**
Because the UI and analytics often want both views:

* “What happened to each mark I made?”
* “Which true errors did I miss?”

---

## 2.16 attempt_remediation_results

Tracks optional practice after grading.

**Fields**

* `id` UUID
* `attempt_id` UUID FK
* `remediation_item_id` UUID FK
* `status` enum (`offered`, `started`, `completed`, `skipped`)
* `submitted_answer` jsonb nullable
* `is_correct` boolean nullable
* `refund_awarded_minor` integer default 0
* `created_at` timestamp
* `updated_at` timestamp

---

## 2.17 financial_ledger_entries

Tracks money changes robustly.

Do not rely only on balance snapshots.

**Fields**

* `id` UUID
* `user_id` UUID FK
* `entry_type` enum (`job_payout`, `practice_refund`, `manual_adjustment`, `purchase`, `reward`)
* `reference_type` enum (`attempt`, `remediation`, `admin_action`, `shop_order`)
* `reference_id` UUID nullable
* `amount_minor` integer
* `balance_after_minor` integer
* `description` text
* `created_at` timestamp

---

## 2.18 rank_history

Optional but useful for analytics and auditing.

**Fields**

* `id` UUID
* `user_id` UUID FK
* `old_rank_level` integer
* `new_rank_level` integer
* `reason` string
* `created_at` timestamp

---

## 2.19 leaderboard_snapshots

Precomputed leaderboard rows for performance and consistency.

**Fields**

* `id` UUID
* `period_type` enum (`daily`, `weekly`)
* `period_start` date
* `period_end` date
* `user_id` UUID FK
* `earnings_minor` integer
* `accuracy_ratio` numeric
* `jobs_completed` integer
* `rank_position_by_earnings` integer nullable
* `rank_position_by_accuracy` integer nullable
* `created_at` timestamp

---

## 2.20 grading_flags

Player-submitted flags for suspected grading issues.

**Fields**

* `id` UUID
* `attempt_id` UUID FK
* `user_id` UUID FK
* `reason_code` enum (`valid_answer_rejected`, `span_too_strict`, `wrong_error_match`, `other`)
* `free_text_note` text nullable
* `status` enum (`pending`, `reviewed`, `resolved`)
* `review_notes` text nullable
* `created_at` timestamp
* `reviewed_at` timestamp nullable
* `reviewed_by` string nullable

---

# Relationship summary

At a high level:

* `users` have many `job_attempts`
* `jobs` belong to `job_templates` and contain many `job_errors`
* `job_errors` can have many `remediation_items`
* `job_attempts` have many `attempt_annotations`
* `attempt_annotations` produce `attempt_annotation_results`
* `job_attempts` also produce `attempt_error_results`
* `daily_contracts` contain many `jobs`
* `financial_ledger_entries` track payouts and refunds
* `grading_flags` attach to `job_attempts`

---

# Suggested scoring summary shape

Store a denormalized summary on `job_attempts.score_summary`.

Example:

```json
{
  "errors_total": 7,
  "errors_found": 5,
  "errors_missed": 2,
  "false_positives": 1,
  "perfect_spans": 3,
  "wide_spans": 2,
  "correct_error_types": 4,
  "correct_corrections": 4,
  "raw_score": 0.71,
  "accuracy_ratio": 0.68,
  "base_fee_minor": 220,
  "penalties_minor": 70,
  "bonus_minor": 15,
  "final_payout_minor": 165
}
```

---

# API Contract

## 1. API style

For v1, I would use REST over JSON with versioned routes:

* `/api/v1/...`

Use server-side scoring only.
The client should never be the source of truth for grade calculations or payouts.

Authentication:

* session cookie or bearer token
* OAuth and password login both terminate into the same auth layer

---

## 2. Authentication

## POST `/api/v1/auth/signup`

Create native account.

**Request**

```json
{
  "email": "alex@example.com",
  "username": "alex",
  "password": "..."
}
```

**Response**

```json
{
  "user": {
    "id": "usr_123",
    "username": "alex",
    "display_name": "alex",
    "rank_level": 1,
    "money_balance": 0
  },
  "token": "..."
}
```

---

## POST `/api/v1/auth/login`

**Request**

```json
{
  "email": "alex@example.com",
  "password": "..."
}
```

**Response**

```json
{
  "user": {
    "id": "usr_123",
    "username": "alex",
    "display_name": "Alex",
    "rank_level": 2,
    "money_balance": 430
  },
  "token": "..."
}
```

---

## POST `/api/v1/auth/oauth/callback`

Backend route to finalize OAuth login.
Exact format depends on provider and auth stack.

---

## POST `/api/v1/auth/logout`

Ends session.

---

## 3. Player profile and settings

## GET `/api/v1/me`

Returns current player profile.

**Response**

```json
{
  "id": "usr_123",
  "username": "alex",
  "display_name": "Alex",
  "rank_level": 3,
  "money_balance": 1280,
  "lifetime_earnings": 5540,
  "preferred_ui_language": "en",
  "preferred_explanation_language_mode": "auto",
  "preferred_script": "simplified",
  "current_department": {
    "id": "dep_news",
    "code": "NEWS_BUREAU",
    "name_en": "News Bureau",
    "name_zh": "新闻局"
  }
}
```

---

## PATCH `/api/v1/me/settings`

**Request**

```json
{
  "preferred_ui_language": "en",
  "preferred_explanation_language_mode": "bilingual",
  "preferred_script": "simplified"
}
```

**Response**

```json
{
  "ok": true
}
```

---

## GET `/api/v1/me/progression`

**Response**

```json
{
  "rank_level": 3,
  "current_department": {
    "id": "dep_news",
    "name_en": "News Bureau"
  },
  "money_balance": 1280,
  "daily_contract_streak": 4,
  "next_unlock": {
    "rank_level": 4,
    "department_id": "dep_civil",
    "department_name_en": "Civil Documentation Dept."
  }
}
```

---

## 4. Daily contract and job discovery

## GET `/api/v1/daily-contract`

Returns today’s contract for the current user.

**Response**

```json
{
  "id": "dc_2026_04_13",
  "contract_date": "2026-04-13",
  "title_en": "Tuesday Department Overflow",
  "title_zh": "周二超额文稿",
  "description_en": "The bureau is behind schedule. Complete these reviews for premium pay.",
  "jobs": [
    {
      "job_id": "job_001",
      "title_en": "Elementary Self-Introduction",
      "title_zh": "自我介绍作文",
      "department": {
        "code": "ELEMENTARY_REVIEW",
        "name_en": "Elementary Assignment Review Dept."
      },
      "difficulty_tier": 1,
      "estimated_duration_seconds": 240,
      "base_fee_minor": 180,
      "status": "available"
    },
    {
      "job_id": "job_002",
      "title_en": "Neighborhood Festival Report",
      "title_zh": "社区节日报道",
      "department": {
        "code": "NEWS_BUREAU",
        "name_en": "News Bureau"
      },
      "difficulty_tier": 3,
      "estimated_duration_seconds": 300,
      "base_fee_minor": 260,
      "status": "locked"
    }
  ]
}
```

---

## GET `/api/v1/jobs`

Browse endless-play jobs filtered by progression.

**Query params**

* `department_id` optional
* `difficulty_tier` optional
* `cursor` optional
* `limit` optional

**Response**

```json
{
  "items": [
    {
      "job_id": "job_101",
      "title_en": "Travel Diary Entry",
      "title_zh": "旅行日记",
      "difficulty_tier": 2,
      "estimated_duration_seconds": 210,
      "base_fee_minor": 200,
      "department": {
        "id": "dep_local",
        "name_en": "Local Correspondence Office"
      }
    }
  ],
  "next_cursor": "..."
}
```

---

## GET `/api/v1/jobs/{jobId}/preview`

Returns metadata only, not the full flawed passage if you want to avoid preloading too much.

**Response**

```json
{
  "job_id": "job_101",
  "title_en": "Travel Diary Entry",
  "title_zh": "旅行日记",
  "context_en": "A student wrote a diary entry about a day trip.",
  "context_zh": "一名学生写了一篇关于一日游的日记。",
  "difficulty_tier": 2,
  "estimated_duration_seconds": 210,
  "base_fee_minor": 200
}
```

---

## 5. Starting and playing a job

## POST `/api/v1/jobs/{jobId}/attempts`

Creates a new attempt.

**Request**

```json
{
  "source_type": "daily_contract",
  "source_id": "dc_2026_04_13"
}
```

**Response**

```json
{
  "attempt_id": "att_555",
  "job": {
    "id": "job_101",
    "title_en": "Travel Diary Entry",
    "title_zh": "旅行日记",
    "context_en": "A student wrote a diary entry about a day trip.",
    "context_zh": "一名学生写了一篇关于一日游的日记。",
    "flawed_text": "昨天我和朋友去公园玩，我们看了很多树，也吃了一个很开心的午饭。",
    "difficulty_tier": 2,
    "estimated_duration_seconds": 210,
    "base_fee_minor": 200
  },
  "annotation_rules": {
    "one_annotation_per_mistake": true,
    "error_types": [
      { "id": "et_1", "code": "WRONG_CHARACTER", "name_en": "Wrong Character", "name_zh": "同音字错误" },
      { "id": "et_2", "code": "WRONG_WORD_CHOICE", "name_en": "Wrong Word Choice", "name_zh": "用词错误" }
    ]
  }
}
```

---

## PATCH `/api/v1/attempts/{attemptId}/heartbeat`

Optional endpoint to track time and abandoned sessions.

**Request**

```json
{
  "client_elapsed_seconds": 87
}
```

**Response**

```json
{
  "ok": true
}
```

This is optional. You can also derive time only from start/submit timestamps for v1.

---

## GET `/api/v1/attempts/{attemptId}`

Returns the current in-progress attempt state.

**Response**

```json
{
  "attempt_id": "att_555",
  "status": "in_progress",
  "job": {
    "id": "job_101",
    "flawed_text": "昨天我和朋友去公园玩，我们看了很多树，也吃了一个很开心的午饭。"
  },
  "annotations": [
    {
      "id": "ann_1",
      "span_start": 20,
      "span_end": 22,
      "selected_text": "开心",
      "error_type_id": "et_2",
      "correction_text": "丰富"
    }
  ]
}
```

---

## PUT `/api/v1/attempts/{attemptId}/annotations`

For simplicity, v1 can replace the full annotation list on each save rather than supporting complex patch semantics.

**Request**

```json
{
  "annotations": [
    {
      "client_id": "tmp_1",
      "span_start": 20,
      "span_end": 22,
      "selected_text": "开心",
      "error_type_id": "et_2",
      "correction_text": "丰富"
    },
    {
      "client_id": "tmp_2",
      "span_start": 0,
      "span_end": 1,
      "selected_text": "昨",
      "error_type_id": "et_1",
      "correction_text": "作"
    }
  ]
}
```

**Response**

```json
{
  "annotations": [
    {
      "id": "ann_11",
      "client_id": "tmp_1"
    },
    {
      "id": "ann_12",
      "client_id": "tmp_2"
    }
  ],
  "updated_at": "2026-04-13T18:14:00Z"
}
```

For higher scale later, you can add:

* `POST /annotations`
* `PATCH /annotations/{id}`
* `DELETE /annotations/{id}`

But full replacement is simpler and reliable for v1.

---

## 6. Submission and grading

## POST `/api/v1/attempts/{attemptId}/submit`

Submits and grades the attempt.

**Request**

```json
{
  "final_annotations": [
    {
      "span_start": 20,
      "span_end": 22,
      "selected_text": "开心",
      "error_type_id": "et_2",
      "correction_text": "丰富"
    }
  ]
}
```

**Response**

```json
{
  "attempt_id": "att_555",
  "status": "graded",
  "summary": {
    "errors_total": 4,
    "errors_found": 3,
    "errors_missed": 1,
    "false_positives": 0,
    "correct_error_types": 3,
    "correct_corrections": 2,
    "accuracy_ratio": 0.74,
    "base_fee_minor": 200,
    "penalties_minor": 40,
    "bonus_minor": 10,
    "final_payout_minor": 170
  },
  "earnings": {
    "money_balance_before_minor": 1280,
    "money_balance_after_minor": 1450,
    "earned_minor": 170
  },
  "annotation_results": [
    {
      "annotation_id": "ann_11",
      "matched_error_id": "err_3",
      "is_false_positive": false,
      "detection_credit": 1.0,
      "span_credit": 0.8,
      "error_type_credit": 1.0,
      "correction_credit": 0.5,
      "total_credit": 0.83,
      "feedback_en": "You found the right issue, but your correction was only partially correct.",
      "feedback_zh": "你找到了正确的问题，但修改答案只部分正确。"
    }
  ],
  "error_results": [
    {
      "error_id": "err_1",
      "status": "corrected",
      "explanation_en": "Here the measure word is incorrect.",
      "explanation_zh": "这里的量词用错了。",
      "remediation_available": true
    },
    {
      "error_id": "err_4",
      "status": "missed",
      "explanation_en": "This sentence is missing a required particle.",
      "explanation_zh": "这个句子缺少一个必要的助词。",
      "remediation_available": true
    }
  ],
  "next_actions": {
    "can_start_remediation": true,
    "can_flag_for_review": true,
    "can_replay": false
  }
}
```

---

## GET `/api/v1/attempts/{attemptId}/results`

Fetches a previously graded result.

Response shape can match submit response.

---

## 7. Remediation flow

## GET `/api/v1/attempts/{attemptId}/remediation`

Returns available practice items for mistakes made or missed.

**Response**

```json
{
  "items": [
    {
      "remediation_item_id": "rem_1",
      "job_error_id": "err_4",
      "exercise_type": "fill_blank",
      "prompt_en": "Fill in the missing word.",
      "prompt_zh": "请填入缺少的词。",
      "payload": {
        "sentence": "你__哪国人？"
      },
      "potential_refund_minor": 20
    }
  ]
}
```

---

## POST `/api/v1/attempts/{attemptId}/remediation/{remediationItemId}/submit`

**Request**

```json
{
  "answer": {
    "text": "是"
  }
}
```

**Response**

```json
{
  "is_correct": true,
  "refund_awarded_minor": 20,
  "money_balance_after_minor": 1470,
  "feedback_en": "Correct. Your refund has been added.",
  "feedback_zh": "正确，退款已发放。"
}
```

---

## 8. Flags and moderation

## POST `/api/v1/attempts/{attemptId}/flag`

Lets the player flag suspected bad grading for developer review.

**Request**

```json
{
  "reason_code": "valid_answer_rejected",
  "free_text_note": "I think my correction should have counted."
}
```

**Response**

```json
{
  "ok": true,
  "flag_id": "flag_101",
  "status": "pending"
}
```

---

## GET `/api/v1/me/flags`

Optional player-facing history of submitted flags.

---

## 9. Leaderboards

## GET `/api/v1/leaderboards/daily`

**Query params**

* `metric=earnings|accuracy`
* `limit=50`

**Response**

```json
{
  "period_type": "daily",
  "period_start": "2026-04-13",
  "period_end": "2026-04-13",
  "metric": "earnings",
  "entries": [
    {
      "position": 1,
      "user": {
        "id": "usr_1",
        "username": "lin"
      },
      "earnings_minor": 880,
      "accuracy_ratio": 0.93,
      "jobs_completed": 4
    }
  ],
  "viewer_entry": {
    "position": 18,
    "earnings_minor": 170,
    "accuracy_ratio": 0.74,
    "jobs_completed": 1
  }
}
```

---

## GET `/api/v1/leaderboards/weekly`

Same shape with weekly period bounds.

---

## 10. Financial and history endpoints

## GET `/api/v1/me/ledger`

Shows payout history.

**Response**

```json
{
  "items": [
    {
      "id": "led_1",
      "entry_type": "job_payout",
      "amount_minor": 170,
      "description": "Travel Diary Entry payout",
      "created_at": "2026-04-13T18:16:00Z"
    },
    {
      "id": "led_2",
      "entry_type": "practice_refund",
      "amount_minor": 20,
      "description": "Refund for remediation",
      "created_at": "2026-04-13T18:18:00Z"
    }
  ],
  "next_cursor": null
}
```

---

## GET `/api/v1/me/attempts`

Attempt history.

**Query params**

* `status`
* `cursor`
* `limit`

---

# Scoring contract detail

This is the main server-side grading sequence that the API should implement.

## Submission input

The server receives:

* attempt id
* final list of annotations with span, error type, correction

## Grading steps

1. Load the job and its `job_errors`
2. Match each annotation to at most one job error
3. Compute span tier credit
4. Compute error-type match credit
5. Compute correction match credit
6. Mark unmatched annotations as false positives
7. Mark unmatched job errors as missed
8. Aggregate credits and penalties
9. Compute final payout
10. Write:

* `attempt_annotations`
* `attempt_annotation_results`
* `attempt_error_results`
* `job_attempts.score_summary`
* `financial_ledger_entries`
* updated `users.money_balance`

## Important invariant

Server must prevent double-submission payouts.
Once an attempt is graded, repeated submit calls should be idempotent or rejected.

Recommended behavior:

* first successful grading writes a submission hash
* later duplicate submits return the existing result

---

# Validation and error handling

Use conventional API errors.

Example:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "span_end must be greater than span_start"
  }
}
```

Suggested error codes:

* `UNAUTHORIZED`
* `FORBIDDEN`
* `NOT_FOUND`
* `JOB_LOCKED`
* `ATTEMPT_NOT_EDITABLE`
* `ATTEMPT_ALREADY_GRADED`
* `INVALID_ANNOTATION`
* `RATE_LIMITED`
* `VALIDATION_ERROR`

---

# Minimal v1 implementation subset

If you want a smaller first cut, this is the leanest version I’d build first:

## Must-have tables

* `users`
* `departments`
* `error_types`
* `jobs`
* `job_errors`
* `remediation_items`
* `job_attempts`
* `attempt_annotations`
* `attempt_annotation_results`
* `attempt_error_results`
* `financial_ledger_entries`
* `grading_flags`

## Must-have endpoints

* `POST /auth/login`
* `GET /me`
* `GET /daily-contract`
* `GET /jobs`
* `POST /jobs/{jobId}/attempts`
* `GET /attempts/{attemptId}`
* `PUT /attempts/{attemptId}/annotations`
* `POST /attempts/{attemptId}/submit`
* `GET /attempts/{attemptId}/results`
* `GET /attempts/{attemptId}/remediation`
* `POST /attempts/{attemptId}/remediation/{remediationItemId}/submit`
* `POST /attempts/{attemptId}/flag`
* `GET /leaderboards/daily`
* `GET /leaderboards/weekly`
