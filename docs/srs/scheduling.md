# 🧩 MODULE: CLASS SCHEDULING

## 1. Purpose

Define when and how every batch's classes happen across the week, and surface the *effective* schedule (recurring routine + date-specific overrides) to every downstream module that needs to know "what is happening, for whom, where, by whom, in what mode." Scheduling owns the routine truth; Teacher Workspace, Student Workspace, Live Class, Attendance, Notification, and Dashboard all consume it. Conflict detection (batch / teacher / room) is enforced here so no other module re-validates.

## 2. Scope

**In Scope**

- One weekly recurring routine per batch (DRAFT → PUBLISHED → ARCHIVED lifecycle)
- Schedule entries within a routine: `(day_of_week, start_time, end_time, subject, teacher, delivery_mode, optional room / live link)`
- Date-specific overrides (one-off changes to a single occurrence)
- Cancellation of a specific occurrence
- Conflict detection: batch / teacher / room time overlap
- Effective-schedule resolution for any date range (recurring + overrides merged)
- Notifications on publish, entry update (recurring), override, cancellation
- Room catalog (lightweight — name, capacity, branch)
- Weekly grid view and list view with filters
- Bangladesh weekly conventions (Friday off-day; configurable)

**Out of Scope**

- Live session execution (Zoom-like engine) — Live Class Management
- Attendance capture for scheduled occurrences — Attendance Management
- Teacher's per-occurrence prep notes, resources, recordings — Teacher Class Workspace
- Student-facing class consumption — Student Class Workspace
- Holiday calendars (full institution holiday module) — flagged as future enhancement
- Teacher availability preferences (the routine does not query teacher leave; conflict check is on assignments, not availability)
- Payroll-relevant hour computation — Payroll consumes scheduling data but owns the math

## 3. Actors

- **Admin** — sole authority to create / edit / publish / override routines and to manage rooms.
- **Teacher** — read-only consumer of their own schedule via `GET /me/schedule`; receives notifications about changes to their assignments.
- **Student** — read-only consumer via the Student Class Workspace (this module exposes the data; the Workspace renders it).
- **System** — runs conflict validation, resolves effective schedules on read, emits notification events on routine writes.

## 4. Core Concepts

### Batch Routine

A weekly recurring schedule attached to one batch. A batch has **at most one** non-`ARCHIVED` routine at any time. Status is `DRAFT`, `PUBLISHED`, or `ARCHIVED`. Routines have an `effective_from` date (when publishing takes effect) and an optional `effective_to` date (when the routine retires).

### Schedule Entry

One recurring slot within a routine: a day-of-week + start/end time + subject + teacher + delivery mode + optional room / live link. An entry is the **pattern**, not an occurrence. A single entry generates many occurrences across calendar dates.

### Occurrence

The materialised calendar instance of a `ScheduleEntry` on a specific date. Not stored as its own table; computed on read from `(entry, date)` unless an `ScheduleOverride` row exists for that `(entry, date)`. Downstream modules (Attendance, Live Class) identify an occurrence by `(schedule_entry_id, occurrence_date)`.

### Schedule Override

A per-`(entry, date)` exception. Status is either:
- `UPDATED` — fields on the occurrence differ from the base entry (e.g., teacher swap, time change, room change, mode change for that day).
- `CANCELLED` — the occurrence does not happen on that date.

Overrides do **not** modify the base entry; they layer on top during resolution.

### Effective Schedule

The result of resolving a routine across a date range: for each calendar date in range, for each `ScheduleEntry` matching the date's `day_of_week`, apply the override (if any) to produce a concrete occurrence. This is what every downstream module reads.

### Delivery Mode

How the class is delivered for that entry (or occurrence, when overridden):
- `ON_SITE` — physical room; requires `room_id`.
- `LIVE_ONLINE` — live session; requires `live_session_ref` (handle to Live Class module).
- `RECORDED_SUPPORT` — no real-time delivery; optional reference to a prerecorded clip.
- `HYBRID` — both on-site and live; requires `room_id` *and* `live_session_ref`.

### Room

A physical space with a name, capacity, and optional branch. Rooms are an institution-level resource catalog; conflict detection treats a room as a non-shareable resource for any `(date, time_range)`.

### Conflict

Two scheduled occurrences are in conflict if they share a time window (any overlap, not just exact match) and share one of: same batch, same teacher, same room (for on-site or hybrid mode).

## 5. Functional Requirements

### 5.1 Routine Lifecycle

- **FR-SCH-001:** Admin shall be able to create a `DRAFT` routine for a batch by selecting the batch and supplying `effective_from`. `effective_to` may be left null.
- **FR-SCH-002:** System shall enforce at most one non-`ARCHIVED` routine per batch at any time. Creating a new routine while one is `PUBLISHED` for the same batch is blocked with `ACTIVE_ROUTINE_EXISTS`.
- **FR-SCH-003:** Admin shall be able to publish a `DRAFT` routine. Publish is rejected if any entry has an unresolved conflict (see FR-SCH-015).
- **FR-SCH-004:** Admin shall be able to archive a `PUBLISHED` routine. Archival sets `effective_to = now` (if not already set) and transitions status to `ARCHIVED`. Archived routines are read-only and remain queryable for history.
- **FR-SCH-005:** Admin shall be able to **replace** a published routine by creating a new `DRAFT`, publishing it, and the system shall automatically archive the prior `PUBLISHED` routine for that batch at the moment the new one is published. The new routine's `effective_from - 1 day` becomes the prior routine's `effective_to`. The archive-prior + publish-new operations **execute inside a single database transaction**; partial completion is impossible. The publish API response includes both the newly published routine and the now-archived prior routine.
- **FR-SCH-005a:** Same-day replacement is allowed: if the new routine's `effective_from = today` and occurrences from the prior routine have already been served earlier today, those occurrences keep their original routine attribution; the new routine governs occurrences for the same day's day_of_week slots whose `start_time` is later than `now()`. Past occurrences from the same day are not retroactively re-attributed.
- **FR-SCH-006:** System shall make a `DRAFT` routine invisible to Teacher and Student actors and to downstream consumer modules.

