# CC-LMS Design System

A design system for a coaching center management platform serving the Bangladesh market. Built for Admin, Teacher, Student, and Accountant roles across mobile and desktop. Institutional, trustworthy, bilingual-ready.

> **Revision note:** This version keeps the navy + amber brand identity and the full original depth. Section 14 has been upgraded from conceptual pattern guidance into a concrete, pixel-level component specification so the system is directly implementable without engineers guessing values. A shared interaction-state model (§14.0) was added so every component references one source of truth for rest / hover / focus / active / disabled behavior.

---

## 1. Product context

**What this product is**
A multi-role LMS for coaching centers covering admissions, scheduling, attendance, assessments, live classes, payments, payroll, accounting, notices, and communication. 15 modules. 4 primary user roles.

**Who uses it**
- **Admin** — institution control room. Approvals, scheduling, oversight, exception handling. Mostly desktop.
- **Teacher** — daily teaching operations. Classes, attendance, evaluation, live sessions. Mobile-heavy.
- **Student** — academic consumption. Joining classes, submitting assessments, viewing results. Mostly mobile, often low-end Android.
- **Accountant** — financial operations. Collections, payroll, ledger, reports. Mostly desktop.

**Where users are**
Bangladesh. Phone-first culture. Manual cash/MFS (mobile financial services) payments are common, not edge cases. SMS is a primary communication channel. Bilingual: English + Bangla (Bengali). Network is often weak; graceful degradation matters.

**Brand personality**
Trustworthy and institutional. Bank-grade calm. Authority without coldness. Confidence without consumer flashiness. Parents handing over school fees should feel the same trust as a bank app — not the energy of a social platform.

**What the product is NOT**
- Not playful or youthful — students are end users but parents and admins are decision-makers
- Not minimalist-to-the-point-of-sparse — administrative density is required
- Not consumer-SaaS-modern — the institutional weight is the point
- Not flat-pastel — that aesthetic reads as casual / unserious in this market

---

## 2. Brand colors

### Primary: Navy `#14315C`
Authority, trust, institutional weight. The color of bank apps, university crests, government portals. Carries calm seriousness without being cold or corporate-generic.

### Accent: Amber `#D97706`
Warmth, action, optimism. Pairs with navy without competing. Warm earth tone that resonates with the South Asian visual context. Reserved for calls-to-action, highlights, and the partial-payment status.

### Why not other directions
- **Not Indigo + Teal** — too generic SaaS; teal sits visually close to success-green and creates ambiguity in dense status-rich screens
- **Not Royal Blue + Coral** — coral sits next to danger-red and risks semantic collision
- **Not Forest Green + Gold** — green as primary conflicts with success states
- **Not single-brand monochrome** — the product has too many states; a second color is needed for hierarchy

### Brand usage rules
- **Navy is the workhorse.** Use for primary buttons, headers, key navigation, primary brand surfaces.
- **Amber is the seasoning.** Reserve for: primary CTAs that need extra emphasis, the "Partial" payment status, brand highlights, key visual accents. If amber appears more than once or twice per screen, you're overusing it — pull back.
- **Never use amber for warning states.** Warning is a separate semantic color (see Section 4). They look similar but carry different meanings.

---

## 3. Color ramps (primitives)

### Navy ramp
```
navy-50   #EEF2F8
navy-100  #D4DDEC
navy-200  #A8BBD9
navy-300  #7C99C6
navy-400  #4F78B3
navy-500  #2D5793
navy-600  #1E4478
navy-700  #14315C  ← Brand primary
navy-800  #0E2447
navy-900  #0A1A35
navy-950  #050E20
```

### Amber ramp
```
amber-50   #FDF6EB
amber-100  #FAE6C3
amber-200  #F5CD8B
amber-300  #EFB252
amber-400  #E2941F
amber-500  #D97706  ← Brand accent
amber-600  #B45F05
amber-700  #8B4906
amber-800  #663608
amber-900  #4A2706
amber-950  #2E1804
```

### Neutral (Slate) ramp
```
neutral-0     #FFFFFF
neutral-50    #F8FAFC
neutral-100   #F1F5F9
neutral-200   #E2E8F0
neutral-300   #CBD5E1
neutral-400   #94A3B8
neutral-500   #64748B
neutral-600   #475569
neutral-700   #334155
neutral-800   #1E293B
neutral-900   #0F172A
neutral-950   #020617
neutral-1000  #000000
```

### Semantic primitives
```
success-50  #ECFDF5   success-500  #10B981   success-700  #047857   success-900  #064E3B
warning-50  #FFFBEB   warning-500  #F59E0B   warning-700  #B45309   warning-900  #78350F
danger-50   #FEF2F2   danger-500   #EF4444   danger-700   #B91C1C   danger-900   #7F1D1D
info-50     #EFF6FF   info-500     #3B82F6   info-700     #1D4ED8   info-900     #1E3A8A
```

---

## 4. Semantic tokens

Components consume these, **never** primitives directly. Semantic tokens swap between light and dark themes automatically.

