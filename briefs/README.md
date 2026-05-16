# briefs/

A brief is a self-contained unit of work. One brief = one developer + one PR + one to three days.

## Layout

- `templates/` — reusable brief shapes
- `active/` — current work in flight

When a brief is merged (PR closed), move it to `briefs/archive/<YYYY-MM>/` so `active/` stays a small list of what's actually in progress.

## How a brief gets created

1. Manager refines SRS in `docs/srs/<module>.md`
2. Manager creates design spec in `docs/design/screens/<module>/`
3. Manager drafts API contract in `docs/api-contracts/<module>.md`
4. Manager invokes the `brief-author` subagent:
   > Use the brief-author subagent to write a brief for Notice Board list view, track A.
5. Subagent reads SRS + design + API contract, writes the brief, returns a summary
6. Manager reviews, edits if needed, hands the brief filename to the dev

## How a dev uses a brief

1. Dev opens Claude Code in their track folder (`track-a/` or `track-b/`)
2. Dev points Claude at the brief:
   > Read `briefs/active/notice-board-list-view-a.md` and start work.
3. Claude reads the brief, the referenced SRS/design/contract, and starts building
4. Dev reviews, iterates, tests
5. Before opening PR, dev runs:
   > Use the design-validator subagent to check my UI changes.
   > Use the code-reviewer subagent to review against the brief.
6. PR opened, manager reviews, merges

## Brief filename convention

`<module>-<short-task>-<track>.md`

- `notice-board-list-view-a.md`
- `dashboard-student-shell-b.md`
- `attendance-roll-call-bulk-a.md`

The track suffix is `-a` or `-b` (not `-track-a`).
