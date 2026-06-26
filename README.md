<p align="center">
  <img src="assets/baton.png" alt="One robot hand passing a rainbow baton to another robot hand" width="560">
</p>

<h1 align="center">baton</h1>

<p align="center"><em>Hand off the work, not the whole conversation.</em></p>

A portable spec → red-tests → implement → review loop for Claude Code. You hand off a
task and a crew of subagents runs it leg by leg, checking each baton pass before the
next runner takes off. The same files run locally, in CI (`claude-code-action`), or on
Claude Code on the web — unchanged.

```
.claude/
  commands/handoff.md     # the starter — /handoff
  agents/
    spec-writer.md        # leg 1: spec → testable acceptance criteria
    test-writer.md         # leg 2: writes failing tests (Red) — example/property/visual
    implementer.md         # leg 3: makes them pass, no test edits (Green)
    reviewer.md            # legs 4 & 6: mutation-test the tests, then adversarial review
    qa-browser.md          # leg 5: verifies the running app in a real browser (Playwright MCP)
  settings.json            # permission defaults
CLAUDE.md                  # per-project commands, gate toggles & conventions (fill this in)
```

## Use it in one project (per-repo)

1. Copy `.claude/` and `CLAUDE.md` into the repo root.
2. Fill in the commands/conventions in `CLAUDE.md`.
3. In a Claude Code session: `/handoff <issue-number | spec text>`.

## Use it everywhere (global) — the symlink pattern

Claude Code reads config from two scopes: the project (`./.claude/`) and the user
(`~/.claude/`). Anything in `~/.claude/agents/` and `~/.claude/commands/` is available
in **every** repo — the same trick as symlinking into your opencode config dir.

Keep this repo as the single source of truth and symlink it into the user scope:

```sh
# from this repo's root
mkdir -p ~/.claude/{agents,commands}
ln -s "$PWD/.claude/commands/handoff.md"    ~/.claude/commands/handoff.md
ln -s "$PWD/.claude/agents/spec-writer.md" ~/.claude/agents/spec-writer.md
ln -s "$PWD/.claude/agents/test-writer.md"  ~/.claude/agents/test-writer.md
ln -s "$PWD/.claude/agents/implementer.md"  ~/.claude/agents/implementer.md
ln -s "$PWD/.claude/agents/reviewer.md"     ~/.claude/agents/reviewer.md
ln -s "$PWD/.claude/agents/qa-browser.md"   ~/.claude/agents/qa-browser.md
```

(Symlink individual files, not the whole dir, so this merges with anything already
in `~/.claude/` instead of clobbering it.)

Now `/handoff` and the four agents work in any repo. **Per-project specifics stay in each
repo's own `CLAUDE.md`** — that's the split: generic baton behavior is global and version-
controlled here; the test command, conventions, and dev-server URL live per-repo. A
project's local `.claude/` also composes with the global one, so you can override or add
per-repo when you want.

## Note on global CLAUDE.md
`~/.claude/CLAUDE.md` loads in every project too. Keep only truly universal preferences
there (e.g. "always run the linter before finishing"). Anything project-shaped belongs in
the repo's CLAUDE.md, or it will leak into unrelated work.

## Deeper testing — two tiers
Baton supports four heavier techniques, toggled per project in `CLAUDE.md` under
"Testing gates." They split across two tiers by cost and environment:

**In-session (cheap, runs inside the pipeline):**
- **Property-based** (Hypothesis): the test-writer writes invariant tests that generate
  hundreds of inputs. Best for parsers, serializers, math, anything with a clear invariant.
- **Mutation testing** (mutmut / Stryker): the reviewer breaks the code on purpose to check
  the tests catch it. Surviving mutants = weak tests = a blocker. This is how you trust tests
  without reading them. It's slow, so enable it where test quality matters most, or move it to CI.

**Better as separate PR-triggered CI checks (expensive, environment-specific):**
- **Browser QA** (qa-browser, via the Playwright MCP server): drives the running app against
  each acceptance criterion, screenshots the result, posts a PASS/FAIL report to the PR.
- **Visual regression** (Playwright snapshots / Percy / Chromatic): screenshot-diffs the UI and
  flags unintended changes.

Why the second tier is better in CI: those gates need a browser and a running app, they take
real time, and — like your test suite — they're the *independent* checks the agent can't
self-report its way past. Run them in-session for thorough local passes; wire them as their own
workflows on the PR for hands-off autonomy.

**Playwright MCP setup** (once, for browser QA): `claude mcp add playwright npx @playwright/mcp@latest`.

A backend library enables only property-based + mutation; a web app turns the rest on too. Keep a
project lean — enable only what fits.

## How baton ships (Ship mode)
The Anchor leg lands the work according to **Ship mode** in the repo's `CLAUDE.md`:
- `auto-merge` (default) — open a PR (the durable artifact) and let CI merge it the moment
  your required PR check goes green. You never click anything; main stays always-verified. This is
  the autopilot path: fire `/handoff`, walk away, come back to a merged-and-checked main. (Bring your
  own CI — any required status check works; baton no longer ships workflow templates.)
- `pr` — open a PR and stop. A human merges. Use when every diff deserves eyes.
- `merge` — merge immediately, no CI wait. Throwaway/solo repos only; the in-session reviewer is
  then the sole gate.

