# 🧩 MODULE: CLASS, BATCH & SUBJECT CONTEXT

## 1. Purpose

Define the academic structure that every other CC-LMS module depends on: **Class** (the grade-level container), **Subject** (a teachable unit), **Batch** (a group of students taught together under a class), and the mappings between them. This module owns the *resolution truth* — given any batch, it can return the effective subject list, the teacher per subject, and the enrolled students. Scheduling, Class Workspaces, Live Class, Attendance, Assessment, and Notification all consume this resolution; none re-derive it.

## 2. Scope

**In Scope**

- Class creation, edit, archive
- Global subject registry (re-usable across classes)
- Class-to-subject assignment (mandatory baseline subject set per class)
- Batch creation under a class
- Batch-level subject override (optional replacement of inherited subject set)
- Teacher assignment per (batch, subject) pair
- Student placement into a batch (one active batch per student) and history of moves
- A canonical resolution API that returns the full context for a batch
- Batch capacity tracking and enforcement
- Status lifecycle (`ACTIVE`, `INACTIVE`, `ARCHIVED`) for class, subject, batch

**Out of Scope**

- Scheduling of class occurrences — Class Scheduling module
- Student admission and registration — Admission Management
- Teacher recruitment, payroll, leave — User Management / Payroll
- Exam logic, evaluation, results — Assessment Management
- Attendance capture — Attendance Management
- Cross-tenant subject sharing (see ADR-0003 — pending)

## 3. Actors

- **Admin** — sole authority to create / edit / archive class, subject, batch, and to assign teachers and place students.
- **Teacher** — read-only view of their own (batch, subject) mappings; cannot edit structure.
- **Student** — read-only consumer of "which batch am I in, what subjects does it cover" via downstream modules; does not interact with this module directly.
- **System** — resolution engine; consumed by every dependent module via the context API.

## 4. Core Concepts

### Class

A grade-level container, e.g. "Class 10", "HSC 1st Year". Has a name, an institution-defined `code`, and a status. A class owns a **mandatory subject set** via `ClassSubject`; every batch under it inherits this set unless overridden.

### Subject

A teachable unit, e.g. "Physics", "Higher Math". Subjects are **global** to the institution by default (`is_global = true`) so they can be reused across classes. A subject becomes part of a class's curriculum only by entry in `ClassSubject`.

### ClassSubject

The mandatory baseline link between a class and a subject. Every batch under a class inherits this set as its default. Removing a `ClassSubject` row affects all batches that have not overridden it.

### Batch

A group of students taught under a single class. Identified by class + name (e.g. "Class 10 — Batch A — Morning"). Has a `batch_type` (`ONLINE`, `OFFLINE`, `HYBRID`), a `capacity`, and a status. A batch may **override** its subject set; if it does, the override entries replace inheritance entirely (not merge).

### BatchSubjectOverride

Optional. If a batch has *any* rows in this table, its effective subjects are those rows; the inherited class subjects are ignored. If empty, inheritance applies. Override subjects must come from the institution's subject pool (existing `Subject` rows).

### BatchSubjectTeacher

The critical operational table: maps `(batch, subject) → teacher`. Every scheduled class, attendance record, assessment, and dashboard widget for a batch's subject reads from here.

### StudentBatch

Records that a student belongs to a batch. A student has **exactly one** active `StudentBatch` row at a time; moves are tracked as separate rows with `joined_at` / `left_at`.

### Resolution

The operation of asking "given a batch_id, what is the effective subject list and the teacher per subject?". Resolution is centralised — every downstream module calls the resolution API rather than running its own joins.

## 5. Functional Requirements

### 5.1 Class Management

- **FR-CBS-001:** Admin shall be able to create a class with `name`, unique `code`, and status `ACTIVE`.
- **FR-CBS-002:** System shall enforce uniqueness of `Class.code` within the institution.
- **FR-CBS-003:** Admin shall be able to edit a class's `name` and `code` at any time. Codes are not foreign keys; downstream reports that embed a code as text show the new code on next render. See OQ for whether to add a confirmation prompt.
- **FR-CBS-004:** Admin shall be able to set a class to `ARCHIVED`. Archived classes cannot have new batches created under them but existing batches continue to operate until separately archived.
- **FR-CBS-005:** System shall block deletion of a class that has any non-archived batch; archival is the supported deactivation path.

### 5.2 Subject Management

- **FR-CBS-006:** Admin shall be able to create a global subject with `name`, unique `code`, and `is_global = true`.
- **FR-CBS-007:** System shall enforce uniqueness of `Subject.code` within the institution.
- **FR-CBS-008:** Admin shall be able to set a subject to `INACTIVE`. Inactive subjects cannot be assigned to new classes or batch overrides, but existing assignments remain valid until separately removed.
- **FR-CBS-009:** System shall block deletion of a subject referenced by any `ClassSubject`, `BatchSubjectOverride`, or `BatchSubjectTeacher`; deactivation is the supported path.