### Surface (backgrounds)
```
                      LIGHT       DARK
surface.canvas        #F8FAFC     #020617    App background
surface.raised        #FFFFFF     #0F172A    Cards, panels
surface.overlay       #FFFFFF     #1E293B    Modals, popovers
surface.sunken        #F1F5F9     #010410    Input wells, sub-areas
surface.inverse       #0F172A     #F8FAFC    Tooltips, inverted contexts
```

### Text
```
                      LIGHT       DARK
text.primary          #0F172A     #F8FAFC    Body and headings
text.secondary        #334155     #CBD5E1    Supporting copy
text.muted            #64748B     #94A3B8    Labels, captions, timestamps
text.disabled         #94A3B8     #475569    Inactive
text.inverse          #FFFFFF     #0F172A    Text on dark/colored backgrounds
text.link             #1E4478     #7C99C6    Inline links
text.brand            #14315C     #A8BBD9    Brand-emphasized text
```

### Border
```
                      LIGHT       DARK
border.subtle         #E2E8F0     #1E293B    Dividers, hairlines
border.default        #CBD5E1     #334155    Inputs, cards
border.strong         #94A3B8     #475569    Hover, emphasis
border.brand          #1E4478     #4F78B3    Selected, brand-tinted
border.focus          #2D5793     #7C99C6    Focus ring
```

### Interactive
```
                              LIGHT       DARK
interactive.primary.rest      #14315C     #2D5793
interactive.primary.hover     #0E2447     #4F78B3
interactive.primary.active    #0A1A35     #7C99C6
interactive.primary.disabled  #E2E8F0     #1E293B
interactive.accent.rest       #D97706     #E2941F
interactive.accent.hover      #B45F05     #EFB252
interactive.accent.active     #8B4906     #F5CD8B
```

---

## 5. Status palette

**This is the most product-specific layer.** The product has many status families and they must never visually collide. Each status is a paired bg + fg.

### Why grouped by domain
A naive system reuses 6 status colors everywhere. But a "Pending" admission and an "Overdue" payment and a "Late" attendance are not the same kind of warning — they belong to different mental models. Grouping by domain prevents drift and lets designers physically pick from the right set.

### Lifecycle family (notices, schedules, assessments)
```
                  LIGHT BG    LIGHT FG    DARK BG     DARK FG
draft             #F1F5F9     #334155     #1E293B     #CBD5E1
scheduled         #EFF6FF     #1D4ED8     #1E3A8A     #93BBFD
published         #ECFDF5     #047857     #064E3B     #6EE7B7
expired           #F1F5F9     #64748B     #1E293B     #64748B
archived          #F1F5F9     #475569     #1E293B     #94A3B8
cancelled         #FEF2F2     #B91C1C     #7F1D1D     #FCA5A5
```

### Approval family (admissions, payment verification)
```
pending           #FFFBEB     #B45309     #78350F     #FCD34D
approved          #ECFDF5     #047857     #064E3B     #6EE7B7
rejected          #FEF2F2     #B91C1C     #7F1D1D     #FCA5A5
```

### Attendance family
```
present           #ECFDF5     #047857     #064E3B     #6EE7B7
absent            #FEF2F2     #B91C1C     #7F1D1D     #FCA5A5
late              #FFFBEB     #B45309     #78350F     #FCD34D
excused           #EFF6FF     #1D4ED8     #1E3A8A     #93BBFD
```

### Payment family
```
paid              #ECFDF5     #047857     #064E3B     #6EE7B7
partial           #FDF6EB     #8B4906     #4A2706     #F5CD8B   ← uses amber, intentional
due               #FFFBEB     #B45309     #78350F     #FCD34D
overdue           #FEF2F2     #B91C1C     #7F1D1D     #FCA5A5
```

### Override markers (schedule occurrences)
Schedule occurrences can be in states like NORMAL, UPDATED, OVERRIDDEN, CANCELLED. Use the lifecycle family for these: NORMAL = no badge, UPDATED = scheduled, OVERRIDDEN = scheduled with subtle marker, CANCELLED = cancelled.

### Status rendering rules
- **Always pair bg + fg.** Never invent a foreground color for a status background.
- **Default rendering: pill (rounded-full)** with leading dot. 11px text, semibold, uppercase, 0.08em letter-spacing.
- **Status colors meet WCAG AA contrast** on their paired surface in both themes. Don't substitute.
- **Don't use status colors for non-status purposes.** A success-green background on a card is wrong — that's `surface.raised`, not `status.published.bg`.

---

## 6. Typography

### Font stack
```
font.sans   Inter             — Latin / English UI
font.bn     Hind Siliguri     — Bangla (Bengali script)
font.mono   JetBrains Mono    — Transaction IDs, codes, technical strings
```

### Why this pairing
- **Inter** — strong institutional choice, excellent legibility at small sizes, supports tabular numerals (critical for finance modules)
- **Hind Siliguri** — pairs visually with Inter (similar x-height ratio, geometric construction), excellent Bangla rendering, free
- **JetBrains Mono** — clear distinction between similar characters (0 vs O, 1 vs l), institutional feel

### Type scale
Format: `name — size / line-height · weight · notes`

