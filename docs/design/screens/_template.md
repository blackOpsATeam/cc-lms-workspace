# Screen: <Module> — <Screen name>

**Role(s):** Student | Teacher | Admin | Accountant
**Density:** comfortable | compact
**Primary device:** mobile | desktop | both

## Purpose

<One sentence: what this screen lets the user do.>

## Layout

<Describe the layout in prose or ASCII. Reference DESIGN.md sections where relevant.>

```
┌─────────────────────────────────────┐
│ Header                              │
├─────────────────────────────────────┤
│ Summary cards                       │
├─────────────────────────────────────┤
│ Main widget(s)                      │
└─────────────────────────────────────┘
```

## Sections

### <Section name>

- Components: KPI card | status badge | queue row | ...
- Tokens: `surface.raised`, `text.primary`, `border.subtle`
- Density: ...
- States: empty | loading | populated | error

## States

### Empty
<What shows when there's no data.>

### Loading
<Skeleton blocks matching the populated layout.>

### Populated
<Normal state.>

### Error
<What shows when data can't load.>

## Interactions

- Tap on <element> → navigates to ...
- Long-press on <element> → ...

## Data needs

- Endpoint: `GET /api/...` (see `docs/api-contracts/<module>.md § ...`)
- Refresh: on mount / pull-to-refresh / every N seconds

## Bangladesh-context

- Currency display: ...
- Date format: ...
- Phone format: ...
- Bangla text expansion: ...

## Accessibility

- Touch targets: ≥44px mobile / ≥32px desktop
- Focus order: ...
- Color-alone: <which info uses redundant labels/icons>

## Out of scope

- <Anything that might be assumed but isn't part of this screen>

## SRS references

- `docs/srs/<module>.md` § <section>
- FR IDs: FR-XX-NNN, FR-XX-NNN, ...