### 5.3 Class-to-Subject Assignment

- **FR-CBS-010:** Admin shall be able to add one or more subjects to a class via `ClassSubject`.
- **FR-CBS-011:** Admin shall be able to remove a subject from a class. The removal is blocked if any batch under that class has neither overridden subjects nor an alternate teacher mapping for the subject (see edge case 12.2).
- **FR-CBS-012:** System shall require a class to have at least one `ClassSubject` row before any batch under it can transition to `ACTIVE`.

### 5.4 Batch Management

- **FR-CBS-013:** Admin shall be able to create a batch under an `ACTIVE` class, supplying `name`, `batch_type` (`ONLINE` / `OFFLINE` / `HYBRID`), `capacity` (positive integer), and initial status `INACTIVE`. Creation is blocked with `CLASS_HAS_NO_SUBJECTS` if the class has no `ClassSubject` rows (since the new batch would have an empty effective subject set on inheritance).
- **FR-CBS-014:** Admin shall be able to transition a batch to `ACTIVE`. The transition is allowed only when the batch has an effective subject set (inherited or overridden) of at least 1 subject **and** every effective subject has an `ACTIVE` `BatchSubjectTeacher` row.
- **FR-CBS-015:** Admin shall be able to edit a batch's `capacity`. Reducing capacity below the current enrolled student count is blocked with `CAPACITY_TOO_LOW`.
- **FR-CBS-016:** Admin shall be able to `ARCHIVE` a batch. An archived batch is read-only for downstream modules.
- **FR-CBS-016a:** System shall emit a `BATCH_ARCHIVED` event on every successful FR-CBS-016 transition. Payload: `{ event_type, batch_id, batch_name, class_id, archived_at, actor_id }`. Consumed by Scheduling (auto-archives PUBLISHED routine), Assessment (auto-closes DRAFT/PUBLISHED assessments for the batch within 5 s), Student Workspace (drops the batch's Today/Upcoming list), Teacher Workspace, Notification Management.
- **FR-CBS-017:** System shall block enrolment when the batch's current student count equals `capacity`; the rejection returns `BATCH_FULL`.

### 5.5 Subject Inheritance and Override

- **FR-CBS-018:** System shall, by default, treat a batch's effective subject set as equal to its class's `ClassSubject` set.
- **FR-CBS-019:** Admin shall be able to add `BatchSubjectOverride` rows for a batch. The presence of *any* override row switches the batch's effective subjects to the override set (replace, not merge).
- **FR-CBS-020:** Admin shall be able to remove all override rows for a batch; removal restores inheritance from the class.
- **FR-CBS-021:** System shall reject an override subject that is not an `ACTIVE` `Subject` row.
- **FR-CBS-022:** System shall, on any change to effective subjects, invalidate `BatchSubjectTeacher` rows whose subject is no longer effective (mark them `INACTIVE`; do not hard-delete; Admin must re-assign).

### 5.6 Teacher Mapping

- **FR-CBS-023:** Admin shall be able to assign a teacher to a `(batch, subject)` pair via `BatchSubjectTeacher`.
- **FR-CBS-024:** System shall enforce that `BatchSubjectTeacher.subject_id` belongs to the batch's effective subject set; otherwise reject `SUBJECT_NOT_IN_BATCH`.
- **FR-CBS-025:** System shall, by default, allow at most one **active** `BatchSubjectTeacher` row per `(batch, subject)` pair. Multi-teacher policy is an open question (see OQ).
- **FR-CBS-026:** Admin shall be able to replace a teacher on a `(batch, subject)` pair. The prior row is deactivated (kept for history); a new active row is created.
- **FR-CBS-026a:** Admin shall be able to reactivate the most recent `INACTIVE` `BatchSubjectTeacher` row for a `(batch, subject)` pair when the subject is back in the effective set and no other active row exists. Reactivation sets `status = ACTIVE` and clears `deactivated_at`. Older `INACTIVE` rows remain as history and cannot be reactivated directly (Admin must create a new row).
- **FR-CBS-026b:** System shall emit a `BATCH_SUBJECT_TEACHER_REASSIGNED` event on every change to the active teacher of a `(batch, subject)` pair (FR-CBS-026 replacement or FR-CBS-026a reactivation). Payload: `{ event_type, batch_id, subject_id, old_teacher_id, new_teacher_id, effective_at, actor_id }`. Consumed by Scheduling (raises stale-entry warnings) and the Workspace modules (drop affected future occurrences from teacher lists).
- **FR-CBS-027:** Teacher shall be able to list their own active `BatchSubjectTeacher` assignments via the read API.

### 5.7 Student Placement

- **FR-CBS-028:** Admin shall be able to place a student into a batch via `StudentBatch`, provided capacity allows.
- **FR-CBS-029:** System shall ensure a student has at most one `StudentBatch` row with `left_at = null` at any time.
- **FR-CBS-030:** Admin shall be able to move a student from one batch to another. The current row is closed (`left_at = now`) and a new row is opened.
- **FR-CBS-031:** System shall preserve the full `StudentBatch` history (do not delete prior rows) so academic records can attribute a student to a batch for any past date.
- **FR-CBS-031a:** System shall emit a `STUDENT_BATCH_CHANGED` event on FR-CBS-030 success. Payload: `{ event_type, student_id, old_batch_id, new_batch_id, changed_at, actor_id }`. Consumed by the Student Class Workspace (refreshes the student's effective list within 5 s SLA) and Notification Management (informs the student via SMS / in-app).

### 5.8 Resolution API

- **FR-CBS-032:** System shall expose a single canonical endpoint `GET /batches/{id}/context` that returns: the batch, its class, the effective subject set, the active teacher per subject, and the current student roster.
- **FR-CBS-033:** System shall guarantee the resolution returns within p95 < 100 ms under nominal load (cached or otherwise — implementation detail).
- **FR-CBS-034:** System shall invalidate any cached resolution for a batch on any write that affects it. Direct triggers (synchronous, before write returns success): batch overrides, teacher mapping for that batch, student placement in/out of that batch, batch status. Fan-out triggers (async, via a queued job): class-level changes that affect N batches under that class — `ClassSubject` add/remove and class status change. The queue ensures bulk operations on a 2,000-batch institution do not block the originating write; downstream readers may observe a stale resolution for up to the queue drain time (target < 5 s p95).

## 6. Business Rules

- A class must have **at least one** mandatory subject before any of its batches can go `ACTIVE`.
- A batch's effective subject set is **either** the class's inherited set **or** the batch's override set — never a merge.
- Teacher is mapped **per (batch, subject)**, never to a batch alone.
- Override subjects must already exist in the global subject pool and be `ACTIVE`.
- One active student-to-batch row per student; history is preserved across moves.
- Batch capacity is a **hard** ceiling for enrolment; cannot be exceeded by enrolment writes.
- Deletion is not supported for class, subject, or batch; archival/deactivation is the path. This protects historical referential integrity for Attendance, Assessment, and Accounting.
- Status enums: Class and Batch use `ACTIVE`, `INACTIVE`, `ARCHIVED`. Subject uses `ACTIVE`, `INACTIVE` only (subjects are a shared catalog; deactivation is always reversible). `ARCHIVED` is terminal; `INACTIVE` is reversible.
- Multi-system support (Coaching / National Institute / Private Tutor) is implemented by configuration, not by separate code paths — see OQ.

## 7. User Flow / Process Flow

### 7.1 Academic Setup (first-time)

1. Admin opens **Class & Subject Setup**.
2. Admin creates the institution's global subjects (one-time seed).
3. Admin creates a class (e.g. "Class 10") with `INACTIVE` status.
4. Admin assigns one or more subjects to the class via `ClassSubject`.
5. Class is set `ACTIVE`.

### 7.2 Batch Creation

1. Admin selects a class on **Batch Management**.
2. Admin creates a batch with name, batch_type, capacity. Status starts `INACTIVE`.
3. System auto-resolves effective subjects = class's `ClassSubject` set.
4. Admin opens the **Teacher Mapping Grid** for the batch.
5. Admin assigns a teacher to each effective subject.
6. Admin sets batch `ACTIVE` (gated by FR-CBS-014).

### 7.3 Subject Override (optional)

1. Admin selects an existing batch.
2. Admin opens the **Subject Override** panel.
3. Admin selects a subset of subjects from the pool.
4. System replaces the effective subject set; any prior `BatchSubjectTeacher` rows for subjects no longer in scope are marked `INACTIVE`.
5. Admin re-assigns teachers for any newly added subjects.

### 7.4 Teacher Replacement

1. Admin opens the **Teacher Mapping Grid** for a batch.
2. Admin selects a `(batch, subject)` row and chooses a new teacher.
3. System deactivates the existing row and creates a new active row (history retained).

### 7.5 Student Move

1. Admin opens the student's profile (in User Management).
2. Admin triggers "Move to another batch" with the target batch id.
3. System validates target batch capacity (FR-CBS-017).
4. System closes the current `StudentBatch` row (`left_at = now`) and opens a new one.

### 7.6 Resolution (System flow)

1. Downstream module (Scheduling, Attendance, etc.) calls `GET /batches/{id}/context`.
2. System checks resolution cache for the batch.
3. On miss: resolve effective subjects, teachers, students; cache.
4. Return the structured response.

## 8. Data Model

### `Class`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | string | e.g. "Class 10" |
| code | string, unique | e.g. "C10" |
| status | enum(`ACTIVE`, `INACTIVE`, `ARCHIVED`) | Default `INACTIVE` on create |
| created_by | UUID, FK → User.id | Admin who created |
| updated_by | UUID, FK → User.id | Admin who last edited |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| archived_at | Timestamp, nullable | Set when status → ARCHIVED |

### `Subject`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | string | e.g. "Physics" |
| code | string, unique | e.g. "PHY" |
| is_global | boolean | Default true; reserved for future per-class private subjects |
| status | enum(`ACTIVE`, `INACTIVE`) | Subjects do not use `ARCHIVED`; deactivation is reversible (see Business Rules — subjects are a shared catalog and may be revived) |
| created_by | UUID, FK → User.id | |
| updated_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| updated_at | Timestamp | |

### `ClassSubject`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| class_id | UUID, FK → Class.id | |
| subject_id | UUID, FK → Subject.id | |
| is_mandatory | boolean | Currently always true (reserved for future "optional" subjects) |
| created_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| | | Unique: (class_id, subject_id) |

### `Batch`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| class_id | UUID, FK → Class.id | |
| name | string | e.g. "Batch A — Morning" |
| batch_type | enum(`ONLINE`, `OFFLINE`, `HYBRID`) | Affects which delivery modules apply |
| capacity | integer, positive | Max active enrolment |
| status | enum(`ACTIVE`, `INACTIVE`, `ARCHIVED`) | |
| created_by | UUID, FK → User.id | |
| updated_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| archived_at | Timestamp, nullable | |
| | | Unique: (class_id, name) |

### `BatchSubjectOverride`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| batch_id | UUID, FK → Batch.id | |
| subject_id | UUID, FK → Subject.id | Must be ACTIVE at insert |
| created_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| | | Unique: (batch_id, subject_id) |

### `BatchSubjectTeacher`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| batch_id | UUID, FK → Batch.id | |
| subject_id | UUID, FK → Subject.id | Must be in batch's effective subject set |
| teacher_id | UUID, FK → User.id | Must have role `TEACHER` |
| status | enum(`ACTIVE`, `INACTIVE`) | History preserved; only one ACTIVE per (batch, subject) |
| created_by | UUID, FK → User.id | |
| created_at | Timestamp | |
| deactivated_at | Timestamp, nullable | Set when replaced |
| | | Partial unique index: (batch_id, subject_id) WHERE status = 'ACTIVE' |

### `StudentBatch`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| student_id | UUID, FK → User.id | Must have role `STUDENT` |
| batch_id | UUID, FK → Batch.id | |
| joined_at | Timestamp | When the student entered this batch |
| left_at | Timestamp, nullable | Set when student moves away; null = currently active |
| created_by | UUID, FK → User.id | Admin who placed the student |
| created_at | Timestamp | |
| | | Partial unique index: (student_id) WHERE left_at IS NULL |

## 9. API Contracts

### Create Class

```http
POST /classes
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

**Request**

```json
{ "name": "Class 10", "code": "C10" }
```

**Response (201)**

```json
{
  "id": "0192c2cb-...",
  "name": "Class 10",
  "code": "C10",
  "status": "INACTIVE"
}
```

**Errors**

- `400` — `INVALID_INPUT`
- `409` — `CODE_CONFLICT` (code already in use)

Satisfies: FR-CBS-001, FR-CBS-002.

### Edit Class

```http
PATCH /classes/{class_id}
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

**Request** (any subset of fields)

```json
{ "name": "Class 10 — Renamed", "code": "C10A" }
```

**Response (200)** — full class object.

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `CLASS_NOT_FOUND`
- `409` — `CODE_CONFLICT`

Satisfies: FR-CBS-003.

### Change Class Status (activate / deactivate / archive)

```http
POST /classes/{class_id}/status
```

**Request**

```json
{ "status": "ACTIVE" }
```

**Response (200)**

```json
{ "id": "0192...", "status": "ACTIVE" }
```

**Errors**

- `400` — `INVALID_INPUT` (unknown status)
- `404` — `CLASS_NOT_FOUND`
- `409` — `INVALID_TRANSITION` (e.g. `ARCHIVED → *`) or `HAS_ACTIVE_BATCHES` (when archiving)

Satisfies: FR-CBS-004, FR-CBS-005.

### Create Subject

```http
POST /subjects
```

**Request**

```json
{ "name": "Physics", "code": "PHY" }
```

**Response (201)**

```json
{ "id": "0192...", "name": "Physics", "code": "PHY", "is_global": true, "status": "ACTIVE" }
```

**Errors**

- `400` — `INVALID_INPUT`
- `409` — `CODE_CONFLICT`

Satisfies: FR-CBS-006, FR-CBS-007.

### Change Subject Status

```http
POST /subjects/{subject_id}/status
```

**Request**

```json
{ "status": "INACTIVE" }
```

**Response (200)** — updated subject object.

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `SUBJECT_NOT_FOUND`

Satisfies: FR-CBS-008.

### Remove Subject from Class

```http
DELETE /classes/{class_id}/subjects/{subject_id}
```

**Response (204)** — empty body.

**Errors**

- `404` — `CLASS_NOT_FOUND` or `SUBJECT_NOT_IN_CLASS`
- `409` — `SUBJECT_IN_USE` (response body lists dependent batches that prevent removal — those without override that have an active mapping for this subject)

Satisfies: FR-CBS-011.

### Assign Subjects to Class

```http
POST /classes/{class_id}/subjects
```

**Request**

```json
{ "subject_ids": ["0192...", "0192..."] }
```

**Response (200)**

```json
{ "class_id": "0192...", "subjects": [{ "id": "0192...", "name": "Physics", "code": "PHY" }] }
```

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `CLASS_NOT_FOUND`
- `409` — `SUBJECT_INACTIVE` (a referenced subject is not ACTIVE)

Satisfies: FR-CBS-010.

### Create Batch

```http
POST /classes/{class_id}/batches
```

**Request**

```json
{
  "name": "Batch A — Morning",
  "batch_type": "OFFLINE",
  "capacity": 40
}
```

**Response (201)**

```json
{
  "id": "0192...",
  "class_id": "0192...",
  "name": "Batch A — Morning",
  "batch_type": "OFFLINE",
  "capacity": 40,
  "status": "INACTIVE"
}
```

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `CLASS_NOT_FOUND`
- `409` — `CLASS_ARCHIVED` or `CLASS_HAS_NO_SUBJECTS` or `BATCH_NAME_CONFLICT`

Satisfies: FR-CBS-013.

### Edit Batch

```http
PATCH /batches/{batch_id}
```

**Request** (any subset)

```json
{ "name": "Batch A — Morning (renamed)", "capacity": 45, "batch_type": "HYBRID" }
```

**Response (200)** — full batch object.

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `BATCH_NOT_FOUND`
- `409` — `CAPACITY_TOO_LOW` (reducing capacity below current enrolment) / `BATCH_NAME_CONFLICT`

Satisfies: FR-CBS-015.

### Archive Batch

```http
POST /batches/{batch_id}/archive
```

**Response (200)**

```json
{ "id": "0192...", "status": "ARCHIVED", "archived_at": "2026-05-18T08:00:00+06:00" }
```

**Errors**

- `404` — `BATCH_NOT_FOUND`
- `409` — `HAS_ENROLLED_STUDENTS` (response lists active student count; Admin must move students out first)

Satisfies: FR-CBS-016.

### List Teacher Assignments (for current teacher)

```http
GET /me/teaching-assignments
```

**Headers**

```
Authorization: Bearer <teacher access_token>
```

**Response (200)**

```json
{
  "assignments": [
    {
      "batch": { "id": "0192...", "name": "Batch A — Morning", "class": { "name": "Class 10", "code": "C10" } },
      "subject": { "id": "0192...", "name": "Physics", "code": "PHY" }
    }
  ]
}
```

Satisfies: FR-CBS-027.

### Set Batch Subject Override

```http
PUT /batches/{batch_id}/subject-override
```

**Request**

```json
{ "subject_ids": ["0192...", "0192..."] }
```

Empty array clears the override and restores inheritance.

**Response (200)**

```json
{ "batch_id": "0192...", "effective_subjects": [ ... ] }
```

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `BATCH_NOT_FOUND`
- `409` — `SUBJECT_INACTIVE`

Satisfies: FR-CBS-019, FR-CBS-020, FR-CBS-021, FR-CBS-022.

### Assign Teacher to Batch+Subject

```http
PUT /batches/{batch_id}/teachers
```

**Request**

```json
{ "subject_id": "0192...", "teacher_id": "0192..." }
```

**Response (200)**

```json
{ "batch_id": "0192...", "subject_id": "0192...", "teacher_id": "0192...", "status": "ACTIVE" }
```

**Errors**

- `400` — `INVALID_INPUT`
- `404` — `BATCH_NOT_FOUND` / `SUBJECT_NOT_FOUND` / `TEACHER_NOT_FOUND`
- `409` — `SUBJECT_NOT_IN_BATCH` / `USER_NOT_TEACHER`

Satisfies: FR-CBS-023, FR-CBS-024, FR-CBS-025, FR-CBS-026.

### Activate Batch

```http
POST /batches/{batch_id}/activate
```

**Response (200)**

```json
{ "id": "0192...", "status": "ACTIVE" }
```

**Errors**

- `409` — `NO_EFFECTIVE_SUBJECTS` or `MISSING_TEACHER_MAPPING` (lists missing subjects)

Satisfies: FR-CBS-014.

### Place Student in Batch

```http
POST /batches/{batch_id}/students
```

**Request**

```json
{ "student_id": "0192..." }
```

**Response (201)**

```json
{ "student_id": "0192...", "batch_id": "0192...", "joined_at": "2026-05-18T08:00:00+06:00" }
```

**Errors**

- `409` — `BATCH_FULL` / `STUDENT_ALREADY_IN_BATCH` / `USER_NOT_STUDENT`

Satisfies: FR-CBS-028, FR-CBS-029.

### Move Student to Another Batch

```http
POST /students/{student_id}/move
```

**Request**

```json
{ "target_batch_id": "0192..." }
```

**Response (200)**

```json
{
  "student_id": "0192...",
  "previous_batch_id": "0192...",
  "left_at": "2026-05-18T08:00:00+06:00",
  "new_batch_id": "0192...",
  "joined_at": "2026-05-18T08:00:00+06:00"
}
```

**Errors**

- `409` — `BATCH_FULL` / `SAME_BATCH`

Satisfies: FR-CBS-030, FR-CBS-031.

### Resolution — Get Batch Context

```http
GET /batches/{batch_id}/context
```

`source` is a computed field (`"INHERITED"` if the subject came from the class's `ClassSubject` set, `"OVERRIDE"` if from `BatchSubjectOverride`). It is not stored.

**Response (200)** — returned for `ACTIVE`, `INACTIVE`, *and* `ARCHIVED` batches (downstream modules use this for historical display; the caller inspects `status` to decide read-only vs writable).

```json
{
  "batch": {
    "id": "0192...",
    "name": "Batch A — Morning",
    "class": { "id": "0192...", "name": "Class 10", "code": "C10" },
    "batch_type": "OFFLINE",
    "capacity": 40,
    "enrollment_count": 12,
    "status": "ACTIVE"
  },
  "subjects": [
    {
      "id": "0192...",
      "name": "Physics",
      "code": "PHY",
      "source": "INHERITED",
      "teacher": { "id": "0192...", "name": "A. Rahman" }
    }
  ],
  "students": [
    { "id": "0192...", "name": "Karim Hossain", "joined_at": "..." }
  ]
}
```

**Errors**

- `404` — `BATCH_NOT_FOUND`

Satisfies: FR-CBS-032.

## 10. UI Components

### Screen: Class & Subject Setup

- **Components:** Class list with status badges; subject panel (multi-select against global subject pool); per-class "Mandatory subjects" count; Archive button (with confirm modal).
- **Actions:** Create class, edit class, archive class, add/remove subjects.
- **States:** loading, empty (no classes yet), populated, archived-filter active.

### Screen: Subject Catalog

- **Components:** Subject list (name, code, status, usage count "in N classes / M batches"), Add Subject form, deactivate/activate toggle.
- **Actions:** Create subject, deactivate (with warning if referenced), edit metadata.
- **States:** loading, populated, deactivation-blocked (shows references).

### Screen: Batch Management

- **Components:** Batch list under a selected class; capacity indicator (`12 / 40`); status badge; batch_type chip; teacher-coverage indicator ("3/4 subjects covered").
- **Actions:** Create batch, edit metadata, archive batch, open Subject Override, open Teacher Mapping Grid, activate batch.
- **States:** loading, empty, populated, full (capacity reached badge), incomplete-mapping (warning).

### Screen: Subject Override Panel

- **Components:** Two columns — "Inherited from class" (read-only chips) and "Override subjects" (multi-select); "Restore inheritance" button.
- **Actions:** Add override, remove individual subject, clear all overrides.
- **States:** inheriting (no overrides), overridden (overrides present), saving, error.

### Screen: Teacher Mapping Grid

- **Components:** Grid with effective subjects as rows, current teacher as cell content, change-teacher action per row; "missing teacher" row highlight.
- **Actions:** Assign teacher (opens teacher picker filtered by role = TEACHER), replace teacher.
- **States:** loading, complete (all subjects mapped), incomplete (one or more missing).

### Screen: Move Student (modal, opened from User Management)

- **Components:** Current batch summary, target batch selector (filtered to ACTIVE, with capacity remaining), reason note (optional), Confirm button.
- **Actions:** Submit move.
- **States:** idle, validating, capacity-full (target rejected), submitting, success.

## 11. Validation Rules

- **Class.code:** 1–16 chars, ASCII letters/digits/hyphens, unique within institution.
- **Class.name:** 1–80 chars, may contain Bangla; trimmed; required.
- **Subject.code:** 1–16 chars, ASCII letters/digits, unique within institution.
- **Subject.name:** 1–80 chars, may contain Bangla; required.
- **Batch.name:** 1–80 chars, unique within `(class_id, name)`; may contain Bangla.
- **Batch.capacity:** integer, `1 ≤ capacity ≤ 500` (institutional sanity ceiling; configurable).
- **Batch.batch_type:** one of `ONLINE`, `OFFLINE`, `HYBRID`.
- **Status transitions:** `INACTIVE → ACTIVE`, `ACTIVE → INACTIVE`, `ACTIVE|INACTIVE → ARCHIVED`. `ARCHIVED → *` is rejected.
- **Subject assignment to class:** referenced `Subject.status` must be `ACTIVE` at insert.
- **Batch override:** referenced `Subject.status` must be `ACTIVE` at insert.
- **Teacher mapping:** target user must have `role = TEACHER` and `is_active = true`.
- **Student placement:** target user must have `role = STUDENT` and `is_active = true`; capacity must allow.
- **Subject expertise check** (FR open — see OQ): currently *not* enforced at the data layer; Admin discretion.

## 12. Edge Cases

- Override removed (last row deleted) → effective subjects fall back to class inheritance; `BatchSubjectTeacher` rows whose subject is in the restored inherited set become valid again (no automatic reactivation — Admin must explicitly reactivate).
- Subject removed from class while a batch (without override) has it scheduled or has active teacher mapping → removal is blocked with `SUBJECT_IN_USE` listing the dependent batches.
- Teacher assigned then their user `is_active = false` → all active `BatchSubjectTeacher` rows for that teacher are marked `INACTIVE`; affected batches show "missing teacher" warning. Triggered by User Management's deactivation flow.
- Subject set to `INACTIVE` while present in an `ACTIVE` batch's effective set → existing mappings continue to work but new mappings or overrides referencing the subject are rejected.
- Batch capacity reduced below current enrolment → rejected with `CAPACITY_TOO_LOW`; Admin must move students out first.
- Concurrent enrolment writes to a batch near capacity → enforced by a row-level lock on the batch's enrolment count or a unique constraint trick; only one of two concurrent writes succeeds at the cap.
- Student moved to a batch with a different subject set → the student's prior `StudentBatch` row is closed; downstream modules see the new batch's effective subjects from the resolution API.
- Batch archived while students are enrolled → archival is blocked; Admin must move students out (or graduate/discharge) first.
- Class archived while batches exist → archival is blocked; archive the batches first.
- Override added that drops a subject the batch had teachers assigned for → those `BatchSubjectTeacher` rows are marked `INACTIVE` automatically (FR-CBS-022).
- Two admins concurrently edit the same batch's override → last-write-wins on the override set; both admins receive the resolution-cache invalidation event so their UI refreshes.
- Subject re-activated after being inactive → does **not** auto-reattach to anything; Admin must explicitly add it back to a class or batch.
- A batch with `HYBRID` type but no `ONLINE` delivery configuration in Class Scheduling → out of scope for this module; Scheduling owns its own validation.
- Student record deleted in User Management → blocked by FK; deactivation flow closes their `StudentBatch` row instead.
- Resolution called for an **`ARCHIVED` batch** → endpoint returns `200` with the full context and `status: "ARCHIVED"`. Downstream modules (Attendance history, Assessment results, transcripts) need this for historical display. Callers inspect `status` and treat the response as read-only when archived.
- A subject is set to `INACTIVE` while present in an `ACTIVE` batch's `BatchSubjectOverride` row → the override row remains (override subjects are treated as a *mapping*, not a fresh assignment); the batch continues operating until Admin removes the override entry. New overrides referencing the inactive subject are still rejected per FR-CBS-021.
- Redis unavailable → resolution falls through to the database (fail-open) at higher latency; the p95 < 100 ms NFR is not honoured during the degraded window. Reads continue to succeed; writes still invalidate on a best-effort basis (retried via queue). Conscious trade-off — see NFR §14.
- Two admins concurrently call `POST /batches/{batch_id}/activate` for the same batch → the activation write uses a row-level lock on `Batch`; the second call returns success idempotently (status already `ACTIVE`) without duplicate side effects.
- A `ClassSubject` row removed for a subject that is the only subject of a batch using inheritance → FR-CBS-011 blocks the removal (the subject has an active teacher mapping there). Admin must add an override on the batch (with different subjects) first, then remove the class subject.
- Removing the **last** override row when the resulting inherited set leaves the batch without a complete teacher mapping → allowed at the data layer; the batch is auto-transitioned to `INACTIVE` and the UI surfaces "missing teacher" warnings until Admin re-assigns (FR-CBS-022 + FR-CBS-014).

## 13. Dependencies

This module is foundational — it is consumed by, not dependent on, most others. Direct dependencies:

- **User Management** — provides `User` rows with `role` (Teacher / Student) and `is_active`. Class/Batch refers to teacher and student via FK to `User.id`. Deactivation flow in User Management triggers cleanup here (FR-CBS-022, edge case "teacher deactivated").
- **Redis** (per ADR-0002) — backs the resolution cache (FR-CBS-033, FR-CBS-034).

Consumed by (downstream — listed for traceability):

- Class Scheduling — uses `GET /batches/{id}/context` to know what to schedule
- Teacher Class Workspace / Student Class Workspace — present resolved context
- Live Class — operates on scheduled occurrences derived from this context
- Attendance — records per (batch, subject, occurrence) using resolved teacher
- Assessment — links assessments to (batch, subject)
- Notification — addresses messages by batch / class roster
- Dashboard — aggregates by batch / class

## 14. Non-Functional Requirements

- **Performance:** `GET /batches/{id}/context` p95 < 100 ms when Redis cache is healthy; uncached cold path p95 < 300 ms. **Degraded mode** — when Redis is unavailable, the API fails open (reads pass through to Postgres); p95 latency rises to the cold-path target and the SLA is explicitly waived until Redis recovers. Writes invalidate the cache on a best-effort basis; failures are retried via a queued job. A `503 Service Unavailable` is returned only when *both* Redis and Postgres are unreachable.
- **Consistency:** Write operations that change effective resolution (FR-CBS-034 triggers) must invalidate the cache before returning success — no stale resolution after a write.
- **Concurrency:** Capacity enforcement and "one active StudentBatch per student" use database-level constraints (partial unique indexes) so the invariants hold under concurrent writes.
- **Scale:** Tested up to 200 classes, 2,000 batches, 100,000 students, 1,000 teachers per institution.
- **Accessibility:** Class & Batch admin screens meet WCAG 2.1 AA; teacher-mapping grid is keyboard-navigable.
- **Internationalization:** Class and subject names support Bangla; codes are ASCII for portability across systems.
- **Auditability:** Every write captures `created_by` / `updated_by` and writes a row to the central `audit_log` (schema location TBD — see Auth SRS section 13 for the same forward-reference).

## 15. Assumptions

- Subjects are seeded once per institution by an Admin; subject catalog evolves rarely.
- Bangladesh curriculum aligns with class-as-grade structure (Class 6 through HSC 2nd Year); the model accommodates this without extension.
- Teacher expertise (which subjects a teacher is qualified for) is *not* enforced by data integrity in v1; Admin chooses appropriately. See OQ.
- A student belongs to exactly one active batch at any time. Multi-batch students (e.g. specialty coaching) are not in v1.
- Multi-system support (Coaching / National Institute / Private Tutor) is realised through configuration (defaults, naming, optional fields) rather than separate schemas — see OQ.
- "Global subjects" means subjects available across classes within one institution. Cross-institution sharing depends on ADR-0003 (multi-tenancy).

## 16. Open Questions

- **Multi-teacher per (batch, subject)?** Original SRS flagged this. Current FRs assume single active teacher. If lifted, FR-CBS-025 changes and the Teacher Mapping Grid must show multiple cells per row.
- **Subject weightage?** Original SRS asked whether subjects carry weight (e.g. credit hours, marks share). Currently no field; if needed, add `ClassSubject.weight` and propagate to Assessment.
- **Teacher subject expertise:** should the system store which subjects a teacher is qualified to teach (e.g. on `User.teacher_subjects`), and reject assignments that violate it? Currently Admin discretion.
- **Capacity enforcement strictness:** hard block (current) vs. soft warning with override (e.g. "exceed only with confirm")? Some coaching centres run 5–10% over capacity by policy.
- **Multi-batch student:** allowed for specialty coaching (e.g. student in "Class 10 — Main" plus "Class 10 — Physics-only")? If yes, FR-CBS-029 must be relaxed.
- **Multi-system modes:** is Coaching / National Institute / Private Tutor a runtime config switch, an ADR-level architectural choice, or merely a default-pack? Needs a decision before per-mode features land.
- **Class.code change after batches reference it:** currently allowed (codes are independent of FKs). Should it be blocked or warning-only? FR-CBS-003 marks it warning-only pending decision.
- **Subject deactivation propagation:** when a subject is set INACTIVE, should existing assignments remain valid (current behaviour) or be auto-deactivated? Different institutions may want different defaults.
- **Override semantics — replace vs. merge:** the SRS specifies replace. Some users have asked for merge (inherit + add). Confirm or capture as future feature.
- **Bangladesh-specific:** does the institution use NCTB subject codes? If so, `Subject.code` should align (e.g. "PHY-100" vs free-form).
- **Override row as mapping vs. assignment:** when a subject is `INACTIVE`-ed, should existing `BatchSubjectOverride` rows referencing it be auto-removed, or stay as "mapping" (current behaviour)? Different institutions may want different defaults.
- **`User.deleted_at` coupling:** auth.md's `User` has both `is_active` and `deleted_at`. This module checks `is_active` only. Confirm with User Management that `deleted_at` is set only via a flow that also sets `is_active = false`, so the single check is sufficient; otherwise add a second check.
- **Class status `ACTIVE`-from-`INACTIVE` confirmation prompt:** since `Class.code` changes are not blocked but downstream reports may reference the old code in stored text, decide whether Admin should see a "this will appear in N reports" confirmation on edit.
