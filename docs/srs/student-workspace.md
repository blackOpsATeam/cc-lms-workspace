# 🧩 MODULE: STUDENT CLASS WORKSPACE

## 1. Purpose

Provide each student with a class-centric home: the list of classes attached to their active batch grouped into Today / Upcoming / Previous, a per-occurrence detail page with teacher-published instructions and resources, a join action for live classes, access to prerecorded support clips and class recordings, and a panel surfacing class-linked assessments. The Student Class Workspace is the student's daily entry point; everything written here is sourced from other modules (Scheduling, Teacher Class Workspace, Live Class, Assessment) — this module is **read-only from the student's perspective** and never mutates schedule, attendance, or assessment state.

## 2. Scope

**In Scope**

- Effective class list for the student's currently active batch, grouped Today / Upcoming / Previous
- Per-occurrence detail rendering: subject, teacher, time, mode, room or live ref, teacher-published instructions, student-visible resources, prerecorded clips, linked recording (for past classes)
- Live class join action gated by a time window and Live Class session state
- Access-control filtering by `StudentBatch` membership (Class Context)
- Linked assessment panel: discovery and one-click open of class-relevant assessments
- Continue-attempt and view-result shortcuts for linked assessments
- Cancelled / updated occurrence badges
- Filters (subject, date range) and search by subject name

**Out of Scope**

