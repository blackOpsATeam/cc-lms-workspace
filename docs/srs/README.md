# docs/srs/

One file per module. Filename matches the module's URL-safe slug:

- `auth.md`
- `admission.md`
- `user.md`
- `profile.md`
- `class-batch-subject.md`
- `class-scheduling.md`
- `teacher-class-workspace.md`
- `student-class-workspace.md`
- `live-class.md`
- `assessment.md`
- `attendance.md`
- `payment-billing.md`
- `payroll.md`
- `accounting.md`
- `notification.md`
- `notice-board.md`
- `dashboard.md`

## Template

Use `_template.md` as the starting structure for any new module SRS. The `cc-lms-srs-author` skill encodes the conventions in detail.

## Refining an existing SRS

1. Create or load the module file
2. Invoke the `srs-reviewer` subagent:
   > Use the srs-reviewer subagent to review `docs/srs/notice-board.md`
3. Address the review output
4. Re-run until verdict is `PASS` or `PASS WITH NOTES`

## Initial seeding

The project started with an SRS PDF that covered all modules. Each module section gets extracted into a separate `<module>.md` file here as it comes up for refinement — we don't pre-load all of them, because we refine as we work. The original PDF stays as a reference but is no longer authoritative once a module file exists here.
