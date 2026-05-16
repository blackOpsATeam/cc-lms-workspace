# CC-LMS Design System

A design system for a coaching center management platform serving the Bangladesh market. Built for Admin, Teacher, Student, and Accountant roles across mobile and desktop. Institutional, trustworthy, bilingual-ready.

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
- Avoid 2px+ — looks heavy and dated for institutional feel

### Rule
Status pills are always `radius.full`. Buttons and cards are `radius.md`. Modals are `radius.lg`. Avatars are `radius.full`. Don't deviate without a specific reason.

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

## 14. Pattern library (component-level guidance)

### Status Badge
- Rounded pill, leading dot
- 11px overline (uppercase, 0.08em letter-spacing)
- Padded 4px vertical, 10px horizontal
- Background + foreground from status palette family
- One per row in tables, max 2 in cards

### KPI Card
- Comfortable density even in admin context (KPIs are scanned, not parsed)
- Small label (11px overline, muted) above large number (display.md or .lg)
- Optional trend indicator (small caret + delta) below
- 24px padding, radius.lg
- Never on `surface.canvas` — always `surface.raised` or `surface.sunken`

### Queue Row (admissions, payment verification, evaluation)
- Compact density (36px row height)
- Avatar/initials → primary identifier → status badge → metadata → actions
- Hover: `surface.sunken` background, no shadow change
- Selected: `border.brand` left edge accent (2px), background unchanged

### Class Occurrence Card
- Subject + batch + time as primary triad
- Delivery mode pill (ON_SITE / LIVE / HYBRID) below
- Status badge for non-normal occurrences (UPDATED / OVERRIDDEN / CANCELLED)
- Action row at bottom (Open / Join / View)
- Comfortable density
- `surface.raised` with `border.subtle`, radius.md

### Planner Widget (today's classes, upcoming exams)
- Section heading (heading.sm) with count badge
- Stacked rows or compact cards
- Empty state if no items — never show blank container
- Each row deep-links to source module

### Action Bar (role-aware)
- Horizontal scroll on mobile, wrap on desktop
- Each action: icon (20px) + label (body.sm, weight 500)
- Highlighted action when timely (live class joinable, exam window open, overdue)
- Hide entirely if no actions exist (don't show empty bar)

### Empty States
- Icon (32px, muted) + heading.sm + body.sm description + optional CTA
- Centered, generous padding
- Never blank space — always explain *why* it's empty

### Loading States
- Skeleton blocks matching final layout (not spinners)
- Match the shape and size of the eventual content
- Use `surface.sunken` for skeleton fill
- Pulse animation respects reduced motion

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

Decisions made deliberately, recorded so they don't drift:

- **No gradients.** Institutional means flat. Gradients read as marketing/consumer.
- **No drop shadows on dark mode.** Use surface lightening instead.
- **No purple.** Saved for nothing. Purple has become the visual cliché of AI/SaaS products — we differentiate by avoiding it.
- **No pure black (`#000000`) for text.** Use `text.primary` (`#0F172A`). Pure black on white is harsh.
- **No pure white (`#FFFFFF`) for canvas.** Use `surface.canvas` (`#F8FAFC`). Pure white tires eyes during long sessions.
- **No emoji in UI.** Icons only. Emojis render inconsistently across platforms and read as casual.
- **No animation longer than 500ms.** Anything longer feels broken or slow.
- **No focus indicator removal.** Every interactive element shows `border.focus` ring on keyboard focus. Never `outline: none` without replacement.
- **No status color as a brand color or vice-versa.** They live in separate palettes for a reason — don't cross them.

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
- **Focus rings always visible** on keyboard navigation.
- **Reduced motion respected** via media query.
- **Bangla screen reader support** — `lang="bn"` attribute on Bangla content blocks.
- **No information conveyed by color alone.** Status pills have text labels. Charts have patterns/labels in addition to color.

---

## End of design system

The system is intentionally opinionated. When generating new screens or components, prefer this system's rules over generic SaaS conventions. When in doubt: choose institutional weight over playfulness, density over generous whitespace for admin/accountant surfaces, comfort over density for student/teacher surfaces, and semantic tokens over primitives in every case.
