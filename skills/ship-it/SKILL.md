---
name: ship-it
description: Validate committed code changes before they ship - independent multi-lens code review (via fresh-context subagents), tests, lint, and a fix-forward loop - with intent-touching findings escalated to the user. A pure-skill gate inspired by no-mistakes. Use when the user asks to ship, gate, validate, or clean up their changes, push safely, asks you to do a task and then validate it, or invokes /ship-it.
user-invocable: true
---

# ship-it

`ship-it` validates committed code changes before they ship. It runs an **independent** code
review (fresh-context subagents, never self-review), runs the project's tests and linters,
applies safe fixes itself, and **escalates anything that touches the user's intent** to the user -
then optionally opens a PR. No daemon, no binary: a pure skill that orchestrates subagents and git.

It mirrors the no-mistakes pipeline (`intent -> review -> test -> lint -> [PR]`) inside one agent
session. What it deliberately does NOT do (a persistent CI monitor, hard interception of
`git push`, crash-safe process isolation) needs a real binary; for that, use no-mistakes itself.

## Request

$ARGUMENTS

If the request is non-empty, the user invoked `/ship-it <task>`: **do the task first**
(Task-first below), then validate. If it is empty, validate the already-committed work on the
current branch (Validate-only).

## Two modes

- **Validate-only** - bare `/ship-it`. The user's changes are already committed; validate them.
- **Task-first** - `/ship-it <task>`:
  1. **Check scope.** Inspect `git status` first. Preserve unrelated uncommitted changes; when you
     commit, commit only what belongs to the task.
  2. **Do the work**, then **commit it on a feature branch**. If the user is on the repo's default
     branch, create a feature branch first - the gate validates committed history on a non-default branch.
  3. **Then validate**, passing the task as the intent.

## Preconditions

- Work **committed** on a **feature branch** (not the default branch).
- **Clean working tree** (commit or stash unrelated changes) - validation may run tests and apply fixes.

If a precondition fails, fix it (commit / branch / stash) before continuing, and say what you did.
Resolve the base and scope:
```sh
DEFAULT=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@origin/@@' || echo main)
BASE=$(git merge-base HEAD "$DEFAULT")
# review scope = $BASE..HEAD
```

## Intent (required, like no-mistakes `--intent`)

Write down **what the user set out to accomplish** - the goal behind the change, not a description
of the diff. Include the decisions and tradeoffs made, constraints ruled in or out, and anything
they explicitly asked for that would look surprising in the diff. Reviewers use this to tell a
**deliberate choice from a mistake**, so a thin one-liner makes them flag things the user already
chose. You know it from the conversation; state it explicitly and pass it to every reviewer.

## Step 1 - Isolate (disposable worktree)

Validate in a throwaway worktree so the user's working tree is never disturbed:
```sh
WT="$(mktemp -d)/ship-it"
git worktree add --detach "$WT" HEAD
```
Run review, tests, lint, and fixes against `$WT` (fixes become new commits on its detached HEAD).
When validation passes, bring the result onto the branch from the main checkout (tree is clean), then clean up:
```sh
git reset --hard "$(git -C "$WT" rev-parse HEAD)"
git worktree remove --force "$WT"
```
If `git worktree` is unavailable, validate in place on the branch instead and say so.

## Step 2 - Review (independent, multi-lens)

**Do NOT review your own work in your own context** - that reproduces the author's blind spots.
Spawn **fresh-context reviewer subagents in parallel**, one per lens, each reading the diff itself
(`git diff $BASE..HEAD` within `$WT`). Give every reviewer the **intent** and this exact rubric:

