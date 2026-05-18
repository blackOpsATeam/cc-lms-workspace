# 🧩 MODULE: TEACHER CLASS WORKSPACE

## 1. Purpose

Provide each teacher with a class-occurrence-centric operational home: a list of their assigned classes (Today / Upcoming / Previous), a per-occurrence detail page where they prepare instructions and resources, and one-click shortcuts to Live Class, Attendance, and Assessment. The workspace owns its own data (instructions, class-linked resources, prerecorded clip attachments) but **never** mutates the schedule, attendance records, or assessment lifecycle — those remain with their owning modules. The Teacher Class Workspace is the teacher's daily entry point; everything else is a redirect.

## 2. Scope

**In Scope**

- Assigned class list grouped into Today / Upcoming / Previous, with filters (subject, batch, delivery mode) and search
- Class detail view per scheduled occurrence
- Class instructions (student-visible and teacher-private notes), per-occurrence or per-recurring-slot
- Class-linked resources: uploaded files, external links, prerecorded support clips
- Resource visibility control (draft / published / hidden) and scope (occurrence vs recurring slot)
- Resource ordering within a class
- Pass-through shortcuts to Live Class, Attendance, Assessment (this module does not implement those)
- Linked assessment panel showing summary metrics for assessments tied to the class
- Class history view (read-only) for previous occurrences

**Out of Scope**

- Schedule creation or editing (Scheduling module)
- Conflict detection (Scheduling module)
- Live session execution (Live Class Management)
- Attendance capture and grading (Attendance Management)
- Assessment authoring, submission, evaluation, results (Assessment Management)
- Generic content library / institution-wide repository — resources here are class-linked, not reusable assets
- File storage internals (delegated to a shared File Asset module — see Dependencies)
- Student-facing rendering of class data — owned by Student Class Workspace (this module is the *write* side; Student Workspace is the *read* side)

## 3. Actors

- **Teacher** — primary user; reads their own assigned classes, writes their own instructions and resources, opens redirects to dependent modules.
- **Admin** — read-only view of any teacher's workspace for audit and oversight; may delete a teacher's resource via admin override path (see FR-TCW-026a).
- **Student** — does **not** interact with this module directly. Student-visible instructions and `PUBLISHED` resources are read by the Student Class Workspace (which exposes them); this module is the *write* side, not the read side.
- **System** — resolves the effective class list from Scheduling, applies schedule overrides before display, marks updated/cancelled occurrences, enforces visibility and ownership rules.

## 4. Core Concepts

### Assigned Class Occurrence

A single calendar instance of a `ScheduleEntry` belonging to the calling teacher, on a specific date, with any `ScheduleOverride` applied. Identified by `(schedule_entry_id, class_date)`. Identity comes from Scheduling (FR-SCH-030a); this module never invents occurrences.

> **Naming alias:** `class_date` in this module is the same value as `occurrence_date` in `scheduling.md` (the calendar date of a specific occurrence). This module uses `class_date` consistently in its data model and APIs; when calling Scheduling endpoints, the same value is passed as `occurrence_date` (e.g. `GET /occurrences/{schedule_entry_id}/{date}`).

### Class Workspace

The operational area for one occurrence. Contains instructions, resources, prerecorded clip slot, and the shortcut buttons. A workspace is rendered on demand; nothing in this module persists "the workspace" as an entity — only its child data (instructions, resources).

### Class Instruction

A note attached to either a recurring slot (`apply_scope = RECURRING_SLOT`, applies to every occurrence of that `ScheduleEntry`) or a specific occurrence (`apply_scope = OCCURRENCE` with `class_date`). Has a type: `STUDENT_VISIBLE` (renders in Student Workspace) or `TEACHER_PRIVATE` (only the author teacher and Admin see it).

### Class Resource

A class-linked artifact: an uploaded file (via the shared File Asset module), an external URL, or a prerecorded support clip. Has a `visibility_status` (`DRAFT`, `PUBLISHED`, `HIDDEN`) controlling student access, and an `apply_scope` (`OCCURRENCE` or `RECURRING_SLOT`). Prerecorded clips are a resource subtype, distinguished by `resource_type = PRERECORDED_CLIP`.

### Linked Assessment

An assessment (owned by Assessment Management) that has been associated with a `(schedule_entry_id, occurrence_date?)` pair. This module shows linked assessments and their summary stats but does not own the assessment data. Read-only from this module's perspective.

### Effective Status (display)

A per-occurrence display marker derived from Scheduling's `ScheduleOverride`:
- `NORMAL` — no override.
- `UPDATED` — override of type `UPDATED` exists; fields changed.
- `CANCELLED` — override of type `CANCELLED` exists.

The teacher's UI surfaces this badge so they know whether the class they see today still matches the routine.

## 5. Functional Requirements

### 5.1 Assigned Class List

- **FR-TCW-001:** System shall list, for the calling teacher, all occurrences from Scheduling where the resolved teacher (post-override) is the caller's `user_id`.
- **FR-TCW-002:** System shall group occurrences into `Today`, `Upcoming` (date > today, ≤ 14 days ahead by default), and `Previous` (date < today, ≥ 90 days back by default). Default windows are configurable client-side.
- **FR-TCW-003:** System shall apply `ScheduleOverride` resolution before grouping; an occurrence's `effective_status` reflects the override state.
- **FR-TCW-004:** System shall hide occurrences whose routine is in `DRAFT` (per Scheduling FR-SCH-006) — only `PUBLISHED` routines are visible to teachers.
- **FR-TCW-005:** Each list item shall include `(batch_id, batch_name, class_name, subject_id, subject_name, date, day_of_week, start_time, end_time, delivery_mode, room_name?, live_session_ref?, effective_status, reason?)`.
- **FR-TCW-006:** System shall visually mark occurrences with `effective_status ∈ { UPDATED, CANCELLED }` distinctly from `NORMAL`; cancelled occurrences are sorted to the end of their group.
- **FR-TCW-006a:** When a teacher is reassigned at the `BatchSubjectTeacher` layer (Class Context FR-CBS-026), the workspace shall continue to show the **historical** occurrences (where this teacher was the assigned teacher at the time) until each occurrence's date passes. Future occurrences in the same `(batch, subject)` are dropped from the list on the **next** `GET /teacher/classes` call after the `BATCH_SUBJECT_TEACHER_REASSIGNED` event has been processed by this module's event consumer. Event processing SLA: ≤ 5 s after publication under nominal load.
- **FR-TCW-006b:** System shall compute `effective_status` as a derived (not stored) field on each list and detail response: `NORMAL` when no `ScheduleOverride` exists for the `(schedule_entry_id, class_date)` pair, `UPDATED` or `CANCELLED` matching the override's status.

