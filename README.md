# AI project practices

Skills and one agent for running a software project with AI coding assistants
over months and years, not one session.

An AI coding agent starts each session without the lessons of the last one.
Your docs drift away from your code, an audit you meant to run monthly stops,
the changelog falls behind, and the test suite grows past the point where
anyone can say which tests protect anything. These skills turn each of those
into something a machine checks or a schedule runs, with an alarm when it
stops.

They work on any stack. The examples use several languages, and every
command is something you fill in for your project.

## The skills

| Skill | What it does |
|---|---|
| `first-run` | Sets up an existing project in one pull request. It surveys the repo, asks you a few questions, then adds the other four skills' setup. You can run it again to find gaps. |
| `rules-that-enforce-themselves` | Each lesson becomes a decision record (why), one entry in `CLAUDE.md` or `AGENTS.md` (what to do), and a guard test that fails the push that breaks the rule. Design docs carry a status line, and runbooks hold what you learn while operating. |
| `health-audit` | A monthly checklist covering docs vs. code, dependencies, reliability, maintainability, tests, security and the agent instructions. A script runs it unattended with an exact tool allowlist. A weekly CI job opens an issue when the newest scorecard is more than 40 days old. |
| `test-audit` | Finds tests that would not notice if the code broke, and proves each finding by breaking the code and rerunning the test. It then fixes them in a safe order: environment switches like CSRF and rate limits, dead code, styling assertions, and browser tests that belong a layer down. |
| `deploy-changelog` | An agent drafts the changelog during each deploy, for people who are not developers. You approve it in the first minute and walk away. Every deploy is tagged, and the audit matches tags to entries. |

The `test-critic` agent runs the test audit's first phase. It changes nothing
it keeps, and it reports only what it proved.

## Install in Claude Code

```
/plugin marketplace add canuk/ai-project-practices
/plugin install project-practices@ai-project-practices
```

Then, in a project:

```
/project-practices:first-run
```

## Use with other agents

The skills in `skills/` follow the open [Agent Skills](https://agentskills.io)
format. If your agent supports it, copy the skill folders into its skills
directory. Check your agent's docs for where that is. The `test-critic` agent
in `agents/` works only in Claude Code. Any agent can follow its file as a
prompt.

## The ideas behind it

- **A rule needs a check behind it.** A rule that lives only in a doc or a
  chat is forgotten. A rule with a test fails the push that breaks it.
- **Guards reflect over the real set.** A guard test lists every route, job,
  table or file the rule covers, and fails on any member nobody classified.
  Its failure message tells the reader how to fix it.
- **Every finding ends as one of four things:** fixed, a guard test, a
  ticket, or accepted with a written reason. A finding that is only a line in
  a report is not closed.
- **Watch the schedule, not the job.** The alarm checks the date of the
  newest merged scorecard. A dead machine, a lost cron line, an expired login
  and an unmerged pull request all show up there.
- **Prove it, do not assert it.** A test counts only once you have seen it
  fail with the code broken.
- **The agent proposes and a person approves.** Agents never push, merge or
  deploy in these workflows. A script pushes one fixed branch name, and a
  person merges.

## License

MIT
