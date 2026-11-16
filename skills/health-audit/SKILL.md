---
name: health-audit
description: Run a recurring codebase health audit, or set one up so it runs on a schedule and raises an alarm when it stops. It reconciles docs with code, checks dependencies, reliability, maintainability, the test suite, security and the agent instructions, turns each mechanical finding into a guard test, and leaves one pull request plus a dated scorecard. Use when asked for a health audit, a codebase review on a schedule, or "why do our audits keep stalling".
license: MIT
---

# Health audit

An audit that runs by hand stops running. Nothing schedules it, nothing says
how to run it, and nothing notices when it does not run. This skill fixes all
three: one checklist that both a person and a scheduler run, a script that
starts it, and an alarm that fires when the newest scorecard is too old.

There are two modes:

- **Set up** (once per project): see "Setting up the schedule" at the end.
- **Run** (each month, or on request): the checklist below.

## Rules for a run

- **Every finding ends as one of four things:** fixed in this pull request, a
  new guard test, a ticket, or accepted with a written reason. Never only a
  line in a report.
- **Fix only small things in the audit pull request:** doc edits, new guards
  with the small violations they find, and fixes of a few lines. Anything
  larger becomes a ticket.
- **Never merge, deploy or push.** Never touch production. The wrapper script
  pushes the branch and opens the pull request after you finish. You need no
  production access.
- **Do not guess.** If a check cannot run (a tool is missing, a login
  expired), write `not checked: <reason>` in the scorecard and open a ticket.
- **Restore every deliberate break.** Step 5 breaks code on purpose to test
  the tests. Restore each file before the next break, and check the diff
  before you commit.

## The checklist

First read the "Project commands" table in the agent instructions file
(`CLAUDE.md`, `AGENTS.md`). It names the test, lint, security scan and
dependency commands for this stack. If there is no table, write one with the
owner before the first run.

**Tickets.** In a run started by a person, open tickets in the project's
tracker. In an unattended run you cannot create tickets. List each one in the
scorecard under "Tickets to open", with a title and two lines of detail, and
the owner opens them when they review the pull request.

### 0. Set up

1. If the wrapper script started you, you are already on the branch
   `health-audit-YYYY-MM`. If a person started you, create that branch from the
   main branch.
2. Run the full test suite. Record runs, failures and time. A red main branch
   is the first finding.

### 1. Reconcile docs with code

1. Each design doc and spec: check its `Status:` line against the code. Fix the
   line and its date.
2. Each audit document: mark each row the code has fixed `CLOSED YYYY-MM-DD`.
3. Each decision record: is it still true? If the code broke it, open a ticket
   or write the superseding record.
4. Each pull request merged since the last scorecard: if it made a lasting
   decision with no record, write the record and add it to the index.
5. Each deploy since the last scorecard: does the changelog have its entry?
   (See the `deploy-changelog` skill.)

### 2. Dependencies

1. Check the language and framework versions against their end-of-life dates.
   An unattended run has no web access, so write `not checked: no web access`
   unless the project keeps the dates in a file.
2. Run the project's "outdated" commands.
3. One ticket per upgrade that is due: a security fix, an end-of-life version,
   or a dependency two major versions behind. Not one ticket per patch release.

### 3. Reliability

1. The error tracker: unresolved issues older than 14 days are findings. An
   unattended run starts no external tools, so write `not checked: no error
   tracker access` unless a person runs the audit with that access.
2. CI history on the main branch: the share of green runs in the last 30, and
   the suite time of the newest three. In an unattended run the wrapper saves
   them to `.audit-input/ci-runs.json` and `.audit-input/ci-timing.tsv`. Never
   commit that folder.
3. Flaky tests: a run that failed and then passed on the same commit
   (`headSha`).
4. Scheduled jobs: if the app records heartbeats, the owner checks them. Do
   not read production yourself.

### 4. Maintainability

1. Hotspots: files changed most in the last 90 days, crossed with the largest
   files. Run `git log --since=90.days --name-only --format=` and count the
   paths (pipe it through `sort`, `uniq -c`, `sort -rn` and `head -20`, each on
   the allowlist exactly as written). A file on both lists is a finding, with a
   suggested split.
2. Dead code: routes nothing links to, templates nothing renders, styles
   nothing uses, services nothing calls. Run the reference scan from the
   `test-audit` skill before calling anything dead.
3. Lint suppressions (`eslint-disable`, `# noqa`, `rubocop:disable`,
   `@SuppressWarnings`): count them and compare with the last scorecard.

### 5. Test suite

1. Tests of dead code: delete them with the code (`git rm`), or open a ticket
   when the code may come back.
2. Tests that pass whatever the code does: pick 10 tests from the slowest or
   most-mocked files, break the code each one covers, and rerun it. A test
   that stays green gets rewritten or deleted. Restore the file after each
   break. The `test-audit` skill has the full method.
3. Assertions on styling (CSS classes, icon names) where behaviour should be
   asserted: replace them with text, status or a `data-test-id` hook.
4. The 20 slowest tests, compared with the last scorecard.

### 6. Security

1. Run the project's security scanners (static analysis, dependency audit).
2. Read every suppression in the scanner's ignore file. Is each one still
   justified?
3. Production-only checks (second-factor coverage, key rotation) belong to the
   owner. Write `owner check` in the scorecard.

### 7. Agent instructions

1. Check every command in the agent instructions file, the onboarding doc,
   and each skill in the repository. Run the safe ones. For the others, check
   that the file or script they name exists. Fix any doc whose command fails or
   whose path moved. An unattended run cannot edit `.agents/`. Put the fix for
   those files in the scorecard as a finding.
2. Check every relative link in `docs/` resolves.

### 8. Ratchet