### 5.2 Schedule Entry Management

- **FR-SCH-007:** Admin shall be able to add, edit, and delete `ScheduleEntry` rows directly only on a routine in `DRAFT` status. Edits to entries on a `PUBLISHED` routine use the scoped flow in FR-SCH-018 (override or future-replace). Direct PATCH on a PUBLISHED entry without a `scope` parameter is rejected.
- **FR-SCH-008:** System shall require every entry to specify `day_of_week` (0–6, Sunday=0), `start_time`, `end_time`, `subject_id`, `teacher_id`, and `delivery_mode`.
- **FR-SCH-009:** System shall reject an entry whose `subject_id` is not in the batch's effective subject set (read via Class Context module).
- **FR-SCH-010:** System shall reject an entry whose `teacher_id` is not the active `BatchSubjectTeacher` for that `(batch, subject)` pair.
- **FR-SCH-011:** System shall require `start_time < end_time` and reject zero-duration entries.
- **FR-SCH-012:** System shall require `delivery_mode = ON_SITE` → `room_id` non-null; `delivery_mode = LIVE_ONLINE` → `live_session_ref` non-null; `delivery_mode = HYBRID` → both non-null; `delivery_mode = RECORDED_SUPPORT` → both optional.
- **FR-SCH-013:** Admin shall be able to attach a free-text `notes` field to any entry (max 500 chars).

### 5.3 Conflict Detection

