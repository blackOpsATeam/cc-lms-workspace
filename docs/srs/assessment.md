# 🧩 MODULE: ASSESSMENT MANAGEMENT

## 1. Purpose

Manage the full academic assessment lifecycle for CC-LMS: creation (quiz, practice, class test, assignment, live exam, monthly exam, model test), question authoring (objective + subjective + file-submission), student attempt and submission, auto-evaluation for objective items, teacher manual evaluation for subjective items, result computation, controlled publication, and performance insights (weak chapter/topic, trend, batch comparison). Assessment is the source of truth for evaluation; Teacher / Student Class Workspaces and Dashboard consume it via read-through endpoints and contextual shortcuts but never compute evaluation themselves.

## 2. Scope

**In Scope**

- Assessment authoring across the six types: `PRACTICE_QUIZ`, `CLASS_TEST`, `ASSIGNMENT`, `LIVE_EXAM`, `MONTHLY_EXAM`, `MODEL_TEST`
- Context mapping: required batch + subject; optional class, chapter/topic, and linked `(schedule_entry_id, class_date)`
- Question authoring: `OBJECTIVE` (MCQ, true/false, single-answer), `SUBJECTIVE` (written), `FILE_SUBMISSION`
- Attempt flow: window enforcement, timer, save-as-progress (configurable), auto-submit on timer expiry, single or multi-attempt per configuration
- Auto-evaluation of objective items via answer key
- Manual evaluation of subjective items and file submissions with awarded marks + optional remarks
- Result computation: total score, percentage, optional rank
- Controlled publication: results invisible until teacher/admin clicks publish
- Per-student performance insights: per-assessment summary, weak chapter/topic (where chapters are tagged), recent trend (for Dashboard)
- Batch-level performance summary
- Contextual shortcuts: Teacher Workspace opens assessment creation with prefilled context; Student Workspace surfaces linked assessments
- File upload via the shared File Asset module

**Out of Scope**