For each mechanical finding, write the guard test that would have caught it
(see the `rules-that-enforce-themselves` skill). Over time the guards take
over the mechanical checks and the audit gets shorter.

### 9. Finish

1. Write `docs/audits/health/YYYY-MM.md` from
   [assets/scorecard-template.md](assets/scorecard-template.md), with a
   "Tickets to open" section if the run could not open them itself.
2. Run the full test suite. It must pass before you commit.
3. Run `git diff <main> --stat` on the source folders. Every file it lists must
   be a fix you meant to make. Restore any file left over from a step 5 break,
   and run the suite again.
4. Commit only the files you changed, on your branch.
5. Stop. Do not push. Do not open the pull request. The wrapper script does it.

## Setting up the schedule

Do these steps with the owner. Copy each template from `assets/` to the path
given, and fill in the project's values. Keep the execute bit on the scripts
(`chmod +x bin/*`).

1. **The command table.** Add a "Project commands" table to the agent
   instructions file: test, lint, security scan, dependency audit, outdated
   dependencies, CI history. The checklist reads it there. Keep one copy only.
2. **The scorecard format.** Copy
   [assets/scorecard-template.md](assets/scorecard-template.md) to
   `docs/audits/health/README.md`. The front matter must keep its `date:` line,
   because the alarm reads it.
3. **The project's checklist.** Copy this file to
   `.agents/skills/health-audit/SKILL.md` and adapt it: remove checks that do
   not apply, and name the project's real folders. The wrapper reads it from
   there, so the unattended run does not depend on any plugin being installed.
4. **The tool allowlist.** Copy
   [assets/health-audit.settings.example.json](assets/health-audit.settings.example.json)
   to `.agents/health-audit.settings.json` and replace each `REPLACE` entry.
   - List exact commands. Never use a wildcard on a command that can run other
     commands: `git:*` lets the agent push, `npm run:*` runs any script,
     `find:*` runs anything through `-exec`, `git fetch:*` runs anything through
     `--upload-pack`, and GNU `sort:*` runs anything through
     `--compress-program`.
   - Keep the file tools scoped to the project (`Read(./**)`, `Edit(./**)`;
     an `Edit` rule covers every file-editing tool), and deny edits
     to `.github/` and `.agents/` (the checklist and the allowlist itself).
   - Deny the git options that read or write files outside the project
     (`--output`, `--no-index`, `-F`, `--file`), and every `gh` command. The
     wrapper collects CI history before the agent starts, so the agent needs no
     GitHub token.
   - The file turns on the OS sandbox. Bash commands can then write only inside
     the audit's clone, cannot read the secret paths in `denyRead`, and reach
     only the domains in `allowedDomains`. Add every path on the machine that
     holds a secret, and only the package registries the outdated and audit
     commands need. `failIfUnavailable` stops the run when the sandbox cannot
     start, rather than running without it. macOS needs nothing extra. Linux
     needs `bubblewrap` and `socat`. If a tool needs a file under `~/.config`,
     replace that entry with the specific secret folders under it.
   - The wrapper runs the agent in restricted mode, so the repository's own
     agent settings cannot widen this list.
5. **Know what the allowlist cannot stop.** The agent runs the test command,
   and a test it writes can run any code. A `git commit` runs the repository's
   hooks. And the audit machine holds credentials that can push, because the
   wrapper pushes. So:
   - run the audit on a machine with no production credentials;
   - give it a token that can push only `health-audit-*` branches, or protect
     the main branch so nothing merges without review;
   - never let the audit's pull request merge itself;
   - keep CI secrets away from builds of non-main branches (for example, put
     them in a protected environment), because CI runs the branch's own tests
     and scripts, and the agent can edit those.
6. **The wrapper script.** Copy
   [assets/run-health-audit](assets/run-health-audit) to
   `bin/run-health-audit`, and set `MAIN` and `SETUP_CMD`. It collects the CI
   history, clones the main branch into `~/health-audit-runs/` with no remote,
   and runs the agent there. Afterwards it never runs git in the agent's clone,
   because the agent could have planted hooks or config in it. It fetches the
   branch into a second, clean clone and checks it there: the branch starts
   from main, it holds a scorecard, and it changes no CI config, hook folder,
   audit config or wrapper. Only then does it push one fixed branch name with
   hooks off, open the pull request, and delete both clones. The agent never
   pushes.
7. **The scheduler.** Run the wrapper once a day on a build box with cron, for
   example `0 2 * * * cd /path/to/repo && bin/run-health-audit >> ~/health-audit-cron.log 2>&1`.
   It does nothing outside days 1 to 7 or once the month's branch is pushed, so
   a missed day is retried the next day. A scheduled CI job is a poor fit: its
   runners hold the repository's secrets, and on GitHub Actions a pull request
   opened with the built-in token does not start the required checks.
8. **The alarm.** Copy [assets/audit-age](assets/audit-age) to `bin/audit-age`
   and [assets/audit-overdue.yml](assets/audit-overdue.yml) to
   `.github/workflows/`. Every week it reads the newest merged scorecard's
   date. When the date is more than 40 days old, it fails and opens an issue,
   and the owner gets an email. It runs on the CI provider's machines, so it
   still fires when the build box, its cron line or its agent login is broken.
   Until the first scorecard is merged it fails every week. That is expected.
9. **The decision record.** Write a record for the audit itself: what it
   checks, where it runs, and how the owner hears when it stops.
10. **A first run.** Run `bin/run-health-audit --now` once. Read the scorecard
    with the owner and merge it. It becomes the baseline, and it stops the
    alarm.

The alarm checks the scorecard's date, not the job's exit status. An audit
can stop in many ways: the machine is off, the cron line was lost, the agent's
login expired, the run failed, or nobody merged the pull request. All of them
leave the newest merged scorecard getting older, so one check catches them
all.