> Review the code changes and return structured findings with a risk assessment.
> - Read the relevant history and diff yourself. Focus on risks introduced by the changed code, but
>   inspect surrounding code, call sites, shared helpers, tests, and invariants to understand root cause.
> - Do NOT run tests during review (a dedicated test step runs after).
> - Analyze for bugs, risks, and simplification opportunities. "Simplification" = reducing complexity
>   through non-functional refactoring (dedup, clearer control flow); NOT removing features or changing behavior.
> - Treat security issues, performance regressions, breaking changes, and insufficient error handling as risks.
> - Do a full pass; do not stop at the first finding. Enumerate every material issue you can substantiate.
> - Anchor every finding to a specific file and one-indexed line.
> - Severity: "error" = must not merge; "warning" = worth addressing, can be a follow-up; "info" = nice to have.
> - Be concise and actionable. No generic advice like "add more tests". Only comment on what genuinely matters.
> - Do NOT report styling, formatting, linting, compilation, or type-checking issues (the lint step covers those).
> - If the change is clean, return an empty findings array.
> - For each finding set `action` to exactly one of:
>   - **ask-user**: about functional requirements or product behavior, or otherwise challenges the author's
>     deliberate intent. Even if it seems obviously wrong, ask the user. Examples: "this feature seems
>     unnecessary", "this hardcoded value should be configurable", "this deletion looks wrong".
>     **When in doubt, default to ask-user.**
>   - **auto-fix**: a non-functional, non user-visible issue (correctness, error handling, security,
>     performance, mechanical code quality) safely fixable without any discussion about the author's intent.
>   - **no-op**: informational only, no action needed.
> - After listing findings, set `risk_level` (low / medium / high) with a one-sentence `risk_rationale`.

Run these lenses concurrently: **correctness/bugs**, **security**, **simplification**. Merge their
findings and dedupe by file+line+gist. Each finding is `{id, severity, file, line, description, action}`.

Pick one or several lenses to match the change's size; for a tiny diff, one general-correctness reviewer is enough.

## Step 3 - Decide on findings (the gate)

Walk the merged findings by `action`:
- **no-op** - nothing to do.
- **auto-fix** - apply it yourself (a fixer subagent, or directly), following Fix-forward rules below,
  then **re-review the changed area**.
- **ask-user** - **STOP and escalate.** Relay the finding to the user **verbatim** (`id`, `file`, full
  `description`); do not paraphrase, summarize away detail, or pre-judge. Ask how to proceed, then act:
  **fix** (with their guidance), **approve** (leave as-is), or **skip**.

Loop review -> fix -> re-review until no **actionable** (non-`no-op`) findings remain. Commit fixes
as separate, clearly-messaged commits (`ship-it(review): <one-line summary>`).

### Fix-forward rules (from no-mistakes)
- First **double-check each finding is legitimate**.
- Prefer the **smallest correct root-cause fix** over patching only the reported line; if a narrow fix
  leaves the same class of bug elsewhere, fix the deeper cause.
- **Never resolve a finding by reverting the author's intentional code.** Fix forward (add validation,
  handle edge cases, tighten logic) rather than deleting; do not restore code they intentionally removed
  unless a real correctness/reliability/security issue requires it.
- **No code comments explaining the fix.**
- **Verify** the issue is resolved before finishing.

## Step 4 - Tests

Run the project's tests. Detect the command (`package.json` scripts, `Makefile`, `go.mod` ->
`go test ./...`, `pytest`, `cargo test`, ...) or use a `ship-it.yaml` if the repo has one. If tests
fail, fix forward (subagent) and re-run until green. **Report failures honestly with the real output;
never claim green without running them.**

## Step 5 - Lint / format

Run the project's linter and formatter (eslint / ruff / golangci-lint / gofmt / prettier, or
`ship-it.yaml`). Apply safe fixes and re-run.

## Step 6 - PR (optional, best-effort)

Only if `gh` is installed, authenticated, and a remote exists:
- push the branch, open a clean PR with `gh pr create` (body derived from the intent + the fixes applied),
- watch checks with `gh pr checks --watch` (or `gh run watch`); on failure, fix forward and push again.

If `gh`, the remote, or auth is missing, **stop here**: report the change is validated locally and ready
to push, and give the exact `git push` command. (This is the part a pure skill cannot fully own.)

## Step 7 - Report

Summarize concisely for the user: what was validated (which review lenses, tests, lint), the `risk_level`,
and a **list of every fix applied** - explicitly acknowledge the issues the original change missed so the
user can review them. If anything was escalated, record the user's decision. End with the PR link, or the
push command if no PR was opened.
