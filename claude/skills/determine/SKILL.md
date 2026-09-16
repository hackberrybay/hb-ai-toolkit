---
name: determine
description: Determine whether the work on the current branch is ready to be turned into a PR for human review. Runs code review, security review, comment-hygiene, reuse and CLEAN/KISS/DRY checks, then returns a READY / NOT READY verdict. Read-only — it never commits, pushes, or edits.
argument-hint: "[base-branch]"
---

Decide one thing: **is the work on this branch ready to become a PR that a human reviews?**

This skill is **read-only**. Do not edit files, do not stage, do not commit, do not push, do not open a PR. Your only output is a verdict plus the blockers behind it. The user runs the `commit` skill themselves afterwards.

Do not soften the verdict to be agreeable. A `NOT READY` that saves a reviewer an hour is the point of this skill. Equally, do not invent blockers to look thorough — a clean branch gets a clean `READY`.

## 0. Preflight

Base branch: use `$ARGUMENTS` if provided, otherwise `dev`. If `dev` does not exist, fall back to `main` and say which base you used.

```bash
git branch --show-current
git fetch origin --quiet || true
git log <base>...HEAD --oneline
git diff <base>...HEAD --stat
git status --short
```

**Scope = everything that will land in the PR**: the committed diff `<base>...HEAD` *plus* uncommitted staged and unstaged changes (the `commit` skill will sweep those in). Review all of it as one body of work.

Guards — stop and report instead of reviewing:
- No commits and no working-tree changes vs. base → nothing to determine, say so.
- The branch **is** the base branch → tell the user to branch first.

Read the commit messages to understand intent (bug fix / feature / refactor / chore). Intent shapes the bar: a bug fix wants a regression test, a refactor must not change observable behaviour, a feature must follow the patterns already in the repo.

## 1. Automated gates