```
display.lg    32 / 40  · 600 · letter-spacing -0.02em
display.md    28 / 36  · 600 · letter-spacing -0.02em
display.sm    24 / 32  · 600 · letter-spacing -0.015em
heading.lg    20 / 28  · 600 · letter-spacing -0.01em
heading.md    18 / 26  · 600
heading.sm    16 / 24  · 600
body.lg       16 / 24  · 400
body.md       14 / 22  · 400  ← default UI text
body.sm       13 / 20  · 400
caption       12 / 18  · 400
overline      11 / 16  · 600 · uppercase, letter-spacing 0.08em
```

### Bilingual line-height rule
Line-heights are calibrated for the **taller** of the two scripts. Bangla glyphs have taller x-height and longer descenders than Latin. A line-height that works for Inter (~1.45) will look cramped for Bangla (~1.7 needed). The scale above uses ~1.6 as the shared safe value, which gives Latin breathing room and accommodates Bangla without re-spacing.

**Practical rule:** never set line-height per-language. The unified value above works for both. If you find yourself wanting tighter line-height for English, you'll break Bangla.

### Tabular numerals
Every type token has a `.tabular` variant using `font-variant-numeric: tabular-nums`. **Always use tabular for:**
- Finance tables (Payment, Payroll, Accounting modules)
- Date/time columns
- Phone numbers in lists
- Any numeric column where alignment matters

Don't use tabular for prose with numbers inline ("3 students attended" reads better with proportional).

### Weight system
Only four weights: 400 / 500 / 600 / 700. Resist adding more.
- 400 — body text
- 500 — emphasized inline text, button labels
- 600 — headings, status pills, table headers
- 700 — rare, reserved for KPI numbers and display sizes

---

## 7. Spacing

### Base unit: 4px
Everything is a multiple of 4. The scale:
```
0, 2, 4, 6, 8, 10, 12, 14, 16, 20, 24, 28, 32, 40, 48, 56, 64, 80, 96, 112, 128
```

### Dual-density rule
Same spacing scale, two density modes. Same screen can mix densities (a compact table inside a comfortable card).

**Comfortable** (default — student/teacher mobile, dashboards)
- Row height: 48px, vertical padding 12px, gutter 16px
- All cards, planners, dashboard widgets

**Compact** (admin tables, accountant queues, dense data)
- Row height: 36px, vertical padding 6px, gutter 12px
- Data tables, queue rows, log views, admin lists

### Touch targets
- **Minimum 44 × 44px** for any tappable element (Apple HIG / WCAG 2.5.5)
- Desktop dense tables allowed minimum: 32 × 32px (mouse precision context)
- Critical for low-end Android — student/teacher demographic

### Why dual density
Admin/accountant surfaces need to display many rows on a single screen — compact is necessary for productivity. Student/teacher surfaces are touch-driven on mobile — comfortable is necessary for accessibility. Trying to use one density everywhere fails both.

---

## 8. Shape

### Radius scale
```
radius.none   0
radius.sm     4px    Inputs, small chips
radius.md     6px    Buttons, cards (default)
radius.lg     8px    Modals, panels
radius.xl     12px   Hero cards, large containers
radius.2xl    16px   Rare, dashboard hero widgets
radius.full   9999px Pills, avatars, status badges
```

### Border widths
- Default: 1px
- Emphasis (selected, focused, brand-tinted): 1.5px
- Avoid 2px+ for structural/decorative borders — looks heavy and dated for institutional feel
- **Exception:** the keyboard focus *ring* is 2px (it is an offset outline, not a structural border — see §14.0)

### Rule
Status pills are always `radius.full`. Buttons and cards are `radius.md`. Inputs are `radius.sm`. Modals are `radius.lg`. Avatars are `radius.full`. Don't deviate without a specific reason.

---

## 9. Elevation

### Shadows (light mode primary, dark mode uses surface lightening instead)
```
shadow.none   none
shadow.xs     0 1px 2px 0 rgb(15 23 42 / 0.04)
shadow.sm     0 1px 3px 0 rgb(15 23 42 / 0.06), 0 1px 2px -1px rgb(15 23 42 / 0.04)
shadow.md     0 4px 8px -2px rgb(15 23 42 / 0.08), 0 2px 4px -2px rgb(15 23 42 / 0.04)
shadow.lg     0 12px 20px -4px rgb(15 23 42 / 0.10), 0 4px 8px -4px rgb(15 23 42 / 0.06)
shadow.xl     0 24px 32px -8px rgb(15 23 42 / 0.14), 0 8px 16px -6px rgb(15 23 42 / 0.06)
```

### Dark mode elevation rule
**Don't try to make shadows visible on dark backgrounds.** They'll either look fake (too dark) or wash out (too light). Instead, elevation in dark mode happens through *surface lightening*:
- `surface.canvas` (darkest) → `surface.raised` (lighter) → `surface.overlay` (lightest)
- This mirrors physical-world depth perception
- It's the opposite of light mode (where raised surfaces are *lighter* than canvas)

