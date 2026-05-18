# Screen: Class, Batch & Subject Context — Class list

**Role(s):** Admin
**Density:** compact (admin convention — DESIGN.md §13)
**Primary device:** desktop

## Purpose

Let an Admin browse every class in the institution, see status and operational health at a glance (mandatory subjects assigned, batch count), and start the create / edit / archive flows.

## Layout

Single-column desktop layout under the standard admin shell (top bar §14.16 + left sidebar §14.16). The screen body is one data table (§14.12) preceded by a page header and a filter bar. No KPI band — the Class list is a working list, not a dashboard.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Top bar (logo · Notifications · Profile)                       z.fixed   │
├──────────┬───────────────────────────────────────────────────────────────┤
│          │ ┌─ Page header ──────────────────────────────────────────────┐│
│ Sidebar  │ │ Classes                              [+ New class] (amber) ││
│  Acad…   │ │ Manage grade-level containers …                            ││
│  > Cls   │ └────────────────────────────────────────────────────────────┘│
│   Subj   │ ┌─ Filter bar ───────────────────────────────────────────────┐│
│   Btch   │ │ [(Search icon) Search by name or code]   [Status ▾]   12 classes      ││
│  Users   │ └────────────────────────────────────────────────────────────┘│
│  ...     │ ┌─ Data table (compact) ─────────────────────────────────────┐│
│          │ │ Code  Name           Subjects  Batches  Status   Updated …││
│          │ ├────────────────────────────────────────────────────────────┤│
│          │ │ C10   Class 10       5         3        ACTIVE   11 May …  ││
│          │ │ C9    Class 9        4         2        ACTIVE   05 May …  ││
│          │ │ HSC1  HSC 1st Year   6         0        INACTIVE 12 Apr …  ││
│          │ │ …                                                          ││
│          │ └────────────────────────────────────────────────────────────┘│
│          │ Pagination · 1–25 of 47                                       │
└──────────┴───────────────────────────────────────────────────────────────┘
```

## Sections

### Page header

- Components: page title (`display.sm`), one-line description (`body.md · text.muted`), primary CTA button (§14.1) on the right.
- Tokens: `text.primary`, `text.muted`, `interactive.accent.*` for the CTA (amber, because creating a class is the highest-value action on this screen — §2 / §14.1 rule "one filled button per view section").
- CTA label: `+ New class`. On click → opens the **Create class** dialog (§14.13, sm width 400px).

### Filter bar

- Components: inline search (§14.17, `md` size), status select (§14.3), result count caption.
- Search debounced; partial match on `Class.name` and `Class.code`. Clear (×) shows when value present.
- Status select options: **All**, **Active**, **Inactive**, **Archived**. Default: **Active** (admins almost always want the live set first).
- Result count uses `body.sm · text.muted · tabular`. Updates as filters change.
- Tokens: `surface.raised` for the bar background, `border.subtle` divider below.

### Data table

Compact data table (§14.12). Columns:

| Column | Type | Format | Notes |
|---|---|---|---|
| Code | text | `font.mono`, body.sm | e.g. `C10`. Left-aligned. Width ~96px. |
| Name | text | body.md | May contain Bangla — `font.bn` per substring (§12). Flex width. |
| Subjects | numeric | tabular, body.sm | Mandatory subjects from `ClassSubject` count. Right-aligned. **When count = 0**, the cell renders `0` + a Lucide `AlertTriangle` icon (16px, per the §11 icon scale) inheriting the `status.cancelled.fg` color (lifecycle family, §5) + a visible inline label "No subjects" (`caption · status.cancelled.fg`). The icon, the colored label text, and the icon's shape together satisfy the "not by color alone" rule (§18). Width ~104px (allow flex on the warning state). |
| Batches | numeric | tabular, body.sm | Total non-archived batches. Right-aligned. Width ~96px. |
| Status | badge | §14.6 | Lifecycle family (§5) — chosen as the closest semantic family since no entity-activation family exists: `ACTIVE` → `published` palette, `INACTIVE` → `draft` palette, `ARCHIVED` → `archived` palette. **The badge label text is the literal class status — `ACTIVE`, `INACTIVE`, `ARCHIVED` — not the palette key (`PUBLISHED`, `DRAFT`).** Only the colors come from the lifecycle family. One badge per row (rule §14.6). Width ~108px. |
| Updated | date | body.sm · tabular | `11 May 2026` format (§15). Sortable, default sort newest first. Width ~120px. |
| ⋯ | icon-only menu button | §14.1 | Trailing column. Opens row menu (see Interactions). Width 40px. |

- Row hover: `surface.sunken` (§14.12).
- Row click anywhere except the ⋯ button: navigates to **Class detail** for that class.
- Tabular numerals on Subjects, Batches, Updated.
- Sticky header on scroll.

### Pagination

- Below the table, right-aligned.
- "1–25 of 47" caption + Previous / Next icon buttons (sm size).
- Page size fixed at 25 for v1.

## States

### Empty

- Trigger: institution has 0 classes (first run after Admin onboarding).
- Layout per §14.18: centered, 48px padding.
- Icon: `GraduationCap` (Lucide) · 32px · `text.muted`.
- Heading: "No classes yet" (`heading.sm`).
- Body: "Create your first class to start defining batches and assigning subjects." (`body.sm · text.muted`).
- Action: **filled navy primary button** `+ New class` (md size, §14.1). The amber variant is reserved for the page-header CTA, which stays visible — only one filled amber button may exist on a screen (§2, §14.1). The empty-state action shares the same target.

A separate **filtered-empty** state: "No classes match these filters." with a `Clear filters` ghost button (§14.1).

### Loading

- Skeleton (§14.19) matching the populated layout:
  - 1 page-header row + filter bar are real (controls remain interactive).
  - Table body shows 8 skeleton rows. Each row: 7 skeleton blocks matching column widths, `surface.sunken`, ~1.5s pulse, 36px row height.

### Populated

- Default state (see ASCII).
- Result count and pagination caption reflect filtered total.

### Error

- Trigger: list fetch fails.
- Render an inline alert *above* the table (not a toast — §14.14 rule "use inline alert for anything the user must act on"):
  - Surface: `surface.raised` + 3px left border in `status.cancelled.fg` (lifecycle family, §5 — the only semantic danger expression available without introducing a new token).
  - Icon: `AlertCircle` 20px · `status.cancelled.fg`.
  - Title: "Couldn't load classes." (`body.md · 500 · text.primary`).
  - Body: server error message (`body.sm · text.secondary`).
  - Action: `Retry` secondary button (§14.1 sm).
- Table area below collapses to a single empty row stating the same.

## Interactions

- **Tap on row (anywhere except ⋯)** → navigate to **Class detail** (subjects + batches tabs).
- **Tap on ⋯ button** → opens a **context-action menu popover** (not a `<select>` element). It borrows visual dimensions from §14.3 — `surface.overlay`, `radius.md`, `shadow.md`, layer `z.dropdown`, option rows 36px high with 12px horizontal padding — but is implemented as a Menu / Popover anchored to the ⋯ button, with keyboard navigation per the standard menu pattern. Options depend on current `status`:
  - `ACTIVE`: `Edit`, `Deactivate`, `Archive`
  - `INACTIVE`: `Edit`, `Activate`, `Archive`
  - `ARCHIVED`: `View` only (no edit, no un-archive — `ARCHIVED → *` is rejected per §11 SRS Validation Rules)
- **Tap `+ New class`** → opens Create class dialog (separate spec).
- **Tap `Archive`** → opens a confirm modal (§14.13 sm). If the class has non-archived batches, the modal shows the error inline before confirm: "This class has 3 non-archived batches. Archive its batches first." (FR-CBS-005) — Confirm is disabled, only Cancel is enabled. Otherwise: title "Archive this class?", body "Archived classes can't have new batches. Existing batches keep operating.", actions: Cancel (secondary) · `Archive` (destructive variant, §14.1).
- **Tap `Activate` / `Deactivate`** → optimistic toggle with success toast (§14.14, 5s auto-dismiss). On failure, revert and show inline alert.
- **Keyboard:** rows are focusable; Enter opens the row. The ⋯ button is reachable via Tab and Space/Enter opens the menu. Focus ring per §14.0 (2px `border.focus`, 2px offset).

## Data needs

- **Endpoint:** `GET /classes?status=<ACTIVE|INACTIVE|ARCHIVED|ALL>&q=<search>&page=<n>&page_size=25` — *to be specified in `docs/api-contracts/class-context.md`*. Returns array of `{ id, name, code, status, subject_count, batch_count, updated_at }` plus `{ total, page, page_size }` envelope. Subject count = mandatory `ClassSubject` rows; batch count = non-archived `Batch` rows. Tenant-scoped via JWT (per SRS §8 multi-tenancy block).
- **Status change endpoint:** `POST /classes/{class_id}/status` (already defined, SRS §9). Used by Activate / Deactivate / Archive row actions.
- **Refresh:** on mount; on filter / search / pagination change; after any row-level write (optimistic, with revalidation).
- **Cache:** not in resolution cache scope (that's per-batch only); a plain SWR/RTK-Query list cache is sufficient.

## Bangladesh-context

- **Names:** class names may be Bangla ("নবম শ্রেণি"). Wrap Bangla substrings in `font.bn` per §12. Allow row height to flex if a long Bangla name wraps.
- **Codes:** ASCII only by validation (§11 SRS) — always render in `font.mono`.
- **Dates:** `11 May 2026` (§15). Western Arabic numerals only, even in Bangla content (§12).
- **No currency on this screen.**

## Accessibility

- **Touch targets:** rows are 36px tall (compact, §7 / §14.12) — desktop's permitted minimum is 32 × 32px (mouse-precision context, §7), so the row and the ⋯ button (32 × 32px) both clear that bar. This screen is desktop-primary; if a future mobile variant is added, it must switch to the comfortable 48px row height to meet the 44 × 44px mobile minimum.
- **Color-alone rule (§18):** status is communicated by both the colored pill *and* the literal status label inside it (ACTIVE / INACTIVE / ARCHIVED). The "no subjects assigned" warning uses the `AlertTriangle` icon shape, the colored cell content, **and** the visible inline label "No subjects" — three independent signals. The icon also carries `aria-label="No mandatory subjects — batches under this class cannot be activated (FR-CBS-012)"` for screen readers.
- **Focus order:** Page CTA → search → status filter → table header (sortable) → row 1 → row 1 ⋯ → row 2 → … → pagination.
- **Sticky header** does not steal focus during keyboard navigation.
- **`aria-label`** on ⋯ button: "Actions for class {name}".
- **Screen reader:** table uses proper `<table>` semantics with `<th scope="col">` and `<caption>`. Bangla cells carry `lang="bn"` (§18).

## Out of scope

- Adding subjects to a class (that's the Class detail spec — Subjects tab).
- Creating / editing / managing batches (Batch list and Batch detail specs).
- Bulk operations (multi-select archive, etc.) — v1 is single-row only.
- Import / export of classes — not in v1 scope.
- Teacher and student counts at the class level — not in the SRS; would require a derived count outside `Class`. Skip in v1.

## SRS references

- `docs/srs/class-context.md` §5.1 (Class Management), §5.3 (Class-to-Subject Assignment for the warning), §6 (Business Rules), §10 (UI Components — Class & Subject Setup), §11 (Validation Rules)
- **FR IDs:** FR-CBS-001, FR-CBS-002, FR-CBS-003, FR-CBS-004, FR-CBS-005, FR-CBS-012 (warning indicator only)