If `auto-merge` is set but the repo has no required check to gate on, Baton does **not** merge
blind — it leaves a plain PR open and tells you. So the seatbelt can't silently come off.

### One-time setup for `auto-merge` (per repo)
GitHub ships with neither branch protection nor auto-merge enabled by default, so a fresh repo
needs two toggles before `auto-merge` actually gates:

```sh
# 1. allow auto-merge on the repo
gh repo edit --enable-auto-merge

# 2. require your CI check job on main (CI-gated, not human-gated)
gh api -X PUT repos/{owner}/{repo}/branches/main/protection \
  -f 'required_status_checks[strict]=true' \
  -f 'required_status_checks[contexts][]=tests' \
  -F 'enforce_admins=false' \
  -F 'required_pull_request_reviews=null' \
  -F 'restrictions=null'
```

`tests` is your CI's job name — change it to match your workflow, and add more `contexts[]` lines for
additional required checks (e.g. `browser`). `required_pull_request_reviews=null` keeps it gated on CI
but not on a human approval, which is the point of autopilot. Until both are set, `auto-merge` safely
falls back to leaving the PR open.

Note: if you run `/handoff` from your own CI workflow, a PR it opens with the default `GITHUB_TOKEN`
won't trigger your other workflows — so auto-merge would wait forever on a check that never fires.
Either run `/handoff` locally (pushes under your creds, CI fires normally) or push from CI with a PAT
instead of the default token. Locally-run pipelines are unaffected.

Heads-up for **private repos on the free plan**: GitHub won't let you require a status check there
(branch protection and rulesets both need Pro or a public repo), so `auto-merge` has nothing to gate on
and would just leave PRs open. Use Ship mode `merge` on those repos until you go Pro or public.

### Keeping branches tidy
Both `merge` and `auto-merge` leave the source branch behind by default — and with worktree fan-out
(`<repo>-wt-<issue>`, see below) those `issue-*` branches add up fast. Turn on auto-delete so the remote
prunes each branch the moment its PR merges:

```sh
gh repo edit <owner>/<repo> --delete-branch-on-merge
```

That handles the *remote*. For the *local* worktrees, `handoff bg` auto-removes a worktree once its PR
merges (see below); `handoff rm <issue>` does it manually for the rest.

## Local parallelism (optional)
`handoff.md` runs one pipeline in whatever tree you're in — it's deliberately generic so it works
identically in GitHub Actions, locally, and on the web. To fan out across issues *locally*, keep the
worktree pre-step **out** of the pipeline (it's machine-specific) and put it in your shell profile instead.
One verb (`handoff`, matching the `/handoff` command), with two subcommands:

```sh
handoff <issue>             # foreground: attached session, watch + steer.        handoff 142
handoff bg <issue> [more…]  # background: detached + logged, fire many.            handoff bg 143 144 145
handoff rm <issue>          # manual cleanup: remove a worktree + its local branch. handoff rm 142
```

`handoff` and `handoff bg` create a sibling worktree `<repo>-wt-<issue>` on branch `issue-<issue>`, so
every run works in isolation — run as many as your machine handles, none stepping on the others. They
ship identically (per the repo's Ship mode); foreground vs background only changes whether you watch it
stream or check `../baton-<issue>.log` later.

**Background auto-cleans itself.** When a `handoff bg` run finishes, it checks whether the issue's PR
actually merged (via `gh pr list --head issue-<issue> --state merged`, which still resolves after
delete-branch-on-merge prunes the branch). If merged, it removes the worktree and local branch
automatically; if **not** merged — a dropped baton, red tests, anything — it **keeps** the worktree so
you can inspect it. You never lose work to cleanup; only the successes get tidied. `handoff rm` is then
just for the leftovers (failed runs you've dealt with, or foreground worktrees).

Typical lifecycle, fanning a few issues out in the background:

```sh
handoff bg 142 143 144     # 3 isolated worktrees, 3 runs detached, each logging to ../baton-<n>.log
tail -f ../baton-142.log   # peek at one if you're curious (optional)
# … each run ships per Ship mode; on merge it auto-removes its own worktree + branch …
handoff rm 143             # only needed for any that DIDN'T merge (the log says "kept … inspect")
```

Start with foreground `handoff` until you trust the permission allowlist end-to-end — a background run
stalls silently on an un-allowed prompt — then graduate routine slices to `handoff bg`. Note: you can't
remove the worktree you're currently `cd`'d into, so run `handoff rm` from the main checkout.

The full functions live in `docs/baton-helpers.zsh`. **Source them** rather than pasting into `~/.zshrc`,
so the baton repo stays the single source of truth (same reasoning as symlinking the agents) and your
shell never drifts from canonical:

```sh
# in ~/.zshrc — guarded so a missing repo doesn't break shell startup
[ -f "$HOME/git/baton/docs/baton-helpers.zsh" ] && source "$HOME/git/baton/docs/baton-helpers.zsh"
```

## Tweak freely
This is a starting point. The handoff logic is plain English in `commands/handoff.md` and
the agent prompts — edit them as your experience dictates. The red gate (leg 2) and the
"never edit tests to pass them" rule (leg 3) are the two batons worth never dropping.
