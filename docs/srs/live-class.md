# 🧩 MODULE: LIVE CLASS MANAGEMENT

## 1. Purpose

Manage the runtime lifecycle of a live online class session bound to a scheduled occurrence: provider session creation or reference binding, secure host (teacher) and participant (student) access, lifecycle state (`UPCOMING` → `READY` → `LIVE` → `ENDED`, with `CANCELLED` short-circuit), join/leave event capture for downstream Attendance, and recording reference linkage when the provider supports it. Live Class does **not** own class listing, schedule authoring, content, or attendance policy — it produces signals the Workspaces and Attendance modules consume.

## 2. Scope

**In Scope**

- One `LiveSession` per scheduled live or hybrid occurrence (`(schedule_entry_id, class_date)`), created on demand
- Provider abstraction: an `External Meeting Provider` integration boundary (Zoom, Google Meet, Jitsi, Whereby; v1 vendor TBD — see OQ)
- Host (teacher) start flow, participant (student) join flow, end flow
- Time-gated join eligibility — server-side join window enforcement
- Secure access — short-lived join tokens minted per `(session, user)` request
- Lifecycle state machine: `UPCOMING` → `READY` → `LIVE` → `ENDED`, with `CANCELLED` from `UPCOMING`/`READY`
- `LiveParticipantEvent` capture: join and leave with timestamps and computed duration
- Recording reference ingestion via webhook + a fallback poll; status tracking
- Admin monitoring view (active sessions, counts)
- Graceful degradation when provider is unavailable

**Out of Scope**

- Class list and assigned-class views — Teacher / Student Workspaces
- Schedule authoring or overrides — Scheduling
- Attendance policy and final-grade computation — Attendance Management
- Class instructions and resources — Teacher Class Workspace
- Assessment workflows — Assessment Management
- Recording publication to a reusable library — outside CC-LMS scope (recording reference belongs to the occurrence)
- Cross-batch broadcasting / multi-batch concurrent sessions

## 3. Actors

- **Teacher** — host of a session for their own assigned class occurrence. Starts and ends the session.
- **Student** — participant; joins through the Student Class Workspace's redirect. Cannot host.
- **Admin** — observes active sessions (monitoring), may end a stuck session, may cancel a session in `READY` state.
- **System** — minting join tokens, enforcing the join window, transitioning state, consuming the provider's webhooks (started / ended / recording-ready), emitting `LIVE_SESSION_*` events.
- **External Meeting Provider** — third-party (Zoom or equivalent). Owns the actual A/V transport. Returns a session reference, host/join URLs or tokens, webhook callbacks for state, and an optional recording reference.

## 4. Core Concepts

### Live Session

A row in `LiveSession` bound to one occurrence `(schedule_entry_id, class_date)`. Created on demand the first time the teacher visits the live-class action for the occurrence, or eagerly minted at `start_time − 15 min` by a scheduled job (see OQ — eager vs. lazy creation). Identified by `id` (UUID); also addressable by `(schedule_entry_id, class_date)` since uniqueness is enforced.

### Session Lifecycle

State machine, enforced server-side:

- `UPCOMING` — created, before `start_time − 10 min`. No join allowed.
- `READY` — within the join window (`start_time − 10 min ≤ now`), provider session is provisioned, but the teacher has not started yet. Students may join (provider's "waiting room" if supported, else queue at the workspace).
- `LIVE` — teacher has started; session is in progress.
- `ENDED` — teacher (or system, via webhook) has ended the session; or `end_time + 30 min` has elapsed with no activity and the system auto-closes.
- `CANCELLED` — Admin or the routine cancellation (Scheduling `ScheduleOverride` with `status = CANCELLED`) ended the session before `LIVE`. Cannot be undone; a new occurrence requires a new session.

### Join Window

Default `[start_time − 10 min, end_time + 30 min]` BST. The join window is the time interval during which `READY` and `LIVE` sessions accept join requests. Outside the window the session returns `JOIN_WINDOW_CLOSED`. Configurable per institution (see OQ).

### Join Token

A short-lived (default 5 min) opaque token minted per `(session, user)` join request, encoding the user's role (HOST / PARTICIPANT) and the session ID. The client uses it to authenticate to the provider's join endpoint. Tokens are single-use to prevent re-entry replay across users.

### Participant Event

A row in `LiveParticipantEvent` capturing each user's join and leave on a session. Multiple join/leave pairs may exist for the same user (network drops, deliberate re-entry). Used by Attendance Management to compute presence.

### Recording Reference

A row in `LiveRecordingReference` capturing the provider's recording handle for a session. Status: `PROCESSING` (provider returned a ref, transcoding not complete) → `AVAILABLE` (playback URL ready) → `FAILED` (provider could not produce a recording). Stored *after* the session ends (provider webhook or scheduled poll).

### Provider Abstraction

A backend interface with concrete adapters per provider. The SRS treats `provider` and `provider_session_ref` as opaque; only the adapter knows the provider-specific payload. v1 ships with **one** adapter (vendor TBD per OQ); the abstraction allows swapping later without schema change.

## 5. Functional Requirements

### 5.1 Session Creation and Binding

- **FR-LCM-001:** System shall create at most one `LiveSession` per `(schedule_entry_id, class_date)` pair where the underlying occurrence has `delivery_mode ∈ { LIVE_ONLINE, HYBRID }`. Uniqueness enforced by a partial unique index.
- **FR-LCM-002:** System shall reject creation for occurrences whose `delivery_mode ∈ { ON_SITE, RECORDED_SUPPORT }` with `INVALID_DELIVERY_MODE`.
- **FR-LCM-003:** System shall set `host_teacher_id` to the occurrence's resolved teacher (post-override) at the time of creation. If a `ScheduleOverride` changing the teacher arrives **before** the `UPCOMING → READY` transition, the System shall re-resolve and update `host_teacher_id` at the moment of that transition. Overrides arriving **after** the session is in `READY` or `LIVE` do **not** mutate `host_teacher_id`; the original host completes the session, and the new teacher is reflected only for future occurrences.
- **FR-LCM-004:** System shall populate `provider_session_ref` by calling the configured provider adapter at creation time. Provider failures surface as `PROVIDER_UNAVAILABLE` to the caller; the session is not persisted.
- **FR-LCM-005:** System shall create sessions either lazily (on first host request) or eagerly (scheduled job at `start_time − 15 min`); the chosen strategy is a configuration value, default lazy. See OQ.

### 5.2 Session Lifecycle and State Transitions

- **FR-LCM-006:** System shall transition a session from `UPCOMING` → `READY` at `start_time − 10 min` (BST). Early-start is **not** supported in v1 (see OQ); attempts to start before the window are rejected per FR-LCM-013.
- **FR-LCM-007:** System shall transition `READY` → `LIVE` when (a) the teacher calls `POST /live-classes/{schedule_entry_id}/start` and the provider adapter's host-start call succeeds, or (b) the provider webhook reports a `session.started` event. Both paths are idempotent — whichever fires first wins; the second is a no-op against the same state.
- **FR-LCM-008:** System shall transition `LIVE` → `ENDED` when (a) the teacher calls `POST /live-classes/{schedule_entry_id}/end`, (b) the provider webhook reports a `session.ended` event, or (c) `end_time + 30 min` has elapsed and the system runs the auto-close job. On every transition to `ENDED`, all `LiveParticipantEvent` rows for the session with `left_at IS NULL` shall have `left_at = ended_at` and `duration_sec` populated.
- **FR-LCM-009:** System shall transition `UPCOMING` or `READY` → `CANCELLED` when the occurrence's `ScheduleOverride.status = CANCELLED` is created, or when Admin calls the cancel endpoint. Transitions from `LIVE` to `CANCELLED` are rejected; ended sessions cannot be cancelled.
- **FR-LCM-009a:** When a `SCHEDULE_OVERRIDE_CREATED` event with `change_kind = CANCELLED` arrives while the session is `LIVE`, System shall **silently discard** the cancellation for this module (the session continues to its natural or forced end). Scheduling's occurrence-level cancellation still applies in Scheduling's domain; the resulting brief inter-system disagreement is accepted.
- **FR-LCM-010:** System shall expose the current session state to teacher/student workspaces via `GET /live-classes/{schedule_entry_id}?class_date=…` and via the `live_session` field on the workspaces' detail responses.

### 5.3 Host (Teacher) Access

- **FR-LCM-011:** Only the user with `User.id = LiveSession.host_teacher_id` shall be permitted to start the session via `POST /live-classes/{schedule_entry_id}/start?class_date=…`. Any other caller receives `403 NOT_HOST`. Calls to `start` while the session is already `LIVE` return `200` with the existing state and a freshly minted host token (re-entry permitted by the host); calls while `ENDED` return `409 SESSION_ENDED`.
- **FR-LCM-012:** System shall mint a single-use host join token on each call to `POST /live-classes/{schedule_entry_id}/start`, with TTL 5 min, encoding `(session_id, user_id, role: HOST)`. The teacher's client uses the token to authenticate to the provider.
- **FR-LCM-013:** A teacher attempting to start a session before `start_time − 10 min` shall be rejected with `JOIN_WINDOW_NOT_OPEN`. The teacher may not start a session before its join window opens (see OQ for early-start policy).
- **FR-LCM-014:** A teacher whose `User.is_active = false` (per Auth's revocation event) shall not be able to start or rejoin; an active session ended by `is_active` flip auto-terminates.

### 5.4 Participant (Student) Access

- **FR-LCM-015:** Only students enrolled in the occurrence's batch at `class_date` (per `class-context.md` FR-CBS-029 + FR-SCW-024a batch-at-the-time rule) shall be permitted to join. Any other caller receives `403 NOT_IN_BATCH`.
- **FR-LCM-016:** System shall reject join requests outside the join window with `JOIN_WINDOW_CLOSED`.
- **FR-LCM-017:** System shall reject join for sessions in state `UPCOMING` (window not opened), `ENDED`, or `CANCELLED` with appropriate codes (`SESSION_NOT_READY`, `SESSION_ENDED`, `SESSION_CANCELLED`).
- **FR-LCM-018:** System shall mint a single-use participant join token on each successful `POST /live-classes/{id}/join`, with TTL 5 min, encoding `(session_id, user_id, role: PARTICIPANT)`. Each call mints a new token (re-joins permitted).
- **FR-LCM-019:** System shall record a `LiveParticipantEvent` row when the provider webhook (`participant.joined`) confirms the user has actually entered the provider session, **not** at the moment of token mint. `joined_at` is set from the webhook timestamp. The pre-webhook `POST /live-classes/{schedule_entry_id}/join` call mints a token only; the row is created on webhook receipt. This ensures token mints without actual provider entry (e.g. failed redirect) do not pollute the participation log.

### 5.5 Attendance Hooks

- **FR-LCM-020:** System shall persist participant join/leave timestamps to `LiveParticipantEvent`. Source of truth is the provider webhook (FR-LCM-019); orphan `participant.left` events with no matching `joined_at` row are logged in `WebhookDeliveryLog` with `outcome = rejected_invalid` and ignored — they typically indicate a lost join webhook and Attendance Management treats the gap as unknown.
- **FR-LCM-021:** System shall compute `duration_sec` as `(left_at - joined_at)` rounded down to seconds, populated on `left_at` write.
- **FR-LCM-022:** System shall emit a `LIVE_PARTICIPATION_RECORDED` event on every join and leave, with payload `{ event_type, session_id, schedule_entry_id, class_date, user_id, role, joined_at, left_at?, duration_sec? }`. Consumed by Attendance Management; this module does not compute attendance presence.
- **FR-LCM-023:** Live participation events shall not by themselves determine final attendance — Attendance Management applies institutional policy (minimum duration threshold, manual override, etc.). FR-LCM-022's payload provides the raw data only.

### 5.6 Recording Reference

- **FR-LCM-024:** If the provider supports recording and the institution has recording enabled in config, System shall ingest the recording reference via the provider's webhook (`recording.ready`) and store one `LiveRecordingReference` row per session.
- **FR-LCM-025:** A fallback poll job shall query the provider for the recording reference if the webhook has not been received within 24 hours of `ENDED`. After 7 days with no recording, status is set to `FAILED`.
- **FR-LCM-026:** Recording reference shall be addressable by `(schedule_entry_id, class_date)` via `GET /live-classes/{schedule_entry_id}/recording?class_date=…`.
- **FR-LCM-027:** Teacher and Admin shall be able to access the recording in any status. Students shall be able to access the recording only when `status = AVAILABLE`. The `playback_url` is returned as the provider's playback URL or a re-signed CDN URL (provider-adapter detail).
- **FR-LCM-028:** Recording playback URL access shall be restricted by the same batch-at-the-time rule as the student workspace's previous-class detail; students from other batches cannot retrieve the recording.

### 5.7 Cancellation and Admin Controls

- **FR-LCM-029:** Admin shall be able to cancel a session in `UPCOMING` or `READY` state via `POST /live-classes/{id}/cancel` with a required `reason`. Transitions the session to `CANCELLED` and writes the cancellation to `audit_log`.
- **FR-LCM-030:** Admin shall be able to forcibly end a `LIVE` session via `POST /live-classes/{schedule_entry_id}/force-end?class_date=…`, typically used when a host's network has dropped and the auto-close job has not yet fired. On force-end: state → `ENDED`; all open `LiveParticipantEvent` rows (`left_at IS NULL`) get `left_at = ended_at` and `duration_sec` computed; the provider is asked to end the meeting (best-effort; provider failure does not block the state change); two concurrent force-end calls on the same session — the second returns `200` with the already-ended state (no-op).
- **FR-LCM-031:** Admin shall be able to view active sessions across the institution via `GET /admin/live-sessions/active`, returning per-session summary `(id, schedule_entry_id, class_date, batch_name, subject_name, host_teacher_name, status, participant_count_live)`.

### 5.8 Provider Webhook Handling

- **FR-LCM-032:** System shall expose a webhook endpoint `POST /webhooks/live-class/{provider}` for the provider to post events (`session.started`, `session.ended`, `participant.joined`, `participant.left`, `recording.ready`).
- **FR-LCM-033:** Webhook signature verification shall be enforced per the provider's documented scheme; unverified payloads return `401`. Replay protection is enforced by the `(provider, provider_event_id)` unique constraint on `WebhookDeliveryLog` plus a 5-minute clock skew window on the `X-Provider-Timestamp` header — payloads with timestamps outside `[now - 5 min, now + 5 min]` are rejected with `401`.
- **FR-LCM-034:** Webhook handler shall be idempotent — re-deliveries of the same event id do not double-write `LiveParticipantEvent` rows or duplicate state transitions.
- **FR-LCM-035:** System shall log webhook receipt outcome (`processed`, `rejected_signature`, `rejected_duplicate`, `rejected_invalid`) for observability.

### 5.9 Monitoring and Resilience

- **FR-LCM-036:** Provider unavailability at create/start time returns `PROVIDER_UNAVAILABLE` to the caller; the teacher's UI surfaces "Live provider is currently unavailable; please retry in a minute or contact Admin."
- **FR-LCM-037:** A `LIVE` session whose `end_time + 30 min` has passed without the teacher ending it shall be auto-closed by a scheduled job; status moves to `ENDED`; all open `LiveParticipantEvent` rows have `left_at = ended_at` set and `duration_sec` populated; teacher and students are notified via Notification Management.
- **FR-LCM-038:** System shall persist essential session metadata for audit (creation, state transitions, host start, host end, force-end, cancellation) in `audit_log` with `actor_id` where applicable.

## 6. Business Rules

- One live session per `(schedule_entry_id, class_date)` for live/hybrid occurrences.
- Only the resolved teacher of the occurrence may host; only enrolled students of the occurrence's batch (at `class_date`) may join.
- Join is permitted only when state is `READY` or `LIVE` and `now ∈ join_window`.
- Cancellation is one-way; a cancelled session cannot be revived. The next occurrence of the same recurring slot gets a fresh session.
- Recording reference belongs to the occurrence, not to a generic library. Reuse via Teacher / Student Class Workspaces is read-through only.
- Live participation events feed Attendance; this module never marks a student "present" or "absent."
- Provider failures degrade gracefully — the host or student sees an actionable error rather than a stuck loading state.
- Sessions store the resolved teacher's identity at creation but follow override changes up until `READY` (so a same-day teacher swap by Admin is honoured).
- Webhook handlers are idempotent; the provider may legitimately retry on its own backoff.
- Session state is the **system's** state, not the provider's — if a provider crashes, the system's auto-close + force-end keep the state consistent.

## 7. User Flow / Process Flow

### 7.1 Teacher Hosts a Live Class

1. Teacher opens class detail in Teacher Class Workspace at `start_time − 10 min` (READY window open).
2. Teacher clicks **Start Live Class**.
3. Workspace calls `POST /live-classes/{schedule_entry_id}/start?class_date=…`.
4. System: creates the `LiveSession` if not eager-created; verifies teacher = host; calls provider adapter to mint a host token; transitions state to `LIVE` on provider confirmation; mints a host join token.
5. Workspace redirects the teacher to the provider's host URL with the join token.
6. Provider sends `participant.joined` webhook for the host → `LiveParticipantEvent` row with `role = HOST` is written.
7. Teacher conducts the class. Provider sends `participant.joined` / `participant.left` webhooks per student.
8. Teacher ends the class. Workspace calls `POST /live-classes/{id}/end`. State → `ENDED`. Provider sends `session.ended` webhook (redundant; idempotent).
9. Provider sends `recording.ready` webhook within minutes/hours. `LiveRecordingReference` row written; status `AVAILABLE`.

### 7.2 Student Joins a Live Class

1. Student opens class detail in Student Class Workspace within the join window.
2. Workspace fetches live state (`live_session.state = READY` or `LIVE`).
3. Student clicks **Join class**.
4. Workspace calls `POST /live-classes/{schedule_entry_id}/join?class_date=…`.
5. System: verifies batch-at-the-time, window, state; mints a participant join token.
6. Workspace redirects to the provider's join URL with the token.
7. Provider sends `participant.joined` webhook → `LiveParticipantEvent` row written.
8. Student attends. If they disconnect, the provider sends `participant.left`; on re-entry, a new `LiveParticipantEvent` row is written.
9. Class ends; the student's `left_at` is finalised.

### 7.3 Cancellation via Scheduling Override

1. Admin creates a `ScheduleOverride` with `status = CANCELLED` for `(schedule_entry_id, class_date)` in Scheduling.
2. Scheduling emits `SCHEDULE_OVERRIDE_CREATED` with `change_kind = CANCELLED`.
3. Live Class consumes the event; if the matching session exists in `UPCOMING` or `READY`, it transitions to `CANCELLED` with `reason` propagated from the override.
4. Teacher and students who had the workspace open see the cancelled banner on next fetch (within the schedule event SLA).

### 7.4 Admin Force-Ends a Stuck Session

1. Admin sees the active sessions list; one session has been `LIVE` for hours past its `end_time`.
2. Admin clicks **Force end**, supplies a reason.
3. System: state → `ENDED`; final participant events are flushed; provider is asked to end the meeting (best-effort).
4. Audit row written.

### 7.5 Auto-Close

1. Background job runs every 5 min.
2. Selects `LIVE` sessions where `end_time + 30 min < now`.
3. Transitions each to `ENDED`; flushes participant events; notifies teacher.

## 8. Data Model

### `LiveSession`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| schedule_entry_id | UUID, FK → ScheduleEntry.id | |
| class_date | Date | Occurrence date |
| provider | enum | e.g. `ZOOM`, `MEET`, `JITSI`, `WHEREBY` (v1 chooses one; see OQ) |
| provider_session_ref | string | Opaque provider handle |
| host_teacher_id | UUID, FK → User.id | The teacher resolved at creation; may be updated up to `READY` per FR-LCM-003 |
| status | enum(`UPCOMING`, `READY`, `LIVE`, `ENDED`, `CANCELLED`) | |
| start_at | Timestamp | Snapshot of the resolved start time at creation; re-resolved on `UPCOMING → READY` if a `ScheduleOverride` changed it (FR-LCM-003 pattern). Frozen once `LIVE` or later |
| end_at | Timestamp | Same snapshot rules as `start_at` |
| started_at | Timestamp, nullable | Actual transition to `LIVE` |
| ended_at | Timestamp, nullable | Actual transition to `ENDED` |
| cancelled_at | Timestamp, nullable | Transition to `CANCELLED` |
| cancel_reason | string, nullable | Required when status = CANCELLED |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| | | **Partial unique index** on `(schedule_entry_id, class_date)` WHERE status ≠ 'CANCELLED' |

### `LiveParticipantEvent`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| session_id | UUID, FK → LiveSession.id | |
| user_id | UUID, FK → User.id | |
| role | enum(`HOST`, `PARTICIPANT`) | |
| joined_at | Timestamp | |
| left_at | Timestamp, nullable | |
| duration_sec | integer, nullable | Computed `(left_at - joined_at)` rounded to seconds |
| provider_event_id | string, nullable | Provider's correlation id for idempotency |
| created_at | Timestamp | |
| | | Index on `(session_id, user_id, joined_at)` |

### `LiveRecordingReference`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| session_id | UUID, FK → LiveSession.id, unique | One recording per session |
| recording_ref | string | Provider handle |
| playback_url | string, nullable | Provider playback URL or re-signed CDN URL (populated when status = AVAILABLE) |
| status | enum(`PROCESSING`, `AVAILABLE`, `FAILED`) | |
| received_at | Timestamp | Webhook or poll receipt |
| failed_reason | string, nullable | When status = FAILED |
| created_at | Timestamp | |
| updated_at | Timestamp | |

### `WebhookDeliveryLog` (for FR-LCM-035)

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| provider | string | |
| event_type | string | e.g. `participant.joined` |
| provider_event_id | string | For idempotency |
| outcome | enum(`processed`, `rejected_signature`, `rejected_duplicate`, `rejected_invalid`) | |
| received_at | Timestamp | |
| payload_excerpt | text | First ~1 KB for diagnostics |
| | | Unique on `(provider, provider_event_id)` to enforce idempotency |

> **Multi-tenancy note:** ADR-0003 is pending. This SRS assumes single-tenant; under multi-tenant, every table gets a `tenant_id` column and the provider adapter must scope provider accounts per tenant. Same caveat as `auth.md` §4.

## 9. API Contracts

### Get Live Session State

```http
GET /live-classes/{schedule_entry_id}?class_date=2026-05-18
```

**Response (200)**

```json
{
  "session": {
    "id": "0192...",
    "schedule_entry_id": "0192...",
    "class_date": "2026-05-18",
    "status": "READY",
    "start_at": "2026-05-18T08:00:00+06:00",
    "end_at": "2026-05-18T09:30:00+06:00",
    "join_window": {
      "from": "2026-05-18T07:50:00+06:00",
      "to": "2026-05-18T10:00:00+06:00"
    },
    "host_teacher_id": "0192...",
    "is_caller_host": false,
    "joinable": true
  }
}
```

`joinable` is true when the caller may join *right now* given their role and the current state/window.

**Response (404)** — `SESSION_NOT_FOUND` (no LiveSession row exists yet; create-on-demand may produce one when the host calls start).

Satisfies: FR-LCM-010.

### Start Session (host)

```http
POST /live-classes/{schedule_entry_id}/start?class_date=2026-05-18
```

**Headers**

```
Authorization: Bearer <teacher access_token>
```

Creates the session if it doesn't exist; transitions to `LIVE`; mints a host join token.

**Response (200)**

```json
{
  "session_id": "0192...",
  "status": "LIVE",
  "host_join_url": "https://provider.example.com/j/...",
  "host_join_token": "<opaque, single-use, 5-min TTL>"
}
```

**Errors**

- `400` — `INVALID_DELIVERY_MODE` (occurrence is not LIVE_ONLINE / HYBRID)
- `403` — `NOT_HOST` (caller is not the resolved teacher)
- `409` — `SESSION_CANCELLED` / `SESSION_ENDED` / `JOIN_WINDOW_NOT_OPEN`
- `503` — `PROVIDER_UNAVAILABLE`

Satisfies: FR-LCM-001 to FR-LCM-005, FR-LCM-007, FR-LCM-011 to FR-LCM-013.

### Join Session (participant)

```http
POST /live-classes/{schedule_entry_id}/join?class_date=2026-05-18
```

**Headers**

```
Authorization: Bearer <student access_token>
```

**Response (200)**

```json
{
  "session_id": "0192...",
  "status": "LIVE",
  "join_url": "https://provider.example.com/j/...",
  "join_token": "<opaque, single-use, 5-min TTL>"
}
```

**Errors**

- `403` — `NOT_IN_BATCH`
- `409` — `JOIN_WINDOW_CLOSED` / `SESSION_NOT_READY` (state `UPCOMING`) / `SESSION_ENDED` / `SESSION_CANCELLED`

Satisfies: FR-LCM-015 to FR-LCM-019.

### End Session (host)

```http
POST /live-classes/{schedule_entry_id}/end?class_date=2026-05-18
```

**Response (200)**

```json
{ "session_id": "0192...", "status": "ENDED", "ended_at": "..." }
```

**Errors**

- `403` — `NOT_HOST`
- `409` — `INVALID_STATUS` (only LIVE may end via this endpoint)

Satisfies: FR-LCM-008.

### Get Recording

```http
GET /live-classes/{schedule_entry_id}/recording?class_date=2026-05-18
```

**Response (200)**

```json
{
  "session_id": "0192...",
  "status": "AVAILABLE",
  "playback_url": "https://...",
  "received_at": "2026-05-18T11:00:00+06:00"
}
```

**Response (404)** — `RECORDING_NOT_FOUND` (no row) or returned with `status: PROCESSING` / `FAILED` per state.

**Errors**

- `403` — `FORBIDDEN` (student outside batch-at-the-time, or recording not in AVAILABLE state for student callers)

Satisfies: FR-LCM-024 to FR-LCM-028.

### Admin — Cancel Session

```http
POST /live-classes/{schedule_entry_id}/cancel?class_date=2026-05-18
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

**Request**

```json
{ "reason": "Public holiday" }
```

**Response (200)** — updated session with `status: "CANCELLED"`, `cancel_reason`.

**Errors**

- `403` — `FORBIDDEN` (not Admin)
- `409` — `INVALID_STATUS` (cannot cancel LIVE or ENDED)

Satisfies: FR-LCM-029.

### Admin — Force End

```http
POST /live-classes/{schedule_entry_id}/force-end?class_date=2026-05-18
```

**Request**

```json
{ "reason": "Host network failure; class effectively over" }
```

**Response (200)** — `{ "session_id": "0192...", "status": "ENDED" }`.

Satisfies: FR-LCM-030.

### Admin — Active Sessions

```http
GET /admin/live-sessions/active
```

**Response (200)**

```json
{
  "sessions": [
    {
      "id": "0192...",
      "schedule_entry_id": "0192...",
      "class_date": "2026-05-18",
      "batch_name": "Class 10 — Batch A",
      "subject_name": "Physics",
      "host_teacher_name": "A. Rahman",
      "status": "LIVE",
      "started_at": "2026-05-18T08:01:00+06:00",
      "participant_count_live": 28
    }
  ]
}
```

Satisfies: FR-LCM-031.

### Provider Webhook

```http
POST /webhooks/live-class/{provider}
```

**Headers**

```
X-Provider-Signature: <hmac>
X-Provider-Timestamp: <unix-ts>
```

Payload schema is provider-specific. Handler verifies signature, deduplicates by `provider_event_id`, applies state transition / writes `LiveParticipantEvent` / `LiveRecordingReference` as appropriate.

**Responses**

- `200` — `processed`
- `401` — invalid signature
- `409` — duplicate event id (still returns 200 to most providers to prevent retry storms; the response body indicates duplicate)
- `422` — malformed payload

Satisfies: FR-LCM-032 to FR-LCM-035.

## 10. UI Components

### Teacher Live Class Panel (inside Teacher Class Workspace detail)

- **Components:** State badge (UPCOMING / READY / LIVE / ENDED / CANCELLED); Start Live Class button (enabled when READY and caller is host); provider status indicator ("Provider OK" / "Provider unavailable — retry"); participant count (live; if provider supports); End button (when LIVE).
- **Actions:** Start, End.
- **States:** disabled (not yet in window), ready-to-start, starting (loading), live (with End button), ended.

### Student Live Class Panel (inside Student Class Workspace detail)

- **Components:** Countdown to `start_time` (UPCOMING); "Class hasn't started yet" (READY before host start); Join Class button (LIVE or READY with provider waiting room); "Class has ended" (ENDED); "Class was cancelled — `<reason>`" (CANCELLED); "Live provider unavailable — please refresh" fallback.
- **Actions:** Join.
- **States:** not_started, joinable, ongoing, ended, cancelled, unavailable.

### Admin Monitoring View

- **Components:** Table of active sessions (host, batch, subject, state, started_at, participant count); filter by state; force-end action per row; cancel action per row.
- **Actions:** Force-end, cancel, drill-down to session log.

### Admin Cancel / Force-end Modal

- **Components:** Reason textarea (required), confirm/cancel buttons.

## 11. Validation Rules

- **Occurrence binding:** `schedule_entry_id` + `class_date` must reference a Scheduling occurrence with `delivery_mode ∈ { LIVE_ONLINE, HYBRID }`.
- **Host identity:** caller `user_id` must equal `LiveSession.host_teacher_id` for `start` / `end`. Re-resolved against override per FR-LCM-003.
- **Student eligibility:** caller must have an open `StudentBatch` row matching the occurrence's batch on `class_date`.
- **Join window:** server clock in BST; default `[start_time − 10 min, end_time + 30 min]`; configurable.
- **State transition matrix:** UPCOMING → READY (auto); UPCOMING|READY → LIVE (host start); LIVE → ENDED (host end / webhook / auto-close); UPCOMING|READY → CANCELLED (override / admin); ENDED is terminal. Any other transition rejected with `INVALID_STATUS`.
- **Cancel reason:** required string, ≤ 500 chars.
- **Join token TTL:** 5 minutes; single-use; minted per request.
- **Webhook signature:** required; rejected with 401 if absent or invalid.
- **Webhook idempotency:** `(provider, provider_event_id)` must be unique; duplicates return 200 with `duplicate: true` body.
- **Recording access:** `status = AVAILABLE` for student callers; any status for teacher/admin.
- **API path convention:** all endpoints in §9 are addressed by `{schedule_entry_id}` with `?class_date=YYYY-MM-DD` query param. The internal `LiveSession.id` (UUID) is not exposed in URLs — clients always identify a session by its occurrence pair. Missing `class_date` on any endpoint returns `400 INVALID_INPUT`.
- **Path consistency:** §5 FR references that previously used `{id}` are read as `{schedule_entry_id}` per this section's convention.

## 12. Edge Cases

- Provider unavailable at start → caller receives `503 PROVIDER_UNAVAILABLE`; the session is not created; teacher retries. The class window remains open until `end_time + 30 min`.
- Teacher starts late → `READY` lingers past `start_time`; students see "Class hasn't started yet" until the host appears. Auto-close still fires at `end_time + 30 min`.
- Student joins after start → allowed within the join window; `LiveParticipantEvent.joined_at` reflects the actual join time; Attendance Management decides whether the late join counts as present.
- Duplicate join attempts (student refreshes the workspace, clicks Join twice) → each successful join mints a new token; the provider deduplicates concurrent sessions for the same user (provider-dependent); multiple `LiveParticipantEvent` rows are written (each a real join/leave pair).
- Network drop and re-entry → provider sends `participant.left` then `participant.joined` again; two separate `LiveParticipantEvent` rows result; Attendance Management sums durations.
- Provider recording delayed → webhook arrives hours later; `LiveRecordingReference.status` stays `PROCESSING` then flips to `AVAILABLE`. The fallback poll job covers the case where the webhook never arrives.
- Session ends without recording (provider didn't capture, or institution disabled recording) → no `LiveRecordingReference` row is created. Student detail shows "Recording not available."
- Join button shown but token mint fails (provider transient error) → UI surfaces "Couldn't join — please retry"; no `LiveParticipantEvent` row is written until the provider confirms join via webhook.
- Scheduling override changes teacher mid-cycle → `host_teacher_id` updates on next `READY` transition (FR-LCM-003); if already `LIVE`, the original host stays.
- Scheduling override cancels the occurrence after `LIVE` → cancellation is ignored for live sessions; the override only affects future occurrences. Teacher may end manually.
- Host's `is_active = false` flips during a `LIVE` session → Auth's session revocation forces them out at the API layer; the auto-close job picks up the stuck session at the deadline. Admin may force-end sooner.
- Two webhooks arrive simultaneously for `session.started` and a teacher manual `start` call → both attempt to transition `READY` → `LIVE`; idempotent at the state machine layer (only first write succeeds; second is a no-op).
- Webhook signature verification fails (replay or tampering) → 401; logged; no state change.
- Provider returns a 5xx during the `start` adapter call → the caller receives `503 PROVIDER_UNAVAILABLE`; the session is not partially created; no row to clean up.
- Recording for a CANCELLED session → not possible; cancellation occurs before LIVE, and the provider never produces a recording.
- Student in HYBRID mode where the live session ends but on-site continues → live class is `ENDED`; student detail loses the Join button; the on-site portion is not affected (Attendance for on-site is separate).
- A student tries to retrieve the recording from a class they were not in (batch-at-time fails) → `403`; this is the same enforcement as the Student Workspace's previous-class detail rule.
- Auto-close job is delayed (Redis or job runner outage) → sessions remain `LIVE` past their window; Admin may force-end; the next job run flushes the backlog.
- **Student receives a valid join token from this module but the provider rejects entry** (waiting room not opened, provider hiccup, expired session ref) → no `LiveParticipantEvent` row is written (rows are webhook-driven per FR-LCM-019); student retries `POST /join`, gets a fresh token, re-attempts. No phantom row remains.
- **`participant.left` webhook arrives without a matching `joined_at` row** (lost join webhook) → orphan event logged in `WebhookDeliveryLog` with `outcome = rejected_invalid`; no `LiveParticipantEvent` mutated. Attendance Management sees a gap in the participation data and applies its own policy.
- **Teacher double-clicks Start** during a slow network → first call mints a host token and transitions to `LIVE`; the second arrives while state is `LIVE`, returns `200` with a fresh host token (FR-LCM-011), no duplicate state change.
- **Two admins force-end the same session simultaneously** → row-level lock on `LiveSession` serialises; first call ends; second receives `200` with the already-ended state (FR-LCM-030 idempotency).
- **`class_date` query param missing on any endpoint** → `400 INVALID_INPUT` per §11.
- **Recording `status = FAILED` is terminal.** No retry mechanism in v1; Admin sees the failed status; future enhancement may add manual re-poll (see OQ).
- **HYBRID occurrence with both online and on-site students** → live session captures only the online participants; on-site attendance is recorded by Attendance Management via a different path (manual marking by the teacher). The `LiveParticipantEvent` log identifies online participation only; Attendance Management is responsible for reconciling the two streams.
- **`SCHEDULE_OVERRIDE_CREATED` with `change_kind = CANCELLED` arrives while session is LIVE** → silently discarded for this module per FR-LCM-009a; the LIVE session continues to its natural or forced end; the Scheduling occurrence and the Live Class session are temporarily out of sync, accepted as design.
- **`BATCH_SUBJECT_TEACHER_REASSIGNED` event consumption** — the event payload (`class-context.md` FR-CBS-026b) names `(batch_id, subject_id, old_teacher_id, new_teacher_id)` but not `schedule_entry_id`. Live Class queries Scheduling for `ScheduleEntry` rows under `(batch_id, subject_id)`, filters to entries on active routines, finds matching `LiveSession` rows in `UPCOMING`, and re-resolves `host_teacher_id` per FR-LCM-003. Sessions in `READY` or later are unaffected.

## 13. Dependencies

- **Class Scheduling (`scheduling.md`)** — source of the occurrence, its time, its resolved teacher, and the `delivery_mode`. Consumes events: `SCHEDULE_OVERRIDE_CREATED` (apply teacher swap / cancellation per FR-LCM-003 and FR-LCM-009a), `ROUTINE_REPLACED` (Live Class queries Scheduling for the affected entries' new schedule_entry_id values to identify which UPCOMING `LiveSession` rows on the now-archived entries should be cancelled or migrated; sessions already in `READY` or later are unaffected).
- **Class Context (`class-context.md`)** — provides `BatchSubjectTeacher` for teacher resolution and `StudentBatch` lineage for participant eligibility. Consumes `BATCH_SUBJECT_TEACHER_REASSIGNED` (FR-CBS-026b) — payload lacks `schedule_entry_id`, so handler queries Scheduling for matching entries on `(batch_id, subject_id)`; only `UPCOMING` `LiveSession` rows are re-resolved (per FR-LCM-003 boundary).
- **Teacher Class Workspace (`teacher-workspace.md`)** — caller for the host start/end actions; consumes `live_session.state` for the detail panel.
- **Student Class Workspace (`student-workspace.md`)** — caller for the participant join action; consumes session state and recording.
- **Attendance Management** — consumer of `LIVE_PARTICIPATION_RECORDED` events.
- **Notification Management** — consumer of `LIVE_SESSION_CANCELLED`, `LIVE_SESSION_FORCE_ENDED`, `LIVE_RECORDING_AVAILABLE` events; sends SMS / in-app to affected teachers and students.
- **External Meeting Provider** — owns A/V; v1 vendor TBD (see OQ). The provider adapter is a backend abstraction.
- **Authentication & Authorization (`auth.md`)** — RBAC; teacher / student / admin role gates on the respective endpoints.
- **Redis** (per ADR-0002) — backs the join-token store (TTL 5 min) and the webhook dedup cache.
- **Central `audit_log`** — receives lifecycle events: created, started, ended, cancelled, force-ended, host-token-minted, participant-token-minted.

## 14. Non-Functional Requirements

- **Performance:** `start` and `join` p95 < 2 s (provider-bound; constrained by the provider adapter). Webhook processing p95 < 500 ms.
- **Security:** all endpoints HTTPS; join tokens single-use with short TTL; webhook signature verification mandatory; teacher and student tokens never returned in URLs that log to the provider (only as POST bodies / headers).
- **Availability:** provider failures degrade gracefully — the host and students see actionable errors, not silent stalls.
- **Concurrency:** webhook idempotency via the `(provider, provider_event_id)` unique constraint; state machine transitions enforced by a row-level lock on `LiveSession`.
- **Mobile-first:** student join flow optimised for low-end Android; redirect to the provider's native app where available; the provider adapter must support a web-only fallback for devices without the app.
- **Scale:** up to 200 concurrent live sessions per institution; webhook handler horizontal-scale-friendly (stateless aside from Redis dedup).
- **Auditability:** every state transition produces an `audit_log` row.
- **Time zone:** all timestamps stored in UTC; presented in BST (+06:00) for UI; join window computed in BST.

## 15. Assumptions

- An External Meeting Provider is integrated; v1 ships with one adapter (vendor TBD per OQ).
- The provider supports webhooks for session and participant events; if it does not, the fallback poll job covers participant events at coarser granularity.
- Recording is provider-supplied; institution can disable per setting.
- Teachers and students access the provider via the workspace redirect, not directly.
- The institution has working internet for the host's side; student-side weak network is expected and handled by the provider, not by this module.

## 16. Open Questions

- **Provider choice for v1:** Zoom, Google Meet, Jitsi, Whereby, or self-hosted? Cost, recording support, Bangladesh data residency, and SDK quality all differ. Blocks vendor adapter implementation.
- **Eager vs. lazy session creation:** create at `start_time − 15 min` via a job, or only when teacher first hits start? Eager is friendlier to provider quirks but costs API calls for classes that never happen.
- **Early-start policy:** may a teacher start more than 10 min before `start_time` (e.g. troubleshooting)? v1 default = no; revisit if institutions ask.
- **Late-student attendance:** should late joins beyond a threshold count as absent or late? This is owned by Attendance; flagged here for cross-module decision.
- **Force-end behaviour:** when Admin force-ends, do students see "class ended" or "class ended by Admin — `<reason>`"?
- **Recording visibility default:** when a recording becomes AVAILABLE, is it auto-published to students or does the teacher have to enable it (Teacher Workspace `visibility_status = PUBLISHED`)? Cross-module policy.
- **Cross-batch joint live class** (one session for two batches simultaneously, e.g. revision class) — explicitly out of v1; will need a different data model (`LiveSession` would need many `schedule_entry_id` links).
- **Provider recording → File Asset module pipeline:** does the recording playback URL live on the provider (passes through) or is it copied to institutional storage (R2) for retention and control? Affects storage costs and data residency. Decision deferred.
- **Webhook fallback when provider is permanently down:** if the provider has a multi-hour outage, what is the recovery plan for ongoing sessions? Manual mark-ended via Admin; document the runbook.
- **Audit log volume:** every join and leave is recorded; for a school with 50 live sessions/day × 30 students × 2 events ≈ 3,000 rows/day. Confirm `audit_log` sizing; or write participant events only to `LiveParticipantEvent` and not to `audit_log`.
