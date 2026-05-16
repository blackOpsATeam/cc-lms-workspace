# docs/design/

## `DESIGN.md`

The design system. Authoritative for tokens, status palette, typography, spacing, density, motion, iconography, and Bangladesh-context patterns. Read this before any UI work.

## `patterns/`

Component-level specs: status badge, KPI card, queue row, class occurrence card, planner widget, action bar, empty state, loading state. One file per pattern.

## `screens/`

Per-module per-screen specs. Folder per module:

```
screens/
├── notice-board/
│   ├── list.md
│   ├── detail.md
│   └── compose.md
├── dashboard/
│   ├── student.md
│   ├── teacher.md
│   └── admin.md
└── ...
```

A screen spec describes the layout, states (empty, loading, error, populated), components used, interactions, and which DESIGN.md tokens to consume. It does NOT include actual code — it's a design contract.

## Validation

After authoring a screen spec, run the `design-validator` subagent against it:

> Use the design-validator subagent to check `docs/design/screens/notice-board/list.md`

The subagent returns `PASS`, `PASS WITH NOTES`, or `NEEDS REWORK`.
