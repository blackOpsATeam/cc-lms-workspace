# Canvas prompt — Class list (populated, desktop)

> **How to use this file.** Open a new conversation at claude.ai. Paste the entire block below the divider (everything between `===8<===` lines) into the chat. Claude will render the screen as an interactive artifact in the Canvas panel. Iterate by saying "now show the empty state" / "now show this on mobile" / "tighten the row height" — Canvas updates in place.
>
> **One prompt = one state = one device.** Don't ask for all four states at once. Quality drops, and the artifact becomes a Frankenstein.
>
> **Variants of this prompt:**
> - Below = **populated, desktop**. The default.
> - Swap the `RENDER` block for the **empty**, **loading**, or **error** state (templates at the bottom of this file).
> - Swap `desktop` for `mobile` to ask for the responsive variant — note Admin is desktop-first per the spec, so the mobile variant is "narrow desktop / tablet" rather than a true phone form factor.

---

===8<=== START COPY ===8<===

You are designing one screen of the **CC-LMS** product — a coaching center management system for Bangladesh.

**Brand:** Institutional, trustworthy, bank-grade calm. Authority without coldness. NOT consumer-SaaS-modern, NOT playful, NOT pastel-flat.

**Output:** a single self-contained **React + Tailwind** artifact in Canvas. No external data fetching, no props — hardcode the realistic Bangladesh data shown in the spec. Use inline Tailwind classes that match the design tokens below (don't try to import a token system). Use the Lucide icon library (`lucide-react`).

---

## DESIGN SYSTEM — authoritative, do not invent

### Colors (semantic, light mode)

```
surface.canvas        #F8FAFC    App background
surface.raised        #FFFFFF    Cards, panels, table
surface.overlay       #FFFFFF    Modals, popovers
surface.sunken        #F1F5F9    Input wells, hover row, table header

text.primary          #0F172A    Body and headings
text.secondary        #334155    Supporting copy
text.muted            #64748B    Labels, captions, timestamps
text.link             #1E4478    Inline links
text.brand            #14315C    Brand-emphasized text

border.subtle         #E2E8F0    Dividers, hairlines
border.default        #CBD5E1    Inputs, cards
border.strong         #94A3B8    Hover, emphasis
border.brand          #1E4478    Selected, brand-tinted
border.focus          #2D5793    Focus ring

interactive.primary.rest      #14315C  (navy)
interactive.primary.hover     #0E2447
interactive.accent.rest       #D97706  (amber)
interactive.accent.hover      #B45F05
```

### Status palette — Lifecycle family (this screen uses these for status badges)

```
                  BG          FG
draft             #F1F5F9     #334155     → use for INACTIVE class
published         #ECFDF5     #047857     → use for ACTIVE class
archived          #F1F5F9     #475569     → use for ARCHIVED class
```

### Typography

```
Font: Inter for English. Fallback "system-ui".
Mono: JetBrains Mono — for the Class.code column.

display.sm  24px / 32 line-height · weight 600
heading.lg  20 / 28 · 600
heading.md  18 / 26 · 600
heading.sm  16 / 24 · 600
body.lg     16 / 24 · 400
body.md     14 / 22 · 400   ← default UI text
body.sm     13 / 20 · 400
caption     12 / 18 · 400
overline    11 / 16 · 600 · uppercase · letter-spacing 0.08em
```

Use `tabular-nums` on numeric and date columns.

### Spacing

4px base unit. Use 4 / 8 / 12 / 16 / 20 / 24 / 32 / 40 / 48.

### Shape

```
radius.sm   4px    Inputs
radius.md   6px    Buttons, cards
radius.lg   8px    Modals, panels
radius.full 9999px Status pills
```

### Elevation (light mode)

```
shadow.xs   0 1px 2px 0 rgb(15 23 42 / 0.04)
shadow.sm   0 1px 3px 0 rgb(15 23 42 / 0.06), 0 1px 2px -1px rgb(15 23 42 / 0.04)
shadow.md   0 4px 8px -2px rgb(15 23 42 / 0.08), 0 2px 4px -2px rgb(15 23 42 / 0.04)
```

### Component specs that apply to this screen

**Top bar (§14.16):** 56px height, `surface.raised` bg, 1px `border.subtle` bottom. Logo left, notification bell + theme toggle + profile avatar right.

**Sidebar (§14.16):** 248px wide, `surface.raised`, 1px `border.subtle` right. Section labels in `overline · text.muted`. Nav items 40px high, 12px padding, 20px icon + body.md label, `radius.md`. Active item: `navy-50` (`#EEF2F8`) bg + `text.brand` text + 2px left-edge accent in navy. Hover: `surface.sunken` bg.

**Button (§14.1):** Heights sm=36 / md=40 / lg=48px. `radius.md` (6px). Primary navy = navy bg + white text. Primary amber = amber bg + white text. Secondary = transparent + 1px `border.default` + `text.primary`. NO hover lift — color shift only. Internal icon→label gap 8px.

**Status badge (§14.6):** pill, `radius.full`, 22px tall, 4px vertical / 10px horizontal padding. 6px leading dot in the foreground color. Label uses `overline` (11px, 600, uppercase, 0.08em letter-spacing). bg + fg from the Lifecycle family above.

**Data table (§14.12, compact):** 36px row height, cell padding 6px vertical / 12px horizontal. Header: `surface.sunken` bg, body.sm weight 600, `text.muted`, 1px `border.subtle` bottom. Row divider: 1px `border.subtle`. Row hover: `surface.sunken` background — NO shadow change. Numeric columns: tabular numerals, right-aligned. NO zebra striping. NO vertical gridlines.

**Input (§14.2):** 40px height (md), 12px horizontal padding, `body.md`, `radius.sm` (4px), 1px `border.default`. Focus: 1px `border.focus` + 2px focus ring (`border.focus` color at 2px offset).

**Select (§14.3):** based on input. Trailing chevron-down icon 16px in `text.muted`.

### Bangladesh rules

- Dates: `11 May 2026` (day-month-year, no leading zero on day). NEVER `05/11/2026` or `2026-05-11`.
- Numerals: Western Arabic (0–9) only, even in Bangla content.
- Use realistic Bangla class names mixed with English: e.g. `নবম শ্রেণি`, `Class 10`, `HSC 1st Year`. For Bangla text, apply font-family `"Hind Siliguri", "Noto Sans Bengali", sans-serif` and `lang="bn"`.

### Hard rules — do not violate

- NO gradients. NO drop-shadow on hover. NO transform / lift on hover.
- NO purple anywhere. Stay in navy + amber + neutral.
- NO emoji in UI — Lucide icons only.
- NO pure black text — use `#0F172A`. NO pure white canvas — use `#F8FAFC`.
- Status pills always `radius.full`. Buttons always `radius.md`. Inputs always `radius.sm`.

---

## SCREEN SPEC — Class list (Admin)

**Purpose:** Admin browses every class in the institution, sees status and operational health (mandatory subjects assigned, batch count), and starts create / edit / archive flows.

**Density:** compact (admin convention).
**Primary device:** desktop.
**Role visible:** Admin only.

### Layout

Standard admin shell — top bar at the top, left sidebar fixed at 248px, main content area to the right with `surface.canvas` (`#F8FAFC`) background and 32px padding.

Main content stack:

1. **Page header** — title `Classes` (`display.sm`, `text.primary`) on the left. One-line description below it: "Manage grade-level containers and their mandatory subjects." (`body.md`, `text.muted`). On the right, a single filled amber button `+ New class` (md size, with `Plus` icon leading). The whole header is on `surface.canvas` (no card wrapper).

2. **Filter bar** — wrapped in a `surface.raised` card with `radius.md`, 1px `border.subtle`, 16px padding, sitting above the table. Contains:
   - Inline search input (md size, leading `Search` icon 16px in `text.muted`, placeholder "Search by name or code"). ~320px wide.
   - Status select (md size, label "Status", default value "Active"). ~160px wide.
   - On the right: result count caption `12 classes` (`body.sm · text.muted · tabular`).

3. **Data table** — `surface.raised` card with `radius.md`, 1px `border.subtle`. Compact density.

   Columns (left to right):

   | Header | Width | Alignment | Format |
   |---|---|---|---|
   | Code | 96px | left | mono, body.sm, text.primary |
   | Name | flex | left | body.md, text.primary; Bangla text uses Bangla font |
   | Subjects | 104px (flex when warning) | right | tabular body.sm; **if count = 0**, render `0` + a 16px Lucide `AlertTriangle` icon + the inline label "No subjects". Icon and label both colored `#B91C1C` (status.cancelled foreground in light mode). 6px gap between number and icon, 4px gap between icon and label. The icon, label, AND its triangle shape are all non-color signals (don't rely on color alone). |
   | Batches | 96px | right | tabular body.sm |
   | Status | 108px | left | status pill (see badge spec) |
   | Updated | 120px | left | tabular body.sm, format `11 May 2026` |
   | (actions) | 40px | center | icon-only `MoreHorizontal` button, 32×32, ghost variant |

   Header row: `surface.sunken` bg, body.sm weight 600, `text.muted`, uppercase NOT applied. Sticky.

   Row hover: `surface.sunken` background. Row cursor: pointer. NO shadow change.

4. **Pagination** — below the table, right-aligned, 16px gap above. Caption "1–25 of 47" (`body.sm · text.muted`) on the left of the control. Previous / Next as two small icon-only ghost buttons (`ChevronLeft`, `ChevronRight`, 32×32). Disabled state on Previous when on page 1.

### Realistic data — render exactly these 8 rows

```
Code     Name                          Subjects   Batches   Status       Updated
C10      Class 10                      5          3         ACTIVE       11 May 2026
C9       Class 9                       4          2         ACTIVE       05 May 2026
HSC1     HSC 1st Year                  6          0         INACTIVE     12 Apr 2026
HSC2     HSC 2nd Year                  6          2         ACTIVE       28 Apr 2026
C8       Class 8                       4          1         ACTIVE       18 Apr 2026
C7       নবম শ্রেণি (preparatory)        3          0         INACTIVE     02 Apr 2026
SSC      SSC Crash Course              5          1         ACTIVE       09 May 2026
C6-OLD   Class 6 (2024)                4          0         ARCHIVED     14 Jan 2026
```

For row `HSC1`, the Subjects cell shows `0` + a 16px `AlertTriangle` icon + the visible label "No subjects" — icon and label both `#B91C1C`. Same for `C7` and `C6-OLD`. The `AlertTriangle` carries `aria-label="No mandatory subjects — batches under this class cannot be activated"` for screen readers. Do NOT use a hover-only tooltip — touch users can't reach it.

The Bangla row (`C7`) renders the Name cell using the Bangla font, with `lang="bn"` on the span. The English suffix "(preparatory)" stays in Inter.

Status badges in the rows take their **colors** from the lifecycle palette but their **label text** is the literal class status (never the palette key). So: `ACTIVE` renders as the label "ACTIVE" with the `published` palette (`#ECFDF5` bg, `#047857` fg, `#047857` dot). `INACTIVE` renders as "INACTIVE" with the `draft` palette (`#F1F5F9` bg, `#334155` fg, `#334155` dot). `ARCHIVED` renders as "ARCHIVED" with the `archived` palette (`#F1F5F9` bg, `#475569` fg, `#475569` dot). The label text is the only thing that distinguishes ACTIVE from INACTIVE on a row at a glance besides the bg color — don't render "PUBLISHED" or "DRAFT" anywhere.

### Sidebar — render this nav

```
SECTION: ACADEMIC
  · Classes        ← active
  · Subjects
  · Batches

SECTION: PEOPLE
  · Students
  · Teachers
  · Admins

SECTION: OPERATIONS
  · Schedule
  · Attendance
  · Assessments

SECTION: FINANCE
  · Payments
  · Payroll
```

Icons (Lucide): `GraduationCap` / `BookOpen` / `Users2` for Academic; `Users` / `UserCog` / `Shield` for People; `Calendar` / `ClipboardCheck` / `FileText` for Operations; `Wallet` / `Banknote` for Finance.

### Top bar — render this

Logo on the left: text "CC-LMS" in `text.brand`, weight 700, 18px. Right side, in order: `Bell` icon (20px, `text.secondary`) with a small `#D97706` dot for unread; `Sun` icon (20px, theme toggle); a 32px circular avatar with initials "RM" on a `#14315C` bg with white text.

### Accessibility cues to honor in the render

- Focus ring (2px `#2D5793` at 2px offset) visible on the search input — show one focused element so the reviewer can see the pattern.
- Status pills include both the dot color AND the text label (never color alone).

---

## RENDER

Produce the **populated** state of this screen on **desktop** (1440px wide reference). One artifact. Self-contained React component with all data inlined. Use Tailwind classes that resolve to the exact hex values above (e.g. `bg-[#FFFFFF]`, `text-[#0F172A]`, `border-[#E2E8F0]`). Use the Inter font (load via Google Fonts in the artifact or set `font-family` inline). Use Lucide icons via `lucide-react`.

Don't add hover transforms, lift, gradients, or animations beyond a subtle row-hover background change. Don't substitute colors. Don't add features that aren't in the spec.

When done, the artifact should look like a screenshot of a production admin tool — calm, dense, institutional.

===8<=== END COPY ===8<===

---

## Variant blocks — swap into the `RENDER` section

### Empty state (no classes exist at all)

```
RENDER

Produce the **empty** state of this screen on **desktop**. Show the page header
(with its amber `+ New class` CTA still visible), the filter bar (controls present
but disabled-looking is fine), and where the table would be: a centered empty-state
block on `surface.raised` with `radius.md`, 48px vertical padding, containing:
  · `GraduationCap` icon, 32px, `text.muted` color
  · Heading "No classes yet" in heading.sm, text.secondary
  · Body "Create your first class to start defining batches and assigning subjects." in body.sm, text.muted, max-width 360px
  · A **filled navy primary** button "+ New class" (md size). Navy, NOT amber —
    only ONE amber filled button on a screen (page header CTA already uses it).
The pagination row is hidden in this state.
```

### Loading state

```
RENDER

Produce the **loading** state of this screen on **desktop**. The top bar, sidebar,
page header, and filter bar render normally and remain interactive-looking.
The table card renders the header row normally. The table body shows 8 skeleton
rows. Each skeleton row is 36px tall and contains 7 skeleton blocks matching the
column widths above. Skeleton fill: `surface.sunken` (#F1F5F9), `radius.sm` (4px),
roughly 60% of the column width. Apply a subtle CSS pulse animation (~1.5s
ease-in-out alternate) — make it static under `prefers-reduced-motion: reduce`.
The pagination row is hidden.
```

### Error state

```
RENDER

Produce the **error** state of this screen on **desktop**. The top bar, sidebar,
page header, and filter bar render normally. Above the table card, render an
inline alert:
  · surface.raised (#FFFFFF) bg, 3px left border #B91C1C (status.cancelled foreground), radius.md
  · 16px padding
  · `AlertCircle` icon 20px, color #B91C1C, leading
  · Title "Couldn't load classes." (body.md, weight 500, text.primary)
  · Body "The server returned an error. Please try again." (body.sm, text.secondary)
  · Trailing on the right: secondary button "Retry" (sm size)
The table card below collapses to a single 96px-tall empty area with caption
"Unable to display classes" centered in text.muted. Pagination hidden.
```

### Mobile / tablet variant

```
RENDER

Produce the **populated** state of this screen on **tablet (768px wide)**. The
sidebar collapses to a 64px icon rail (icons only, active item still shows the
2px left-edge brand accent). The page header keeps the title + CTA on one line.
The filter bar wraps: search full-width on the first row, status select +
result count on the second row. The table compresses by hiding the "Updated"
column (lowest information density). All other rules unchanged.
```

---

## Iteration prompts (after the first render)

These are short follow-ups you can paste into the same Canvas conversation:

- "Make the row hover state more obvious — try `#F1F5F9` background as the spec says, with a 100ms transition."
- "The status pills look slightly too tall. Target 22px exactly, with 4px vertical padding and `overline` text (11px, 600, uppercase, 0.08em letter-spacing)."
- "Show me how the Archive confirmation modal would overlay this screen — `surface.overlay` (#FFFFFF), `radius.lg` (8px), `shadow.xl`, 520px wide, with `neutral-950 at 50% opacity` backdrop. Title 'Archive this class?', body 'Archived classes can't have new batches. Existing batches keep operating.', actions: Cancel (secondary) and Archive (destructive — transparent + 1px #EF4444 border + #B91C1C text)."
- "Now show the row's ⋯ menu open. Menu surface `#FFFFFF`, `radius.md`, `shadow.md`, anchored under the ⋯ button. Items: Edit, Deactivate, Archive — each row 36px, 12px horizontal padding, body.md, hover `surface.sunken`."

---

## Validation checklist — eyeball before saving the screenshot

- [ ] Status badges use **lifecycle family** colors (published / draft / archived), not random green/grey
- [ ] One filled amber button on the page (`+ New class`). No second amber button anywhere
- [ ] Class.code column is in `JetBrains Mono` or another mono fallback
- [ ] Numeric columns are right-aligned with tabular numerals
- [ ] Dates render as `11 May 2026` — not `05/11/2026`, not `2026-05-11`
- [ ] Bangla row's name uses a Bangla font, not Inter
- [ ] No hover lift, no transform, no shadow change on row hover
- [ ] Focus ring is 2px `#2D5793` with 2px offset, visible on the focused input
- [ ] No purple, no gradient, no emoji, no pure black, no pure white canvas
