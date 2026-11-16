---
name: rules-that-enforce-themselves
description: Turn a project's hard-won lessons into rules that people and AI coding agents both follow and that a machine checks. A decision record explains why, the agent instructions file (CLAUDE.md, AGENTS.md) links each rule to its record, and a guard test fails the push that breaks the rule. Use when a bug class recurs, when a decision must outlive the session that made it, when docs drift from the code, or when setting up a repository for AI-assisted work.
license: MIT
---

# Rules that enforce themselves

An AI coding agent starts every session with no memory of last month's
outage. A person forgets too, only more slowly. A rule that lives only in a
chat, a pull request comment or an agent's memory file is lost. A rule that
lives only in a document drifts away from the code. A rule with a test behind
it fails the push that breaks it, so it holds.

Every rule in the project follows this chain:

1. **A decision record** says what was decided and why. People and agents read
   it when they need the reason.
2. **One entry in the agent instructions file** says what to do, in a few
   lines, and links to the record. Agents read this file every session.
3. **A guard test** checks the rule wherever a machine can see it. Its failure
   message says how to fix the problem.

Write all three in the same change. If you write the rule but skip the guard,
the rule will be broken again.

## 1. Write the decision record

Use [references/decision-record-template.md](references/decision-record-template.md).
Rules for the records:

- One file per decision in `docs/decisions/`, numbered `NNNN-short-slug.md`.
- Each record has a `## Status` heading. The first line under it starts with
  `Accepted (YYYY-MM-DD)`, `Proposed`, or `Superseded by NNNN (YYYY-MM-DD)`.
- Do not rewrite an old decision. Write a new record that supersedes it, and
  mark the old one.
- `docs/decisions/README.md` lists every record in a table. A guard test
  checks the list (see section 3).
- The Context section names the real incident: what broke, when, and what
  it cost. A record without the incident reads as an opinion, and people
  argue with it.

Write a record when a decision changes how future code must be written, when
a bug class has happened twice, or when someone would otherwise ask "why is it
done this way?" a year from now. Do not write one for a choice that is easy to
reverse and affects one file.

## 2. Add the rule to the agent instructions file

The agent instructions file (`CLAUDE.md`, `AGENTS.md`, or your agent's
equivalent) loads into every session. Keep it short and link out. Add a
section called **Discipline** with one entry per bug class you have already
paid for:

```markdown
## Discipline

### Read requests never write data — [ADR 0002](docs/decisions/0002-no-writes-on-read.md)
- A GET or HEAD handler must not insert rows or change a lifecycle flag.
- `find_or_create` in a read handler is the smell. Build the object in memory
  and save it on the write that follows.
```

Rules for the entries:

- The heading is the rule, written as a plain statement.
- Two to five bullets: the smell an agent can spot in a diff, and the fix.
- The link goes to the decision record. The full reasoning stays there.
- Only list bug classes that have really happened. A list of every good
  practice is too long for anyone to read, and agents will skim it too.
- When a rule has a guard test, name the test file, so an agent that breaks
  it knows where the failure comes from.

## 3. Write the guard test

A guard reflects over the **real set** of the things the rule covers and
fails when one of them does not follow the rule. It does not check a sample.
The set can be files, routes, database tables, scheduled jobs, services that
call an AI model, or the decision records themselves.

The shape:

1. Find every member of the set from the source of truth: the file system,
   the router, the schema, the scheduler config, or a code scan.
2. Classify each member: it follows the rule, or it is exempt with a written
   reason.
3. Fail when a member is neither. A new member that nobody classified fails
   the build on the day someone adds it.
4. Write the failure message as instructions: what is wrong, which items are
   affected, and the exact fix. An agent that hits this failure reads the
   message and fixes the problem without asking.

Rules for guards:

- **Check the code, never the calendar.** No test fails because a date
  passed. Watch schedules with a separate alarm (see the `health-audit` skill).
- **Exemptions carry a reason** in the test itself, and a ticket number when
  the exemption is temporary.
- **Prove the guard bites.** Break the rule once, watch the test fail, and
  restore the code.
- **Keep guards fast.** They run on every push. Prefer a file scan or a
  reflection over the router to booting a browser.

[references/guard-tests.md](references/guard-tests.md) has a catalogue of
guards with examples in several languages.

## 4. Make documents say whether to trust them

- **Design docs and specs carry a status line** as their first line:
  `Status: shipped | partial (<what is missing>) | deferred (<why>) — last reconciled YYYY-MM-DD`.
  A build is done when its doc says so. The person or agent that ships the
  last piece updates the line in the same change. A guard test checks that
  every design doc has a valid status line.
- **Audit findings are closed in place.** Mark a fixed row
  `CLOSED YYYY-MM-DD (<commit>)` in the audit document itself. Do not write a
  new report that will also go stale.
- **Operational knowledge goes into a runbook in the same session that
  learns it.** Deploy steps, workarounds, and recovery steps go into
  `docs/runbooks/`. An agent's memory file may point to a runbook. It must
  never hold the only copy.
- **Fix the doc when you fix the code.** If a change makes a doc wrong, the
  same change fixes the doc.

## 5. Close every finding

Every problem found in a review, an audit or an incident ends as exactly one
of four things:

1. **Fixed** in the current change.
2. **A guard test**, so the problem cannot come back unseen.
3. **A ticket** with an owner, for work too large to do now.
4. **Accepted**, with the reason written down where the next reader will see it.

A finding that is only a line in a report is not closed.

## Setting up a repository

When asked to set up a project this way, do these steps in order:

1. Create `docs/decisions/README.md` with an empty index table, and the first
   record: "Record architecture decisions".
2. Add the index guard test (no two records share a number, every record
   has a status, the index lists every record).
3. Add a Discipline section to the agent instructions file. Ask the owner for
   the two or three bugs that have hurt the most. Write a record and an entry
   for each one, and a guard where a machine can see the rule.
4. Add the status-line guard for design docs, if the project keeps them.
5. Create `docs/runbooks/` and tell the owner that operational steps go there.
6. Run the full test suite. The new guards may find real violations. Fix the
   small ones. Exempt the large ones with a reason and a ticket.
