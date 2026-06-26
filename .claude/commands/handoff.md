---
description: Hand off a task to the baton crew of subagents — spec → red tests → implement → review
argument-hint: <github-issue-number | issue-url | inline spec text>
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task
---

You run baton. You take the baton — the task below — and hand it to each
specialist subagent in turn, checking the baton was passed cleanly before the next
runner takes off. You don't run the legs yourself; you make sure each handoff is good.

The task:

$ARGUMENTS

Run the legs below **in order**. Do not skip ahead. The handoffs between legs are
the whole point — if a leg's exit condition isn't met, STOP and report rather than
working around it. A dropped baton ends the run; it doesn't get quietly picked up.

Read `CLAUDE.md` first for the project's test command, lint command, and
conventions. Everything project-specific lives there; this command stays generic.

The heavy gates — mutation testing, browser QA, visual regression — are expensive and
environment-specific. Run them in-session when they're enabled and feasible, but for
hands-off CI they're often better as separate PR-triggered checks (see README). Which
gates are on is set per project in CLAUDE.md: a backend library won't enable browser QA;
a web app will. Skip any leg whose gate is off.

## Leg 1 — Spec (spec-writer)
- If the argument is a GitHub issue number or URL, fetch it: `gh issue view <n>`.
- Delegate to the **spec-writer** subagent to turn the input into a concrete
  spec with explicit, testable acceptance criteria and an explicit out-of-scope list.
- If you are running interactively (a human is watching), print the refined spec
  and wait for confirmation before continuing. If running headless (CI / web),
  proceed automatically.

## Leg 2 — Red (test-writer) (this is the most important gate)
- Delegate to the **test-writer** subagent to write tests covering the acceptance
  criteria, using the styles enabled in CLAUDE.md (example-based always; property-based
  and visual-snapshot where enabled). Tests only — no implementation.
- Run the project's test command.
- **Confirm the new tests FAIL, and fail for the right reason** (the feature does
  not exist yet), not because of import errors, syntax errors, or typos.
  - If a new test PASSES before any implementation → the test is vacuous. STOP and report.
  - If a new test ERRORS (not a clean assertion failure) → the test is malformed. STOP and report.
- Only a clean, meaningful red proves the tests are real constraints. Do not
  proceed to implementation until you have it.

## Leg 3 — Green (implementer)
- Delegate to the **implementer** subagent to make the failing tests pass.
- It must NOT modify the tests to make them pass. If a test is genuinely wrong,
  surface it for human decision rather than editing it away.
- Run the new tests → confirm green.
- Run the **full** suite → confirm no regressions. If anything else broke, fix it.

## Leg 4 — Harden the tests (mutation testing) — if enabled
- If mutation testing is enabled in CLAUDE.md, delegate to the **reviewer** to run it on
  the changed code (e.g. mutmut / Stryker).
- Surviving mutants mean the tests don't actually constrain the code. Hand them back to the
  **test-writer** to add the missing assertions, then re-run leg 3.
- This is how you trust the tests without reading them. Skip only if disabled.

## Leg 5 — Verify the running app (qa-browser) — if enabled / web UI
- If this is a web app and QA is enabled in CLAUDE.md, start the dev server (or use the
  preview URL), then delegate to the **qa-browser** subagent.
- It exercises each acceptance criterion in a real browser via the Playwright MCP server,
  runs visual-regression checks if enabled, and produces a PASS/FAIL report with screenshots.
- Any FAIL is a blocker → back to the **implementer** (leg 3). Keep the report for the PR.

## Leg 6 — Review (reviewer) (loop until clean)
- Delegate to the **reviewer** subagent (fresh, adversarial, read-only).
- It weighs the mutation results and QA report as evidence alongside its own read.
- If it raises blocking issues, hand them back to the **implementer**, then re-run the
  relevant checks and re-review. Repeat until the reviewer reports no blockers.
- Non-blocking suggestions: note them in the summary, don't gold-plate.

## Anchor leg — Ship
- Read **Ship mode** from `CLAUDE.md` (default `auto-merge` if unset).
- Open or update the PR: `gh pr create` with a body that includes the acceptance criteria,
  what changed, the QA report (if any), and the reviewer's verdict. The PR is the artifact —
  the durable record of the work — regardless of mode.
- Then land it per Ship mode:
  - **`auto-merge`** (default): enable GitHub auto-merge so CI merges it the moment the
    independent gate passes — `gh pr merge --auto --squash`. You never click anything, but
    `pr-checks.yml` still has to go green first. If the repo has no required check (auto-merge
    can't engage), DO NOT merge — leave the plain PR open and say so in the summary, so the
    fall-through to an unchecked merge is never silent.
  - **`pr`**: stop here. Leave the PR open for a human to merge.
  - **`merge`**: merge now — `gh pr merge --squash`. Only valid when CLAUDE.md explicitly sets
    this; it skips the independent gate, so the in-session reviewer is the sole check.
- Print a short summary: criteria met, unit + full-suite status, mutation result (if run),
  QA verdict (if run), review verdict, the Ship mode taken (and whether auto-merge engaged or
  fell back to a plain PR), and anything left for human judgment.

Throughout: keep changes scoped to this one task. If the task is too large to fit
cleanly (tests sprawl, the diff balloons), STOP and recommend splitting the issue.