### 5.2 Class Detail View

- **FR-TCW-007:** Teacher shall be able to open a detail view for any occurrence in their list, identified by `(schedule_entry_id, class_date)`.
- **FR-TCW-008:** Detail shall include all fields from FR-TCW-005 plus: instructions (split into STUDENT_VISIBLE and TEACHER_PRIVATE blocks), resources list, prerecorded clip panel, live class action panel, attendance shortcut, assessment panel.
- **FR-TCW-009:** For `delivery_mode ∈ { ON_SITE, HYBRID }`, detail shall display `room_name` and capacity if available.
- **FR-TCW-010:** For `delivery_mode ∈ { LIVE_ONLINE, HYBRID }`, detail shall display the live session status (pulled from Live Class Management) and a "Start session" or "Join session" button per state.
- **FR-TCW-010a:** For `delivery_mode = RECORDED_SUPPORT`, detail shall hide live action and attendance buttons (no real-time class to attend); the prerecorded clip panel is the primary content area, and student instructions/resources behave normally.
- **FR-TCW-011:** When a prerecorded clip resource is attached to the occurrence (directly or via the recurring slot), detail shall surface it in a dedicated panel above generic resources.
- **FR-TCW-012:** For occurrences with `class_date < today` (server clock in BST), detail shall be read-only for instructions and resources — existing rows remain visible but `POST`, `PATCH`, `DELETE` are rejected with `OCCURRENCE_PAST`. Linked recordings (from Live Class) and historical assessment links remain visible and updateable in their owning modules. (v1: not configurable; flagged in OQ for future per-institution tuning.)

### 5.3 Instructions

- **FR-TCW-013:** Teacher shall be able to create a `ClassInstruction` of type `STUDENT_VISIBLE` or `TEACHER_PRIVATE` on a class detail, with `apply_scope ∈ { OCCURRENCE, RECURRING_SLOT }`. `OCCURRENCE` scope requires `class_date`.
- **FR-TCW-014:** Teacher shall be able to edit or delete only instructions they authored (`created_by = caller`).
- **FR-TCW-015:** System shall enforce non-empty `content` for `STUDENT_VISIBLE` instructions; `TEACHER_PRIVATE` may be empty (draft).
- **FR-TCW-016:** System shall render `STUDENT_VISIBLE` instructions to the Student Workspace; `TEACHER_PRIVATE` is filtered out at the API layer.
- **FR-TCW-017:** Admin shall be able to read any teacher's `TEACHER_PRIVATE` notes for oversight; Admin cannot edit them. Reads via `GET /admin/teacher-workspace/classes/{schedule_entry_id}/detail?teacher_id=&class_date=`. The read is recorded in `audit_log` so teachers have a transparent trail. (Decision closed: institutional oversight is required in the Bangladesh coaching context; teachers are informed during onboarding.)

### 5.4 Resources

- **FR-TCW-018:** Teacher shall be able to create a `ClassResource` of `resource_type ∈ { FILE, LINK, PRERECORDED_CLIP }` with `title`, optional `description`, `visibility_status`, `apply_scope`, and the type-specific source (`file_asset_id` for FILE, `external_url` for LINK, either for PRERECORDED_CLIP).
- **FR-TCW-018a:** **File asset handshake.** When `file_asset_id` is provided, System shall verify with the File Asset module that the asset is in `COMMITTED` state and that `caller.user_id` is the asset's `created_by` (or has explicit attach permission). A `PENDING`, `FAILED`, or foreign-owned asset is rejected with `FILE_ASSET_NOT_COMMITTED` or `FILE_ASSET_NOT_OWNED`. The Add Resource client therefore must call this endpoint only **after** receiving the File Asset module's commit confirmation; no `ClassResource` row is created against an uncommitted asset.
- **FR-TCW-019:** System shall reject a resource without the source field appropriate to its type: `FILE` requires `file_asset_id`; `LINK` requires `external_url`; `PRERECORDED_CLIP` requires one of the two.
- **FR-TCW-020:** Teacher shall be able to edit `title`, `description`, `visibility_status`, `display_order` of resources they authored. Source fields (`file_asset_id`, `external_url`) are immutable post-create — replacing the source requires delete + re-add.
- **FR-TCW-021:** Teacher shall be able to delete only resources they authored.
- **FR-TCW-022:** Teacher shall be able to reorder resources within a `(schedule_entry_id, class_date)` scope by updating `display_order` (one or many rows in a single call).
- **FR-TCW-023:** System shall support resource type labels (subtype tags) such as `worksheet`, `note`, `slide_deck`, `revision`, `assignment_brief` via an optional `subtype` field; subtypes are display-only and do not affect behaviour.
- **FR-TCW-024:** Resources with `apply_scope = RECURRING_SLOT` shall appear in detail for every occurrence of the slot; resources with `apply_scope = OCCURRENCE` shall appear only on that specific `class_date`.
- **FR-TCW-025:** When an occurrence has `effective_status = CANCELLED`, recurring-slot resources still appear on that date's detail (read-only for the teacher; not surfaced to students since the class did not happen).
- **FR-TCW-026:** Teacher shall not be able to modify another teacher's resource. The Admin override path is **delete-only**: Admin can delete any teacher's resource via FR-TCW-026a; Admin cannot edit a teacher's resource content.
- **FR-TCW-026a:** Admin shall be able to delete a `ClassInstruction` or `ClassResource` authored by any teacher via an Admin endpoint; the action writes an `audit_log` row including `actor_id` (Admin), `original_author_id`, and `reason` (required).

