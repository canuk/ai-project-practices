# NNNN — <the decision, as a short statement>

## Status

Accepted (YYYY-MM-DD).

<!-- Or: Proposed. Or: Superseded by NNNN (YYYY-MM-DD). -->

## Context

What happened. Name the incident or the pattern: what broke, when, how it was
found, and what it cost. Include the numbers you have (how many files, how
many weeks, how many users saw it).

Say why the existing setup did not catch it.

## Decision

1. The rule, as a numbered list of plain statements.
2. Where the rule lives: the agent instructions entry, and the guard test file.
3. What is exempt, and why.

## Consequences

- What becomes impossible or loud (the guard fails the push that breaks the rule).
- What it costs (a status line per doc, one test per promise, slower builds by N seconds).
- What is still not covered, and how it will be noticed (an audit step, an alarm).