### Z-index layers (named, numbered)
```
z.base       0
z.dropdown   1000
z.sticky     1100     Sticky table headers
z.fixed      1200     Fixed navigation
z.backdrop   1300
z.modal      1400
z.popover    1500
z.toast      1600
z.tooltip    1700     Always on top
```

---

## 10. Motion

Kept deliberately minimal — institutional products lose trust when they feel "bouncy."

### Durations
```
motion.instant    0ms
motion.fast       150ms   Hover, focus, simple state changes
motion.base       200ms   Most transitions (default)
motion.slow       300ms   Modals, sheets, drawers
motion.slower     500ms   Full-page transitions (rare)
```

### Easings
```
motion.standard   cubic-bezier(0.2, 0, 0, 1)     Default
motion.entrance   cubic-bezier(0, 0, 0, 1)       Elements coming in
motion.exit       cubic-bezier(0.4, 0, 1, 1)     Elements leaving
motion.emphasis   cubic-bezier(0.2, 0, 0, 1.4)   Rare, celebratory moments
```

### Reduced motion
All motion respects `prefers-reduced-motion: reduce` and falls back to instant. This is non-negotiable for accessibility.

### Rule
Don't bounce. Don't spring. Don't elastic. Don't celebrate every interaction. The product handles money and children's education — gravity matters more than delight.

**Specifically: components do not move on hover.** Hover communicates through color and background change, never through transform/lift. (This is the deliberate institutional adaptation — a consumer "card lifts on hover" pattern is rejected here. See §14.)

---

## 11. Iconography

- **Library:** Lucide icons (consistent stroke, large coverage, pairs with Inter)
- **Sizes:** 16 / 20 / 24 / 32 (always even, always 4px-aligned)
- **Stroke weight:** 1.5px (1px reads weak on Android, 2px reads heavy on desktop)
- **Color:** icons inherit `currentColor` by default; semantic tokens otherwise
- **Tap target:** even when icon is 20px, surrounding tap area is min 44px

---

## 12. Bilingual rules

### English-first, Bangla-ready
The product launches with English primary, Bangla parity coming after. The token system already supports both — no rework needed when Bangla is enabled.

### Text rendering rules
- **Language detection per content block, not per page.** A page may have English navigation, Bangla student name, English subject, Bangla notice content. Each block uses the right font.
- **Use `font.bn` for any Bangla text.** Don't rely on font fallback chains — they produce inconsistent rendering.
- **Mixed-script text in one block** (English label : Bangla value) — use `font.sans` as the container default and wrap Bangla sub-strings in `font.bn`.

### Layout rules
- **Don't size containers based on English string length.** Bangla strings can be 20-40% longer for the same meaning. Always allow text to wrap.
- **Buttons should not have fixed widths** — let Bangla labels breathe.
- **Tables must have flexible column widths** for name/address/notice fields.

### Number rendering
- **Always use Western Arabic numerals** (0-9), not Bengali numerals (০-৯), even in Bangla content. This is the LMS convention in Bangladesh — schools, banks, and government forms all use Western numerals. Bengali numerals would feel anachronistic.
- **Currency:** ৳ (Taka symbol) precedes the amount with one space. Example: `৳ 3,500.00`.
- **Phone numbers:** `+880` country code prefix, then 10 digits. Example: `+880 1712 345 678`.

---

## 13. Role-based UI orientation

The same component renders differently for different roles. The design system supports this through *patterns*, not separate components.

### Admin
- **Mood:** institution control room
- **Density:** compact (tables, queues, exception lists)
- **Priority:** pending approvals → exceptions → operational summaries
- **Surface:** desktop-first, full-width containers

### Teacher
- **Mood:** today's operations center
- **Density:** comfortable on mobile, compact on desktop
- **Priority:** today's classes → pending tasks → upcoming work
- **Surface:** mobile-first, occurrence-centric cards

### Student
- **Mood:** academic planner + urgent actions
- **Density:** comfortable everywhere
- **Priority:** today → urgent (live class, due exam, overdue payment) → upcoming → history
- **Surface:** mobile-first, simple, single-column

### Accountant
- **Mood:** daily finance operations board
- **Density:** compact (queue-first design)
- **Priority:** verification queue → collection queue → payroll → reports
- **Surface:** desktop-first, table-heavy

---

## 14. Component specifications

The previous version of this section described patterns conceptually. This version specifies them — exact sizes, padding, radii, and interaction states — so a component can be built without guessing. Every value below resolves to a token from §3–§10; nothing here introduces a new primitive.

