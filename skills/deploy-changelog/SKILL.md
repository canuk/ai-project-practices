---
name: deploy-changelog
description: Keep a changelog for non-developers current by drafting each entry during the deploy with a headless AI agent, so no deploy ships undescribed. The owner reviews the draft in the first minute of the deploy and can then walk away, the changelog commit skips CI, and each deploy is tagged so an audit can match tags to entries. Use when a project's changelog has fallen behind, when setting up a deploy script, or when asked to write release notes for users or staff.
license: MIT
---

# Deploy changelog

A changelog that someone writes "later" stops being written. Tie it to the one
step that always happens, the deploy, and have an agent draft it from the
merged pull requests. A person only approves or edits.

## The design

1. **Draft during the deploy, in the background.** The deploy script starts the
   draft as soon as it knows what is being deployed. It works from the pull
   request titles and descriptions merged since the last deploy tag.
2. **All interaction happens early.** Put the review right after the deploy's
   first interactive step (a password manager approval, a confirmation
   prompt). Once the owner approves the entry, nothing else asks a question, so
   they can walk away while the build and rollout run.
3. **A shape check, not taste alone.** A small test checks each new entry:
   a title, 1 to 10 items, one sentence per item, no code. The deploy refuses
   an entry that fails it and shows why.
4. **The changelog commit skips CI.** The entry is a one-file commit. The CI
   config ignores pushes that touch only the changelog file (and docs), so the
   deploy does not wait for a full test run it does not need. A guard test
   keeps that ignore list from growing to cover code.
5. **Each deploy gets a tag**, for example `production-YYYY-MM-DD-HHMM`, with the
   entry title in the tag message (`changelog: <title>` or `changelog: skipped`).
   The monthly audit matches tags to entries, so a skipped entry is a choice
   that someone made, never one nobody noticed.
6. **Failures never block the deploy.** If the agent is missing, logged out or
   slow, the draft step times out (90 seconds is plenty) and the review offers
   an empty entry to write by hand, or skip.

## The entry

Audience: people who use the product or support its users, not developers.
Copy [assets/changelog-prompt.md](assets/changelog-prompt.md) to
`.agents/prompts/changelog.md` and fill in the product name and the audience.
Its rules:

- A title of at most 8 words naming the main change.
- 1 to 10 items. One item per change a person would notice.
- Never split one change into several items to make the list longer. Never
  join unrelated changes to fit the limit. When more than 10 qualify, keep the
  10 that matter most.
- Each item is one sentence of at most 30 words, in active voice, naming who
  can now do what.
- Leave out internal work completely: tests, CI, refactors, tooling, docs,
  dependency updates. Do not summarize it.
- No file names, class names, issue numbers or test counts.
- If nothing qualifies, the agent replies `NONE` and the deploy tags
  `changelog: skipped`.

## The shape check

Write it as a test in the project's language, run against every entry newer
than the last deploy. It fails when:

- the title is missing, or is not plain text (a YAML title containing `: `
  parses as a map unless quoted);
- there are fewer than 1 or more than 10 items;
- an item is longer than 30 words;
- an item contains something that reads as code: `::`, a file path, `#123`,
  a `snake_case` word, or words like "migration", "refactor", "table".

## Setting it up

1. Pick where entries live: a YAML file the app renders, or a `CHANGELOG.md`.
   The prompt asks for YAML. For Markdown, change the reply format at the end of
   the prompt. Write the shape check against the file.
2. Copy [assets/draft-changelog](assets/draft-changelog) to
   `bin/draft-changelog` and keep its execute bit. It finds the last deploy tag,
   collects the merged pull requests since then, and runs the agent headless
   with no tools, so the model only writes text.
3. In the deploy script, right after its first interactive step:
   1. Start the draft in the background.
   2. Show the draft. Offer: accept, edit in `$EDITOR`, write from empty, or skip.
   3. Run the shape check on the result. On failure, show the problems and
      offer the same choices again.
   4. Commit the entry alone, with the subject `Changelog: <title>`, push it,
      and continue with the deploy. The draft script finds the last entry by
      that subject, so keep it exact.
   5. When the deploy succeeds, create the annotated tag and push it.
4. If the deploy script is started without a terminal (from a chat tool, a CI
   job), it must stop before it deploys and say so, or deploy with
   `--skip-changelog` explicitly. It must never hang waiting for input nobody
   can give.
5. Add the changelog file to the CI ignore list, and a guard test that the
   list holds only docs and the changelog.
6. Backfill: run the draft script with a date range for each past release that
   has no entry, and review each one like a normal entry.
7. Write a decision record for the process, and a runbook entry for "the
   draft failed" and "I skipped the entry by mistake".
