---
name: commit
description: Commit staged files, push, and create or update a PR against dev
---

Follow these steps precisely:

## 1. Secret scan (hard gate — do this FIRST)

Before anything else, scan both staged AND unstaged changes for secrets. Run `git diff --cached` and `git diff` (and `git status` to catch newly added files), then inspect the contents for things that must never be committed:

- API keys, access tokens, bearer tokens, OAuth client secrets, session tokens
- Private keys / certificates (`-----BEGIN ... PRIVATE KEY-----`), SSH keys
- Cloud credentials (AWS `AKIA...` access key IDs + secret keys, GCP service-account JSON, Azure keys)
- Database connection strings or URLs containing inline passwords
- `.env` / `.env.*` files (or any file) that contain real assigned secret values — not empty placeholders or `.env.example` templates
- Any high-entropy string assigned to a variable/key whose name contains `secret`, `token`, `password`, `passwd`, `apikey`, `api_key`, `private_key`, `credential`, etc.

Use judgement: a variable *name* or an empty/placeholder value (e.g. `API_KEY=`, `token: "<your-token>"`) is fine; a real-looking value is not.

**If you find anything that looks like a real secret — staged or unstaged — STOP immediately.** Do not commit, push, or create a PR. Tell the user exactly which file(s) and line(s) look like a secret and what you found (mask the actual value, e.g. show `AKIA****`), and let them decide how to proceed. Only continue past this step when the scan is clean.

## 2. Analyze and group changes into logical commits

Run `git status` and `git diff` (plus `git diff --cached` for anything already staged) to understand **all** pending changes — staged and unstaged. Run `git log --oneline -5` to see recent commit style.

Then **partition the changed files into logical groups**, where each group becomes its own commit. Do NOT default to one commit containing every file. Group by *concern*, so each commit tells a coherent story. Typical seams to split on:

- **Distinct features / tickets** — unrelated pieces of work go in separate commits.
- **Production code vs. its tests** — only split these apart when the tests cover materially different scope or stand alone; a small feature plus its own tests usually belongs together.
- **Refactors / cleanups vs. behavior changes** — a rename or reformat shouldn't ride along with a feature.
- **Tooling / config / deps / seed / docs** (chore:, build:, docs:) vs. application code.
- **Bug fixes** that are incidental to the main change.

Guidelines:
- Each commit should build/stand on its own where practical and have a message that accurately describes *just that group*.
- Don't over-split: if everything genuinely serves one change, a single commit is correct. Aim for the smallest number of commits that keeps each one coherent — quality of separation over quantity.
- Grouping is at **file granularity** (stage specific paths per commit). Interactive hunk staging (`git add -p` / `git add -i`) is not available in this environment, so if a single file mixes unrelated concerns, keep it in one commit and pick the message that best fits its dominant change — mention the mix to the user rather than trying to split the file.
- Order the commits sensibly (e.g. refactor or config before the feature that builds on it).

Briefly tell the user the grouping you chose before committing.

## 3. Commit each group

For **each** group, in order, stage only that group's files and commit it:

```
git add <paths for this group>
git commit -m "$(cat <<'EOF'
subject line here

Optional body here.
EOF
)"
```

Each commit message must:
- Use conventional commit format (feat:, fix:, refactor:, chore:, docs:, test:, etc.)
- Have a short subject line (under 72 chars) summarizing the "why" of *that group*
- Include a body if that group is non-trivial, explaining motivation and what changed
- Does NOT include any "Co-Authored-By" lines or mention AI/Claude in any way

Repeat until every intended change is committed. Run `git status` at the end to confirm nothing meant for inclusion was left behind.

## 4. Push

Push the current branch to origin:
```
git push -u origin HEAD
```

## 5. Create or update the Pull Request

First check whether this branch already has an open PR:

```
gh pr list --head "$(git branch --show-current)" --state open --json number,url,baseRefName
```

The PR description must cover **all** the commits on this branch (not just the ones from this run), targeting the `dev` branch. Write a description that:
- Has a title that follows the **PR title** standard below
- Includes a "## Summary" section with 2-4 bullet points explaining what and why
- Includes a "## Changes" section listing key changes (one bullet per logical commit reads well)
- Does NOT mention Claude, AI, or any code assistant anywhere
- If there are arguments provided: $ARGUMENTS — use them as additional context for the PR description

### PR title

Use `<type>(<scope>): <summary> (<TICKET>)` — the shape this repo's merged PRs already follow.

- **`<type>`** — the conventional-commit type of the PR's *dominant* change: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `build`, `perf`, `style`.
- **`<scope>`** — the area touched, lowercase kebab-case: a screen or feature (`home`, `guide`, `results`, `tabs`, `craving-help`) or a component folder (`goal-form`, `nav-bar`). Omit `(<scope>)` only when the change is genuinely repo-wide; never claim a scope broader than what actually changed.
- **`<summary>`** — imperative mood, lowercase first word, no trailing period. Say what the change *does*, not which files moved: "paint the safe-area strip per tab", not "update TabLayout".
- **`(<TICKET>)`** — the issue ID, uppercased, in parentheses at the very end. Derive it from the branch name, which is issue-generated: `feature/slu-198-safe-area` → `(SLU-198)`. If the branch carries no ID, look in the commit messages and in any arguments passed to this skill. Omit it only when there is genuinely no ticket — and say so when you report the PR, so the user can add one.
- Keep the whole line under 70 characters *including* the ticket. If it does not fit, cut words from the summary — never drop the type or the ticket to make room.

Before writing the title, check what this repo actually does:

```
gh pr list --state merged --limit 15 --json title --jq '.[].title'
```

If those titles have settled on a different shape, match them instead — the repo wins over this template.

Good:
- `fix(tabs): paint the safe-area strip per tab (SLU-198)`
- `feat(craving-help): add the Kontakta SLR block (SLU-162)`
- `refactor(log-chart): draw each point with its own segment (SLU-201)`

Bad:
- `Glide the home mood dot to its new row on selection` — no type, no scope, no ticket
- `feat: updates` — says nothing
- `feat(home): Added animation.` — past tense, capitalised, trailing period
- `feat(app): tweak the mood dot (SLU-201)` — scope inflated to the whole app

### If no open PR exists — create one

```
gh pr create --base dev --title "title" --body "$(cat <<'EOF'
## Summary
...

## Changes
...
EOF
)"
```

### If an open PR already exists — update it instead of creating a new one

Do NOT run `gh pr create` (it would error or open a duplicate). Read the current PR so you build on what's there rather than discarding it:

```
gh pr view <number> --json title,body
```

Then review the full set of commits now on the branch (`git log --oneline origin/dev..HEAD`) and rewrite the description so it reflects **everything** the PR now contains — fold the new commits into the existing Summary/Changes rather than appending a disjointed section. Keep the existing title unless the scope materially changed. Update with:

```
gh pr edit <number> --title "title" --body "$(cat <<'EOF'
## Summary
...

## Changes
...
EOF
)"
```

Tell the user you updated the existing PR (vs. created a new one).

Return the PR URL when done.