Read `package.json` (or the project's equivalent) and run whichever of these exist, in this order, stopping at nothing:

- typecheck (`tsc --noEmit`, `npm run typecheck`, `check`)
- lint
- tests
- build — only if it is not slow enough to be disruptive; skip it and say so if the repo has no fast build

Any failure is a **blocker**. Report the failing command and the first real error, not the whole log. If a script does not exist, note it as *not run* rather than as a pass — never claim a check succeeded that you did not run.

## 2. Review agents (run in parallel)

Send these in a **single message** so they run concurrently. Give each one the branch, the base, and the list of changed files, and tell it to weight findings toward changed lines rather than pre-existing code.

1. **`hb-ai-toolkit:code-reviewer`** — code quality, type safety, logic, error handling, function design across every changed `.ts/.tsx/.js/.jsx` file.
2. **`hb-ai-toolkit:security-reviewer`** — OWASP review of the changed files. Always run it, even when the diff looks like pure UI: auth, input handling, data exposure and secrets hide in ordinary-looking changes.
3. **`Explore`** — reuse hunt. Brief it with every new file and every new component, hook, util, type and constant in the diff, and ask it to search the repo, very thorough, for existing equivalents: same purpose under a different name, a shared `ui`/`components`/`lib`/`utils`/`hooks` folder, an existing design-system primitive, an existing wrapper around the same library. It reports what exists; you judge.

While they run, do sections 3–5 yourself.

If the repo is not TypeScript/JavaScript, still run the security reviewer and substitute a `general-purpose` agent for the code reviewer, briefed on the actual language.

## 3. Comment hygiene

Read the full diff of added lines and judge every comment that this branch introduces. **Superfluous comments are a blocker** — they are the clearest tell that code was written by an assistant and not edited afterwards.

Flag:
- Comments that restate the next line in English — `// set the user name`, `// loop over items`, `// return the result`
- Narration of the *change* or the conversation rather than the code — `// added this to fix X`, `// NEW`, `// updated`, `// changed from useState`, `// as requested`
- Banner or divider comments (`// ===== HELPERS =====`) that the surrounding files do not already use
- JSDoc that only repeats the signature and types, adding no meaning, in a repo that does not document that layer
- Commented-out code, dead scaffolding, `console.log` left behind
- `TODO` / `FIXME` added in this diff — each one either gets done or gets a ticket before review

Keep — do not flag:
- Why a non-obvious choice was made, a workaround with its reason, a link to a ticket or spec
- Warnings about a real constraint, ordering requirement, or footgun
- Public API docs where the repo already documents its public API that way

**Calibrate against the repo, not against a rule.** Open two or three untouched files near the change and compare comment density and style. The diff should be indistinguishable from code the team already wrote.

## 4. Reuse

Using the `Explore` findings plus your own reading:

- Every **new component** — does an equivalent already exist? Was a design-system primitive available? Is a near-duplicate being introduced alongside an existing one?
- Every **new util / hook / helper** — is this a reimplementation of something in `lib`, `utils`, or `hooks` (date formatting, class merging, fetching, validation, slugify, currency, storage)?
- Every **new dependency** in `package.json` — does the repo already ship something that covers it?
- Copy-pasted blocks within the diff itself — the same logic in two files is a new duplication the branch is creating.

A new thing is fine when it is genuinely new, or when reusing would have forced the existing piece into a shape that does not fit. State the justification. An unjustified duplicate is a blocker.

## 5. CLEAN / KISS / DRY / YAGNI

Judge the diff, not the repo:

- **KISS** — is there a materially simpler shape? Needless abstraction layers, indirection with one caller, a config object where two arguments would do, a state machine where a boolean would do.
- **DRY** — repeated logic, magic values repeated across files, duplicated types that should be derived (`Pick`, `Omit`, inference).
- **YAGNI** — options, props, flags, generic parameters and exports nothing currently uses. Speculative generality is a blocker, not a nice-to-have.
- **CLEAN** — names that say what the thing is, functions doing one thing, no functions that grew past what a reader can hold, no mixed levels of abstraction in one function, no dead code, consistent file and folder placement with the rest of the repo.
- **Scope** — does the diff contain changes unrelated to the branch's stated intent? Unrelated drive-by edits make a PR harder to review; call them out so the user can split them.
- **Tests** — bug fix without a regression test, or new logic with no coverage in a repo that tests that layer, is a blocker.
- **Leftovers** — `.env` values, debug flags, commented-out imports, stray scratch files, fixture data, `.only` / `.skip` left in tests.

## 6. Verdict

Fold everything — the automated gates, the three agents, and sections 3–5 — into one decision. Deduplicate: the same issue found by two agents is one finding.

Classify each finding:
- **Blocker** — must be fixed before a human reviews this. Correctness bugs, any security finding above informational, a failing gate, superfluous comments, unjustified duplication, speculative code, missing regression test on a bug fix.
- **Non-blocking** — worth fixing, would not embarrass the branch in review.

Do not take an agent's severity at face value — verify each blocker against the actual code before you report it. If you cannot point to the file and line, it does not go in the list.

Output exactly this, and nothing after it:

```
## Determine: {branch} → {base}

**Scope:** {n} files, {n} commits{, plus uncommitted changes}
**Intent:** {Bug fix | Feature | Refactor | Chore}

### Gates
- Typecheck: {pass | fail | not run}
- Lint: {pass | fail | not run}
- Tests: {pass | fail | not run}
- Build: {pass | fail | not run | skipped}
- Code review: {n blockers, n non-blocking}
- Security review: {n blockers, n non-blocking}
- Comment hygiene: {pass | n issues}
- Reuse: {pass | n issues}
- CLEAN / KISS / DRY / YAGNI: {pass | n issues}

### Blockers ({n})
1. `{file}:{line}` — {what is wrong, and what it should be instead}

### Non-blocking ({n})
- `{file}:{line}` — {finding}

---

## VERDICT: {READY | NOT READY}

{One or two sentences. If NOT READY, name the single most important thing to fix.}
```

`READY` requires zero blockers. Non-blocking findings do not stop a `READY` — list them and let the user decide.

Then stop. Do not fix anything, do not run the `commit` skill, do not offer a plan unless the user asks. If the verdict is `NOT READY`, one short line telling the user you can fix the blockers on request is enough.
