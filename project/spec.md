# 📄 Product Specification: LLM-Powered Language Editing Game (v1)

## 1. Overview

### 1.1 Product Summary

A browser-based educational game where players act as editors within a satirical bureaucratic organization, identifying and correcting mistakes in short Chinese-language texts. The experience emphasizes short, engaging sessions (3–5 minutes per job), combining language learning with a progression-based game loop.

### 1.2 Core Value Proposition

* Learn Chinese through **active error detection and correction**
* Experience a **game-first loop** with scoring, money, and progression
* Engage with **varied, contextualized content** generated via templates
* Receive **clear feedback and optional practice reinforcement**

---

## 2. Target Audience

### Primary Audience

* Adult English-speaking learners of Chinese

### Secondary (Future)

* Other L1 learners (e.g., French → Chinese)
* Intermediate learners seeking practice (HSK1–4 initially)

---

## 3. Core Gameplay Loop

### Loop Steps

1. Player selects a job
2. Reads scenario + passage
3. Annotates mistakes:

   * highlight text
   * assign error type
   * enter correction
4. Submits work
5. Receives grading breakdown (“moment of truth”)
6. Earns money and progresses
7. Optionally reviews explanations and completes practice

---

## 4. Game Structure

### 4.1 Session Design

* Each job: **3–5 minutes**
* Daily session:

  * 1 “Daily Contract” (higher payout)
  * Optional additional jobs (endless play)

### 4.2 Job Composition

Each job includes:

* Scenario context (1–3 sentences, narrative flavor)
* One passage (<300 words)
* Multiple injected errors (variable count)

---

## 5. Annotation System

### 5.1 Interaction Model

* Player selects text via click + drag
* System creates annotation
* Player completes annotation via UI panel

### 5.2 Annotation Object

```json
{
  "span_start": int,
  "span_end": int,
  "error_type": enum,
  "correction": string
}
```

### 5.3 Constraints

* **1 annotation = 1 mistake**
* No multi-error annotations allowed
* Overly broad selections reduce score

---

## 6. Error System

### 6.1 Error Object (Ground Truth)

```json
{
  "id": string,
  "type": enum,
  "span_start": int,
  "span_end": int,
  "incorrect_text": string,
  "accepted_corrections": [string],
  "explanation_en": string,
  "explanation_zh": string
}
```

---

### 6.2 Error Types (v1)

1. Wrong Character (同音字错误)
2. Wrong Word Choice (用词错误)
3. Word Order Error (语序错误)
4. Missing Word (缺少词语)
5. Extra Word (多余词语)
6. Grammar Pattern Error (语法结构错误)
7. Measure Word Error (量词错误)

Notes:

* Objective errors only
* No style/register in v1

---

## 7. Scoring System

### 7.1 Scoring Dimensions

Each error is evaluated on:

1. **Detection**
2. **Span Accuracy**
3. **Error Type**
4. **Correction Accuracy**

---

### 7.2 Span Accuracy (Tier Model)

| Tier          | Description          | Score Impact |
| ------------- | -------------------- | ------------ |
| Perfect       | Exact or tight match | 100%         |
| Slightly Wide | Minor extra text     | 70–90%       |
| Very Wide     | Significant extra    | 10–50%       |
| Excessive     | Too large            | 0%           |

---

### 7.3 Detection

* Found → base credit
* Missed → penalty

---

### 7.4 Error Type

* Correct → full credit
* Incorrect → partial or zero

---

### 7.5 Correction

* Matches accepted list → full
* Close but invalid → partial or zero

---

### 7.6 False Positives

* Penalized when:

  * annotation does not correspond to real error

---

### 7.7 Final Job Score

Calculated after submission:

* Sum of all error scores
* Minus penalties
* Converted into money payout

---

## 8. Economy & Progression

### 8.1 Currency

* **Money (primary)**

  * persistent
  * earned per job
  * used for cosmetics / progression gating (future)

### 8.2 Rank System

* Integer-based level
* Represents career progression
* Unlocks:

  * new departments
  * harder jobs

---

### 8.3 Payment Model

Per job:

* Base fee (hidden)
* Final payout = computed after grading

Includes:

* performance bonuses
* penalties

---

## 9. Narrative & Setting

### 9.1 World

* Single large bureaucratic organization
* Satirical and slightly absurd tone

### 9.2 Progression Structure

Departments (example progression):

1. Elementary Assignment Review Dept.
2. Local Correspondence Office
3. News Bureau
4. Civil Documentation Dept.
5. Academic Review Board

---

## 10. Learning & Feedback

### 10.1 Post-Submission Feedback

* Full breakdown of:

  * missed errors
  * incorrect annotations
  * correct answers

---

### 10.2 Explanations

* Provided for each error
* Available in:

  * English
  * Chinese

Language behavior:

* English-heavy early
* Chinese default at HSK3–4+
* User override available

---

### 10.3 Practice Exercises

* Offered per mistake (missed or incorrect)
* Optional
* Types:

  * fill-in-the-blank
  * multiple choice

### 10.4 Refund Mechanic

* Completing exercises returns money
* Refund scaled by job level

---

## 11. Leaderboards

### 11.1 Types

* Daily
* Weekly

### 11.2 Metrics

* Total earnings
* Accuracy rate

---

## 12. Content Generation Pipeline

### 12.1 Approach: Template-Driven

Each job is generated from:

* scenario template
* content template
* error injection rules

---

### 12.2 Generation Flow

1. Generate clean passage

2. Inject errors based on taxonomy

3. Produce:

   * flawed text
   * error objects
   * accepted corrections
   * explanations

4. Store in DB

---

### 12.3 Storage Model

Each job stores:

* scenario context
* clean text
* flawed text
* structured errors
* explanations
* practice items

---

### 12.4 LLM Usage

* Used **offline / pre-generation**
* Not used for scoring
* Focus on:

  * low-cost models
  * batch generation

---

## 13. UI / UX

### 13.1 Core Screen Layout

* Left: scenario context
* Center: passage
* Right or overlay: annotation editor

---

### 13.2 Annotation UX

* Click + drag highlight
* Opens annotation panel:

  * error type dropdown
  * correction input

---

### 13.3 Visual Style

* Pre-PC office desk aesthetic
* Paper documents
* Red pen annotations

---

## 14. Technical Architecture (High-Level)

### 14.1 Frontend

* SPA (React or Next.js recommended)
* Desktop-first

### 14.2 Backend

* API server (Node.js / similar)
* Handles:

  * job retrieval
  * scoring
  * persistence

### 14.3 Database

Stores:

* users
* jobs
* annotations
* scores
* leaderboard data

---

### 14.4 Auth

* OAuth (Google, etc.)
* Native account option

---

## 15. Non-Goals (v1)

* Multiplayer gameplay
* Real-time collaboration
* Style/register correction
* Fully dynamic generation at runtime
* Handwriting input
* Mobile-first optimization

---

## 16. Future Extensions

* Style and tone errors
* Social features (sharing jobs)
* Advanced analytics
* Multi-language support (other L1s)
* Adaptive difficulty
* LLM-assisted personalized feedback

---

## 17. Success Criteria (v1)

* Players can complete a job in under 5 minutes
* Scoring feels fair and understandable
* Error detection improves over repeated play
* Daily engagement loop is viable
* Content quality is consistent and believable
