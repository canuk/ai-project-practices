# Health audit scorecards

One file per run, `docs/audits/health/YYYY-MM.md`. The alarm
(`bin/audit-age`) reads the `date:` line. Keep the front matter flat, one
value per key, so a script or a dashboard can plot trends.

A value that could not be measured is a quoted string: `"not checked: <reason>"`.

```markdown
---
date: 2026-12-07
ticket: 123                      # or "none: unattended run"
suite_runs: 4520
suite_failures: 0
suite_minutes: 8.4
main_green_share: 0.97
flaky_tests: 1
error_tracker_issues_over_14_days: 2
outdated_due: 3
lint_suppressions: 41
test_findings: 3
findings: 12
fixed: 4
guards_added: 2
tickets: 5
accepted: 1
---

# Health audit 2026-12

| # | Finding | Outcome |
|---|---|---|
| 1 | Design doc X says shipped, but Y is missing | fixed: status line |
| 2 | Two templates have no caller | ticket #124 |
| 3 | A scheduled job has no test | guard: test/jobs/schedule_coverage_test |
| 4 | Pinned dependency Z is one major behind | accepted: blocked by W, revisit in March |

## Tickets to open

Only in an unattended run, which cannot open tickets itself.

1. **Split the 900-line billing module.** Changed in 31 of the last 90 days.
   Suggested split: invoices, payments, refunds.
```

The commands the audit runs are in the "Project commands" table in the agent
instructions file (`CLAUDE.md`, `AGENTS.md`).
