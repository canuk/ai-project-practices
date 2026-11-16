# Guard test catalogue

Each guard enumerates a real set, classifies every member, and fails with
instructions. Pick the ones whose promise your project makes. Adapt the
examples to your test framework.

## Guards for the documentation itself

| Promise | The set | Fails when |
|---|---|---|
| Decision records are numbered and indexed | `docs/decisions/NNNN-*.md` | Two records share a number, a record has no Status line, or the index and the folder disagree |
| Every design doc says whether it shipped | `docs/design/*.md`, `docs/specs/*.md` | The first lines have no `Status:` line in the agreed format |
| Every relative link in the docs resolves | `docs/**/*.md` | A link points to a file that does not exist |
| Every command in the agent instructions file still exists | Code blocks in `CLAUDE.md` / `AGENTS.md` that name a script | The script path does not exist |
| CI skips only paths that cannot break the build | The `paths-ignore` list in the CI config | It lists anything outside `docs/` or the changelog file |

## Guards for code-wide promises

| Promise | The set | Fails when |
|---|---|---|
| Every scheduled job exists, has a test, and reports what it did | Entries in the scheduler config | A class is missing, no test names it, or it does not record a heartbeat |
| Read requests never write data | Every GET route | A request to it inserts or updates a row (wrap each request and count writes) |
| Every call to an AI model records its cost and owner | Classes that reference the model client | A class does not use the usage-tracking module and is not exempt |
| Every database table is in the data export, or excluded with a reason | Tables in the schema | A new table is in neither list |
| Account deletion clears every user-owned table | Associations on the user model | A new association is neither cleared nor kept with a reason |
| Every browser event listener is removed when its component unmounts | Front-end controllers or components | A file adds a `window`/`document` listener with no matching removal |
| No stylesheet hides the keyboard focus ring | Stylesheets | A rule sets `outline: none` outside the one file allowed to |
| The architecture diagram shows every service that calls the model | Services that reference the model client | A service is neither drawn nor listed as not drawn with a reason |
| Environment variables the app reads are documented | `ENV[...]` / `process.env.X` / `os.environ[...]` reads | A name is missing from `.env.example` |

## Example: the decision record index (Python, pytest)

```python
import re
from pathlib import Path

DIR = Path(__file__).resolve().parents[1] / "docs" / "decisions"
STATUS = re.compile(r"^(Accepted \(\d{4}-\d{2}-\d{2}\)|Proposed|Superseded by \d{4} \(\d{4}-\d{2}-\d{2}\))")


def records():
    return sorted(p for p in DIR.glob("[0-9][0-9][0-9][0-9]-*.md"))


def status_line(path):
    lines = path.read_text().splitlines()
    for i, line in enumerate(lines):
        if line.strip() == "## Status":
            return next((l.strip() for l in lines[i + 1:] if l.strip()), "")
    return ""


def test_no_two_records_share_a_number():
    seen = {}
    for p in records():
        seen.setdefault(p.name[:4], []).append(p.name)
    shared = {n: names for n, names in seen.items() if len(names) > 1}
    assert not shared, (
        f"These records share a number: {shared}. Keep the older one. Give the "
        "newer one the next free number, rename the file, fix its title line, "
        "update every link to it, and add its row to docs/decisions/README.md."
    )


def test_every_record_has_a_status():
    bad = [p.name for p in records() if not STATUS.match(status_line(p))]
    assert not bad, (
        f"These records have no valid status: {bad}. Add a '## Status' heading. "
        "The first line under it starts with 'Accepted (YYYY-MM-DD)', "
        "'Proposed' or 'Superseded by NNNN (YYYY-MM-DD)'."
    )


def test_index_lists_every_record_and_nothing_else():
    listed = set(re.findall(r"\]\((\d{4}-[^)]+\.md)\)", (DIR / "README.md").read_text()))
    files = {p.name for p in records()}
    assert files <= listed, f"docs/decisions/README.md does not list {sorted(files - listed)}. Add a row for each."
    assert listed <= files, f"docs/decisions/README.md links to missing files {sorted(listed - files)}. Fix or remove the rows."
```

## Example: every scheduled job is real, tested and reporting (TypeScript, Vitest)

```ts
import { readFileSync, readdirSync } from "node:fs";
import { join } from "node:path";
import { describe, expect, test } from "vitest";
import { schedule } from "../src/jobs/schedule"; // [{ name, handler }]

const testsText = readdirSync("test", { recursive: true })
  .filter((f) => String(f).endsWith(".test.ts"))
  .map((f) => readFileSync(join("test", String(f)), "utf8"))
  .join("\n");

describe("scheduled jobs", () => {
  test("every scheduled job has a test that names it", () => {
    const untested = schedule.map((j) => j.name).filter((n) => !testsText.includes(n));
    expect(
      untested,
      `No test names these scheduled jobs: ${untested.join(", ")}. ` +
        "Add test/jobs/<name>.test.ts that runs the handler once.",
    ).toEqual([]);
  });

  test("every scheduled job records a heartbeat", () => {
    const silent = schedule.filter((j) => !j.handler.recordsHeartbeat).map((j) => j.name);
    expect(
      silent,
      `These jobs do not record what they did: ${silent.join(", ")}. ` +
        "Wrap the handler with withHeartbeat(). Record 'skipped' on any early return.",
    ).toEqual([]);
  });
});
```

## Example: every service that calls the model records usage (Ruby, Minitest)

```ruby
class UsageTrackingCoverageTest < ActiveSupport::TestCase
  # Each exemption needs a reason a reviewer can check.
  EXEMPT = {
    "ModelCheck" => "lists model ids at deploy time, sends no prompt"
  }.freeze

  test "every class that calls the model client records usage" do
    callers = Dir["app/**/*.rb"].select { File.read(_1).include?("ModelClient") }
    untracked = callers.reject do |path|
      source = File.read(path)
      source.include?("include UsageTracking") || EXEMPT.keys.any? { source.include?("class #{_1}") }
    end

    assert_empty untracked, <<~MSG
      These files call the model but do not record usage:
        #{untracked.join("\n  ")}
      Add `include UsageTracking` to the class, or add it to EXEMPT with a reason.
    MSG
  end
end
```

## Writing a good failure message

A failure message is the instruction an agent reads when it breaks the rule.
Compare:

- Weak: `Expected [] to be empty.`
- Strong: `These scheduled jobs have no test: NightlyDigestJob. Add test/jobs/nightly_digest_job_test.rb that runs it once.`

The strong message names the items, says what is wrong with them, and gives
the exact fix. An agent can act on it without reading the test.
