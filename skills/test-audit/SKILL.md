---
name: test-audit
description: Audit a test suite for whether its tests would notice if the code broke, and fix what the audit proves. Every finding is proved by breaking the production code and rerunning the test, never by reading alone. Covers tests the test environment quietly switches off (CSRF, rate limits, caching), tests that assert their own mocks, dead code that only tests still call, styling asserted as behaviour, slow or duplicated layers, and serial runs. Use when asked whether a large suite is useful, after a bug shipped behind green tests, or before pruning tests or code.
license: MIT
---

# Test audit

A green suite is not evidence. The question for each test is:

> What single, plausible change to the production code would make this test fail?

If the answer is "none", "a syntax error" or "deleting the whole function",
the test protects nothing. You answer the question by changing the code and
running the test, not by reading.

The audit has two phases. Phase 1 finds and proves. Phase 2 fixes. Phase 1
changes nothing it keeps. Ask the owner which findings to act on before
phase 2, unless they have already said.

## Phase 1: find and prove

Work in an isolated copy (a git worktree), never in a checkout someone else is
using. For each suspect test:

1. Predict the mistake a person could make that the test should catch: invert
   a condition, drop a filter, return early, remove a guard, move an
   error handler outside a loop, change a boundary from `>=` to `>`.
2. Make that change in the **production** code.
3. Run only the affected test file.
4. Record whether it failed, and whether the failure message was clear.
5. Restore the file before the next change. Check `git diff` is empty at the end.

A test that survives a change it should have caught is a confirmed finding. A
test you only think is weak is not a finding. Prove it or drop it. A critic
that cries wolf gets ignored, and then its real findings go unread.

The `test-critic` agent in this plugin runs this phase. Point it at a folder,
a feature, or "the 20 most-mocked files".

### What to look for, most damaging first

1. **Switched off by the test environment.** The test exercises something the
   test configuration disables, so it cannot fail. Read the test environment
   config before you trust any test of these:
   - CSRF or forgery protection (often off in tests, on in production).
     A JavaScript save that forgets the token passes every test and fails
     for every user.
   - Rate limiting (a null or no-op cache never counts, so no limit fires).
   - Caching, background job adapters, email delivery, host checks, SSL.
   - External identifiers no test can check: AI model names, API versions,
     webhook URLs. A test that compares a constant with itself proves nothing.
     Check these against the real service at deploy time instead.
2. **Asserting the mock.** The test stubs a collaborator, then asserts the stub
   was called with what the code passed it. It proves the code calls the code.
3. **Passes with the feature deleted.** The setup creates a record and the
   assertion checks that it exists. The assertion restates the fixture.
4. **Rendering mistaken for behaviour.** A 200 response is the whole test. It
   proves the template compiled. It does not prove the save persisted or the
   job ran. Any write that goes through the browser (autosave, chat, forms
   submitted by script) needs one end-to-end test that performs it and checks
   the stored result.
5. **One case standing for many.** A loop that must survive one item's failure
   is tested with one item. A history that must send every turn is tested with
   one turn. Use at least two, and make the first one fail.
6. **Assertions that cannot fail.** A refutation of a CSS class no template can
   render. `assert not_nil` on an object built two lines above.
7. **Over-broad matchers.** `match /error/`, where any error passes. A word
   that also appears in the navigation. An HTML-escaped string that matches an
   attribute instead of the visible text.
8. **Styling asserted as behaviour.** Counting icon classes to mean
   "completed". Replace with the text a person reads, or a
   `data-test-id="kebab-case"` hook.

### Also measure

- **Dead code that only tests call.** List classes and functions whose only
  references are their own file and their own tests.
- **Duplicated layers.** Browser tests that never click, type, run a script or
  press a key, and that repeat a request-level test.
- **Speed.** Is the suite running on one worker by accident? A single global
  setting (for example a parallelism call in one shared base class) can force
  the whole suite onto one process.

### What not to flag

- Simple tests of simple code.
- Fixture and factory setup.
- Smoke tests where rendering really is the contract, and the file says so.
- Tests that call real external services on purpose and skip without a flag.
- Any weakness you could not confirm with a code change.

### The phase 1 report

Write it to `docs/audits/YYYY-MM-DD-test-suite-audit.md` in the main checkout. For each confirmed
finding: the file and test, the exact change you made, what happened, the
assertion that would make it real, and what a user would see if it shipped
broken. End with what you checked that came back clean, and what you could not
verify. Rank by what could ship broken, not by how easy it is to describe.

## Phase 2: fix

Write a plan with one task per kind of fix, in this order. Each task's rule:
**a new or changed test must fail when the code it guards is broken. Prove it
once and restore the code.**

1. **Speed first.** Fix accidental serial runs, so every later run is faster.
2. **Cover live code before you delete its only test.** If dead code holds the
   only test of a rule the live code also enforces, write the live test first.
3. **The environment switches.** Turn the protection on where the tests that
   need it run, for example CSRF on for every browser test. Rules for this task:
   - Change only the test setup. Never change how production checks it.
   - Production works today, so a test that starts failing is far more likely
     a test problem than an app bug. Change app code only after a real browser
     against a development server (which has the protection on) shows the same
     failure. Put each app fix in its own commit and flag it for review.
   - If more than 10 tests fail, or the cause is unclear, stop and report.
   - Run the whole browser suite twice before you call it done.
   - For rate limits, add a test helper that swaps in a counting store for one
     block and restores the null store after, even when the block raises.
   - For external identifiers, add a deploy-time check that lists what the
     service serves. A network failure must say "not checked", never "wrong id",
     and must not skip the deploy's other checks.
4. **Weak tests.** Two members where the loop must survive a failure, three
   turns where the history must be complete, both sides of every boundary.
5. **Dead code.** Before deleting anything, scan for every reference:
   - the name as a word, as a string and as a symbol, across source, config,
     templates, scripts, scheduler config and deploy hooks;
   - dynamic calls (`send`, `getattr`, `obj[name]`, reflection), route strings,
     and templates or helpers that build names;
   - diagrams or docs that name it (fix them in the same change).
   A reference only from the code's own file, its own tests, or other code
   deleted in the same change does not keep it alive. Put the scan result in
   the commit message, one line per name.
6. **Styling assertions.** Replace with text or `data-test-id` hooks. Delete
   refutations of classes no template, helper, or script can produce (watch for
   names built by string interpolation and classes added by scripts).
7. **Move page-load-only browser tests down a layer.** A browser test that only
   loads a page and reads server-rendered text belongs in a request test. Keep
   it in the browser when the check depends on a script: content a component
   inserts, visibility, a lazy-loaded frame, attributes a script sets. Prove a
   sample of moved tests fail when the view breaks.
8. **Close the audit.** Mark each finding `CLOSED YYYY-MM-DD (<commit>)` in the
   audit document, or `OPEN: <what is left>`.

Have each task reviewed against its own text before the next one starts. After
all tasks, review the whole branch once. Report the suite's run count and time
before and after.