- Government board registration / external certification authorities
- AI proctoring, identity verification, webcam capture
- OCR of handwritten answer scans
- Advanced plagiarism detection
- Public certificate generation (PDF transcripts beyond an institution's letterhead)
- Generic question bank reuse across institutions
- Marketplace / external question sources

## 3. Actors

- **Teacher** — authors assessments for their assigned `(batch, subject)`; evaluates subjective items and file submissions; publishes results.
- **Student** — views assigned published assessments; takes attempts; views results once published.
- **Admin** — has full authority across the institution: create on behalf of any batch, override published results, configure ranking and result-publication policy.
- **Accountant** — out of scope for this module; the Accountant role does not interact with academic evaluation.
- **System** — auto-evaluates objective items, computes totals/percentages/ranks, enforces attempt window and timer, emits notification and dashboard events.

## 4. Core Concepts

### Assessment

A unit of academic evaluation with one of six `assessment_type` values. Identified by UUID. Always belongs to one `(batch, subject)`; may optionally link to a `chapter`, a `class` (via batch), and a specific occurrence `(schedule_entry_id, class_date)`.

### Assessment Context

The tagging that makes an assessment discoverable from the right places:
- **Required:** `batch_id`, `subject_id`.
- **Optional:** `chapter_id` / `topic_id` (for weak-area insight), `linked_schedule_entry_id` + `linked_class_date` (for class-detail surfacing in Workspaces).

Linked class context is one-way: setting it does not duplicate the assessment, only attaches a navigational link.

### Objective Question

A question evaluated automatically via the stored answer key. Subtypes: `MCQ` (single correct), `MCQ_MULTI` (multiple correct), `TRUE_FALSE`, `SINGLE_ANSWER` (short text matched against an answer list — see OQ for fuzzy matching).

### Subjective Question

A written-answer question evaluated by a teacher. Optional rubric or hint text; total marks set on the question; student response is free text up to a configured length.

### File-Submission Question

A question where the student's "answer" is one or more uploaded files (via File Asset module). Evaluated by the teacher reviewing the file(s). Typical for `ASSIGNMENT` type but allowed in any assessment.

### Attempt

A student's single instance of taking an assessment. Carries `status` (`IN_PROGRESS`, `SUBMITTED`, `EVALUATED`), `started_at`, `submitted_at`, `total_score`. One assessment may allow multiple attempts (configured per assessment); each attempt produces an `AssessmentAttempt` row.

### Answer

A student's response to one question within an attempt. May contain structured answer data (e.g. `{ "selected": ["A", "C"] }` for MCQ), free text, or a `file_asset_id`. Stores `awarded_marks` after evaluation.

### Result

The published summary per (assessment, student): score, percentage, optional rank, published flag. Visible to students only when `published = true`. Created on assessment closure or on publish, depending on policy.

### Publication

The act of releasing computed results to students. Teacher/Admin initiates; once published, students can view their results. A result may be **unpublished** (reverted to hidden) by Admin under exceptional circumstances; see OQ.

### Attempt Window

The time interval during which a student may start and submit. Default: from `open_at` to `close_at` (both required for `LIVE_EXAM` and `MONTHLY_EXAM`; optional for `PRACTICE_QUIZ` and `ASSIGNMENT`). A `duration_minutes` adds a per-attempt clock independent of the window.

## 5. Functional Requirements

### 5.1 Assessment Creation

- **FR-ASM-001:** Teacher shall be able to create an assessment for any `(batch, subject)` pair where a `BatchSubjectTeacher` row exists with `status = ACTIVE` and `teacher_id = caller.user_id`. Admin shall be able to create for any pair regardless.
- **FR-ASM-002:** Every assessment shall have `assessment_type ∈ { PRACTICE_QUIZ, CLASS_TEST, ASSIGNMENT, LIVE_EXAM, MONTHLY_EXAM, MODEL_TEST }`.
- **FR-ASM-003:** Every assessment shall be required to set `batch_id` (existing `ACTIVE` batch) and `subject_id` (must be in the batch's effective subject set per Class Context resolution).
- **FR-ASM-004:** Optional `chapter_id` and `topic_id` shall be supported for weak-area tagging. The catalog of chapters/topics is institution-scoped (see OQ — chapter taxonomy SRS or simple free-form tags in v1).
- **FR-ASM-005:** Optional `linked_schedule_entry_id` + `linked_class_date` shall be supported. When set, the link is read by the Workspaces' "linked assessments" panels.
- **FR-ASM-006:** Every assessment shall have `title`, optional `instructions` (Markdown), and `total_marks` (decimal, positive).
- **FR-ASM-007:** `duration_minutes` (positive integer) is required when `assessment_type ∈ { CLASS_TEST, LIVE_EXAM, MONTHLY_EXAM, MODEL_TEST }`, optional for `PRACTICE_QUIZ` and `ASSIGNMENT`.
- **FR-ASM-008:** `open_at` and `close_at` (timestamps with timezone) define the attempt window. `LIVE_EXAM` and `MONTHLY_EXAM` require both; other types may leave one or both null (open-ended).
- **FR-ASM-009:** Assessments shall be created in `DRAFT` status by default; transition to `PUBLISHED` releases the assessment to assigned students. `PUBLISHED → CLOSED` is automatic at `close_at`, or manual by Teacher/Admin.
- **FR-ASM-010:** A `DRAFT` assessment shall be editable in every field. On a `PUBLISHED` assessment, editability depends on whether any `AssessmentAttempt` row exists:
  - **Always editable on PUBLISHED:** `instructions`, `close_at` extension, adding new questions.
  - **Frozen once any attempt exists:** existing question text, options, marks, answer key, `total_marks`, `duration_minutes`, `open_at`, `max_attempts`, `ranking_enabled`, `answer_selection_policy`.
  A `CLOSED` assessment is fully read-only.
- **FR-ASM-011:** Publishing an assessment shall require `total_marks > 0` and at least one question or a file-submission requirement. Publish-time validations fail with `INVALID_FOR_PUBLISH` listing the issues.

### 5.2 Question Authoring

- **FR-ASM-012:** Teacher/Admin shall be able to add questions of type `OBJECTIVE`, `SUBJECTIVE`, or `FILE_SUBMISSION` to a `DRAFT` assessment (and add new questions to a `PUBLISHED` assessment per FR-ASM-010).
- **FR-ASM-013:** `OBJECTIVE` questions shall carry an `answer_key` (JSON) describing the correct option(s) or expected single answer. Subtypes: `MCQ`, `MCQ_MULTI`, `TRUE_FALSE`, `SINGLE_ANSWER`.
- **FR-ASM-014:** `SUBJECTIVE` questions shall optionally carry a rubric / model-answer text shown to evaluators only.
- **FR-ASM-015:** `FILE_SUBMISSION` questions shall configure: allowed file types (e.g. `pdf,jpg,png`), max file size MB, max number of files.
- **FR-ASM-016:** Every question shall carry `marks` (decimal, positive). The sum of question marks across the assessment shall be validated against `total_marks` at publish; mismatch returns `MARKS_SUM_MISMATCH` listing the delta. Auto-tolerance: ±0.01.
- **FR-ASM-017:** Questions shall support `display_order` (integer) for student-facing ordering.

### 5.3 Student Attempt

- **FR-ASM-018:** A student shall see an assessment in their assigned list when: `status = PUBLISHED`, `batch_id` matches their active `StudentBatch`, current time is between `open_at` (or null) and `close_at` (or null).
- **FR-ASM-019:** A student shall be able to start an attempt within the window. Each call to `POST /assessments/{id}/start` creates an `AssessmentAttempt` row only if the student has not exceeded the configured `max_attempts` (default 1; configurable per assessment).
- **FR-ASM-020:** Time-bound assessments shall display a server-derived deadline equal to `min(now + duration_minutes, close_at)`. The client renders a timer; the server is the authority for "time up."
- **FR-ASM-021:** System shall save attempt progress on every answer write (`PATCH /attempts/{attempt_id}/answers/{question_id}`); progress is durable across browser refreshes (FR-ASM-022).
- **FR-ASM-022:** A student who refreshes the browser mid-attempt shall be able to resume the same `AssessmentAttempt` row by calling `GET /attempts/{attempt_id}`; partially-entered answers persist.
- **FR-ASM-023:** System shall auto-submit an attempt when the server-side deadline elapses, transitioning `IN_PROGRESS → SUBMITTED` with `submitted_at = deadline`. Late client submits within a 30-second grace window after the deadline are accepted (network slack); beyond that, returns `409 ATTEMPT_CLOSED`.
- **FR-ASM-024:** System shall prevent the student from starting a new attempt once `max_attempts` has been reached; `409 MAX_ATTEMPTS_REACHED`.
- **FR-ASM-025:** Student shall be able to submit a `FILE_SUBMISSION` answer by passing a committed `file_asset_id` (committed per File Asset handshake, see `teacher-workspace.md` FR-TCW-018a pattern). Multiple files per question are supported up to the configured `max_files`.
- **FR-ASM-026:** Duplicate `submit` calls on the same attempt shall be idempotent: the first transitions `IN_PROGRESS → SUBMITTED`; subsequent calls return `200` with the existing state.

### 5.4 Evaluation

- **FR-ASM-027:** System shall auto-evaluate all `OBJECTIVE` answers on submission by comparing `answer_data` to the question's `answer_key`. Partial-credit policy for `MCQ_MULTI` is configurable per assessment (default: all-or-nothing).
- **FR-ASM-028:** System shall set the auto-evaluated questions' `awarded_marks` immediately on submission; the attempt's `status` becomes `SUBMITTED` (not yet `EVALUATED`).
- **FR-ASM-029:** Teacher shall be able to evaluate `SUBJECTIVE` and `FILE_SUBMISSION` answers via `POST /attempts/{attempt_id}/evaluate`, supplying `awarded_marks` (≥ 0, ≤ question marks) and optional `remarks`. Authorised callers: the assessment's `created_by` user (even if no longer the active teacher); any user with an `ACTIVE BatchSubjectTeacher` row for the assessment's `(batch, subject)`; Admin (always). Others receive `403 NOT_AUTHORIZED`.
- **FR-ASM-030:** System shall transition an attempt to `EVALUATED` when **all** questions have `awarded_marks` set (including auto-evaluated objective). Partial evaluation keeps the attempt in `SUBMITTED`. For all-objective assessments, the `SUBMITTED → EVALUATED` transition is atomic with submission (FR-ASM-027 sets every `awarded_marks`); the row may transition `IN_PROGRESS → SUBMITTED → EVALUATED` within a single transaction.
- **FR-ASM-031:** System shall expose a pending-evaluation count per teacher per assessment, consumed by Teacher Workspace's linked-assessment panel (FR-TCW-037) and Dashboard.
- **FR-ASM-032:** Teacher shall be able to re-evaluate a question (correction); the new `awarded_marks` replaces the previous. Audit trail recorded.

### 5.5 Result and Publication

- **FR-ASM-033:** System shall compute `AssessmentResult.score` per student as the sum of `AssessmentAnswer.awarded_marks` across the student's most recent `EVALUATED` attempt.
- **FR-ASM-034:** `percentage` shall be `(score / total_marks) × 100` rounded to two decimal places.
- **FR-ASM-035:** `rank` shall be computed per assessment using dense rank (ties share a rank, no gap to the next) when ranking is enabled at the assessment level. Default: ranking on for `MONTHLY_EXAM` and `MODEL_TEST`; off for others. Admin override allowed per assessment.
- **FR-ASM-036:** Results shall be hidden from students (`published = false`) by default. Teacher or Admin publishes via `POST /assessments/{id}/publish-results`. Publication requires the **policy-selected attempt per student** (per `answer_selection_policy`) to be `EVALUATED` for every student who attempted. Older non-canonical attempts may remain `SUBMITTED` without blocking publication. If any required attempt is not yet `EVALUATED`, return `INCOMPLETE_EVALUATION` listing pending student IDs and attempt IDs.
- **FR-ASM-037:** A student shall be able to fetch their own `AssessmentResult` only when `published = true`; otherwise `404 RESULT_NOT_PUBLISHED`.
- **FR-ASM-038:** Admin shall be able to **unpublish** a result with a required `reason` and an audit trail; unpublication hides the result from students. Teacher cannot unpublish (only Admin) — see OQ.
- **FR-ASM-039:** System shall surface the per-student result summary endpoint and the per-assessment leaderboard endpoint (Admin/Teacher) with the same row shape.

### 5.6 Performance Insights

- **FR-ASM-040:** System shall expose a per-student performance summary: assessments taken (count by type), average percentage, trend over the last 6 publications.
- **FR-ASM-041:** Where assessments have `chapter_id` / `topic_id`, System shall compute a weak-area breakdown per student: chapters/topics with average score below 60% (configurable threshold) flagged as weak.
- **FR-ASM-042:** Recent-trend data (last 6 publications per student) shall be exposed in a Dashboard-friendly endpoint.
- **FR-ASM-043:** Teacher and Admin shall be able to view batch-level performance summaries per assessment: distribution buckets (0–40, 40–60, 60–80, 80–100), median, mean.

### 5.7 Workspace Linkage

- **FR-ASM-044:** System shall expose `GET /assessments/by-class?student_id=&schedule_entry_id=&class_date=` returning all assessments where `linked_schedule_entry_id` matches (and optionally `linked_class_date`), plus assessments where `(batch_id, subject_id)` match the occurrence's batch+subject (configurable; default true — see OQ). Used by Student Class Workspace.
- **FR-ASM-045:** System shall expose `GET /assessments/by-class?schedule_entry_id=&class_date=` (without `student_id`) returning the teacher-facing summary list for Teacher Class Workspace's linked-assessment panel (FR-TCW-036).
- **FR-ASM-046:** Assessment shall remain the source of truth for all evaluation logic; the Workspaces are read-only consumers of `(assessment_id, student_status, score)` data.

### 5.8 Event Consumption

- **FR-ASM-047:** System shall transition every `DRAFT` and `PUBLISHED` assessment for a batch to `CLOSED` within 5 s of receiving a `BATCH_ARCHIVED` event for that batch (Class Context FR-CBS-016a). Any `IN_PROGRESS` attempt on those assessments is auto-submitted (`submitted_at = now`, `auto_submitted = true`); subsequent answer-save calls return `409 ASSESSMENT_CLOSED`.
- **FR-ASM-048:** System shall revoke student access on the next list fetch after a `STUDENT_BATCH_CHANGED` event (Class Context FR-CBS-031a). Existing `AssessmentAttempt` and `AssessmentResult` rows are retained; the student can still view their own past results if published, but no longer sees assessments of the old batch in their assigned list.

## 6. Business Rules

- An assessment is bound to exactly one `(batch, subject)` pair; it cannot be shared across batches in v1.
- Linked class context (`linked_schedule_entry_id` + `linked_class_date`) is optional and navigational only; removing the link does not delete the assessment.
- Unpublished assessments and unpublished results are never visible to students.
- Objective auto-evaluation and subjective manual evaluation may coexist on the same assessment.
- The teacher of the linked class is **not** automatically the evaluator; evaluation rights default to the assessment's `created_by` and any teacher with `BatchSubjectTeacher` for `(batch, subject)`. Admin can always evaluate. See OQ for delegation policy.
- File-submission answers reference `file_asset_id`; the File Asset module owns storage; Assessment owns the reference.
- A `PUBLISHED` assessment with no attempts may be edited freely; once one attempt exists, question text and marks are frozen.
- Rank computation uses **dense rank**; ties share a rank.
- A student's most recent `EVALUATED` attempt is the canonical attempt for that assessment. Multi-attempt assessments may show all attempts to the student but only one feeds the published result (configurable: best, last, average — see OQ).
- Class context is optional and should not be mandatory for every assessment.
- Weak-area analysis is only meaningful when chapters/topics are correctly tagged; assessments without tags do not contribute to the weak-area report.

## 7. User Flow / Process Flow

### 7.1 Teacher Creates an Assessment (from Workspace)

1. Teacher opens a class detail in Teacher Class Workspace.
2. Clicks **Create assessment** in the linked-assessments panel.
3. Workspace fetches `GET /teacher/classes/{schedule_entry_id}/assessment-context?class_date=…` (prefill).
4. UI navigates to Assessment Management's create form with `batch_id`, `subject_id`, `linked_schedule_entry_id`, `class_date` prefilled.
5. Teacher selects `assessment_type`, enters `title` + `instructions`, sets `total_marks` and `duration_minutes`, sets `open_at` / `close_at`.
6. Teacher adds questions (objective with answer keys, subjective with optional rubric, file-submission with allowed types).
7. Teacher saves as `DRAFT`. On Publish, system validates marks sum, attempt window, and at least one question.
8. On Publish success: students in the batch see the assessment; if linked to a class, the assessment appears in both Teacher and Student Workspaces' linked panels.

### 7.2 Student Takes an Assessment

1. Student opens an assessment from the Student Workspace's linked panel or the dedicated Assessment list.
2. Reads instructions.
3. Clicks **Start**. System creates `AssessmentAttempt` (`IN_PROGRESS`); server returns the deadline.
4. Student answers questions one by one; each answer write is saved via `PATCH /attempts/{attempt_id}/answers/{question_id}`.
5. For file-submission: student uploads via File Asset module → receives committed `file_asset_id` → submits the answer with the asset id.
6. Student clicks **Submit** (or timer expires; system auto-submits).
7. System auto-evaluates objective items; attempt is `SUBMITTED`.
8. Student sees a "Submitted" confirmation; result is `RESULT_NOT_PUBLISHED` until teacher publishes.

### 7.3 Teacher Evaluates and Publishes

1. Teacher opens the pending-evaluations list (Teacher Workspace or Assessment module).
2. Selects an attempt with subjective or file-submission items.
3. Reviews student response, sets `awarded_marks` and optional `remarks`.
4. Saves. If all questions are evaluated, attempt transitions to `EVALUATED`.
5. After all attempts are `EVALUATED`, Teacher clicks **Publish results**.
6. System computes `score`, `percentage`, optional `rank`; sets `published = true`.
7. Students may now view their result; Notification Management may push an in-app / SMS message.

### 7.4 Admin Unpublish (exceptional)

1. Admin opens the assessment's published results.
2. Clicks **Unpublish** with a required reason.
3. System sets `published = false`; audit row written; students lose access to the result on next fetch.

### 7.5 Class Workspace Discovery (student)

1. Student opens a class detail in Student Workspace.
2. Workspace calls `GET /assessments/by-class?student_id=…&schedule_entry_id=…&class_date=…`.
3. Assessment Management returns linked assessments + assessments matching the batch+subject context, with per-student `student_status`.
4. Student clicks an active assessment → redirect to Assessment Management's attempt flow.

## 8. Data Model

### `Assessment`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| title | string | ≤ 200 chars; Bangla allowed |
| instructions | text, nullable | Markdown; ≤ 5,000 chars |
| assessment_type | enum(`PRACTICE_QUIZ`, `CLASS_TEST`, `ASSIGNMENT`, `LIVE_EXAM`, `MONTHLY_EXAM`, `MODEL_TEST`) | |
| batch_id | UUID, FK → Batch.id | Required |
| subject_id | UUID, FK → Subject.id | Required |
| chapter_id | UUID, nullable | Optional for weak-area |
| topic_id | UUID, nullable | Optional |
| linked_schedule_entry_id | UUID, FK → ScheduleEntry.id, nullable | Navigational link to a class slot |
| linked_class_date | Date, nullable | Used together with `linked_schedule_entry_id` for specific occurrence |
| total_marks | Decimal(8,2) | Positive |
| duration_minutes | integer, nullable | Required for timed types |
| open_at | Timestamp, nullable | Attempt window start |
| close_at | Timestamp, nullable | Attempt window end |
| max_attempts | integer | Default 1 |
| ranking_enabled | boolean | Default per `assessment_type` (FR-ASM-035) |
| answer_selection_policy | enum(`LAST`, `BEST`, `AVERAGE`) | Default `LAST` for multi-attempt assessments |
| status | enum(`DRAFT`, `PUBLISHED`, `CLOSED`) | |
| created_by | UUID, FK → User.id | Author |
| updated_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| published_at | Timestamp, nullable | Set on DRAFT → PUBLISHED |
| closed_at | Timestamp, nullable | Set on PUBLISHED → CLOSED |

### `AssessmentQuestion`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| assessment_id | UUID, FK → Assessment.id | |
| question_type | enum(`OBJECTIVE`, `SUBJECTIVE`, `FILE_SUBMISSION`) | |
| objective_subtype | enum(`MCQ`, `MCQ_MULTI`, `TRUE_FALSE`, `SINGLE_ANSWER`), nullable | For OBJECTIVE only |
| question_text | text | Bangla allowed |
| options | JSONB, nullable | For MCQ / MCQ_MULTI: array of `{ key, text }` |
| answer_key | JSONB, nullable | For OBJECTIVE: structured correct answer; for SUBJECTIVE: optional rubric/model answer; for FILE_SUBMISSION: file rules |
| rubric | text, nullable | Evaluator-visible only |
| marks | Decimal(6,2) | Positive |
| display_order | integer | |
| created_at | Timestamp | |

### `AssessmentAttempt`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| assessment_id | UUID, FK → Assessment.id | |
| student_id | UUID, FK → User.id | |
| attempt_number | integer | 1-indexed; ≤ `max_attempts` |
| started_at | Timestamp | |
| deadline_at | Timestamp | Server-derived `min(started_at + duration, close_at)` |
| submitted_at | Timestamp, nullable | |
| status | enum(`IN_PROGRESS`, `SUBMITTED`, `EVALUATED`) | |
| total_score | Decimal(8,2), nullable | Set on transition to EVALUATED |
| auto_submitted | boolean | True when system auto-submits on timer expiry |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| | | Unique: `(assessment_id, student_id, attempt_number)` |

### `AssessmentAnswer`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| attempt_id | UUID, FK → AssessmentAttempt.id | |
| question_id | UUID, FK → AssessmentQuestion.id | |
| answer_data | JSONB, nullable | OBJECTIVE: `{ "selected": [...] }`; SUBJECTIVE: `{ "text": "..." }` |
| file_asset_ids | UUID[], nullable | FILE_SUBMISSION; up to `max_files` |
| awarded_marks | Decimal(6,2), nullable | Set after auto- or manual evaluation |
| evaluator_remarks | text, nullable | Manual evaluator notes |
| evaluated_by | UUID, FK → User.id, nullable | |
| evaluated_at | Timestamp, nullable | |
| updated_at | Timestamp | |
| | | Unique: `(attempt_id, question_id)` |

### `AssessmentResult`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| assessment_id | UUID, FK → Assessment.id | |
| student_id | UUID, FK → User.id | |
| attempt_id | UUID, FK → AssessmentAttempt.id | The attempt selected by `answer_selection_policy` |
| score | Decimal(8,2) | |
| percentage | Decimal(5,2) | Two decimal places |
| rank | integer, nullable | Set when `ranking_enabled` |
| published | boolean | Default false |
| published_at | Timestamp, nullable | |
| unpublished_at | Timestamp, nullable | Set when Admin reverts |
| unpublish_reason | string, nullable | Required when unpublished |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| | | Unique: `(assessment_id, student_id)` |

> **Multi-tenancy:** Per ADR-0003 (accepted, shared-schema), `Assessment`, `AssessmentQuestion`, `AssessmentAttempt`, `AssessmentAnswer`, and `AssessmentResult` each carry a `tenant_id` column (UUID, NOT NULL, FK → `Tenant.id`). All uniqueness constraints are tenant-scoped: `(tenant_id, assessment_id, student_id, attempt_number)` on `AssessmentAttempt`; `(tenant_id, attempt_id, question_id)` on `AssessmentAnswer`; `(tenant_id, assessment_id, student_id)` on `AssessmentResult`. Performance summaries and batch comparisons are computed within a single tenant only.

## 9. API Contracts

All endpoints require `Authorization: Bearer <access_token>`. Teacher endpoints reject non-teacher callers; student endpoints reject non-student callers.

### Create Assessment

```http
POST /assessments
```

**Request**

```json
{
  "title": "Class Test — Algebra",
  "assessment_type": "CLASS_TEST",
  "batch_id": "0192...",
  "subject_id": "0192...",
  "chapter_id": "0192...",
  "linked_schedule_entry_id": "0192...",
  "linked_class_date": "2026-05-18",
  "total_marks": 20,
  "duration_minutes": 30,
  "open_at": "2026-05-18T10:00:00+06:00",
  "close_at": "2026-05-20T22:00:00+06:00",
  "max_attempts": 1,
  "ranking_enabled": false,
  "answer_selection_policy": "LAST"
}
```

**Response (201)** — full assessment object.

**Errors**

- `400` — `INVALID_INPUT`
- `403` — `NOT_AUTHORIZED` (caller is not the active teacher for `(batch, subject)` and not Admin)
- `409` — `BATCH_INACTIVE` / `SUBJECT_NOT_IN_BATCH`

Satisfies: FR-ASM-001 to FR-ASM-008.

### List Assessments (teacher / admin)

```http
GET /assessments?status=&batch_id=&subject_id=&type=&cursor=&limit=
```

Returns assessments authored by the caller (Teacher) or any assessment (Admin), filtered by status, batch, subject, type.

**Response (200)** — list of summary rows: `{ id, title, type, batch_id, subject_id, status, total_marks, open_at, close_at, attempt_count, pending_evaluation_count, result_published }`.

Satisfies: FR-ASM-001, FR-ASM-031.

### Get Assessment (read)

```http
GET /assessments/{assessment_id}
```

Returns the full assessment object including questions. For students, questions' `answer_key` and `rubric` are omitted from the response.

Satisfies: read side of FR-ASM-010, FR-ASM-018.

### Update Assessment

```http
PATCH /assessments/{assessment_id}
```

Editable fields depend on status per FR-ASM-010.

**Errors**

- `400` — `INVALID_INPUT` / `FIELD_FROZEN` (attempt to edit a frozen field on PUBLISHED-with-attempts)
- `403` — `NOT_AUTHORIZED`
- `409` — `ASSESSMENT_CLOSED`

Satisfies: FR-ASM-010.

### Close Assessment (manual)

```http
POST /assessments/{assessment_id}/close
```

Transitions `PUBLISHED → CLOSED`. Teacher (creator or active `BatchSubjectTeacher`) or Admin. Open attempts are auto-submitted per FR-ASM-047 semantics.

**Errors**

- `409` — `INVALID_STATUS` (only PUBLISHED may be closed manually)

Satisfies: FR-ASM-009.

### Resume Attempt

```http
GET /attempts/{attempt_id}
```

Returns the in-progress attempt with all currently saved answers and the server-derived `deadline_at`. Allows the student to resume after a refresh or device switch.

**Response (200)**

```json
{
  "attempt_id": "0192...",
  "assessment_id": "0192...",
  "status": "IN_PROGRESS",
  "started_at": "...",
  "deadline_at": "...",
  "answers": [
    { "question_id": "0192...", "answer_data": { "selected": ["B"] }, "file_asset_ids": null }
  ]
}
```

**Errors**

- `403` — `NOT_OWNER` (caller is not the student of this attempt)
- `409` — `ATTEMPT_COMPLETED` (already SUBMITTED or EVALUATED; use a different endpoint to view result)

Satisfies: FR-ASM-022.

### Add Question

```http
POST /assessments/{assessment_id}/questions
```

**Request**

```json
{
  "question_type": "OBJECTIVE",
  "objective_subtype": "MCQ",
  "question_text": "What is 2 + 2?",
  "options": [{"key":"A","text":"3"},{"key":"B","text":"4"},{"key":"C","text":"5"},{"key":"D","text":"22"}],
  "answer_key": {"correct": ["B"]},
  "marks": 1,
  "display_order": 1
}
```

**Response (201)** — full question object.

**Errors**

- `400` — `INVALID_INPUT` / `MARKS_INVALID`
- `403` — `NOT_AUTHORIZED`
- `409` — `ASSESSMENT_CLOSED` / `QUESTION_LOCKED` (PUBLISHED + has attempts; FR-ASM-010)

Satisfies: FR-ASM-012 to FR-ASM-017.

### Publish Assessment

```http
POST /assessments/{assessment_id}/publish
```

**Response (200)** — `{ "id": "0192...", "status": "PUBLISHED", "published_at": "..." }`.

**Errors**

- `409` — `INVALID_FOR_PUBLISH` (lists issues: no questions, marks mismatch, missing required fields) / `MARKS_SUM_MISMATCH`

Satisfies: FR-ASM-009, FR-ASM-011.

### Get Student Assessments (assigned list)

```http
GET /student/assessments?status=ACTIVE&subject_id=&cursor=&limit=
```

`status` filter: `ACTIVE` (current attempt window), `UPCOMING` (open_at in future), `SUBMITTED`, `RESULT_PUBLISHED`, `CLOSED`.

**Response (200)** — list of `{ assessment_id, title, type, subject_name, open_at, close_at, duration_minutes, student_status, score?, percentage?, attempt_id? }`.

Satisfies: FR-ASM-018.

### Start Attempt

```http
POST /assessments/{assessment_id}/start
```

**Response (201)**

```json
{
  "attempt_id": "0192...",
  "started_at": "2026-05-18T10:05:00+06:00",
  "deadline_at": "2026-05-18T10:35:00+06:00",
  "questions": [ ... ]
}
```

**Errors**

- `403` — `NOT_IN_BATCH`
- `409` — `ATTEMPT_WINDOW_CLOSED` / `MAX_ATTEMPTS_REACHED` / `ASSESSMENT_NOT_PUBLISHED`

Satisfies: FR-ASM-019, FR-ASM-020.

### Save Answer (incremental)

```http
PATCH /attempts/{attempt_id}/answers/{question_id}
```

**Request**

```json
{ "answer_data": { "selected": ["B"] } }
```

or

```json
{ "answer_data": { "text": "Long written answer..." } }
```

or

```json
{ "file_asset_ids": ["0192...","0192..."] }
```

**Response (200)** — updated answer object.

Satisfies: FR-ASM-021, FR-ASM-022, FR-ASM-025.

### Submit Attempt

```http
POST /attempts/{attempt_id}/submit
```

**Response (200)** — `{ "attempt_id": "0192...", "status": "SUBMITTED", "submitted_at": "...", "auto_submitted": false }`.

Idempotent: re-calling on a SUBMITTED or EVALUATED attempt returns 200 with the existing state.

**Errors**

- `409` — `ATTEMPT_CLOSED` (deadline passed by > 30 s)

Satisfies: FR-ASM-023, FR-ASM-026, FR-ASM-027.

### Evaluate Attempt (per question)

```http
POST /attempts/{attempt_id}/evaluate
```

**Request**

```json
{
  "evaluations": [
    { "question_id": "0192...", "awarded_marks": 4.5, "remarks": "Good explanation" }
  ]
}
```

**Response (200)** — updated answer rows; if all questions are now evaluated, the attempt transitions to `EVALUATED` and `total_score` is set.

**Errors**

- `400` — `MARKS_OUT_OF_RANGE` (awarded > question.marks)
- `403` — `NOT_AUTHORIZED`

Satisfies: FR-ASM-029, FR-ASM-030, FR-ASM-031, FR-ASM-032.

### Publish Results

```http
POST /assessments/{assessment_id}/publish-results
```

**Response (200)** — `{ "assessment_id": "0192...", "published_count": 28 }`.

**Errors**

- `409` — `INCOMPLETE_EVALUATION` (lists pending attempts)

Satisfies: FR-ASM-036.

### Admin Unpublish Result

```http
POST /assessments/{assessment_id}/unpublish-results
```

**Request**

```json
{ "reason": "Question 3 had an incorrect answer key" }
```

Satisfies: FR-ASM-038.

### Get Student Result

```http
GET /student/results/{assessment_id}
```

**Response (200)**

```json
{
  "assessment_id": "0192...",
  "title": "Class Test — Algebra",
  "score": 18,
  "percentage": 90.00,
  "rank": 3,
  "total_marks": 20,
  "remarks": null,
  "weak_areas": [
    { "chapter_id": "0192...", "name": "Quadratic equations", "score_percentage": 50.0 }
  ]
}
```

**Errors**

- `404` — `RESULT_NOT_PUBLISHED`
- `403` — `NOT_IN_BATCH`

Satisfies: FR-ASM-037, FR-ASM-041.

### By-Class Assessment Listing (Workspaces)

```http
GET /assessments/by-class?schedule_entry_id=&class_date=&student_id=
```

Returns linked assessments (via `linked_schedule_entry_id`) plus `(batch, subject)`-matched assessments (configurable). For Student Workspace, `student_id` enables per-student `student_status` computation. For Teacher Workspace, omit `student_id`.

**Response (200)**

```json
{
  "linked_assessments": [
    {
      "assessment_id": "0192...",
      "title": "Chapter 4 quiz",
      "type": "PRACTICE_QUIZ",
      "status": "PUBLISHED",
      "open_at": "2026-05-18T08:00:00+06:00",
      "close_at": "2026-05-20T22:00:00+06:00",
      "submission_count": 12,
      "pending_evaluation_count": 4,
      "result_status": "NOT_PUBLISHED",
      "student_status": "ACTIVE",
      "due_at": "2026-05-20T22:00:00+06:00",
      "score_available": false
    }
  ]
}
```

`student_status` is included only when `student_id` is provided. Possible values: `UPCOMING`, `ACTIVE`, `IN_PROGRESS`, `SUBMITTED`, `RESULT_PUBLISHED`, `CLOSED`.

Satisfies: FR-ASM-044, FR-ASM-045.

### Performance Summary (student dashboard)

```http
GET /student/performance/summary
```

Returns counts by type, average percentage, trend (last 6 publications), weak areas.

Satisfies: FR-ASM-040, FR-ASM-041, FR-ASM-042.

### Batch Performance (teacher/admin)

```http
GET /assessments/{assessment_id}/batch-performance
```

Returns distribution buckets, median, mean.

Satisfies: FR-ASM-043.

## 10. UI Components

### Screen: Teacher Assessment List

- **Components:** Tabs (Draft / Published / Closed); filter by subject and batch; create button; pending-evaluation badge per row.
- **Actions:** Create, edit, evaluate, publish results.
- **States:** loading, empty, populated.

### Screen: Assessment Create / Edit

- **Components:** Type selector (radio of 6 types); batch + subject pickers (prefilled from Workspace context); optional chapter/topic; total marks; duration; open/close times; instructions Markdown editor; question builder (drag-to-reorder list, modal per type); rubric input for subjective; file-rules input for file-submission.
- **Actions:** Save draft, publish.
- **States:** idle, saving, publishing, validation-error (per-field).

### Screen: Student Assessment List

- **Components:** Tabs (Active / Upcoming / Submitted / Results); subject filter; per-card status (`UPCOMING` / `ACTIVE` / `IN_PROGRESS` / `SUBMITTED` / `RESULT_PUBLISHED`).
- **Actions:** Open, Continue, View result.
- **States:** loading, empty, populated.

### Screen: Attempt

- **Components:** Question list with display order; per-question answer area (radio for MCQ, textarea for subjective, file uploader for file-submission); timer (when timed); auto-save indicator; submit button.
- **Actions:** Answer, save (auto), upload file, submit.
- **States:** in-progress, deadline-approaching (last 5 min), submitting, submitted, auto-submitted.

### Screen: Evaluation (Teacher)

- **Components:** Student list with submission status; per-attempt detail (questions + student answers); awarded-marks input per question; remarks textarea; objective items marked auto-evaluated; save evaluation; bulk-publish button.
- **Actions:** Evaluate per question, save, publish all results.

### Screen: Result (Student)

- **Components:** Score summary card (total, percentage, rank if enabled); remarks if any; weak-areas chart (if chapters tagged); per-question breakdown (correct / incorrect / your-answer / correct-answer for objective; teacher remarks for subjective).
- **Actions:** Open class link if assessment was linked to a class.

## 11. Validation Rules

- **Title:** required, 1–200 chars, Bangla allowed.
- **Batch / Subject:** required; subject must be in batch's effective set; cross-call to Class Context at create.
- **Total marks:** decimal, > 0.
- **Duration:** positive integer when required by `assessment_type`.
- **Open / close window:** if both set, `open_at < close_at`; `close_at > now` at publish time.
- **Question marks sum:** equals `total_marks` ± 0.01 at publish time.
- **Objective answer key:** required and structurally valid for the subtype (e.g. `MCQ` requires exactly one entry in `correct`).
- **Awarded marks:** ≥ 0, ≤ question.marks.
- **File submission:** asset must be `COMMITTED` in File Asset and owned by the caller; type / size / count per question configuration.
- **Attempt start:** student must be in batch; `now ∈ [open_at, close_at]`; `attempt_number ≤ max_attempts`.
- **Submit:** allowed up to deadline + 30 s grace; thereafter `409 ATTEMPT_CLOSED`; auto-submit applies on the deadline.
- **Publish:** all attempts EVALUATED; `total_marks > 0`; at least one question.
- **Unpublish reason:** required string, ≤ 500 chars; Admin-only.

## 12. Edge Cases

- **Timer expires during file upload** → auto-submit at the deadline finalises whatever answers are saved; partial uploads are not retained as the answer; the question is unanswered for evaluation. Student sees "Submitted at <time> (auto)."
- **Student refreshes browser mid-attempt** → resume via `GET /attempts/{attempt_id}`; partial answers persist (FR-ASM-022); timer reflects remaining time (server-derived).
- **Teacher tries to publish results before all evaluations are done** → `409 INCOMPLETE_EVALUATION` with pending list; Teacher must finish first.
- **Linked class is replaced (Scheduling FR-SCH-005)** → `linked_schedule_entry_id` may point at an archived entry; the assessment remains valid and surfaces under the archived occurrence's history. Future occurrences of the same slot do not auto-link.
- **Assignment file missing after submission** (file purged by Admin override or retention) → student's `file_asset_ids` array references the now-missing asset; evaluator sees "File no longer available"; can still award marks based on prior review notes if any. Audit trail captures the missing reference.
- **Duplicate submit calls** → idempotent per FR-ASM-026; first call wins state transition.
- **Ranking tie** → dense rank — same rank shared, next rank is immediately next integer. Example: 100, 95, 95, 90 → ranks 1, 2, 2, 3.
- **Student batch changes after assessment publication** → student loses access to the assessment from their workspace (no longer in the assigned batch). Existing `AssessmentAttempt` rows remain in history; student can still see their result on the result screen if published, because access control on results uses `student_id`, not current batch. (See OQ — revisit if this is undesirable.)
- **Two teachers both authorized for `(batch, subject)` evaluate the same question simultaneously** → last-write-wins on `awarded_marks`; audit row captures each write. Re-evaluation is allowed per FR-ASM-032.
- **Admin unpublishes after students viewed result** → students lose access on next fetch; result was likely cached client-side; UI shows "Result temporarily unavailable" until republished.
- **`SINGLE_ANSWER` objective question** with student answer differing in whitespace or case → exact match in v1; case-insensitive and whitespace-trimmed match flagged as OQ.
- **Student takes multiple attempts and `answer_selection_policy = BEST`** → result picks the EVALUATED attempt with highest `total_score`. If two attempts tie at the highest score, the earlier one wins.
- **Open-ended assessment (`open_at` null, `close_at` null)** → `PRACTICE_QUIZ` can be taken any time after publish; never auto-closes; teacher must explicitly close via FR-ASM-009.
- **Network failure during answer save** → client retries on the same `PATCH` (idempotent on the same `(attempt_id, question_id)`); persistent failure surfaces as "Answer not saved — please retry."
- **Auto-submit fires but the server has stored stale answers (some unsynced from the client)** → only synced answers contribute; unsynced answers are lost. UI warns the student in the last 30 s before deadline if there are unsaved changes.
- **Bangla input in subjective answer** → fully supported; storage is UTF-8 text; rendering uses the institution's chosen Bangla font.
- **Concurrent attempts by the same student from two devices** → only one attempt is `IN_PROGRESS` at any time per the unique `(assessment_id, student_id, attempt_number)` rule + a server-side check; the second device's `start` call returns `409 ATTEMPT_IN_PROGRESS` and a redirect to the existing attempt.
- **File-submission with `max_files = 1` and student uploads a different file before submit** → the new `file_asset_ids` replaces the prior; the prior asset is not auto-deleted (File Asset retention applies).
- **Teacher edits a question on a PUBLISHED assessment after the first attempt** → FR-ASM-010 blocks; teacher must either close the assessment or add a new corrective question. To fix an incorrect answer key after attempts exist, Admin may unpublish results, the teacher re-evaluates per FR-ASM-032, and Admin republishes (manual recovery flow).
- **Recording or class linkage missing when student opens linked-assessment context** → linkage is decorative; the assessment is taken regardless. The Student Workspace's "Open from class" deeplink still works.
- **`BATCH_ARCHIVED` event received while attempts are `IN_PROGRESS`** → per FR-ASM-047, the assessment auto-closes within 5 s; open attempts are auto-submitted with whatever answers are saved (`auto_submitted = true`). Students mid-attempt see a "Class archived — your attempt has been submitted" message on the next answer-save call.
- **Teacher account deactivated while PUBLISHED assessment has pending evaluations** → the assessment's `created_by` remains valid (FK is not deleted), but the teacher cannot evaluate. Admin or another active `BatchSubjectTeacher` for `(batch, subject)` must complete the evaluation. If neither exists, Admin is the only path.
- **`close_at` extended after auto-close fired** → FR-ASM-010 allows extension while PUBLISHED; auto-close to CLOSED is terminal. To "extend" a closed assessment, Teacher must create a follow-up assessment (no resurrect path in v1; documented to avoid implementer confusion).
- **`answer_selection_policy = AVERAGE` with one EVALUATED and one still SUBMITTED attempt** → FR-ASM-036's gate fires (publication is blocked) because the selection requires all multi-attempt rows to be EVALUATED for AVERAGE to compute. Teacher must finish evaluating the SUBMITTED attempt before publishing.
- **Multi-attempt + file-submission overwrite race** → when a student starts attempt N+1 while attempt N is still being evaluated, the new attempt's file references are independent (different `attempt_id`); the evaluator's view is pinned to the specific `attempt_id` they opened. No data loss.
- **`SINGLE_ANSWER` objective with null `answer_data`** → student never answered; auto-evaluation sets `awarded_marks = 0`; no error.
- **`total_marks` mismatch with question sum after question edits on DRAFT** → publish-time validation (FR-ASM-016 ± 0.01) catches it; teacher must adjust either `total_marks` or question marks before publishing.

## 13. Dependencies

- **Class Context (`class-context.md`)** — provides `BatchSubjectTeacher` authority for create, batch's effective subject set for `subject_id` validation, `StudentBatch` for student access. Consumes events: `BATCH_ARCHIVED` (auto-close all DRAFT/PUBLISHED assessments for the batch), `STUDENT_BATCH_CHANGED` (revoke student's access on next list fetch).
- **Class Scheduling (`scheduling.md`)** — referenced for `linked_schedule_entry_id` validity; not a writer.
- **Teacher Class Workspace (`teacher-workspace.md`)** — initiator of contextual creation; consumer of per-class linked-assessment summaries via FR-ASM-045.
- **Student Class Workspace (`student-workspace.md`)** — consumer of per-class linked assessments via FR-ASM-044 (`student_id`-scoped).
- **Dashboard System** — consumer of FR-ASM-040 / FR-ASM-042 endpoints.
- **Notification Management** — consumer of `ASSESSMENT_PUBLISHED`, `ASSESSMENT_RESULT_PUBLISHED`, `ASSESSMENT_DUE_SOON` (24 h / 1 h pre-close) events.
- **File Asset module** (shared, schema TBD) — stores file submissions and any teacher-attached files for evaluation.
- **Authentication & Authorization (`auth.md`)** — RBAC.
- **Central `audit_log`** — receives publish/unpublish events, re-evaluation events, admin overrides.
- **Redis** — caches attempt deadlines, max-attempt counts, per-student `student_status` for quick Workspace responses; not a source of truth.

## 14. Non-Functional Requirements

- **Performance:** assessment list (student) p95 < 1.5 s; start attempt p95 < 1 s; answer save (PATCH) p95 < 500 ms; submit p95 < 2 s; publish-results p95 < 3 s for up to 100 students.
- **Timer stability:** the server is authoritative on the deadline; the client computes display time from `deadline_at` returned at start, with a single server timestamp echoed in every answer-save response to correct clock drift.
- **Weak-network tolerance:** answer-save retries with exponential backoff; the client warns the student in the last 30 s if there are unsaved answers; auto-submit can succeed with only synced answers.
- **Concurrency:** unique `(attempt_id, question_id)` on `AssessmentAnswer` and unique `(assessment_id, student_id, attempt_number)` on `AssessmentAttempt` enforce no double-submission.
- **Mobile-first:** all student screens responsive down to 360 px; file-uploader supports the institution's S3-compatible storage's resumable upload protocol if available.
- **Accessibility:** WCAG 2.1 AA on all student-facing screens. Attempt screen requires keyboard-accessible answer selection (radio / checkbox), screen-reader-compatible timer announcements (live region updates at 5-min, 1-min, 30-sec, 10-sec marks), and clearly labelled form fields. Bangla content rendered with appropriate language attributes for screen readers.
- **Internationalization:** title, instructions, question text, options, remarks all support Bangla.
- **Auditability:** every publish, unpublish, mark change, and admin override writes to `audit_log`.
- **Scale:** up to 5,000 students per batch event (monthly exam); up to 200 concurrent live exams per institution.
- **Storage:** answer data inline in JSONB; file submissions in File Asset's S3-compatible store.
- **Time zone:** all timestamps stored UTC; presented in BST (+06:00) for UI.

## 15. Assumptions

- Assessments are batch + subject driven; cross-batch / cross-subject is not in v1.
- Linked class context is optional and used only for navigation.
- The institution's chapter/topic catalog exists (free-form text for v1; structured chapter SRS in future).
- Many students take assessments on mobile (low-end Android); the attempt screen is engineered accordingly.
- Live exams happen during scheduled class times for the batch; the LMS does not separately enforce physical proctoring.
- File submissions are reviewed manually; no OCR or AI evaluation in v1.
- A student belongs to exactly one batch at a time (Class Context invariant); assessments inherit this scoping.

## 16. Open Questions

- **Max attempts default per type:** v1 default = 1. Should `PRACTICE_QUIZ` default to unlimited? Confirm with academic team.
- **Show solutions immediately on submit, or only after result publication?** Two camps: instant feedback vs. teacher-controlled. v1 default = after publication.
- **Class-linked assessments auto-appear in class detail** (FR-ASM-044 second clause): v1 default = batch+subject match contributes to the linked list. Some institutions want strict-linked-only; configurable per institution.
- **Single-answer fuzzy matching:** exact string match in v1; case-insensitive and whitespace-trim are common asks. Confirm.
- **Chapter/topic taxonomy:** free-form tags in v1, or a structured `Chapter` SRS? Weak-area report is more meaningful with structure.
- **Evaluator delegation:** can a teacher delegate evaluation to a TA / sub-teacher? Currently only the active `BatchSubjectTeacher` (and Admin) may evaluate.
- **Teacher unpublish vs. Admin-only:** FR-ASM-038 restricts unpublish to Admin. Some teachers want self-service unpublish for own assessments. Confirm.
- **Multi-attempt result selection policy:** v1 supports `LAST`, `BEST`, `AVERAGE`. Which is default per type?
- **Student loses access after batch change but keeps result history** — confirm acceptable, or should past assessments remain accessible to the student forever?
- **Auto-evaluation partial credit for `MCQ_MULTI`:** v1 = all-or-nothing; partial-credit policy is configurable. Default per type?
- **Notification cadence:** "due soon" reminders at 24 h and 1 h pre-close. Some institutions want SMS pushes; confirm channel and frequency caps.
- **`PUBLISHED` → `DRAFT` revert:** when a publish was made in error, can Teacher pull back to DRAFT before any attempt? v1 default = no; Teacher must close and recreate.
- **File-submission storage retention:** how long after assessment closes are uploaded files retained? Cost vs. transcript-quality trade-off.
- **Cross-question rubric / mark distribution within one subjective question** — currently flat `awarded_marks` per question; some teachers want per-criterion sub-marks. Out of v1.