- Schedule mutation — Scheduling
- Resource and instruction *authoring* — Teacher Class Workspace
- Live session execution — Live Class Management
- Attendance capture, evaluation, results computation — Attendance / Assessment
- Generic content library — resources are class-linked only
- File storage internals — File Asset module
- Teacher-private notes — never exposed here
- Cross-batch access (a student cannot view another batch's classes)

## 3. Actors

- **Student** — primary user; reads classes from their active batch, opens detail, clicks redirects to Live Class / Assessment.
- **System** — resolves the student's effective class list from Class Context + Scheduling, applies overrides, filters resources by `visibility_status = PUBLISHED` and cancellation rules, gates live join by time window.
- **Teacher** — indirect contributor (writes via Teacher Class Workspace; visible here as the assigned teacher per occurrence).
- **Admin** — indirect actor (owns the published schedule, batch placement, role policies); does not directly use this module.

## 4. Core Concepts

### Effective Class Occurrence

A class occurrence resolved from Scheduling, restricted to the calling student's active batch (per `StudentBatch.left_at IS NULL`), with overrides applied. Identified by `(schedule_entry_id, class_date)` — the same canonical identity used by `scheduling.md` (where it is also called `occurrence_date`) and `teacher-workspace.md`. The student sees only `PUBLISHED` routines; `DRAFT` routines are invisible.

### Class Detail

The student-facing rendering of one occurrence: subject, teacher, time, mode, room or live link, `STUDENT_VISIBLE` teacher instructions (filtered from `ClassInstruction`), `PUBLISHED` resources (filtered from `ClassResource`), prerecorded clip panel, linked recording (for past occurrences), live join action (for current occurrences), linked assessments panel.

### Student-Visible Resource

A `ClassResource` row (owned by Teacher Class Workspace) where `visibility_status = PUBLISHED` and the occurrence is not cancelled (FR-SCW-022a). For `apply_scope = RECURRING_SLOT`, the resource appears on every non-cancelled occurrence; for `apply_scope = OCCURRENCE`, only on the matching `class_date`.

### Join Window

A time interval around an `ON_SITE`/`LIVE_ONLINE`/`HYBRID` occurrence during which the join action is enabled. Default: from `start_time − 10 min` to `end_time + 30 min` (BST), configurable. Outside the window the action is hidden or shows a not-started / ended state; inside the window the action is enabled if Live Class Management reports the session is joinable.

### Linked Assessment

An assessment (owned by Assessment Management) associated with the `(schedule_entry_id, class_date?)`. The student sees it in the class detail's assessment panel; eligibility (whether the student may attempt it) is decided by Assessment Management. This module surfaces the link and the student's per-assessment status; it never decides eligibility.

### Display Phase

A per-occurrence derived time marker:
- `UPCOMING` — `class_date > today`, or same-day before `start_time`.
- `ONGOING` — current time inside the join window for a live/hybrid mode, or between `start_time` and `end_time` for on-site.
- `PREVIOUS` — `class_date < today`, or same-day after `end_time + 30 min` (BST).

### Override Status

A per-occurrence derived override marker, independent of time:
- `NORMAL` — no `ScheduleOverride` exists for `(schedule_entry_id, class_date)`.
- `UPDATED` — `ScheduleOverride.status = UPDATED` exists; one or more fields differ from the base entry.
- `CANCELLED` — `ScheduleOverride.status = CANCELLED` exists.

Both `display_phase` and `override_status` are computed by this module and returned as separate fields on every occurrence response. They are not stored.

## 5. Functional Requirements

### 5.1 Class List

- **FR-SCW-001:** System shall list, for the calling student, only occurrences whose `batch_id` matches the student's active `StudentBatch.batch_id` (the row with `left_at IS NULL`). Cross-batch leakage is rejected at the query layer.
- **FR-SCW-002:** System shall group occurrences into `Today`, `Upcoming` (date > today, default ≤ 14 days ahead), `Previous` (date < today, default ≥ 90 days back). Default windows are configurable client-side.
- **FR-SCW-003:** System shall apply Scheduling's `ScheduleOverride` resolution before rendering. Each occurrence response carries both `display_phase` (UPCOMING/ONGOING/PREVIOUS) and `override_status` (NORMAL/UPDATED/CANCELLED) as separate fields. Override resolution may also change the occurrence's time, teacher, mode, or room, which are reflected in the response fields directly.
- **FR-SCW-004:** System shall hide occurrences whose underlying routine is in `DRAFT` (per `scheduling.md` FR-SCH-006).
- **FR-SCW-005:** System shall visually mark occurrences with `override_status ∈ { UPDATED, CANCELLED }` distinctly from `NORMAL`; cancelled occurrences are sorted to the end of their `display_phase` group.
- **FR-SCW-006:** Each list item shall include `(batch_id, batch_name, class_name, subject_id, subject_name, teacher_id, teacher_name, date, day_of_week, start_time, end_time, delivery_mode, room_name?, live_session_ref?, display_phase, override_status, reason?)`.
- **FR-SCW-006a:** When a student is moved to a different batch (`StudentBatch` change via `class-context.md` FR-CBS-030), the workspace shall serve the **new** batch's effective list from the next fetch after the `STUDENT_BATCH_CHANGED` event is consumed (SLA ≤ 5 s after publication). The student's view of past classes remains scoped to their batch *at the time of each class* (see FR-SCW-024a). The `GET /student/classes` endpoint is the testable surface for this SLA.

### 5.2 Class Detail

- **FR-SCW-007:** Student shall be able to open a detail view for any occurrence in their list, identified by `(schedule_entry_id, class_date)`. The system rejects with `403 NOT_IN_BATCH` if the student was not in the occurrence's batch on `class_date` (per the batch-at-the-time rule in FR-SCW-024a).
- **FR-SCW-008:** Detail shall include all fields from FR-SCW-006 plus: student-visible instructions, student-visible resources, prerecorded clip panel, linked recording panel (PREVIOUS only), live join action (ONGOING/UPCOMING live/hybrid), linked assessments panel.
- **FR-SCW-009:** Student shall see only `ClassInstruction` rows where `instruction_type = STUDENT_VISIBLE` and not soft-deleted; teacher-private notes are filtered at the API layer and never returned.
- **FR-SCW-010:** Student shall see only `ClassResource` rows where `visibility_status = PUBLISHED` and not soft-deleted; `DRAFT` and `HIDDEN` resources are filtered at the API layer.
- **FR-SCW-011:** Student shall see class-linked prerecorded clips (`resource_type = PRERECORDED_CLIP` with `visibility_status = PUBLISHED`) in a distinct panel above generic resources.
- **FR-SCW-012:** For `PREVIOUS` occurrences, detail shall display the linked recording (sourced from Live Class Management's recording entity) when its `recording_status = AVAILABLE`.
- **FR-SCW-013:** System shall **never** return `TEACHER_PRIVATE` instructions on the student endpoint, regardless of caller filters.

### 5.3 Live Class Join

- **FR-SCW-014:** Student shall be able to join an occurrence with `delivery_mode ∈ { LIVE_ONLINE, HYBRID }` only inside the join window (default `start_time − 10 min` to `end_time + 30 min` BST).
- **FR-SCW-014a:** For `delivery_mode = RECORDED_SUPPORT`, no join action is offered; the prerecorded clip panel is the primary content area; the `live_session` field is null in the response; the `display_phase` follows the same time-based rules (UPCOMING / ONGOING / PREVIOUS) but `ONGOING` has no special meaning for the student (no live action to take).
- **FR-SCW-014b:** For `delivery_mode = ON_SITE`, no join action is offered (the class is physical); `live_session` is null; the `display_phase` reflects time as usual.
- **FR-SCW-015:** For occurrences with `effective_status = UPCOMING` and current time outside the join window, system shall display a `NOT_STARTED` state with a countdown to `start_time`.
- **FR-SCW-016:** For occurrences with `effective_status = ONGOING`, system shall display an active `Join class` button if Live Class Management reports the session as joinable.
- **FR-SCW-017:** If `live_session_ref` is missing or Live Class Management reports the session as `NOT_CONFIGURED`/`UNAVAILABLE` inside the join window, system shall display an `UNAVAILABLE` state with guidance to contact the teacher.
- **FR-SCW-018:** Clicking `Join class` shall redirect to Live Class Management's join flow with `(schedule_entry_id, class_date)` and the student identity; this module never proxies the live session.

### 5.4 Resources

- **FR-SCW-019:** Student shall be able to list and open class-specific resources via `GET /student/classes/{schedule_entry_id}/resources?class_date=…`.
- **FR-SCW-020:** Resources may be `FILE`, `LINK`, or `PRERECORDED_CLIP`. For `FILE`, this module returns a short-lived signed URL minted by the File Asset module; for `LINK` / `PRERECORDED_CLIP` with `external_url`, the URL is passed through as-is.
- **FR-SCW-021:** Student shall be able to download or open resources only for their own batch's classes; cross-batch resource access is rejected with `403`.
- **FR-SCW-022:** System shall not return `DRAFT` or `HIDDEN` resources to the student endpoint.
- **FR-SCW-022a:** When an occurrence's `effective_status = CANCELLED`, student-facing resource and instruction endpoints shall **not** return `OCCURRENCE`-scoped rows for that `class_date`; `RECURRING_SLOT` rows continue to appear on other (non-cancelled) occurrences of the slot.

### 5.5 Previous Class Review

- **FR-SCW-023:** System shall move an occurrence to the `Previous` group when `class_date < today` (server clock BST) **or** same-day and current time > `end_time + buffer` (default 30 min).
- **FR-SCW-024:** Detail of a previous occurrence shall remain accessible: student-visible instructions, resources (as published at the time of last fetch — Teacher Class Workspace's read-only past rule means the set is stable), prerecorded clip if still published, linked recording when available.
- **FR-SCW-024a:** Past-occurrence visibility shall respect *batch-at-the-time*: a previous occurrence is visible to a student only if their `StudentBatch` row for that occurrence's `batch_id` was open (`joined_at ≤ class_date AND (left_at IS NULL OR left_at > class_date)`). Moving a student to a new batch does not retroactively grant access to the new batch's history nor remove access to the old batch's history.
- **FR-SCW-025:** Previous classes shall remain visible for a default retention window of **365 days back** from today; configurable per institution (see OQ). Older occurrences are hidden from the default `group=previous` list but still queryable by explicit `date_from` / `date_to` parameters that may target any historical range, paged in ≤ 92-day windows (the per-request range cap; the student steps backward through time using successive requests).

### 5.6 Linked Assessments

- **FR-SCW-026:** Student shall see assessments linked to a class occurrence in the detail's assessment panel, sourced via `GET /assessments/by-class?schedule_entry_id=…&class_date=…&student_id=<caller>` (Assessment Management endpoint).
- **FR-SCW-027:** Each linked assessment row shall include `(assessment_id, title, type, student_status, due_at?, score_available)`, where `student_status ∈ { UPCOMING, ACTIVE, IN_PROGRESS, SUBMITTED, RESULT_PUBLISHED, CLOSED }` as computed by Assessment Management for the calling student.
- **FR-SCW-028:** Student shall be able to click an active or in-progress linked assessment; this module redirects to Assessment Management with `(assessment_id, schedule_entry_id, class_date)` context.
- **FR-SCW-029:** Student shall be able to click a `RESULT_PUBLISHED` linked assessment; this module redirects to Assessment Management's result view with the same context.
- **FR-SCW-030:** Eligibility (whether the student is permitted to attempt the assessment) is **always** decided by Assessment Management. This module surfaces the link; the actual `start attempt` is rejected by Assessment Management if the student is not eligible. The detail panel must not pre-emptively block based on local heuristics.
- **FR-SCW-031:** When the linked assessment's `student_status = CLOSED` or `UPCOMING`, the row is read-only (no button); the student sees the listing for informational context only.
- **FR-SCW-031a:** When `student_status = SUBMITTED`, the row is read-only and shows "Submitted on `<submitted_at>`"; no result button until `student_status` transitions to `RESULT_PUBLISHED`.

### 5.7 Filtering, Search, Pagination

- **FR-SCW-032:** Student shall be able to filter the class list by one or more `subject_id`.
- **FR-SCW-033:** Student shall be able to filter by `date_from` and `date_to` for the Upcoming and Previous groups.
- **FR-SCW-034:** Student shall be able to free-text search by subject name.
- **FR-SCW-035:** List responses shall be paginated with default page size 30 and max 100; cursor-based.

## 6. Business Rules

- Workspace is derived from the **published** effective schedule only; draft routines are invisible.
- Student access is gated by **active batch membership at the time of access** for Today/Upcoming and **batch-at-the-time** for Previous. The two rules together prevent a recent batch move from leaking another batch's future schedule or erasing the student's own academic history.
- Live join is allowed only inside the configured join window AND when Live Class Management reports the session as joinable.
- Class-linked prerecorded clips and class-linked recordings live within the class; this module never surfaces them as a generic library.
- Previous class detail is read-only by definition (Teacher Workspace blocks writes on past occurrences); the student sees whatever was published before the cutoff.
- Teacher-private notes never reach the student endpoint under any circumstance.
- Schedule changes (override / replacement) are reflected on the next fetch; the student may see updated time/teacher/room when they next open the list or detail.
- Class-linked assessments are surfaced for discovery only; *attempt eligibility, submission, evaluation, result* all live in Assessment Management.
- A cancelled occurrence shows the class header with the `CANCELLED` badge and the reason; resources for that specific date are hidden from the student.

## 7. User Flow / Process Flow

### 7.1 Today's Class

1. Student opens the Student Class Workspace (default home).
2. System resolves the active `StudentBatch.batch_id`.
3. System fetches the `Today` group (lazy-loads `Upcoming` and `Previous` on tab click).
4. Student sees their classes for today with status badges (UPCOMING / ONGOING / PREVIOUS / CANCELLED).
5. Student opens the next class.

### 7.2 Future Live Class

1. Student opens an upcoming live class detail.
2. System checks the join window — current time < `start_time − 10 min`.
3. Detail shows `NOT_STARTED` with a countdown.
4. Join action is hidden.

### 7.3 Ongoing Live Class

1. Student opens a class detail during the join window.
2. System calls Live Class Management for session state.
3. If state is `JOINABLE`, the Join button is enabled.
4. Student clicks; system redirects to Live Class Management's join flow.

### 7.4 Previous Class Review

1. Student switches to the Previous tab.
2. System loads occurrences within the configured retention window (default 365 days) where the student was in the batch at the time.
3. Student opens one; detail shows instructions, resources, prerecorded clip if still published, linked recording if available.

### 7.5 Open Linked Assessment

1. Student opens a class detail.
2. System fetches linked assessments from Assessment Management for the caller.
3. Student sees a list with `student_status` and due timing.
4. Student clicks an `ACTIVE` or `IN_PROGRESS` assessment.
5. System redirects to Assessment Management's attempt flow.

### 7.6 View Result From Class

1. Student opens a previous class detail.
2. Sees a linked assessment with `student_status = RESULT_PUBLISHED` and `score_available = true`.
3. Clicks View Result.
4. System redirects to Assessment Management's result view.

## 8. Data Model

This module **owns no tables**. All data is read-through from upstream modules. The entities listed in the original SRS are projections (views) composed at the resolver layer.

### Read-through projections

- **`StudentClassView`** — composed by calling Scheduling's `GET /batches/{batch_id}/schedule?from=&to=` for the student's active `StudentBatch.batch_id` (server-side, with the student's identity passed for access logging) plus Class Context's `GET /batches/{id}/context` for `class_name`. The Scheduling endpoint is callable by the Student Workspace resolver as a service caller; see `scheduling.md` for the access-control note.
- **`StudentClassInstructionView`** — composed by calling Teacher Class Workspace's `GET /teacher/classes/{schedule_entry_id}/instructions?class_date=&student_visible_only=true` (the new endpoint added to `teacher-workspace.md` Section 9 for this purpose). The endpoint accepts a system / service caller token and filters to STUDENT_VISIBLE only.
- **`StudentClassResourceView`** — composed by calling Teacher Class Workspace's `GET /teacher/classes/{schedule_entry_id}/resources?class_date=&visibility_status=PUBLISHED`, then applying the cancellation rule (FR-SCW-022a) at the resolver layer.
- **`StudentClassRecordingView`** — composed by calling Live Class Management's recording endpoint with `(schedule_entry_id, class_date)`.
- **`StudentClassAssessmentView`** — composed by calling Assessment Management's `GET /assessments/by-class?student_id=…&schedule_entry_id=…&class_date=…`.

Because the module is stateless, no DDL is required for this SRS. Indexes and partial unique constraints needed for the underlying tables live in their owning module's SRS.

> **Multi-tenancy:** Per ADR-0003 (accepted, shared-schema). This module owns no tables, but every upstream read (Scheduling, Teacher Workspace, Live Class, Assessment, Class Context, File Asset) is tenant-scoped via the JWT `tenant_id` claim. The Student Workspace resolver passes the caller's `tenant_id` on every service call; an upstream service that receives a mismatched `tenant_id` returns `404` (existence-hiding).

## 9. API Contracts

All endpoints require `Authorization: Bearer <student access_token>`. All return `403 FORBIDDEN` if the caller's role is not `STUDENT` (and `403 NOT_IN_BATCH` if the student is not in the requested batch's lineage for the requested resource).

### List Classes

```http
GET /student/classes?group=today&subject_id=&date_from=&date_to=&search=&cursor=&limit=
```

**Response (200)**

```json
{
  "occurrences": [
    {
      "schedule_entry_id": "0192...",
      "class_date": "2026-05-18",
      "batch": { "id": "0192...", "name": "Batch A — Morning" },
      "class": { "id": "0192...", "name": "Class 10", "code": "C10" },
      "subject": { "id": "0192...", "name": "Physics", "code": "PHY" },
      "teacher": { "id": "0192...", "name": "A. Rahman" },
      "day_of_week": 0,
      "start_time": "08:00",
      "end_time": "09:30",
      "delivery_mode": "ON_SITE",
      "room": { "id": "0192...", "name": "Room 201" },
      "live_session_ref": null,
      "display_phase": "ONGOING",
      "override_status": "NORMAL",
      "reason": null
    }
  ],
  "next_cursor": "..."
}
```

**Errors**

- `400` — `INVALID_INPUT` / `RANGE_TOO_LARGE` (> 92 days)
- `401` — missing/invalid token
- `403` — `FORBIDDEN` (caller role is not STUDENT)

Satisfies: FR-SCW-001 to FR-SCW-006, FR-SCW-006a, FR-SCW-032 to FR-SCW-035.

### Get Class Detail

```http
GET /student/classes/{schedule_entry_id}?class_date=2026-05-18
```

**Response (200)**

```json
{
  "occurrence": { "...same shape as list item..." },
  "instructions": [
    { "id": "0192...", "content": "Bring your math notebook tomorrow.", "apply_scope": "OCCURRENCE" }
  ],
  "resources": [
    {
      "resource_id": "0192...",
      "title": "Worksheet Chapter 4",
      "resource_type": "FILE",
      "download_url": "https://r2.example.com/...signed...",
      "external_url": null,
      "display_order": 0,
      "apply_scope": "OCCURRENCE"
    }
  ],
  "prerecorded_clips": [
    { "resource_id": "0192...", "title": "Pre-class explainer", "resource_type": "PRERECORDED_CLIP", "external_url": "https://...", "download_url": null }
  ],
  "recording": { "recording_id": "0192...", "title": "Class recording", "recording_url": "...", "recording_status": "AVAILABLE" },
  "linked_assessments": [
    { "assessment_id": "0192...", "title": "Chapter 4 quiz", "type": "PRACTICE_QUIZ", "student_status": "ACTIVE", "due_at": "2026-05-20T22:00:00+06:00", "score_available": false }
  ],
  "live_session": { "state": "NOT_STARTED", "join_window": { "from": "2026-05-18T07:50:00+06:00", "to": "2026-05-18T10:00:00+06:00" } }
}
```

`recording` is null when none is available (always for current/upcoming, and for past when status ≠ AVAILABLE). `live_session` is null for `delivery_mode = ON_SITE`.

**Errors**

- `400` — missing `class_date`
- `403` — `NOT_IN_BATCH` (caller's batch at `class_date` ≠ the occurrence's batch)
- `404` — `OCCURRENCE_NOT_FOUND`

Satisfies: FR-SCW-007 to FR-SCW-013, FR-SCW-014 to FR-SCW-017, FR-SCW-014a, FR-SCW-014b, FR-SCW-026, FR-SCW-027.

### Get Class Resources

```http
GET /student/classes/{schedule_entry_id}/resources?class_date=2026-05-18
```

Same `resources` array shape as in detail. Use this for paginated resource browsing when the detail's inline list is too long.

Satisfies: FR-SCW-019 to FR-SCW-022a.

### Get Class Instructions

```http
GET /student/classes/{schedule_entry_id}/instructions?class_date=2026-05-18
```

Returns the `STUDENT_VISIBLE` instructions only. Same shape as in detail.

Satisfies: FR-SCW-009, FR-SCW-013.

### Get Class Recording

```http
GET /student/classes/{schedule_entry_id}/recording?class_date=2026-05-18
```

**Response (200)**

```json
{ "recording_id": "0192...", "title": "Class recording", "recording_url": "...", "recording_status": "AVAILABLE" }
```

**Response (404)** — `RECORDING_NOT_AVAILABLE` (none exists, or status is PROCESSING / UNAVAILABLE).

Satisfies: FR-SCW-012, FR-SCW-024.

### Get Linked Assessments

```http
GET /student/classes/{schedule_entry_id}/assessments?class_date=2026-05-18
```

Returns `linked_assessments` (same shape as in detail). Suitable for an Assessment panel refresh without a full detail re-fetch.

Satisfies: FR-SCW-026, FR-SCW-027.

### Open Linked Assessment (redirect-context)

```http
GET /student/classes/{schedule_entry_id}/assessments/{assessment_id}?class_date=2026-05-18
```

**Response (200)** — context for the redirect:

```json
{
  "assessment_id": "0192...",
  "schedule_entry_id": "0192...",
  "class_date": "2026-05-18",
  "deeplink": "/assessments/0192.../attempt"
}
```

The client uses `deeplink` to navigate within Assessment Management. This endpoint does not check eligibility; that happens at Assessment Management's attempt endpoint.

Satisfies: FR-SCW-028, FR-SCW-029.

## 10. UI Components

### Screen: Student Class List

- **Components:** Tab group (Today / Upcoming / Previous), subject filter chips, date range picker (Upcoming / Previous), search input, class card list. Each card shows: subject (primary), teacher, time, mode chip, status badge (UPCOMING / ONGOING / PREVIOUS / CANCELLED). Primary action button per card: `Open`, `Join`, or `Review`.
- **Actions:** Open detail, Join live (when ONGOING), Review (when PREVIOUS).
- **States:** loading, empty (no classes today), populated, filtered-empty, error, batch-changing (after `STUDENT_BATCH_CHANGED` event, brief refresh state).

### Screen: Class Detail

- **Components:** Header (subject, teacher, batch chip, time, mode, room or live status, status badge); Instructions panel (rendered Markdown); Resources panel (each row with file icon + title + open action); Prerecorded clip panel (distinct); Linked recording panel (PREVIOUS only); Live join area (when relevant; countdown + Join button); Assessment panel (active items at top, results below).
- **Actions:** Open resource (file download or external link), Play prerecorded clip, Play recording, Join live, Open assessment, View result.
- **States:** loading, populated, cancelled (header has prominent CANCELLED banner + reason; resources for that date hidden), live-not-started (countdown), live-ongoing (Join button), live-ended (no Join), recording-processing (placeholder).

### Mobile-First Considerations

- Class cards stack vertically on width < 480 px.
- Touch targets ≥ 44 px (Bangladesh low-end Android primary audience).
- Inline play for short prerecorded clips; external links open the system player to avoid in-app data costs.
- File downloads show progress with cancellable indicator; resumable where the File Asset module supports it.

## 11. Validation Rules

- **Caller role:** must be `STUDENT`; other roles get 403 even with a valid token.
- **Batch membership at query time:** `StudentBatch` row exists with `left_at IS NULL` and `batch_id` matching the requested occurrence's batch for Today/Upcoming; **batch-at-the-time** rule (FR-SCW-024a) for Previous.
- **Date range:** `from ≤ to`; range ≤ 92 days. Same convention as Scheduling.
- **`class_date`** on detail / resource / instruction / recording / assessment endpoints: required; must be a valid date; the underlying occurrence must exist.
- **Join window:** `now ∈ [start_time − 10 min, end_time + 30 min]` (BST) AND Live Class state is `JOINABLE`.
- **Resource type → returned source:** FILE returns `download_url` (signed, short TTL — default 10 min); LINK returns `external_url`; PRERECORDED_CLIP returns `external_url` or `download_url` depending on source.
- **No DRAFT / HIDDEN / TEACHER_PRIVATE leakage:** API guards filter at the persistence layer; integration tests verify these filters are not bypassed by query params.
- **Pagination cursor:** opaque; server validates issuance.

## 12. Edge Cases

- Student moved to a new batch on the same day → on the next list fetch, Today shows the new batch's classes; the student's open detail of an old-batch class still works only if their `StudentBatch` row for that date was open. The old batch's *previous* classes remain visible per FR-SCW-024a.
- Class cancelled after the student already opened it → on next refresh, the detail header carries the CANCELLED banner; occurrence-scoped resources for that date disappear; the join button is hidden.
- Live session reference missing inside join window → state `UNAVAILABLE`; student sees guidance to contact the teacher; this module does not page Live Class on the student's behalf.
- Prerecorded support clip unpublished after the student already opened it once → on next fetch the clip is filtered out. If the student is mid-playback on an `external_url`, the external host decides; on a signed `download_url`, the URL expires after its TTL and re-fetch fails.
- Previous class recording is still processing → `recording_status = PROCESSING`; UI shows "Recording is being processed; check back later"; no playback URL returned.
- External link resource broken → returned to client unchanged; client navigates and the external host's 404 is shown. No periodic link-health check in v1.
- Override changes same-day time (Scheduling FR-SCH-020) → on next list fetch the time updates; if the student had the join button enabled at the old time, it disappears outside the new window. Notification Management may push a `SCHEDULE_OVERRIDE_CREATED` SMS/in-app message.
- Class has multiple active assessments → all surface in the panel; UI groups by `student_status` (ACTIVE first).
- Assessment due date is after the class date (post-class homework) → row is visible on the class detail with `due_at` in the future; sorted to the top of the assessment panel.
- Class cancelled but the linked assessment remains valid → the assessment row stays visible on the detail with the CANCELLED banner on the class header; student can still open the assessment (Assessment Management decides eligibility).
- Student changed batch after an assessment was published for the old batch → Assessment Management's `student_status` reflects whether the student is still eligible; this module just surfaces the status.
- Result published after the class moved to PREVIOUS → on next assessment-panel refresh, `student_status = RESULT_PUBLISHED`, `score_available = true`; "View Result" button appears.
- Linked assessment closed before the student opens the detail → row shows `student_status = CLOSED`; informational only, no button.
- Student tries to open a class for a date when they were in a different batch → `403 NOT_IN_BATCH`; UI displays "This class wasn't part of your schedule on that date".
- Cross-batch leakage attempt by parameter tampering (student passes another batch's `schedule_entry_id`) → query layer cross-checks `StudentBatch` lineage; rejected with `403`.
- Mobile client on weak network mid-detail load → API responses are JSON, small per endpoint; clients should fetch detail in stages (list → header → resources → assessments) to remain useful on 2G/3G.
- File asset bound to a deactivated teacher or deleted by Admin override → the student's `download_url` request fails with `404` from File Asset; UI shows "Resource no longer available".
- Retention window cutoff: a class older than 365 days does not appear in the default Previous list → student can still query it by explicit `date_from`/`date_to` until institutional retention policy purges (out of scope for this module). Each request is capped at 92 days; the student steps backward through time across multiple requests.
- `RECORDED_SUPPORT` occurrence opened → the detail page shows the prerecorded clip panel as the primary content area; no Join button; `live_session: null`; the recording panel applies only to PREVIOUS-phase occurrences with a `recording_status = AVAILABLE`.
- File asset hard-deleted (purged from File Asset storage, not just soft-deleted) → signed `download_url` request returns `404` from this module; UI shows "Resource no longer available. Contact your teacher." Distinct from the soft-delete case where the resource itself is filtered out at the list layer.
- Student account deactivated mid-session → next request fails with `401` (Auth's session revocation flow); the already-loaded detail page is shown as-is until the next API call. The client should treat any `401` as a forced logout.
- Routine replacement on same-day as current Today class (`scheduling.md` FR-SCH-005a) → the morning class today remains attributed to the prior routine; an afternoon class today is governed by the new routine. The student's Today group may temporarily display occurrences from both routines (no data inconsistency — both are valid for their respective time windows).
- Concurrent signed URL requests for the same FILE resource from two devices → each request mints a separate URL with the configured TTL (default 10 min); v1 does not cache URL issuance server-side. Acceptable for the BD context; revisit if bandwidth costs spike.
- Assessment panel becomes stale after ONGOING → PREVIOUS transition → the client polls the panel endpoint every 60 s when the detail tab is in foreground, OR re-fetches on tab focus. Stale `student_status` is corrected within one poll cycle.
- Brief inconsistency window during `STUDENT_BATCH_CHANGED` event consumption → up to the 5 s SLA, the student may see the old batch's list. After the next fetch following event consumption, the new batch's list is served. Detail-open requests during this window may receive `403 NOT_IN_BATCH` if the student tries to open an old-batch detail after the cutover; UI should refresh the list on `403`.

## 13. Dependencies

- **Class Context (`class-context.md`)** — provides `StudentBatch` lineage for batch-membership checks (active and historical) and `batch.class.name`. Consumes events: `STUDENT_BATCH_CHANGED` (next-fetch consistency), `BATCH_ARCHIVED` (drop the batch's Today/Upcoming).
- **Class Scheduling (`scheduling.md`)** — source of effective schedule and overrides. Consumes events: `SCHEDULE_OVERRIDE_CREATED`, `ROUTINE_REPLACED`, `ROUTINE_PUBLISHED` (refresh next fetch).
- **Teacher Class Workspace (`teacher-workspace.md`)** — source of student-visible instructions and `PUBLISHED` resources. Calls Teacher Workspace's list endpoints with student-visible filter applied at the API contract level (Teacher Workspace's `student_visible_only` parameter — to be added to that module's API).
- **Live Class Management** — source of session state for the join action and the linked recording for previous classes.
- **Assessment Management** — source of linked-assessment metadata and per-student status; redirect target for attempts and results.
- **File Asset module** (shared, schema location TBD) — mints signed `download_url`s for FILE-type resources. Signed URL TTL default 10 min.
- **Notification Management** — push SMS / in-app on schedule changes (`SCHEDULE_OVERRIDE_CREATED`, `ROUTINE_REPLACED`) and on new STUDENT_VISIBLE instruction publishes (see OQ).
- **Authentication & Authorization (`auth.md`)** — RBAC; `STUDENT` role enforced on all `/student/*` endpoints.
- **Central `audit_log`** — schema TBD; this module writes lightweight access logs (resource opens, recording plays) for institutional audit; verbose write disabled by default for storage volume reasons. See OQ.

## 14. Non-Functional Requirements

- **Performance:** list (Today) p95 < 1.5 s; detail p95 < 1 s; resource open (signed URL mint) p95 < 300 ms.
- **Mobile-first:** all screens responsive down to 360 px; touch targets ≥ 44 px; low-end Android (Android 8+, 1 GB RAM) is a primary device class.
- **Weak-network tolerance:** API responses are small per endpoint to support 2G/3G; client should fetch detail in stages; offline cache of last-fetched list survives short network drops (client-side responsibility).
- **Access control:** zero cross-batch leakage; the membership and batch-at-time check is enforced at the API guard layer with integration tests. A student cannot list, detail, resource, or assessment-link another batch by parameter tampering.
- **Internationalization:** all UI strings translatable EN ↔ BN; instruction Markdown renders Bangla correctly; resource titles and subject names support Bangla.
- **Accessibility:** WCAG 2.1 AA on all screens.
- **Cache:** list endpoint cacheable for 30 s on the client (private cache); detail not cached; signed `download_url` cached only until expiry.
- **Data cost (BD context):** prerecorded clips and recordings open in an external player by default to use the platform's native data optimizations; in-app inline play is opt-in.
- **Auditability:** access logs (resource opens, recording plays) are written sampled/aggregated by default; verbose mode configurable.

## 15. Assumptions

- A student belongs to exactly one active batch at a time (Class Context FR-CBS-029).
- The student's identity is `User.role = STUDENT` and visibility flows through `StudentBatch`.
- Students access classes primarily on mobile (low-end Android dominant in the Bangladesh coaching market).
- Prerecorded clips are class-linked, not part of a reusable library.
- Previous class review is important for the coaching context (revision before tests); the default retention is 365 days back.
- The published effective schedule is the source of truth; this module never invents occurrences.
- "Today" is computed in Bangladesh Standard Time on the server; clients respect the server boundary.

## 16. Open Questions

- **Previous-class retention:** is 365 days the right default, or should past-batch occurrences persist longer (3 academic years for transcript-style access)?
- **Conditional access to prerecorded clip:** today gated only by `visibility_status = PUBLISHED`. Should access also require attendance on the live occurrence (some institutions enforce this to drive live attendance)? v1 default: no.
- **Viewed/downloaded indicator:** should the UI show whether the student has viewed/downloaded each resource? Adds an access log row per resource open; storage cost and privacy trade-off.
- **Auto-notification on new student-visible resource / instruction:** push SMS or in-app when teacher publishes? Default v1: no (avoid spam during exam season).
- **Join-window defaults (10 min / 30 min):** confirm with institutions; some prefer tighter windows.
- **Cross-batch historical access for transcript:** when a student graduates and moves out of any active batch, can they still see their full history (e.g., to share with another institution)? Out of v1 scope unless flagged.
- **Live class missing ref behaviour:** today shows `UNAVAILABLE`. Should the student get an automated SMS apology and/or an Admin/Teacher alert? Out of v1; flagged.
- **Recording playback inside the app vs external:** the BD-cost-driven default is external; should institutions be able to force in-app to enforce DRM / no-download?
- **Cross-module call cost:** each detail open hits Class Context + Scheduling + Teacher Workspace + Live Class + Assessment. Consider a server-side composite endpoint to reduce round-trips on slow networks; v1 ships with parallel composition in the resolver layer; revisit if mobile perf degrades.
- **Audit log: access-logging volume** — every resource open writes a row; for 5,000 students × ~5 opens/day = 25k rows/day. Confirm `audit_log` is sized for this or split to a separate `student_access_log` table.