### 5.5 Prerecorded Clip

- **FR-TCW-027:** Teacher shall be able to attach a prerecorded clip directly to a class via `POST /teacher/classes/{schedule_entry_id}/prerecorded-clips`. The clip is stored as a `ClassResource` row with `resource_type = PRERECORDED_CLIP`.
- **FR-TCW-028:** Prerecorded clips are class-linked, **not** library-owned. They do not appear in any reusable content list; deleting the class (theoretical — see Edge Cases) would orphan the clip's metadata even if the underlying file persists.
- **FR-TCW-029:** Teacher shall control student access via `visibility_status`. `DRAFT` is invisible to students; `PUBLISHED` is visible; `HIDDEN` was once published but is now retracted (still visible in teacher history).
- **FR-TCW-030:** System shall render prerecorded clips distinctly from generic resources in the class detail (FR-TCW-011).

### 5.6 Live Class Shortcut

- **FR-TCW-031:** Detail shall expose a "Start session" / "Join session" / "View recording" action for `delivery_mode ∈ { LIVE_ONLINE, HYBRID }`. The button text and target reflect the session state from Live Class Management.
- **FR-TCW-032:** Live actions shall **redirect** to Live Class Management endpoints (FR-LCM-*); this module does not run sessions.
- **FR-TCW-033:** When the occurrence is `CANCELLED`, live actions are hidden.

### 5.7 Attendance Shortcut

- **FR-TCW-034:** Detail shall expose an "Open attendance" action that deep-links to Attendance Management with the `(batch_id, schedule_entry_id, class_date)` context preloaded.
- **FR-TCW-035:** When the occurrence is `CANCELLED`, the attendance shortcut is hidden (no class, no roll).

### 5.8 Linked Assessments

- **FR-TCW-036:** Detail shall display a panel listing all assessments linked to the `(schedule_entry_id, class_date)` pair, plus assessments linked to the recurring slot (no date).
- **FR-TCW-037:** Each linked assessment row shall include `assessment_id, title, type, status (DRAFT|PUBLISHED|CLOSED), submission_count, pending_evaluation_count, result_status (NOT_PUBLISHED|PUBLISHED)`.
- **FR-TCW-038:** Teacher shall be able to click "Create assessment" from the panel; this redirects to Assessment Management with prefilled context: `batch_id`, `subject_id`, `linked_schedule_entry_id`, `class_date`.
- **FR-TCW-039:** Teacher shall be able to open any linked assessment, redirecting to Assessment Management with the assessment's edit/manage URL.
- **FR-TCW-040:** Linked assessment data is sourced from Assessment Management; this module caches the summary row only as a read-through view and never writes assessment state.

### 5.9 Filtering, Search, Pagination

- **FR-TCW-041:** Teacher shall be able to filter the class list by one or more `subject_id`, `batch_id`, `delivery_mode`.
- **FR-TCW-042:** Teacher shall be able to free-text search by batch name or subject name.
- **FR-TCW-043:** System shall support date-range navigation for `Previous` and `Upcoming` groups: `date_from` and `date_to` query params; default windows in FR-TCW-002 apply when omitted.
- **FR-TCW-044:** List responses shall be paginated with default page size 30 and max page size 100; cursor-based for stable ordering across schedule changes.

## 6. Business Rules

