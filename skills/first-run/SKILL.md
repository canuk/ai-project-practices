---
name: first-run
description: Set up an existing project to work this way, in one reviewed pull request. Surveys the repository, asks the owner a few questions, then adds decision records with an index guard, a Discipline section in the agent instructions file, status lines on design docs, the scheduled health audit with its alarm, the deploy-time changelog, and a baseline test audit. Safe to run again, because it detects what is already in place. Use when adopting these practices in a project that has not used them, or when asked to "set this project up properly for AI coding agents".
license: MIT
---

# First run

This skill adopts the other four skills in an existing project:
`rules-that-enforce-themselves`, `health-audit`, `test-audit` and
`deploy-changelog`. It does the setup each of them describes, in a safe order,
and leaves one pull request the owner can review.

## Rules

- **Read first, change later.** Steps 1 and 2 change nothing.
- **Use the project's conventions.** If the project already keeps decision
  records in `doc/adr/`, use that folder. If it has `AGENTS.md` and not
  `CLAUDE.md`, edit that. Do not create a second copy of anything.
- **Never change production behaviour on your own.** Deploy scripts, CI
  required checks, security settings and environment config change only with
  the owner's explicit yes, given for that change.
- **Never commit a secret.** Templates use placeholders. Tokens and keys go in
  the owner's secret store.
- **One commit per part**, so the owner can drop a part without losing the rest.
- **Every guard you add must bite.** Break the rule once, watch the guard fail,
  restore the code.
- **If something is already in place, check it instead of rebuilding it.**
  Report what you found and what you would improve.

## 1. Survey the repository

Find and note, in a scratch file outside the repository:

1. The stack: languages, frameworks, package managers, test framework.
2. The commands: run the tests, lint, security scan, dependency audit,
   outdated dependencies, start the app. Look at the README, `package.json`
   scripts, `Makefile`, `bin/`, `justfile`, CI config.
3. The test suite: how many tests, how long a full run takes, whether it runs
   in parallel. Run it once. A red suite is the first thing to tell the owner.
4. CI: the provider, the workflow files, the required checks, any path filters.
5. Deploy: the deploy script or pipeline, whether it is interactive, how it
   authenticates, whether deploys are tagged.
6. Docs: the agent instructions file and its length, decision records, design
   docs or specs, runbooks, a changelog, an onboarding doc.
7. Schedulers and background jobs, and how failures are noticed.
8. The error tracker, and the issue or ticket tracker.
9. What is already in place from the four skills.

## 2. Ask the owner

Ask these, one message, numbered. Skip any the survey already answered.

1. Which two or three bugs or incidents hurt the most? (These become the first
   Discipline rules.)
2. Who reads the changelog, if anyone? (Users, staff, developers, nobody yet.)
3. Where can a monthly audit run unattended? It needs an always-on machine
   with cron, no production credentials, and a token that can push only audit
   branches. Or should it stay manual for now?
4. Where should tickets go? (The issue tracker, a board, or the audit's own
   scorecard.)
5. Should I run the test audit's phase 1 now (it reports and changes nothing),
   or later?
6. Is there anything I must not touch?

Then write the plan: the parts you will add, the files for each, and the steps
only the owner can do. If the survey found the suite running on one worker by
accident, put the fix in the plan. Show the plan and wait for a yes.

## 3. Apply, in this order

Work on a branch named `adopt-practices`. Each part follows the setup steps
of its skill (for `test-audit`, its phase 1). Read that skill before you start
the part.

1. **Decision records.** From `rules-that-enforce-themselves`:
   - The folder, the index, and the first record ("Record architecture
     decisions").
   - The index guard test.
   - One record for each incident the owner named.
2. **The agent instructions file.**
   - Keep it short. Move long sections into `docs/` and link to them.
   - Add a "Project commands" table, from the survey. This is the only copy.
     The health audit reads it here.
   - Add a Discipline section with one entry per record from part 1. Add a
     guard test for each rule a machine can check.
   - Add a "Where things live" list: decision records, runbooks, design docs,
     audits.
3. **Docs that say whether to trust them.**
   - A status line on every design doc and spec. Check each line against the
     code, and ask the owner when you cannot tell.
   - The status-line guard.
   - A `docs/runbooks/` folder. Move any operational steps you found in the
     README or in agent memory files into runbooks.
4. **Health audit.** From `health-audit`, "Setting up the schedule":
   - The scorecard format.
   - The project's adapted checklist in `.agents/skills/health-audit/SKILL.md`.
   - The tool allowlist in `.agents/health-audit.settings.json`.
   - The wrapper script.
   - The alarm workflow and its date script. Tell the owner the alarm fails
     every week until the first scorecard is merged.
   - A decision record.
   - Do not install the scheduler yourself. Write the exact cron line into a
     runbook, for the owner to install.
5. **Deploy changelog.** From `deploy-changelog`:
   - The prompt, the draft script and the shape check.
   - The CI ignore rule for the changelog file, with its guard.
   - If a deploy script exists, write the integration as a separate commit,
     and list it under "Changes to the deploy" in the pull request. If there is
     no deploy script, leave the integration steps in the runbook.
6. **Test audit baseline.** If the owner said yes, run phase 1 of `test-audit`
   (or start the `test-critic` agent). It writes the audit document and changes
   no code. If the owner approved it in the plan, also fix an accidental serial
   run, because it speeds up everything after.

After each part, run the full test suite. Run the new guards and prove each
one bites.

## 4. Open the pull request

The pull request description has these sections:

1. **What this adds**: one line per part, with its decision record.
2. **What the guards found**: real violations fixed, and any exempted with a
   reason and a ticket.
3. **Changes to the deploy or CI**: each one listed for the owner to check, or
   "none".
4. **Owner steps**: set up the audit machine (agent login, a token that can
   push only audit branches), install the cron line, keep CI secrets away from
   branch builds, merge, run the first audit by hand, and merge its scorecard.
   Make each one an exact command.
5. **Next**: the test audit's findings (if phase 1 ran), and when the first
   scheduled audit will run.

Do not merge. Do not install the schedule.

## Running it again

On a project that already has some parts, step 1 finds them. For each existing
part, check it against its skill:

- Is the guard still in the suite?
- Is the newest scorecard younger than 40 days?
- Does every deploy tag since the last audit have a changelog entry?

Report the gaps, and fix them in the same kind of pull request.