- **FR-SCH-014:** System shall detect conflicts on every entry insert or update: **batch overlap** (same batch already scheduled in the time window on the same day), **teacher overlap** (teacher already assigned in another routine's entry on the same day-time window), **room overlap** (room already booked in another entry on the same day-time window). Two entries conflict when their time ranges share any non-zero overlap.
- **FR-SCH-015:** System shall allow `DRAFT` routines to contain unresolved conflicts (so Admin can fix them iteratively), but shall block `PUBLISH` until all conflicts are resolved.
- **FR-SCH-016:** Conflict detection shall complete within p95 < 2 s for an institution of up to 2,000 batches.
- **FR-SCH-017:** System shall return conflicts as a structured response listing each conflicting `(entry_id, conflicting_entry_id, conflict_type)` so the UI can render inline indicators.

### 5.4 Routine Update (Post-Publish)

- **FR-SCH-018:** Admin shall be able to edit an entry on a `PUBLISHED` routine and choose: **(a)** apply to **this occurrence only** — creates a `ScheduleOverride` for one specific date; **(b)** apply **going forward from a date** — splits the routine: the existing routine is archived with `effective_to = chosen_date - 1`, a new `PUBLISHED` routine is created with the edited entry and `effective_from = chosen_date`.
- **FR-SCH-019:** System shall re-run conflict validation **and all structural validations** (FR-SCH-008 to FR-SCH-013 — required fields, delivery-mode → resource requirements, subject in batch's effective set, teacher matches `BatchSubjectTeacher`) against the chosen scope (single date for overrides, the new routine for going-forward edits) before applying.

### 5.5 Override Behaviour

- **FR-SCH-020:** Admin shall be able to override a single occurrence: change `start_time`, `end_time`, `teacher_id`, `delivery_mode`, `room_id`, `live_session_ref`, with an optional `reason`. Any combination of fields may be overridden; null fields inherit from the base entry.
- **FR-SCH-021:** Admin shall be able to cancel a single occurrence (`ScheduleOverride.status = CANCELLED`) with a required `reason`.
- **FR-SCH-022:** Admin shall be able to delete a `ScheduleOverride`, reverting the occurrence to the base entry's values. Deleting an `UPDATED` override removes the changed fields and the occurrence reverts to base. Deleting a `CANCELLED` override **reinstates** the occurrence as `SCHEDULED` with the base entry's field values. Deletion writes an audit row; the historical override is retained in the audit log even though the table row is removed.
- **FR-SCH-023:** Override values shall take precedence over base-entry values during effective-schedule resolution.
- **FR-SCH-024:** System shall re-run conflict validation on every override insert/update against the resulting occurrence's resource use (teacher, room, batch) for that calendar date. Conflict scope includes: (a) the base entries of the same batch on that date, (b) other base entries across all batches sharing the teacher or room on that date, **and (c) other `ScheduleOverride`-derived occurrences on the same date that share the teacher or room with the override-in-flight**. A conflict in any of these dimensions rejects the override with `CONFLICT_DETECTED` listing the offending occurrence id and type.
- **FR-SCH-024a:** When `ScheduleOverride.new_subject_id` is provided, System shall additionally validate that `new_teacher_id` (or, if null, the base entry's `teacher_id`) is the active `BatchSubjectTeacher` for the `(batch, new_subject_id)` pair. Reject with `TEACHER_NOT_MAPPED_FOR_NEW_SUBJECT` otherwise.
- **FR-SCH-025:** System shall block creating two `ScheduleOverride` rows for the same `(schedule_entry_id, override_date)`; updating the existing override is the supported path.

### 5.6 Effective Schedule Resolution

- **FR-SCH-026:** System shall expose `GET /batches/{batch_id}/schedule?from=<date>&to=<date>` returning every occurrence in the range with overrides applied. The response includes the originating `schedule_entry_id`, the `batch_id` / `batch_name` on each occurrence, and, when overridden, the `override_id`. **Caller matrix:** Admin (any batch); the assigned teacher of any entry in the batch (read-only); the Student Class Workspace resolver via internal service token, scoped to the calling student's `StudentBatch.batch_id`; direct STUDENT callers receive `403`. The resolver layer (not the student client) calls this endpoint.
- **FR-SCH-027:** System shall expose `GET /me/schedule?from=<date>&to=<date>` returning the calling teacher's scheduled occurrences (across all batches) with overrides applied.
- **FR-SCH-028:** Date-range queries shall be capped at 92 days (one quarter) per request to bound response size; ranges beyond are rejected with `RANGE_TOO_LARGE`.
- **FR-SCH-029:** System shall guarantee resolution returns within p95 < 2 s for any in-range query under nominal load.
- **FR-SCH-030a:** System shall expose `GET /occurrences/{schedule_entry_id}/{date}` returning the single effective occurrence for a `(schedule_entry_id, date)` pair, with overrides applied. This is the canonical per-occurrence read for Attendance, Live Class, and Payroll. The endpoint works for past, current, and future dates and returns `200` for any date that falls within the routine's `effective_from..effective_to` window (or, for archived routines, the historical window). Returns `404` if no occurrence exists for that pair.
- **FR-SCH-030b:** Admin shall be able to list a batch's routines via `GET /batches/{batch_id}/routines` returning all routines (DRAFT, PUBLISHED, ARCHIVED) ordered by `effective_from` desc. Used by the Routine Builder load flow and by historical attribution queries.
- **FR-SCH-030:** Cancelled occurrences shall appear in the response with `status: CANCELLED`, the `reason`, **and the full base-entry fields** (`subject`, `teacher`, `day_of_week`, `start_time`, `end_time`, `delivery_mode`, `room` if applicable). Consumers need this to render "X class on date Y was cancelled" without a follow-up query. Consumers decide presentation (Student Workspace shows a struck-through tile, etc.).

### 5.7 Notifications

- **FR-SCH-031:** System shall emit a `ROUTINE_PUBLISHED` event on FR-SCH-003 success. Payload: `{ event_type, routine_id, batch_id, batch_name, class_name, effective_from, entry_count, published_at, published_by }`. Recipients (resolved by Notification Management): assigned teachers (deduped from entries) and enrolled students of the batch.
- **FR-SCH-032:** System shall emit a `SCHEDULE_OVERRIDE_CREATED` event on FR-SCH-020 or FR-SCH-021 success. Payload: `{ event_type, override_id, schedule_entry_id, batch_id, batch_name, subject_id, subject_name, occurrence_date, change_kind (UPDATED|CANCELLED), original_start_time, original_end_time, original_teacher_id, original_teacher_name, new_start_time?, new_end_time?, new_teacher_id?, new_teacher_name?, new_delivery_mode?, new_room_name?, reason?, created_at, created_by }`. Recipients: the affected teacher (original and new if changed) and enrolled students of the batch.
- **FR-SCH-033:** System shall emit a `ROUTINE_REPLACED` event on FR-SCH-005 success. Payload: `{ event_type, old_routine_id, new_routine_id, batch_id, batch_name, cutover_date, published_at, published_by }`. Recipients: assigned teachers across both old and new routines (deduped) and enrolled students.
- **FR-SCH-034:** Notification payloads in FR-SCH-031 to FR-SCH-033 shall be self-contained: every field a downstream channel (SMS, email, in-app) needs to compose a human-readable message is present in the payload. Notification Management does **not** call back to Scheduling to enrich the message.

### 5.8 Room Management

- **FR-SCH-035:** Admin shall be able to create, edit, and deactivate rooms with `name`, `capacity`, optional `branch_id`, and `is_active`.
- **FR-SCH-036:** System shall block deactivation of a room referenced by any non-archived `ScheduleEntry`; Admin must reassign first.
- **FR-SCH-037:** Conflict detection shall ignore rooms with `is_active = false` (treated as deleted for new bookings).

## 6. Business Rules

- One non-`ARCHIVED` routine per batch at any time.
- Conflicts are **hard blocks** at publish and at override write; soft (allowed) only in `DRAFT`.
- Two time windows conflict when they share any non-zero overlap (boundary-touching, e.g. `10:00–11:00` and `11:00–12:00`, is **not** a conflict).
- Override resolution: presence of an override row for `(entry_id, date)` replaces the corresponding base-entry fields for that occurrence; null override fields inherit the base.
- Cancelled overrides count as no class — Attendance is not generated, Live Class is not started, notifications go out.
- Schedules are stored in Bangladesh Standard Time (UTC+6); ISO 8601 with offset on the wire.
- Friday is treated as the default off-day (no entries by convention, configurable per institution); the system does not block Friday entries, but the UI surfaces a warning.
- A routine's `effective_from` cannot be in the past relative to its publish time; publishing a routine dated in the past is rejected.
- Replacing a routine (FR-SCH-005) creates a clean cutover at `effective_from` of the new routine; no occurrence is "between" two routines.
- Recording, attendance, and assessment data attached to past occurrences are not affected by routine archival or replacement (downstream modules keep their own foreign-key snapshots).

## 7. User Flow / Process Flow

### 7.1 Weekly Routine Creation

1. Admin opens **Routine Builder** for a batch.
2. System loads the batch's effective subject set and the active teacher per subject from Class Context.
3. System loads the existing `DRAFT` routine if any, else opens an empty one.
4. Admin drags / types schedule entries into the weekly grid.
5. System validates each entry inline (FR-SCH-008 to FR-SCH-013) and runs conflict detection (FR-SCH-014).
6. Admin reviews the conflict panel and resolves any flagged entries.
7. Admin clicks **Publish**.
8. System re-validates the full routine and either publishes (FR-SCH-003) or returns the conflict list.

### 7.2 Conflict Resolution

1. Admin sees inline conflict marker on an entry.
2. Admin opens the conflict detail (lists the conflicting entry, type, and offending resource).
3. Admin edits time, teacher, or room.
4. System re-runs conflict check on save; marker clears if resolved.

### 7.3 Recurring Update (post-publish)

1. Admin opens a published entry.
2. Admin edits a field.
3. UI prompts: **"This occurrence only"** / **"This and future occurrences"**.
4. **This occurrence only:** Admin selects a date; system creates a `ScheduleOverride` (FR-SCH-020).
5. **This and future:** Admin selects an effective date; system archives the current routine at `date - 1`, creates a new `PUBLISHED` routine with the edited entry (FR-SCH-018b).
6. Conflict validation runs against the chosen scope.
7. On success, the relevant notification event fires.

### 7.4 Single-Occurrence Override

1. Admin selects a specific calendar date on the routine.
2. Admin chooses **Override** or **Cancel** for that occurrence.
3. Admin enters new field values (override) or a reason (cancel).
4. System validates conflict on the resulting occurrence.
5. System creates the `ScheduleOverride` row and emits the event.

### 7.5 Resolution (downstream consumer flow)

1. Teacher Workspace calls `GET /me/schedule?from=2026-05-18&to=2026-05-24`.
2. System fetches every active routine entry the caller teaches, expands occurrences for the date range, applies overrides.
3. Response includes one entry per occurrence with all resolved fields.

## 8. Data Model

### `BatchRoutine`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| batch_id | UUID, FK → Batch.id | One non-archived routine per batch (partial unique index where status ≠ ARCHIVED) |
| status | enum(`DRAFT`, `PUBLISHED`, `ARCHIVED`) | |
| effective_from | Date | When this routine starts taking effect after publish |
| effective_to | Date, nullable | Set on archival or replacement |
| published_at | Timestamp, nullable | Set on first transition to PUBLISHED |
| archived_at | Timestamp, nullable | |
| created_by | UUID, FK → User.id | Admin |
| updated_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| updated_at | Timestamp | |

### `ScheduleEntry`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| routine_id | UUID, FK → BatchRoutine.id | |
| day_of_week | smallint, 0–6 | Sunday = 0 |
| start_time | Time | Local time (BST) |
| end_time | Time | start_time < end_time |
| subject_id | UUID, FK → Subject.id | Must be in batch's effective subject set |
| teacher_id | UUID, FK → User.id | Must be the active `BatchSubjectTeacher` for the pair |
| delivery_mode | enum(`ON_SITE`, `LIVE_ONLINE`, `RECORDED_SUPPORT`, `HYBRID`) | |
| room_id | UUID, FK → Room.id, nullable | Required for ON_SITE and HYBRID |
| live_session_ref | string, nullable | Required for LIVE_ONLINE and HYBRID; opaque handle to Live Class |
| notes | string, nullable | ≤ 500 chars |
| created_by | UUID, FK → User.id | |
| updated_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| updated_at | Timestamp | |

### `ScheduleOverride`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| schedule_entry_id | UUID, FK → ScheduleEntry.id | |
| override_date | Date | The specific calendar date overridden |
| status | enum(`UPDATED`, `CANCELLED`) | |
| new_start_time | Time, nullable | |
| new_end_time | Time, nullable | |
| new_teacher_id | UUID, FK → User.id, nullable | |
| new_subject_id | UUID, FK → Subject.id, nullable | Override may swap subject in unusual cases (rare; surfaced as warning) |
| new_delivery_mode | enum, nullable | |
| new_room_id | UUID, FK → Room.id, nullable | |
| new_live_session_ref | string, nullable | |
| reason | string, nullable | Required when status = CANCELLED; ≤ 500 chars; may contain Bangla |
| created_by | UUID, FK → User.id | |
| updated_by | UUID, FK → User.id | Set on every override update (FR-SCH-020 PUT path) |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| | | Unique: (schedule_entry_id, override_date) |

### `Room`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | string | Display name |
| capacity | integer | Positive |
| branch_id | UUID, nullable | Future multi-branch support |
| is_active | boolean | Soft delete |
| created_by | UUID, FK → User.id | |
| updated_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| updated_at | Timestamp | |

> **Multi-tenancy note:** ADR-0003 is pending. This SRS assumes single-tenant deployment. If ADR-0003 selects shared-schema multi-tenant, a `tenant_id` column must be added to `BatchRoutine`, `ScheduleEntry`, `ScheduleOverride`, and `Room`, and every query, conflict-detection scope, and unique index must be tenant-scoped. Same caveat as `auth.md` §4 and `class-context.md` §15.

## 9. API Contracts

### Create Routine

```http
POST /routines
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

**Request**

```json
{ "batch_id": "0192...", "effective_from": "2026-06-01" }
```

**Response (201)**

```json
{ "id": "0192...", "batch_id": "0192...", "status": "DRAFT", "effective_from": "2026-06-01" }
```

**Errors**

- `404` — `BATCH_NOT_FOUND`
- `409` — `ACTIVE_ROUTINE_EXISTS`

Satisfies: FR-SCH-001, FR-SCH-002.

### Add Schedule Entry

```http
POST /routines/{routine_id}/entries
```

**Request**

```json
{
  "day_of_week": 0,
  "start_time": "08:00",
  "end_time": "09:30",
  "subject_id": "0192...",
  "teacher_id": "0192...",
  "delivery_mode": "ON_SITE",
  "room_id": "0192...",
  "notes": "Bring practice book"
}
```

**Response (201)** — full entry object plus a `conflicts: []` array (empty if no conflicts in draft).

**Errors**

- `400` — `INVALID_INPUT` (zero duration, missing required fields per delivery mode)
- `404` — `ROUTINE_NOT_FOUND` / `SUBJECT_NOT_FOUND` / `TEACHER_NOT_FOUND` / `ROOM_NOT_FOUND`
- `409` — `ROUTINE_ARCHIVED` / `SUBJECT_NOT_IN_BATCH` / `TEACHER_NOT_MAPPED`

Satisfies: FR-SCH-007 to FR-SCH-013.

### Update Schedule Entry

```http
PATCH /entries/{entry_id}
```

**Request** — any subset of entry fields. For `PUBLISHED` routines, the request must include a `scope` field:

```json
{ "scope": "OCCURRENCE", "occurrence_date": "2026-06-15", "new_teacher_id": "0192..." }
```

or

```json
{ "scope": "FUTURE", "effective_from": "2026-06-15", "new_start_time": "09:00", "new_end_time": "10:30" }
```

**Response (200)** — for `DRAFT`, the updated entry object (same shape as Add Schedule Entry response):

```json
{
  "id": "0192...",
  "routine_id": "0192...",
  "day_of_week": 0,
  "start_time": "09:00",
  "end_time": "10:30",
  "subject_id": "0192...",
  "teacher_id": "0192...",
  "delivery_mode": "ON_SITE",
  "room_id": "0192...",
  "live_session_ref": null,
  "notes": "Bring practice book",
  "conflicts": []
}
```

For `OCCURRENCE` scope on `PUBLISHED`: the created `ScheduleOverride` row. For `FUTURE` scope: the same publish-response shape as `POST /routines/{routine_id}/publish` (new `published` + `archived_prior`).

**Errors**

- `400` — `INVALID_INPUT` or missing `scope` on a `PUBLISHED` routine
- `403` — `FORBIDDEN` (caller is not Admin)
- `404` — `ENTRY_NOT_FOUND` (returned in preference to 403 when the entry does not belong to the caller's institution, to avoid existence-leak; relevant once ADR-0003 lands)
- `409` — `CONFLICT_DETECTED` (response body lists conflicts) / `ROUTINE_REPLACED_CONCURRENTLY` (FUTURE-scope race)

Satisfies: FR-SCH-018, FR-SCH-019.

### Publish Routine

```http
POST /routines/{routine_id}/publish
```

The publish + archive-prior operations execute in a single database transaction (FR-SCH-005). The response includes both routines so the caller can verify the cutover happened.

**Response (200)** — first-time publish (no prior routine for the batch)

```json
{
  "published": { "id": "0192...", "status": "PUBLISHED", "effective_from": "2026-06-01", "published_at": "2026-05-25T10:00:00+06:00" },
  "archived_prior": null
}
```

**Response (200)** — replacement publish (prior routine auto-archived)

```json
{
  "published": { "id": "0192...new...", "status": "PUBLISHED", "effective_from": "2026-06-15", "published_at": "2026-06-10T10:00:00+06:00" },
  "archived_prior": { "id": "0192...old...", "status": "ARCHIVED", "effective_to": "2026-06-14", "archived_at": "2026-06-10T10:00:00+06:00" }
}
```

**Errors**

- `404` — `ROUTINE_NOT_FOUND`
- `409` — `INVALID_STATUS` / `CONFLICTS_PRESENT` (response body lists conflicts) / `EFFECTIVE_FROM_IN_PAST` / `ROUTINE_REPLACED_CONCURRENTLY` (lost race against a parallel publish)

Satisfies: FR-SCH-003, FR-SCH-005, FR-SCH-005a.

### Archive Routine

```http
POST /routines/{routine_id}/archive
```

**Response (200)** — updated routine. Always sets `effective_to` if null.

**Errors**

- `409` — `ALREADY_ARCHIVED`

Satisfies: FR-SCH-004.

### Create / Update Override

```http
PUT /entries/{entry_id}/overrides/{date}
```

`date` in path is the `override_date` in `YYYY-MM-DD`. Idempotent.

**Request**

```json
{
  "status": "UPDATED",
  "new_start_time": "12:00",
  "new_teacher_id": "0192...",
  "reason": "Teacher swap due to leave"
}
```

or

```json
{ "status": "CANCELLED", "reason": "Public holiday" }
```

**Response (200/201)** — created or updated override.

**Errors**

- `400` — `INVALID_INPUT` / `MISSING_REASON_ON_CANCEL`
- `404` — `ENTRY_NOT_FOUND`
- `409` — `CONFLICT_DETECTED` (response body lists)

Satisfies: FR-SCH-020, FR-SCH-021, FR-SCH-024, FR-SCH-025.

### Delete Override

```http
DELETE /entries/{entry_id}/overrides/{date}
```

**Response (204)** — empty body.

Satisfies: FR-SCH-022.

### Get Batch Schedule (range)

```http
GET /batches/{batch_id}/schedule?from=2026-05-18&to=2026-05-24
```

**Response (200)**

```json
{
  "batch_id": "0192...",
  "from": "2026-05-18",
  "to": "2026-05-24",
  "occurrences": [
    {
      "schedule_entry_id": "0192...",
      "override_id": null,
      "batch_id": "0192...",
      "batch_name": "Class 10 — Batch A — Morning",
      "date": "2026-05-18",
      "day_of_week": 0,
      "start_time": "08:00",
      "end_time": "09:30",
      "subject": { "id": "0192...", "name": "Physics", "code": "PHY" },
      "teacher": { "id": "0192...", "name": "A. Rahman" },
      "delivery_mode": "ON_SITE",
      "room": { "id": "0192...", "name": "Room 201", "capacity": 40 },
      "live_session_ref": null,
      "status": "SCHEDULED"
    },
    {
      "schedule_entry_id": "0192...",
      "override_id": "0192...",
      "date": "2026-05-19",
      "day_of_week": 1,
      "start_time": "08:00",
      "end_time": "09:30",
      "subject": { "id": "0192...", "name": "Physics", "code": "PHY" },
      "teacher": { "id": "0192...", "name": "A. Rahman" },
      "delivery_mode": "ON_SITE",
      "room": { "id": "0192...", "name": "Room 201", "capacity": 40 },
      "status": "CANCELLED",
      "reason": "Public holiday"
    }
  ]
}
```

**Errors**

- `400` — `RANGE_TOO_LARGE` (> 92 days)
- `404` — `BATCH_NOT_FOUND`

Satisfies: FR-SCH-026, FR-SCH-028, FR-SCH-029, FR-SCH-030.

### Get My Schedule (teacher)

```http
GET /me/schedule?from=2026-05-18&to=2026-05-24
```

**Headers**

```
Authorization: Bearer <teacher access_token>
```

**Response (200)** — same occurrence shape as the batch endpoint (each occurrence includes `batch_id` and `batch_name` so the caller can distinguish across batches), scoped to the calling teacher across all their batches.

**Errors**

- `400` — `INVALID_INPUT` / `RANGE_TOO_LARGE` (> 92 days) / `RANGE_INVERTED` (`from > to`)
- `401` — missing or invalid token
- `403` — `FORBIDDEN` (caller role is not `TEACHER`)

Satisfies: FR-SCH-027, FR-SCH-028.

### Get Single Occurrence (point-in-time)

```http
GET /occurrences/{schedule_entry_id}/{date}
```

Canonical per-occurrence read used by Attendance, Live Class, and Payroll. Works for past, present, and future dates. Applies any `ScheduleOverride` for the pair; returns the same occurrence shape as the range endpoint.

**Response (200)** — single occurrence object (same shape as one entry in the range response).

**Errors**

- `404` — `OCCURRENCE_NOT_FOUND` (no entry, no matching day_of_week, or date outside routine's effective window and no override exists)

Satisfies: FR-SCH-030a.

### List Routines for a Batch

```http
GET /batches/{batch_id}/routines
```

Returns all routines (DRAFT, PUBLISHED, ARCHIVED) for the batch, ordered by `effective_from` desc.

**Response (200)**

```json
{
  "routines": [
    { "id": "0192...", "status": "PUBLISHED", "effective_from": "2026-06-15", "effective_to": null, "published_at": "2026-06-10T10:00:00+06:00" },
    { "id": "0192...", "status": "ARCHIVED", "effective_from": "2026-06-01", "effective_to": "2026-06-14", "archived_at": "2026-06-10T10:00:00+06:00" }
  ]
}
```

Satisfies: FR-SCH-030b.

### List Rooms

```http
GET /rooms?is_active=true
```

**Response (200)**

```json
{
  "rooms": [
    { "id": "0192...", "name": "Room 201", "capacity": 40, "branch_id": null, "is_active": true, "in_use_entry_count": 7 }
  ]
}
```

Satisfies: FR-SCH-035.

### Get Routine (read)

```http
GET /routines/{routine_id}
```

Returns the routine metadata + all its entries.

**Response (200)** — `{ routine: {...}, entries: [...] }`.

Used by the Routine Builder load flow (§7.1 step 3) and the historical attribution queries.

### Create Room

```http
POST /rooms
```

**Request**

```json
{ "name": "Room 201", "capacity": 40, "branch_id": null }
```

**Response (201)** — full room object.

Satisfies: FR-SCH-035.

### Edit / Deactivate Room

```http
PATCH /rooms/{room_id}
```

**Request** (any subset)

```json
{ "name": "Room 201 (renovated)", "capacity": 45, "is_active": false }
```

**Errors**

- `409` — `ROOM_IN_USE` (deactivation blocked by referencing entries)

Satisfies: FR-SCH-035, FR-SCH-036.

## 10. UI Components

### Screen: Routine Builder (Weekly Grid)

- **Components:** Sunday-to-Saturday columns; rows are time slots (15-min granularity); empty cells clickable to add an entry; entry tile shows subject + teacher; conflict marker (red dot) on conflicting tiles; Friday column visually muted (off-day convention).
- **Actions:** Drag tile to move (DRAFT only); click to edit; right-click → override (PUBLISHED); Publish button (gated by zero conflicts).
- **States:** loading, empty (no entries), draft (editable), published (read-only for grid edits, override available), with-conflicts (publish disabled).

### Screen: Routine Builder (List View)

- **Components:** Sortable, filterable table of entries with columns: day, time, subject, teacher, mode, room, notes; bulk select for delete; filter bar (class, batch, teacher, subject, date range, status).
- **Actions:** Edit row, delete row, jump to grid view.
- **States:** loading, empty, populated.

### Screen: Entry Editor (modal)

- **Components:** Day-of-week selector, start/end time pickers, subject dropdown (scoped to batch's effective subjects), teacher field (auto-filled from `BatchSubjectTeacher`, read-only), delivery-mode radio, conditional room picker / live-link input, notes textarea.
- **Actions:** Save (creates or updates), Save & Add Another, Cancel.
- **States:** idle, validating, conflict-detected (shows conflict panel inline).

### Screen: Recurring Edit Prompt (modal, post-publish)

- **Components:** Two-choice radio (**This occurrence only** / **This and future occurrences**), date picker, optional "Remember my choice for this session" checkbox.
- **Actions:** Continue (proceeds with the chosen scope), Cancel.
- **States:** idle, applying, conflict (rolls back, shows conflict).

### Screen: Override / Cancel (modal)

- **Components:** Date display (read-only), field overrides (time, teacher, mode, room), reason textarea (required for cancel), confirm button.
- **Actions:** Save override, Cancel occurrence.
- **States:** idle, validating, conflict, success.

### Screen: Room Catalog

- **Components:** Room list (name, capacity, branch, status, in-use count), Add Room button, deactivate toggle with confirm.
- **Actions:** Create, edit, deactivate.
- **States:** loading, empty, populated, deactivation-blocked (shows referencing entries).

## 11. Validation Rules

- **day_of_week:** integer 0–6, with 0 = Sunday.
- **start_time / end_time:** `HH:MM` (24-h), local BST, `start_time < end_time`, both within 06:00–22:00 (institutional sanity bound; configurable).
- **Entry overlap (within a routine):** prevented by FR-SCH-014; system blocks same-batch overlap at save.
- **room_id:** must reference an `is_active = true` room when assigned.
- **live_session_ref:** string, ≤ 256 chars, opaque to this module.
- **notes:** ≤ 500 chars, may contain Bangla.
- **override.override_date:** must fall on the same `day_of_week` as the base entry; otherwise rejected with `DATE_DAY_MISMATCH`.
- **override.new_start_time / new_end_time:** if either provided, both must be provided (no half-overrides on time).
- **override.reason:** required for `status = CANCELLED`; ≤ 500 chars.
- **effective_from:** ISO date, must be ≥ today at publish time.
- **Date range queries:** `from ≤ to`; range ≤ 92 days. Applies to both `GET /batches/{id}/schedule` and `GET /me/schedule`.
- **Override null fields:** when an override row has a null `new_*` field, the resolved occurrence inherits the base entry's value for that field (FR-SCH-020). Validation does not require all `new_*` fields to be present; the only paired-field rule is `new_start_time` and `new_end_time` (both or neither).
- **Friday entries:** `day_of_week = 5` (Friday) is **not** blocked at the API layer; the UI surfaces a warning per Bangladesh convention (Friday is the default off-day). Configurable per institution.

## 12. Edge Cases

- Teacher becomes inactive (`User.is_active = false`) after their entries are scheduled → User Management calls `BatchSubjectTeacher` deactivation (Class Context); Scheduling's stored `teacher_id` on entries becomes orphaned. System surfaces a "teacher unassigned" warning per entry; Admin must override per occurrence or re-publish a new routine. Existing future occurrences continue to render with the now-inactive teacher until corrected.
- Batch is archived (Class Context) while a routine is PUBLISHED → routine is auto-archived by a triggered flow (Class Context calls Scheduling on batch archive). All future occurrences disappear from downstream reads; past occurrences remain queryable.
- Override conflicts with another override → second override creating the conflict is rejected at write; the existing override is unaffected.
- Two admins concurrently create overrides for the same `(entry, date)` → unique index on `(schedule_entry_id, override_date)` ensures only one succeeds; the other returns `OVERRIDE_EXISTS` and the UI refreshes.
- Room deleted (deactivated) after assignment → deactivation is blocked by FR-SCH-036; Admin must reassign first.
- LIVE_ONLINE entry with missing `live_session_ref` 15 minutes before class → out of scope here; Live Class module surfaces "session not configured" to teacher/student. Scheduling exposed the ref as null because FR-SCH-012 only enforced presence, not validity.
- Holiday overlap → not modelled in v1 (no holiday calendar). In v1 Admin must cancel each affected occurrence individually using the per-occurrence override endpoint. Bulk cancel across batches is deferred (see OQ-1). Holiday Calendar module is a future enhancement.
- Partial routine then publish attempt → FR-SCH-015 blocks publish; conflicts list returned. Empty routine (zero entries) is allowed to publish — a batch may legitimately have no scheduled classes in some weeks (e.g. exam break). The published-empty case produces zero occurrences in resolution queries; downstream modules show "no classes scheduled" empty states.
- Daylight Saving Time → Bangladesh does not observe DST; not an edge case here, but stated to head off implementation confusion.
- Override on a date before the routine's `effective_from` → rejected with `DATE_BEFORE_ROUTINE`.
- Override on a date after the routine's `effective_to` → rejected with `DATE_AFTER_ROUTINE`.
- Two routines for the same batch with overlapping `effective_from..effective_to` → prevented at write; routine replacement (FR-SCH-005) handles the cutover atomically.
- A teacher's leave request (handled outside Scheduling) for a date → not auto-applied; Admin must create overrides. Future cross-module integration with a Leave module is an OQ.
- Boundary-touching time windows (`09:00–10:00` and `10:00–11:00`) → not a conflict (open interval semantics).
- Schedule range query spanning a routine replacement → response includes occurrences from both routines (the old one up to `effective_to`, the new one from `effective_from`).
- Network failure during publish → publish is transactional (single DB transaction wrapping status flip + audit row); on failure routine stays `DRAFT`, Admin retries.
- Deleting an entry on a `PUBLISHED` routine → not allowed via the entry endpoint; Admin must either override-cancel all future occurrences (one-by-one or via a bulk endpoint, OQ) or replace the routine.
- Two admins concurrently call `PATCH /entries/{entry_id}` with `scope: FUTURE` and the same `effective_from` for the same batch → serialised by a row-level lock on `BatchRoutine`; the loser receives `409 ROUTINE_REPLACED_CONCURRENTLY` and the UI refreshes to show the winning admin's replacement.
- Batch transitioned to `INACTIVE` (reversible, not archived) while a `PUBLISHED` routine exists → routine remains `PUBLISHED` and queryable; resolution API continues to return occurrences. Downstream modules (Attendance, Notification) decide whether to act based on the *batch* status, not the routine status. If the batch is set `ACTIVE` again, no action needed.
- Teacher reassigned at the `BatchSubjectTeacher` layer (Class Context FR-CBS-026) while existing `ScheduleEntry` rows reference the old teacher → existing entries are **snapshots**; their `teacher_id` is preserved on PUBLISHED routines and continues to be served for historical occurrences. For future occurrences, Class Context emits a `BATCH_SUBJECT_TEACHER_REASSIGNED` event; Scheduling surfaces a "teacher mismatch" warning per affected entry in the Routine Builder. Admin chooses whether to override per occurrence, replace the routine, or leave as-is (e.g. the original teacher is finishing the term).
- Override with `new_subject_id` swapping to a subject whose `BatchSubjectTeacher` is a different teacher than the override's `new_teacher_id` (or the base entry's `teacher_id` when `new_teacher_id` is null) → rejected with `TEACHER_NOT_MAPPED_FOR_NEW_SUBJECT` per FR-SCH-024a.
- Admin's role changed from `ADMIN` to a non-admin role while they have an active session → their existing access token still carries `role: ADMIN` until expiry (per `auth.md` access-token TTL, default 24 h). Scheduling writes continue to succeed during that window. Mitigation lives in Auth's revocation deny-list (auth.md FR-AUTH-012); Scheduling assumes the auth layer is sufficient and does not re-check role-source-of-truth on every write.

## 13. Dependencies

- **Class Context (class-context.md)** — provides the batch, its effective subject set, the `BatchSubjectTeacher` mapping. FR-SCH-009 and FR-SCH-010 read from there. **Inter-module events Scheduling consumes:** `BATCH_ARCHIVED` (triggers auto-archive of the batch's PUBLISHED routine), `BATCH_SUBJECT_TEACHER_REASSIGNED` (raises a "teacher mismatch" warning on affected entries; does not auto-rewrite). Scheduling stores `teacher_id` as a snapshot on `ScheduleEntry`; historical occurrences are not retroactively re-attributed.
- **User Management** — provides teacher/admin identities, `is_active` flag for the orphan-teacher case.
- **Notification Management** — consumes the events emitted in section 5.7.
- **Live Class Management** — owns the lifecycle of `live_session_ref` strings; Scheduling stores the handle but does not interpret it.
- **Teacher Class Workspace, Student Class Workspace, Attendance, Assessment, Dashboard, Payroll** — downstream consumers of `GET /batches/{id}/schedule` and `GET /me/schedule`.
- **Redis** (per ADR-0002) — caches the effective-schedule resolution for hot ranges (today / this week). Invalidation triggers on every override write and on every routine status change.
- **Central `audit_log`** (schema location TBD — same forward-reference as `auth.md` §13) — Scheduling writes events for routine status changes, entry edits, override creates/deletes.

## 14. Non-Functional Requirements

- **Performance:** conflict validation on entry save p95 < 2 s for ≤ 2,000-batch institution; schedule resolution range query p95 < 2 s for a 7-day range, p95 < 5 s for the full 92-day range.
- **Consistency:** routine status changes and entry writes are transactional; partial publish is impossible.
- **Concurrency:** override uniqueness on `(entry_id, override_date)` enforced by a partial unique index; one-active-routine-per-batch by a partial unique index where `status ≠ ARCHIVED`.
- **Cache degraded mode:** when Redis is unavailable, resolution falls through to Postgres (fail-open) at higher latency; p95 SLA is waived. Writes still invalidate the cache on a best-effort basis with queued retry.
- **Scale:** 200 active routines, 5,000 entries, 50,000 occurrences per quarter per institution.
- **Auditability:** every routine status change, entry write, and override write produces an `audit_log` row with `actor_id`, `entity_type`, `entity_id`, `diff`.
- **Internationalization:** subject names and notes may contain Bangla; day-of-week labels are translated EN ↔ BN in the UI; times are presented in 12-h format with AM/PM in EN and ২৪-ঘণ্টা in BN (configurable).
- **Time zone:** all times stored and validated as Bangladesh Standard Time (UTC+6). API responses include the offset on ISO 8601 timestamps; bare time fields (`HH:MM`) are implicitly BST.

## 15. Assumptions

- Admin is the sole authority for scheduling; teachers and students never write.
- Conflict checks are limited to batch / teacher / room; teacher leave / unavailability is not part of v1.
- A batch's `BatchSubjectTeacher` mapping is the only valid teacher for any entry under that `(batch, subject)`; Admin cannot route around the mapping at the entry level.
- Friday is the convention-default off-day; institutions running 7-day weeks are supported by the data model but the UI flags Friday entries.
- Recording links and live-session handles are opaque strings; Scheduling does not validate them beyond presence.
- The institution operates in a single time zone (Bangladesh Standard Time); multi-time-zone is not in v1.

## 16. Open Questions

- **Bulk cancel** for a single date across many batches (e.g. national holiday): is a one-click bulk-cancel needed, or do we add a Holiday Calendar module that creates overrides automatically?
- **Holiday Calendar module:** flagged as future enhancement in the original SRS — confirm whether v1 ships without it (current assumption) or whether it lands as a peer module.
- **Teacher leave integration:** when a teacher's leave (from a future Leave module) covers a date, should Scheduling auto-create cancellation or "needs substitute" overrides, or remain manual?
- **Sub-routine within routine:** some institutions run two separate weekly schedules per batch (e.g. main + remedial). Current model is one routine per batch — confirm v1 limit.
- **Per-occurrence room override and capacity check:** if an override changes the room to a smaller-capacity room, should the system warn about enrolment exceeding the new room's capacity?
- **Recurring "delete entry" on PUBLISHED routine:** current FRs require override-cancel or routine replacement. Should there be a streamlined "stop this entry from date X" flow?
- **Friday default:** institutions running Friday-Saturday weekend (some Middle East deployments) need configurable weekend days. Confirm Bangladesh-only for v1.
- **Override on `override_date` that's a different day_of_week than the entry** (FR-SCH-008 / validation): blocked today; some institutions want to *move* a class to a different weekday for one week. Should this be a different operation type (e.g. one-off "added occurrence") rather than an override of a recurring slot?
- **Audit log diff format:** confirm the conventions in the shared `audit_log` doc once it exists (cross-reference to Auth and Class Context OQs).
- **Override delete audit trail blocked until `audit_log` lands:** FR-SCH-022 promises override history is retained in `audit_log` after delete. If `audit_log` is not ready when Scheduling ships, this guarantee is unfulfilled. Decide: delay Scheduling, or temporarily disallow override delete, or write a Scheduling-local fallback history table.
- **Mass-effective-from change:** if Admin needs to shift the entire routine by 30 minutes (e.g. exam season), is a bulk-shift operation needed or is "replace the routine" sufficient?