**How to read this section.** Each component lists *anatomy* (its parts), *dimensions* (exact px, all 4px-grid aligned), *states* (referencing §14.0), and *rules* (the do/don't that keep it from drifting). Where a component has a comfortable and a compact form, both are given.

### 14.0 Interaction states — the shared model

Every interactive component supports the same five states. Components reference this model rather than redefining it.

```
rest         Default. Token group at .rest.
hover        Pointer devices only. Background/color shifts one step
             (e.g. interactive.primary.rest → .hover). NO transform,
             NO lift, NO shadow change. Transition: motion.fast.
focus-visible 2px outline in border.focus, 2px offset, on keyboard
             focus. Always visible. Never removed without replacement.
active       Pressed. One step darker than hover (.active token).
disabled     interactive.*.disabled background, text.disabled label,
             cursor: not-allowed, no hover/active response, aria-disabled.
```

Two optional states, used only where the component supports them:
```
selected/checked   Brand-tinted: border.brand, navy-50 (light) / navy-800
                   (dark) fill, or interactive.primary fill for controls.
loading            Inline 16px spinner replaces the label, or skeleton
                   replaces content. Component stays its rest size.
```

**Rules**
- All state transitions use `motion.fast` (150ms) with `motion.standard` easing, and collapse to instant under `prefers-reduced-motion`.
- The focus ring is 2px — this is the one sanctioned exception to the "avoid 2px borders" rule (§8), because it is an offset outline, not a structural border.
- Hover never moves an element. This is the deliberate institutional adaptation of the common consumer "lift on hover" pattern.

### 14.1 Buttons

**Anatomy:** optional leading icon · label · optional trailing icon. Label is always present except for the icon-only variant.

**Sizes** — `height / horizontal padding / label token / icon size`
```
sm   36px / 12px / body.sm  weight 500 / 16px   Compact contexts (admin, accountant, table row actions)
md   40px / 16px / body.md  weight 500 / 16px   Default
lg   48px / 20px / body.md  weight 500 / 20px   Primary CTA on mobile; matches the comfortable row height
```
Radius: `radius.md` (6px). Icon-only buttons are square: 36 / 40 / 48px. Internal gap between icon and label: 8px.

**Variants**
```
Primary (navy)    bg interactive.primary.*  ·  text.inverse        The default action.
Primary (amber)   bg interactive.accent.*   ·  text.inverse        The single most important CTA on a screen.
Secondary         transparent · 1px border.default · text.primary  Hover: surface.sunken bg + border.strong.
Ghost             transparent · no border · text.brand             Hover: surface.sunken bg.
Destructive       transparent · 1px danger-500 · danger-700 text   Hover: danger-50 bg. Filled danger-500 only inside a confirm modal.
```

**States:** per §14.0. Disabled primary uses `interactive.primary.disabled` bg with `text.disabled`.

**Rules**
- Do place at most **one filled button** (navy or amber) per view section — it is the primary path.
- Do reserve the amber variant for the highest-value action; **never two amber buttons on one screen** (echoes §2).
- Do make primary form-submit buttons **full-width on mobile**.
- Don't add a hover lift or shadow — color shift only (§10, §14.0).
- Don't give buttons fixed pixel widths — Bangla labels must be free to expand (§12).
- Icon-only buttons require an `aria-label` and a tooltip (§14.15).

### 14.2 Text input & textarea

**Anatomy:** label (above) · field · optional helper or error text (below). Optional leading/trailing icon or affix inside the field.

**Dimensions**
```
height        sm 36px · md 40px (default) · lg 48px
horizontal padding  12px
font          body.md (14px); font.mono for code/transaction-ID fields
radius        radius.sm (4px)   ← inputs are 4px, distinct from buttons at 6px
label         body.sm weight 500 · text.secondary · 6px gap above field
helper text   caption (12px) · text.muted · 4px gap below field
error text    caption (12px) · danger-700
textarea      same padding · min-height 80px · vertical resize only
```

**States**
```
rest      1px border.default
hover     1px border.strong
focus     1px border.focus + 2px focus ring (§14.0)
filled    1px border.default (has a value)
error     1px danger-500 + error text shown
disabled  surface.sunken bg · text.disabled · border.subtle
```
Placeholder text uses `text.muted`. (See §15 for the phone-input affix pattern.)

**Rules**
- Do keep the label visible above the field — placeholder text is not a substitute for a label.
- Do show error text as well as the red border (color is never the only signal — §18).
- Don't use a fixed-width field where a Bangla value may appear (name, address).

### 14.3 Select / dropdown

**Anatomy:** input field (per §14.2) + trailing chevron-down icon 16px `text.muted`. Opens a menu.

**Menu:** `surface.overlay` · `radius.md` · `shadow.md` · layer `z.dropdown`. Option rows 36px high, 12px horizontal padding, `body.md` label. Option hover: `surface.sunken`. Selected option: `text.brand` + trailing check icon 16px. Max menu height ~280px, then scroll.

**Rules**
- Do support type-to-filter for any select with more than ~8 options.
- Do use a searchable variant for batch / class / subject pickers (these grow over time).
- Don't nest selects more than one level — use a dedicated picker screen instead.

### 14.4 Checkbox, radio & toggle

These are three different controls. Don't substitute one for another.

**Checkbox** — multi-select, table-row selection, "Read & Acknowledged".
```
box        18 × 18px · radius.sm · 1.5px border.default (unchecked)
checked    interactive.primary.rest fill + 14px white check (Lucide)
indeterminate  filled + white dash
label      body.md · 8px gap · tap area padded to 44px on mobile
```

**Radio** — one-of-many.
```
ring       18 × 18px · circle · 1.5px border.default
checked    1.5px border.brand + 8px inner dot interactive.primary.rest
```

**Toggle / switch** — a setting that takes effect immediately (notification preferences, feature on/off).
```
track  36 × 20px · radius.full
knob   16px circle · 2px inset · white · shadow.xs · slides on motion.fast
off    track neutral-300 (light) / neutral-600 (dark)
on     track interactive.primary.rest
```

**Rules**
- Do use a **checkbox** in forms and tables, a **toggle** only for immediately-applied settings.
- Don't render a toggle as a circle the way a generic "preferences switch" might — a circular toggle reads as a checkbox here. Keep the pill track.
- For the attendance Present/Absent/Late/Excused control, use the segmented control (§14.5), not a toggle.

### 14.5 Tabs & segmented control

**Underline tabs** — top-level grouping (Today / Upcoming / Previous; Day / Month; Draft / Published).
```
tab height    40px
label         body.md weight 500 · 16px horizontal padding
active        text.brand + 2px bottom bar in interactive.primary.rest
inactive      text.muted · hover text.secondary
baseline      1px border.subtle hairline runs full width under the tab row
count badge   optional pill on a tab: neutral-100 bg · text.secondary · 18px tall · caption text
```

**Segmented control** — a compact set of mutually exclusive choices in a tight space.
```
container     surface.sunken · radius.md
segment       equal width · height 32px (compact) / 36px (comfortable)
selected      surface.raised fill + text.primary + shadow.xs
unselected    text.muted
```
For attendance status entry, the segments adopt the **attendance status palette** (§5) so Present/Absent/Late/Excused are color-coded as the user toggles.

**Rules**
- Do use underline tabs for navigation between views, segmented controls for a setting within a view.
- Don't exceed ~5 tabs — beyond that, use a select or a side menu.

### 14.6 Status badge

**Anatomy:** pill · leading dot · label.
```
shape    radius.full
height   22px
padding  4px vertical · 10px horizontal
dot      6px circle, fill = the status foreground color
label    overline (11px · 600 · uppercase · 0.08em letter-spacing)
color    bg + fg from the correct §5 status family — always the paired set
```

**Rules**
- Do pick from the domain family that matches the data (approval / attendance / payment / lifecycle).
- Do limit to **one badge per table row, max two per card**.
- Don't make a badge clickable — it is a label, not a control. If an action is needed, use a button.
- Don't invent a foreground color (§5).

### 14.7 KPI card

**Anatomy:** small label · large value · optional trend indicator.
```
surface     surface.raised (never surface.canvas)
radius      radius.lg (8px)
padding     24px
min-height  120px
elevation   shadow.xs (light) / surface lightening (dark)
label       overline (11px) · text.muted
value       display.md (28px) weight 700 · text.primary · tabular if numeric
trend       caption (12px) + 16px caret icon
            up = success-700 · down = danger-700 · flat = text.muted
```
Density is always **comfortable**, even on admin surfaces — KPIs are scanned, not parsed.

**Rules**
- Do show a sign-correct trend (a rising due-amount is *bad* — use danger, not success, when "up" is negative for the user).
- Don't place a status-palette background on the card; the card surface is `surface.raised`.

### 14.8 Queue row

For admissions, payment verification, and evaluation queues. **Compact** density.
```
height        36px
padding       6px vertical · 12px horizontal · gutter 12px
anatomy       28px avatar/initials → primary identifier (body.md 500)
              + sub-label (caption · text.muted) → status badge
              → metadata (body.sm · tabular) → action buttons (sm / icon)
divider       1px border.subtle between rows
hover         surface.sunken background (no shadow change)
selected      2px border.brand left-edge accent · background unchanged
```
The whole row deep-links to the source workflow.

**Rules**
- Do keep one status badge per row (§14.6).
- Do right-align numeric metadata and use tabular numerals.
- Don't put more than ~2 action buttons inline — overflow into a "more" menu.

### 14.9 Class occurrence card

A single scheduled class, in teacher and student workspaces. **Comfortable** density.
```
surface     surface.raised · 1px border.subtle · radius.md
padding     16px
primary triad   subject (heading.sm) · batch (body.sm · text.secondary) · time (body.sm · tabular)
delivery mode   small label badge (ON_SITE / LIVE / HYBRID) — neutral palette, not status palette
status badge    top-right, only for non-normal occurrences (UPDATED / OVERRIDDEN / CANCELLED), lifecycle family
action row      bottom, 1–3 sm buttons (Open / Join / View), 12px gap above
```
A **cancelled** card dims its content to `text.muted` but keeps the cancelled badge legible.

**Rules**
- Do use the delivery-mode label in the neutral palette — it is a category, not a status.
- Don't hide a cancelled class — keep it visible with its badge (per SRS visibility rules).

### 14.10 Planner widget

A grouped list of time-oriented items (today's classes, upcoming exams, payment queue).
```
section heading   heading.sm + count badge
items             stacked rows or compact cards · 8px gap
empty state       mandatory (§14.18) — never a blank container
overflow          "View all" link (body.sm · text.link) when the list exceeds ~5 items
```
Each row deep-links to its source module.

### 14.11 Action bar

A row of role-aware quick actions.
```
container   horizontal scroll on mobile · wrap on desktop · 8px gap
action      min 44 × 44px tap target · 20px icon + label (body.sm · 500)
            · 8px vertical / 12px horizontal padding · radius.md
highlighted action   when an action is timely (live class joinable,
            exam window open, payment overdue): amber-50 bg + amber-700
            text + 1.5px amber border, or filled interactive.accent for
            the strongest emphasis
```

**Rules**
- Do hide the bar entirely when no actions exist — never render an empty bar (§ Dashboard rules).
- Do highlight at most one action at a time.
- Don't show an action for a workflow that does not exist in the current build.

### 14.12 Data table

The workhorse of admin and accountant surfaces. **Compact** default; a **comfortable** variant exists for lighter contexts.
```
row height    compact 36px (cell padding 6 × 12) · comfortable 48px (cell padding 12 × 16)
header        surface.sunken bg · body.sm weight 600 · text.muted
              · sticky (z.sticky) · 1px border.subtle bottom
row divider   1px border.subtle  (no zebra striping, no vertical gridlines)
hover         surface.sunken row background
selected      2px border.brand left-edge accent
numeric cols  tabular numerals · right-aligned
status col    one status pill per row (§14.6)
```
Column widths are flexible for name / address / notice / Bangla fields (§12). Empty and loading states are required (§14.18, §14.19). Long tables paginate or infinite-scroll.

**Rules**
- Do keep separation with dividers, not fills — zebra striping fights the status palette.
- Do freeze the header on scroll for any table taller than the viewport.
- Don't right-align text columns; don't left-align numeric columns.

### 14.13 Modal / dialog & bottom sheet

**Desktop dialog**
```
surface     surface.overlay · radius.lg · shadow.xl · layer z.modal
backdrop     neutral-950 at ~50% opacity · layer z.backdrop
width        sm 400px · md 520px · lg 720px · max 92vw
padding      24px
header       heading.md + close icon-button (20px icon, top-right)
body         body.md · scrolls internally if tall
footer       right-aligned action row · 8px gap · secondary then primary
             (primary rightmost) · destructive confirm uses filled danger-500
motion       enter 300ms motion.entrance (fade + 4px rise) · exit 200ms motion.exit
```

**Mobile:** the same dialog renders as a **bottom sheet** — full width, `radius.lg` on the top corners only, slides up from the bottom. Optional drag handle.

**Rules**
- Do keep the primary action rightmost (desktop) / bottom (mobile sheet).
- Don't stack modals — resolve one before opening another.

### 14.14 Toast

Transient feedback.
```
surface      surface.inverse, or surface.raised with a 3px colored left
             border indicating severity
radius       radius.md · shadow.lg · layer z.toast
width        max 400px · padding 12 × 16
content      leading status icon 20px · title (body.md · 500) · optional body.sm line
dismiss      auto after 5s · CRITICAL severity is manual-dismiss only
motion       slide + fade on motion.base
stacking     bottom (mobile) / bottom-right (desktop) · 8px gap
```

**Rules**
- Do reserve toasts for confirmations and recoverable errors; use an inline alert for anything the user must act on.
- Don't put a critical, must-act message in an auto-dismissing toast.

### 14.15 Tooltip

A real component — **do not** rely on the native `title` attribute (inconsistent rendering, and unreachable on touch devices).
```
surface   surface.inverse bg · text.inverse
text      caption (12px)
shape     radius.sm · padding 6 × 8 · shadow.sm
layer     z.tooltip (always on top)
width     max 240px
timing    300ms appear delay · instant dismiss · motion.fast fade
```

**Rules**
- Do provide a tooltip for every icon-only button (§14.1).
- Don't put essential information *only* in a tooltip — it is unreachable on touch.

### 14.16 Navigation

**Top bar** (all roles)
```
height    56px · surface.raised · 1px border.subtle bottom · layer z.fixed
left      brand / logo
center    primary nav links on desktop — body.md 500 · 40px tap height
          · active = text.brand + indicator
right     notification bell (with unread count badge) · theme toggle
          · profile avatar dropdown (32px avatar · radius.full)
```

**Sidebar** (admin / accountant, desktop)
```
width        248px · surface.raised · 1px border.subtle right
              · collapsible to a 64px icon rail
section label  overline · text.muted
nav item     40px height · 12px horizontal padding · 20px icon + body.md label · radius.md
              rest text.secondary · hover surface.sunken
              active = navy-50 (light) / navy-800 (dark) bg + text.brand
                       + 2px left-edge accent in interactive.primary
```

**Mobile:** hamburger → drawer (slides from the left, `surface.raised`, `z.modal`, `motion.slow`), or a bottom tab bar (max 5 items, 56px height, 24px icon + caption label, active `text.brand`).

### 14.17 Search

**Inline search field** — the primary search pattern.
```
based on the text input (§14.2), sm or md size
leading   magnifier icon 16px · text.muted
trailing  clear (×) icon, shown only when the field has a value
behavior  debounced input · partial match (required by the User Management module)
```

**Command palette** — *optional* enhancement for admin power users.
```
trigger   ⌘K / Ctrl+K
surface   centered overlay modal · search input + grouped, keyboard-navigable results
scope     optional — not required for student / teacher surfaces
```

**Rules**
- Do make the inline search field the default; treat the command palette as additive, never the only way to search.

### 14.18 Empty states

Never a blank container — always explain why it is empty and what to do next.
```
layout    centered · 48px vertical padding
icon      32px · text.muted
heading   heading.sm · text.secondary
body      body.sm · text.muted — explains why it is empty
action    optional single button
```

### 14.19 Loading & skeleton states

```
skeleton   blocks that match the final layout's shape and size
fill       surface.sunken
radius     matches the eventual content
animation  ~1.5s ease-in-out pulse · static under prefers-reduced-motion
spinner    inline 16px only — for button-loading and small inline waits
```

**Rules**
- Do prefer skeletons over spinners for content areas — they reduce layout shift.
- Do load dashboard zones independently (per the Dashboard module) so one slow source does not block the page.

---

## 15. Common Bangladesh-context patterns

### Phone input
- `+880` prefix is fixed and visually separated (sunken or muted)
- Input accepts 10 digits, formats automatically (`1712 345 678`)
- Validation: must start with `1` (mobile prefix in BD)

### Currency display
- BDT default. `৳` symbol with single space before amount.
- Tabular numerals always.
- Negative values: `-৳ 500.00`, never `৳ -500.00`
- Zero amounts: `৳ 0.00`, never blank or em-dash

### Payment method indicators
The product handles five payment methods (Cash, MFS, POS, Online, Bank). Each gets a small label badge — neutral palette, not status palette:
- Cash → neutral-100 bg, neutral-700 fg
- MFS → amber-50 bg, amber-700 fg  (most common, slight accent)
- POS → neutral-100 bg, neutral-700 fg
- Online → info-50 bg, info-700 fg
- Bank → neutral-100 bg, neutral-700 fg

### Date format
- Display: `11 May 2026` (day-month-year, no leading zero on day)
- Compact: `11 May` (when year is contextually obvious)
- With time: `11 May 2026, 14:30` (24-hour, comma separator)
- Never `05/11/2026` (US format causes confusion) or `2026-05-11` (ISO is for APIs, not UI)

---

## 16. What this design system rejects

Decisions made deliberately, recorded so they don't drift. Each is paired with the practice to follow instead.

| Don't | Do instead |
|---|---|
| No gradients — they read as marketing/consumer | Flat fills; institutional weight comes from color and type, not effects |
| No drop shadows in dark mode | Use surface lightening for elevation (§9) |
| No purple anywhere | Stay in the navy + amber + neutral system; purple is the AI/SaaS cliché we differentiate from |
| No pure black (`#000000`) for text | Use `text.primary` (`#0F172A`) |
| No pure white (`#FFFFFF`) for the canvas | Use `surface.canvas` (`#F8FAFC`) |
| No emoji in UI | Lucide icons only (§11) |
| No animation longer than 500ms | Stay within the §10 duration scale |
| No `outline: none` without a replacement | Every interactive element shows `border.focus` on keyboard focus (§14.0) |
| No status color used as a brand color, or vice-versa | Keep the two palettes separate (§2, §5) |
| No hover lift / movement on components | Hover communicates through color and background only (§10, §14.0) |
| No indigo as the primary color | Navy `#14315C` is the deliberate primary (§2) — indigo collides semantically and reads as generic SaaS |
| No fixed-width buttons or tightly-sized containers | Let Bangla text wrap and expand (§12) |
| No reliance on the native `title` tooltip | Use the real tooltip component (§14.15) |

---

## 17. Theme switching

- **Mechanism:** `data-theme` attribute on `<html>`. Values: `light`, `dark`, `system`.
- **System mode** respects `prefers-color-scheme` media query.
- **Persistence:** localStorage key `cc-lms-theme`.
- **First paint:** inline script in `<head>` sets the theme before React hydrates — no flash of incorrect theme.
- **All semantic tokens swap automatically.** Components never check the theme — they read from semantic tokens.

---

## 18. Accessibility commitments

- **WCAG AA contrast minimum** for all text/background combinations. Status colors verified in both themes.
- **44 × 44px minimum touch target** on mobile, 32 × 32px on desktop.
- **Focus rings always visible** on keyboard navigation (2px outline, `border.focus`, 2px offset — §14.0).
- **Reduced motion respected** via media query.
- **Bangla screen reader support** — `lang="bn"` attribute on Bangla content blocks.
- **No information conveyed by color alone.** Status pills have text labels. Charts have patterns/labels in addition to color. Form errors show text, not just a red border.

---

## End of design system

The system is intentionally opinionated. When generating new screens or components, prefer this system's rules over generic SaaS conventions. When in doubt: choose institutional weight over playfulness, density over generous whitespace for admin/accountant surfaces, comfort over density for student/teacher surfaces, and semantic tokens over primitives in every case. Build components from the §14 specifications — they are the source of truth for sizing and state behavior.