- The workspace consumes the published effective schedule only. Schedule changes are admin-owned; no flow in this module mutates `ScheduleEntry`, `ScheduleOverride`, or `BatchRoutine`.
- A teacher can only manage their own assigned classes. Cross-teacher access is read-only (linked-assessment context may show another teacher's name, e.g. co-teaching, but not their workspace).
- Resources and instructions are class-linked, not reusable. The same file may legitimately appear in two classes via two `ClassResource` rows pointing at the same `file_asset_id` (the file asset is shared; the class linkage is not).
- Cancelled classes remain visible in the workspace history with their `effective_status` badge; their student-visible resources are not surfaced to students for that date.
- A class occurrence may have zero, one, or many linked assessments.
- Linked assessments and recordings persist independently of schedule changes; a routine replacement (Scheduling FR-SCH-005) does not delete past linked assessments.
- Teacher private notes are never published to students under any circumstance; the API never returns them on the student-facing endpoint.
- Live, attendance, and assessment shortcuts are pure redirects — this module sees the user click and routes them; it never proxies the dependent module's writes.
- Admin has a **delete-only override** path for teacher resources and instructions (FR-TCW-026a); Admin cannot edit a teacher's content. Admin reads via the dedicated admin endpoint and writes nothing except the audit row on delete.
- A `RECURRING_SLOT`-scoped resource attached to a slot inherits visibility across all occurrences; a teacher who wants per-occurrence variation must use `OCCURRENCE` scope.

## 7. User Flow / Process Flow

### 7.1 Teacher Daily Flow

1. Teacher logs in and lands on the Teacher Class Workspace (default home).
2. System fetches the calling teacher's `Today` group first (lazy-loads `Upcoming` and `Previous` on tab click).
3. Teacher reviews the Today list and opens the next class.
4. System loads detail: scheduling fields, instructions, resources, prerecorded clip (if any), live/attendance/assessment shortcuts.
5. Teacher takes one or more actions:
   - Add instruction (occurrence or recurring scope).
   - Upload / link a resource.
   - Attach a prerecorded clip.
   - Click Start Live Class (redirect to Live Class).
   - Click Open Attendance (redirect to Attendance).
   - Click Create Assessment (redirect to Assessment).
6. Teacher returns to the list when done.

### 7.2 Attach a Resource

1. Teacher opens class detail.
2. Clicks **Add resource**.
3. Selects type: File / Link / Prerecorded clip.
4. For File: uploads via the shared File Asset module (presigned URL flow), receives `file_asset_id`, submits to this module.
5. For Link / Clip URL: pastes URL.
6. Provides title, optional description, picks visibility (`DRAFT` / `PUBLISHED`), picks scope (`OCCURRENCE` / `RECURRING_SLOT`).
7. Saves; system validates (FR-TCW-019), creates the row, returns the resource object.
8. Resource appears in the detail with the chosen order (default: appended).

### 7.3 Edit Visibility of an Existing Resource

1. Teacher opens detail.
2. Clicks the visibility toggle on a resource row.
3. System updates `visibility_status` and emits a `RESOURCE_VISIBILITY_CHANGED` event (see notifications in OQ).
4. Student Workspace sees the change on next fetch.

### 7.4 Create Assessment from Class

1. Teacher opens detail, clicks **Create assessment**.
2. This module fetches `GET /teacher/classes/{schedule_entry_id}/assessment-context` (returns batch_id, subject_id, linked_schedule_entry_id, optional class_date).
3. UI navigates to Assessment Management's create flow with context preloaded.
4. Teacher completes assessment authoring in Assessment Management (out of scope here).
5. On save, Assessment Management records the link (writes to its own table, with `schedule_entry_id` and optional `class_date`).
6. Teacher returns to the class detail; assessment now appears in the linked panel.

### 7.5 Previous Class Review

1. Teacher selects **Previous** group.
2. System lists past occurrences (default 90 days).
3. Teacher opens one; detail is read-only for instructions/resources (per FR-TCW-012).
4. Teacher views attendance summary (redirect), linked recording (from Live Class), linked assessments and their results status.

## 8. Data Model

This module owns `ClassInstruction` and `ClassResource`. The other entities in the original SRS (`TeacherAssignedClassView`, `ClassAssessmentLinkView`) are **read-through projections** sourced from Scheduling and Assessment respectively — they are not stored here.

### `ClassInstruction`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| schedule_entry_id | UUID, FK → ScheduleEntry.id | The recurring class slot |
| class_date | Date, nullable | Required when `apply_scope = OCCURRENCE`; null when `RECURRING_SLOT` |
| apply_scope | enum(`OCCURRENCE`, `RECURRING_SLOT`) | |
| instruction_type | enum(`STUDENT_VISIBLE`, `TEACHER_PRIVATE`) | |
| content | text | Body (Bangla allowed); ≤ 2,000 chars |
| created_by | UUID, FK → User.id | Author teacher |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| deleted_at | Timestamp, nullable | Soft delete; visible to Admin for audit |

### `ClassResource`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| schedule_entry_id | UUID, FK → ScheduleEntry.id | |
| class_date | Date, nullable | Required when `apply_scope = OCCURRENCE` |
| apply_scope | enum(`OCCURRENCE`, `RECURRING_SLOT`) | |
| resource_type | enum(`FILE`, `LINK`, `PRERECORDED_CLIP`) | |
| title | string | ≤ 200 chars; required; Bangla allowed |
| description | text, nullable | ≤ 1,000 chars |
| subtype | string, nullable | Display-only label, e.g. `worksheet`, `revision` |
| file_asset_id | UUID, nullable | FK → File Asset module; required when type = FILE; allowed for PRERECORDED_CLIP |
| external_url | string, nullable | ≤ 2,048 chars; required when type = LINK; allowed for PRERECORDED_CLIP |
| visibility_status | enum(`DRAFT`, `PUBLISHED`, `HIDDEN`) | Default `DRAFT` |
| display_order | integer | Default 0; **unique within `(schedule_entry_id, class_date, apply_scope)`** via a partial unique index. Reorder API rewrites the set atomically to avoid duplicates |
| created_by | UUID, FK → User.id | Author teacher |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| deleted_at | Timestamp, nullable | Soft delete |

### Read-through views (not tables)

- `TeacherAssignedClassView` — composed by calling Scheduling `GET /me/schedule` and merging with the calling teacher's own filters. Lives entirely in this module's controller / resolver layer.
- `ClassAssessmentLinkView` — composed by calling Assessment Management's per-class assessment-link endpoint (`GET /assessments/by-class?schedule_entry_id=&class_date=`). Read-only.

> **Multi-tenancy:** Per ADR-0003 (accepted, shared-schema), `ClassInstruction` and `ClassResource` each carry a `tenant_id` column (UUID, NOT NULL, FK → `Tenant.id`). All ownership checks (`created_by = caller`) are implicitly tenant-scoped via the JWT claim. The Admin override path verifies the caller's `tenant_id` matches the target row's `tenant_id`; cross-tenant admin reach is rejected with `404`.

## 9. API Contracts

All endpoints require `Authorization: Bearer <teacher access_token>` unless noted. Endpoints prefixed `/admin/teacher-workspace/...` require Admin role.

### List Assigned Classes

```http
GET /teacher/classes?group=today&subject_id=&batch_id=&mode=&date_from=&date_to=&cursor=&limit=
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
      "day_of_week": 0,
      "start_time": "08:00",
      "end_time": "09:30",
      "delivery_mode": "ON_SITE",
      "room": { "id": "0192...", "name": "Room 201", "capacity": 40 },
      "live_session_ref": null,
      "effective_status": "NORMAL",
      "reason": null
    }
  ],
  "next_cursor": "..."
}
```

**Errors**

- `400` — `INVALID_INPUT` / `RANGE_TOO_LARGE` (> 92 days)
- `401` — missing or invalid token
- `403` — `FORBIDDEN` (caller role is not `TEACHER`)

Satisfies: FR-TCW-001 to FR-TCW-006, FR-TCW-041 to FR-TCW-044.

### Get Class Detail

```http
GET /teacher/classes/{schedule_entry_id}?class_date=2026-05-18
```

`class_date` query param is required for canonical addressing of an occurrence.

**Response (200)**

```json
{
  "occurrence": { ... same shape as list item ... },
  "instructions": {
    "student_visible": [{ "id": "0192...", "content": "Bring practice book", "apply_scope": "OCCURRENCE", "updated_at": "..." }],
    "teacher_private": [{ "id": "0192...", "content": "Cover problem set 4", "apply_scope": "OCCURRENCE", "updated_at": "..." }]
  },
  "resources": [
    {
      "id": "0192...",
      "title": "Worksheet Chapter 4",
      "resource_type": "FILE",
      "file_asset_id": "0192...",
      "external_url": null,
      "visibility_status": "PUBLISHED",
      "apply_scope": "OCCURRENCE",
      "display_order": 0,
      "subtype": "worksheet"
    }
  ],
  "prerecorded_clips": [ ... ClassResource rows with resource_type=PRERECORDED_CLIP ... ],
  "linked_assessments": [
    { "assessment_id": "0192...", "title": "Chapter 4 quiz", "type": "PRACTICE_QUIZ", "status": "PUBLISHED", "submission_count": 12, "pending_evaluation_count": 4, "result_status": "NOT_PUBLISHED" }
  ],
  "live_session": { "state": "NOT_STARTED", "join_url": null }
}
```

**Errors**

- `400` — missing `class_date`
- `403` — `NOT_ASSIGNED` (caller is not the teacher of this occurrence)
- `404` — `OCCURRENCE_NOT_FOUND`

Satisfies: FR-TCW-007 to FR-TCW-012, FR-TCW-031, FR-TCW-036, FR-TCW-037.

### Add Instruction

```http
POST /teacher/classes/{schedule_entry_id}/instructions
```

**Request**

```json
{
  "apply_scope": "OCCURRENCE",
  "class_date": "2026-05-18",
  "instruction_type": "STUDENT_VISIBLE",
  "content": "Bring your math notebook tomorrow."
}
```

**Response (201)** — full instruction object.

**Errors**

- `400` — `INVALID_INPUT` / `MISSING_CLASS_DATE` (when `apply_scope = OCCURRENCE`) / `EMPTY_CONTENT` (when type = STUDENT_VISIBLE)
- `403` — `NOT_ASSIGNED`
- `404` — `SCHEDULE_ENTRY_NOT_FOUND`

Satisfies: FR-TCW-013, FR-TCW-015.

### Update / Delete Instruction

```http
PATCH /teacher/instructions/{id}
DELETE /teacher/instructions/{id}
```

**Errors**

- `403` — `NOT_AUTHOR` (caller did not create this instruction)
- `404` — `INSTRUCTION_NOT_FOUND`

Satisfies: FR-TCW-014.

### Add Resource

```http
POST /teacher/classes/{schedule_entry_id}/resources
```

**Request**

```json
{
  "apply_scope": "OCCURRENCE",
  "class_date": "2026-05-18",
  "resource_type": "FILE",
  "title": "Worksheet Chapter 4",
  "description": null,
  "subtype": "worksheet",
  "file_asset_id": "0192...",
  "external_url": null,
  "visibility_status": "PUBLISHED",
  "display_order": 0
}
```

**Response (201)** — full resource object.

**Errors**

- `400` — `INVALID_INPUT` / `MISSING_SOURCE` (no file_asset_id or external_url per type) / `FILE_ASSET_NOT_COMMITTED` (asset is still PENDING or FAILED in File Asset module) / `FILE_ASSET_NOT_OWNED` (asset belongs to another user)
- `403` — `NOT_ASSIGNED` / `OCCURRENCE_PAST` (class_date is in the past)

Satisfies: FR-TCW-018, FR-TCW-018a, FR-TCW-019.

### Attach Prerecorded Clip

```http
POST /teacher/classes/{schedule_entry_id}/prerecorded-clips
```

Same shape as Add Resource but `resource_type` is fixed to `PRERECORDED_CLIP` and at least one of `file_asset_id` / `external_url` is required.

Satisfies: FR-TCW-027 to FR-TCW-030.

### Update / Delete Resource

```http
PATCH /teacher/class-resources/{id}
DELETE /teacher/class-resources/{id}
```

**PATCH Request** (any subset; source fields immutable)

```json
{ "title": "Worksheet Ch 4 (rev 2)", "visibility_status": "HIDDEN", "display_order": 2 }
```

**Errors**

- `400` — `IMMUTABLE_FIELD` (attempt to modify `file_asset_id` / `external_url`)
- `403` — `NOT_AUTHOR`
- `404` — `RESOURCE_NOT_FOUND`

Satisfies: FR-TCW-020, FR-TCW-021, FR-TCW-022.

### Reorder Resources (bulk)

```http
PUT /teacher/classes/{schedule_entry_id}/resources/order
```

**Request**

```json
{
  "class_date": "2026-05-18",
  "orders": [
    { "id": "0192...", "display_order": 0 },
    { "id": "0192...", "display_order": 1 }
  ]
}
```

**Response (200)** — list of updated resources.

**Errors**

- `403` — `NOT_AUTHOR` (if any `id` in `orders` has a different `created_by` than the caller; the entire request is rejected before any row is updated)
- `404` — `RESOURCE_NOT_FOUND` (any id missing)

The endpoint applies all updates within a single transaction; partial reorder is impossible.

Satisfies: FR-TCW-022.

### Get Assessment Context (prefill)

```http
GET /teacher/classes/{schedule_entry_id}/assessment-context?class_date=2026-05-18
```

**Response (200)**

```json
{
  "batch_id": "0192...",
  "subject_id": "0192...",
  "linked_schedule_entry_id": "0192...",
  "class_date": "2026-05-18"
}
```

Satisfies: FR-TCW-038.

### List Linked Assessments

```http
GET /teacher/classes/{schedule_entry_id}/assessments?class_date=2026-05-18
```

**Response (200)** — `linked_assessments` array, same shape as inside the Class Detail response.

Satisfies: FR-TCW-036, FR-TCW-037.

### List Instructions for a Class (server-callable)

```http
GET /teacher/classes/{schedule_entry_id}/instructions?class_date=&student_visible_only=true
```

**Headers**

```
Authorization: Bearer <teacher access_token>   OR
X-Service-Token: <internal service token>
```

Callable by: (a) the teacher of the class (returns both STUDENT_VISIBLE and TEACHER_PRIVATE rows, ignoring `student_visible_only`); (b) Admin (same as teacher); (c) the Student Class Workspace resolver via internal service token (must pass `student_visible_only=true`; returns STUDENT_VISIBLE rows only; TEACHER_PRIVATE rows are filtered at the persistence layer regardless of parameter).

**Response (200)**

```json
{
  "instructions": [
    { "id": "0192...", "content": "Bring practice book", "apply_scope": "OCCURRENCE", "instruction_type": "STUDENT_VISIBLE", "updated_at": "..." }
  ]
}
```

**Errors**

- `400` — missing `class_date`
- `403` — `NOT_AUTHORIZED` (caller is neither the assigned teacher nor Admin nor a recognised service token)
- `404` — `SCHEDULE_ENTRY_NOT_FOUND`

Satisfies: FR-TCW-016, FR-TCW-018 (read side); supports Student Workspace's read-through projection.

### List Resources for a Class (server-callable)

```http
GET /teacher/classes/{schedule_entry_id}/resources?class_date=&visibility_status=PUBLISHED
```

Same caller matrix as instructions. The `visibility_status` filter accepts `PUBLISHED` (student-facing usage) or any of `DRAFT|PUBLISHED|HIDDEN` for teacher/admin callers (default: all). Service-token callers are restricted to `PUBLISHED` at the API guard layer.

**Response (200)**

```json
{
  "resources": [
    {
      "id": "0192...",
      "title": "Worksheet Chapter 4",
      "resource_type": "FILE",
      "file_asset_id": "0192...",
      "external_url": null,
      "visibility_status": "PUBLISHED",
      "apply_scope": "OCCURRENCE",
      "display_order": 0,
      "subtype": "worksheet"
    }
  ]
}
```

Satisfies: FR-TCW-018 to FR-TCW-024 (read side); supports Student Workspace's read-through projection.

### Admin — Read Any Teacher's Class Detail (incl. private notes)

```http
GET /admin/teacher-workspace/classes/{schedule_entry_id}/detail?teacher_id=&class_date=
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

Returns the same shape as the teacher detail endpoint, scoped to the named teacher's data, including `TEACHER_PRIVATE` instructions. Writes an `audit_log` row capturing the admin read (`actor_id`, `target_teacher_id`, `schedule_entry_id`, `class_date`) so teachers have a transparent trail.

**Errors**

- `403` — `FORBIDDEN` (caller is not Admin)
- `404` — `OCCURRENCE_NOT_FOUND` / `TEACHER_NOT_FOUND`

Satisfies: FR-TCW-017.

### Admin — Delete Any Resource / Instruction

```http
DELETE /admin/teacher-workspace/resources/{id}
DELETE /admin/teacher-workspace/instructions/{id}
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

**Request**

```json
{ "reason": "Inappropriate content reported by parent" }
```

**Response (204)** — empty body. Writes audit row with `actor_id`, `original_author_id`, `reason`.

**Errors**

- `400` — `MISSING_REASON`
- `403` — `FORBIDDEN` (not Admin)

Satisfies: FR-TCW-017 (read), FR-TCW-026a.

## 10. UI Components

### Screen: Teacher Class List (Workspace home)

- **Components:** Tab group (Today / Upcoming / Previous), filter chips (subject, batch, mode), search input, date range picker (for Previous / Upcoming), class card list. Each card shows: subject + batch (primary), time (large), mode chip, status badge (NORMAL / UPDATED / CANCELLED), quick-action row (Open, Start live, Attendance, Resources).
- **Actions:** Open detail, jump to live, jump to attendance, quick-add resource.
- **States:** loading, empty (no assignments), populated, filtered-empty, error.

### Screen: Class Detail

- **Components:** Header strip (subject + batch + time + room + mode + status badge); Instructions panel (split into Student-visible / Private with separate add buttons); Resources panel (sortable list, each with visibility toggle + edit/delete); Prerecorded clip panel (distinct from generic resources); Live action panel (button reflects session state); Attendance shortcut button; Linked assessment panel (read-only list + Create assessment button).
- **Actions:** Add instruction, add resource, attach clip, toggle visibility, reorder (drag), start live, open attendance, create assessment, open assessment.
- **States:** loading, populated, read-only (past class), cancelled (most actions hidden), live-in-progress (Start → Join).

### Modal: Add Resource

- **Components:** Type segmented control (File / Link / Clip); title, description, subtype dropdown, visibility radio (Draft / Published), scope radio (Occurrence / Recurring slot), source field (file uploader OR URL input).
- **Actions:** Upload (delegates to File Asset module's presigned flow), Save, Cancel.
- **States:** idle, uploading (with progress), validating, saving, error (per-field).

### Modal: Add Instruction

- **Components:** Type radio (Student-visible / Private), scope radio, content textarea (rich-text-light: bold, list, link), save.
- **Actions:** Save, Cancel.
- **States:** idle, saving, error.

### Drawer: Linked Assessments

- **Components:** List of linked assessment summary rows (title, type, status, submission count, pending eval, result status). Each row opens the assessment in Assessment Management (redirect).
- **Actions:** Open assessment, Create new (with prefilled context).
- **States:** loading, empty, populated.

## 11. Validation Rules

- **Class membership:** Teacher may only call class-scoped endpoints for `schedule_entry_id` rows where they are the resolved teacher; otherwise `403 NOT_ASSIGNED`.
- **`apply_scope = OCCURRENCE`** requires `class_date` non-null and matching the `ScheduleEntry.day_of_week`. Validation cross-calls Scheduling's `GET /occurrences/{schedule_entry_id}/{class_date}` and rejects with `DATE_DAY_MISMATCH` if no occurrence exists for the pair.
- **`apply_scope = RECURRING_SLOT`** requires `class_date` null.
- **Past-occurrence write block:** writes to instructions and resources are rejected when `class_date < today` (BST) with `OCCURRENCE_PAST`. The teacher detail GET still returns the historical data.
- **Content storage format:** `ClassInstruction.content` is stored as **Markdown** (CommonMark subset: bold, italic, lists, links). The character limit (≤ 2,000) applies to the raw stored Markdown, not rendered HTML. The frontend renders Markdown safely (no HTML pass-through).
- **Bulk reorder ownership:** all `id` values in a reorder request must have `created_by = caller.user_id`; if any do not, the entire request is rejected with `403 NOT_AUTHOR`.
- **Instruction content:** `STUDENT_VISIBLE` requires non-empty after trim; max 2,000 chars. Bangla and English both supported.
- **Resource title:** required, ≤ 200 chars.
- **Resource type → source field:** FILE requires `file_asset_id`; LINK requires `external_url`; PRERECORDED_CLIP requires one of the two.
- **External URL:** must parse as a valid `http(s)` URL; ≤ 2,048 chars; no other validity check at write time (the link may break later; see edge case).
- **File asset:** `file_asset_id` must exist and the caller must have permission to attach it (delegated check to File Asset module).
- **Visibility status:** one of `DRAFT`, `PUBLISHED`, `HIDDEN`.
- **Display order:** integer ≥ 0.
- **Authorship:** edit and delete endpoints check `created_by = caller.user_id`; Admin override is a separate endpoint (FR-TCW-026a).
- **Pagination cursor:** opaque; server validates issuance.
- **Date range:** for list queries, range ≤ 92 days (same convention as Scheduling).
- **Admin reason** on delete: required, ≤ 500 chars.

## 12. Edge Cases

- Schedule overridden after teacher attached an occurrence-scoped resource → resource remains attached to the original `class_date`; if the override changed time / teacher, the resource still appears in the detail since it is keyed by `(schedule_entry_id, class_date)`, not by the override.
- Schedule occurrence cancelled after a `PUBLISHED` student-visible resource was attached → the detail still shows the resource to the teacher (with the cancelled badge); the Student Workspace **does not** surface the resource for that date because the class did not happen. Recurring-slot resources are unaffected on future occurrences.
- Live session ref missing 15 min before class → detail shows live state from Live Class Management ("session not configured"); the teacher can either contact Admin or use the schedule-edit redirect (Admin-only).
- Teacher attaches the same file (same `file_asset_id`) twice to the same class → allowed (two `ClassResource` rows with the same source); UI surfaces a "duplicate file" hint but does not block, since a teacher may have intentional reasons.
- External URL becomes a 404 after publish → not detected at the API layer; future enhancement is a periodic link-health check (see OQ).
- Recurring-slot resource is unsuitable for one override occurrence (e.g. the override swapped subject) → teacher cannot exclude a recurring resource from a single occurrence in v1; they must convert it to occurrence-scope on each remaining date, or delete the recurring one and re-add per occurrence. OQ asks whether per-occurrence opt-out is needed.
- Teacher reassigned at the `BatchSubjectTeacher` layer (Class Context FR-CBS-026) after resources were created → the original author's resources stay on the slot. The new teacher cannot edit them (`NOT_AUTHOR`), but they can add their own resources. Admin may delete the old teacher's resources via override if needed.
- Previous class read-only window (FR-TCW-012) — exact cutoff is "occurrence date < today, server clock"; an occurrence in the same calendar date but past `end_time` is still considered Today (configurable in OQ).
- Class has multiple linked assessments → all surface in the panel; the linked-assessments call paginates with default page size 20 if more than that exist (rare).
- Linked assessment exists for a schedule entry; the routine is replaced (Scheduling FR-SCH-005) → the assessment's `schedule_entry_id` continues to point at the now-archived entry. Assessment Management owns its own snapshot; this module surfaces the assessment under the archived occurrence's history.
- Teacher reassigned after a linked assessment was created → the assessment row still references the original `linked_teacher_id` (snapshot); the new teacher sees the link in the panel but cannot edit the assessment unless Assessment Management also adopts the reassignment rule (see Assessment SRS OQ).
- Assessment published but class later cancelled → the linked panel still shows the assessment with status `PUBLISHED`; the class card carries the cancelled badge. Students may still submit per Assessment Management rules.
- Result published after class moved to history (Previous group) → panel still updates the `result_status` (read-through), even though the class itself is read-only.
- Resource deleted while a student is mid-download → File Asset module's S3-compatible storage permits in-flight downloads to complete; subsequent fetches return 404.
- `OCCURRENCE`-scoped resource attached to a date that is later affected by a `FUTURE`-scope routine replacement (Scheduling FR-SCH-018b) → the resource's `schedule_entry_id` points at the now-archived entry. The class detail for that date renders from the **archived** routine (Scheduling returns the historical entry), so the resource is still surfaced on its original occurrence. The new routine's entry for the same `day_of_week` does not inherit it; teacher must re-create on the new entry if desired.
- `RECURRING_SLOT`-scoped resource where the underlying `ScheduleEntry` is on an archived routine → resource remains queryable in historical views; not surfaced on the new routine's entries. The new teacher must attach equivalents to the new entry.
- Teacher submits `POST .../resources` with a `file_asset_id` belonging to another user → rejected with `FILE_ASSET_NOT_OWNED` (FR-TCW-018a); no `ClassResource` row is created.
- `PRERECORDED_CLIP` attached as `external_url` to a third-party host (YouTube, Drive) vs as `file_asset_id` to institutional R2 storage → both supported; institutional storage gives Bangladesh students lower data usage and access control via signed URLs, while third-party links cost more data and have no institutional ACL. The teacher UI flags this trade-off as an informational hint at attach time.
- Two devices submit identical instruction `POST` from the same teacher in quick succession → both writes succeed (no uniqueness constraint on content); UI client-side deduplicates by checking the recent list before submit. Backend does not enforce uniqueness; teachers can delete duplicates manually.
- Cancelled occurrence and student-visibility enforcement → this module's detail API returns `student_accessible: false` on every resource and instruction when the occurrence's `effective_status = CANCELLED`. Student Workspace honours this flag. Recurring-slot resources still show in the teacher's view for context but are not student-accessible for that specific date.
- Two concurrent PATCH calls on the same resource (e.g. visibility toggle from two devices) → last-write-wins on the row; both responses succeed; client must re-fetch.
- A teacher's `is_active` flips to false → Auth's session-revocation flow (auth.md FR-AUTH-035) invalidates their tokens; existing workspace data remains in the database under their `created_by`. Admin can read or delete their resources via override.
- Admin deletes a teacher's resource → audit row written; the resource is soft-deleted (`deleted_at` set); subsequent reads omit it.

## 13. Dependencies

- **Class Scheduling (`scheduling.md`)** — provides the source-of-truth schedule and overrides. This module calls `GET /me/schedule` and `GET /occurrences/{schedule_entry_id}/{date}`. **Inter-module events consumed:** Scheduling's `BATCH_SUBJECT_TEACHER_REASSIGNED` (drops the affected slot's future occurrences from the list); `SCHEDULE_OVERRIDE_CREATED` and `ROUTINE_REPLACED` (refresh client cache on next fetch — no live invalidation needed because reads are always live).
- **Class Context (`class-context.md`)** — indirect; Scheduling consults Class Context for subject/teacher truth, this module trusts Scheduling's resolved view.
- **Live Class Management** — provides session state for live-mode detail (`live_session.state`, `join_url`); the live action button is a redirect.
- **Attendance Management** — provides the deep-link for the attendance shortcut.
- **Assessment Management** — provides the linked-assessments summary and the create/edit redirect targets. Read-only from this module.
- **File Asset module** (shared, schema location TBD) — owns physical file storage (S3-compatible per ADR-0002: Cloudflare R2). This module stores `file_asset_id` references only. Upload flow is presigned-URL via the File Asset module.
- **Notification Management** — may consume `RESOURCE_PUBLISHED` events to alert students of new study material (see OQ).
- **Authentication & Authorization** (`auth.md`) — RBAC; teacher role enforced on all `/teacher/*` endpoints.
- **Central `audit_log`** — schema location TBD; this module writes rows on Admin overrides (FR-TCW-026a) and on resource deletes.

## 14. Non-Functional Requirements

- **Performance:** class list (Today group) p95 < 1.5 s; class detail p95 < 1 s; resource upload completion driven by file size and File Asset module SLA (not directly controlled here).
- **Mobile:** all teacher screens responsive down to 360 px width; touch targets ≥ 44 px; image and video resource previews lazy-load. Teachers on low-end Android are a primary audience.
- **Weak-network tolerance:** uploads use chunked or resumable transfer where the File Asset module supports it; partial writes do not create orphan `ClassResource` rows (resource is created only after the file asset is committed).
- **Internationalization:** all UI strings translatable EN ↔ BN; instruction and resource title fields accept Bangla input; the rich-text-light editor renders Bangla correctly.
- **Accessibility:** WCAG 2.1 AA on all screens; keyboard navigation for the reorder action.
- **Cache:** the class-list endpoint is cacheable for 30 s on the client (private cache); the detail endpoint is not cached (fresh fetch each open).
- **Auditability:** every Admin override (delete instruction / delete resource) writes to `audit_log` with `actor_id`, `original_author_id`, `reason`. Every soft-delete (any actor) writes `deleted_at` on the row.
- **Storage:** instructions and resources are class-scoped and small (text, references); per-class storage cap not enforced in v1. Per-teacher resource count soft cap: 500 per class slot (warning at 80%); see OQ.

## 15. Assumptions

- Teachers do not edit the official routine; all schedule mutations go through Admin → Scheduling.
- Prerecorded clips are class-linked, not part of a reusable library.
- Many teachers use mobile devices; design and NFRs assume mobile-first.
- A class may host both live delivery and support resources concurrently (e.g. HYBRID).
- Files uploaded here are visible only to the class roster + the author teacher + Admin; the File Asset module enforces this via signed URLs scoped to those identities.
- A teacher's identity is `User.role = TEACHER` and the linkage from teacher to classes flows through `BatchSubjectTeacher` (Class Context) → `ScheduleEntry.teacher_id` (Scheduling).
- "Today" is interpreted in Bangladesh Standard Time (UTC+6) on the server; the API computes the date boundary, the client respects it.

## 16. Open Questions

- **Teacher private notes — Admin visibility:** can Admin read a teacher's `TEACHER_PRIVATE` notes for oversight (FR-TCW-017)? Some institutions require this for review; teachers may prefer privacy. Default v1 = Admin can read but not edit; confirm.
- **Mark class as completed:** original SRS asked whether teachers should mark a class completed independent of attendance. v1 default = no (presence of attendance records implies completion); confirm.
- **Conditional access to prerecorded clip:** today the gate is `visibility_status = PUBLISHED`. Some institutions want access tied to attendance (only students who attended) or to assessment completion. Confirm not in v1.
- **Read-only cutoff for past classes:** is the cutoff "class_date < today" or "end_time < now()" (same calendar day, after class ends)? Default v1 = "class_date < today" (simpler).
- **Per-occurrence opt-out of recurring-slot resources:** v1 forces conversion to occurrence-scope; should there be a "hide this recurring resource for this date" mechanism without changing scope?
- **Notification on `RESOURCE_PUBLISHED`:** should publishing a resource send an SMS / in-app push to enrolled students? Default v1 = no auto-notification (avoid spam); teachers can post a STUDENT_VISIBLE instruction separately.
- **External URL link-health check:** periodic background job to ping URLs and mark broken links visually for the teacher; needed in v1?
- **Per-class resource cap:** soft cap at 500 — too high / too low for the Bangladesh coaching context?
- **Bulk operations:** mass-delete, mass-publish, mass-copy-to-batch — needed in v1 or future?
- **Audit-log schema cross-reference:** consistent with Auth and Scheduling OQs — confirm location and field list once the shared `audit_log` doc is created.
- **Per-class resource cap enforcement:** the 500-resource soft cap in §14 — should the API return an `X-Soft-Limit-Approaching` response header at 80%, or is this purely a client-side warning? v1 default: client-side only; no server enforcement.
