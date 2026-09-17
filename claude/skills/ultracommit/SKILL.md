---
name: ultracommit
description: Commit staged files, screenshot any UI changes, push, and create or update a PR against dev with the screenshots attached
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
- A browser storage-state file used by step 2 (e.g. `.claude/.auth.json`) — it holds live session cookies. It must be gitignored, never committed, and never uploaded in step 6.

Use judgement: a variable *name* or an empty/placeholder value (e.g. `API_KEY=`, `token: "<your-token>"`) is fine; a real-looking value is not.

**If you find anything that looks like a real secret — staged or unstaged — STOP immediately.** Do not commit, push, create a PR, or capture screenshots. Tell the user exactly which file(s) and line(s) look like a secret and what you found (mask the actual value, e.g. show `AKIA****`), and let them decide how to proceed. Only continue past this step when the scan is clean.

## 2. Detect UI changes and capture screenshots

This step runs against the **working tree, before anything is committed** — that is the state whose appearance we want to record. It is best-effort: **no failure here may ever stop the commit.**

### 2a. Decide whether the change is UI-visible

Build the list of changed paths from the union of:

```bash
{ git diff --name-only; git diff --cached --name-only; git ls-files --others --exclude-standard; } | sort -u
```

Treat a path as **UI-visible** if it matches the project's shape:

- **Next.js App Router** — `**/app/**/page.tsx`, `**/app/**/layout.tsx`, `**/app/**/template.tsx`, `**/components/**`, `packages/ui/**`, `**/*.css`, `tailwind.config.*`, `**/*.style.ts`
- **Flutter** — `lib/**/*.dart` (excluding generated files below), plus local UI packages such as `packages/*_ui/**`
- **Expo router** — `app/**/*.tsx`, `components/**`

Treat a path as **not UI** if it matches: `**/*.test.*`, `**/*.spec.*`, `**/api/**/route.ts`, `**/*.g.dart`, `**/*.freezed.dart`, `**/*.gr.dart`, `**/*.mocks.dart`, `prisma/migrations/**`, `**/*.md`, lockfiles (`pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`, `pubspec.lock`), `.github/**`, and any `apps/api/**` / backend-only workspace.

**If nothing UI-visible changed, skip the rest of step 2**, say so in one line (e.g. `No UI-visible changes — skipping screenshots.`), and go straight to step 3. Repos that are pure backend, library or tooling always skip.

### 2b. Work out which screens to capture

Map each changed file to a screen:

- **Next.js** — path to route: drop `src/app`, drop route groups like `(marketing)`, so `src/app/(marketing)/fund/page.tsx` → `/fund`. A changed shared component has no route of its own; pick the route(s) that import it, or fall back to `/`.
- **Flutter** — recover the route from the router table by searching for the changed widget's class name:
  ```bash
  rg -n "MyChangedScreen" --glob '*rout*.dart' --glob '*go_router*.dart' lib
  ```
  If no path is found, capture the landing screen and say that is what you did.
- **Expo router** — file-based, with group segments stripped from the URL: `app/(tabs)/profile.tsx` → the `/profile` tab, `src/app/(tabs)/(results)/results.tsx` → `/results`. That stripped path is what the deep link in step 2d takes.

**Ask the user** — do not guess — when a route contains a dynamic segment (`[slug]`, `:id`) and you have no real value, when the screen sits behind a login, or when you cannot infer a route at all. Never invent or reuse credentials.

### 2c. Honour a per-repo override if present

If `.claude/screenshots.json` exists in the repo, each key it sets wins over inference — **keys it omits still fall back to inference**, so a repo can pin the plumbing (target, AVD, app id, scheme) and still let the diff decide which screens to capture. A `notes` array, if present, is guidance for you to read, not config.

```json
{
  "target": "web",
  "devCommand": "pnpm dev",
  "readyUrl": "http://localhost:3000",
  "routes": ["/", "/fund"],
  "viewport": "1440,900",
  "storageState": ".claude/.auth.json"
}
```

A mobile project uses the same file with the emulator fields instead:

```json
{
  "target": "expo-android",
  "devCommand": "EXPO_PUBLIC_ENV=dev expo run:android",
  "avd": "Pixel_10_Pro",
  "appId": "com.example.app.dev",
  "scheme": "my-app",
  "routes": ["/results"],
  "navigate": [{ "route": "/results", "tapText": "Resultat" }]
}
```

`target` is one of `web`, `flutter-web`, `flutter-ios`, `expo-android`, `expo-ios`, `android`. Use this for projects whose dev command is non-standard (a tenant argument, a Flutter flavor, a dart-define file) rather than hardcoding project knowledge here. `navigate` gives a fallback for routes a deep link cannot reach: tap that text instead.

### 2d. Capture

Put every PNG in a throwaway directory, never in the repo:

```bash
SHOT_DIR=$(mktemp -d)
```

Tell the user before you start if the chosen recipe is a mobile one — **those take over two minutes.** Always poll for readiness rather than sleeping a fixed amount, and always tear down whatever you started.

**Web (Next.js) and Flutter web**

Start the dev server in the background, keeping its PID and log:

```bash
cd "$APP_DIR" && pnpm dev > "$SHOT_DIR/dev.log" 2>&1 &
DEV_PID=$!
curl -sf --retry 60 --retry-delay 2 --retry-all-errors -o /dev/null "$READY_URL"
```

For Flutter web, force a deterministic port instead:

```bash
flutter run -d chrome --web-port=5599 > "$SHOT_DIR/dev.log" 2>&1 &
```

Then capture each route:

```bash
npx --yes playwright@1.62.1 screenshot \
  --viewport-size="1440,900" --full-page --wait-for-timeout=1500 \
  "$READY_URL/<route>" "$SHOT_DIR/<name>.png"
```

If the run needs an authenticated session, add `--load-storage="<storageState>"`. If a route needs real navigation (click through a flow, fill a login form), write a short Playwright script into `$SHOT_DIR` and run it with `npx --yes playwright@1.62.1` instead of using the CLI — do not write that script into the repo. If any flag above is rejected, check `npx --yes playwright@1.62.1 screenshot --help` and adapt rather than giving up.

The first run may download a browser once (~120MB) if the cached build does not match the pinned version; that is expected and is cached afterwards.

Tear down:

```bash
kill "$DEV_PID" 2>/dev/null; lsof -ti:3000 | xargs -r kill 2>/dev/null
```

**Flutter on an iOS simulator**

Record whether the simulator was already booted, so you only shut down what you started:

```bash
UDID=$(xcrun simctl list devices available -j | python3 -c "import json,sys;d=json.load(sys.stdin)['devices'];print([x['udid'] for v in d.values() for x in v if 'iPhone' in x['name']][0])")
WAS_BOOTED=$(xcrun simctl list devices | grep "$UDID" | grep -c Booted)
xcrun simctl boot "$UDID" 2>/dev/null
xcrun simctl bootstatus "$UDID" -b
flutter run -d "$UDID" $FLAVOR_ARGS > "$SHOT_DIR/run.log" 2>&1 &
timeout 420 tail -F "$SHOT_DIR/run.log" | grep -m1 -q "Flutter run key commands"
xcrun simctl io "$UDID" screenshot "$SHOT_DIR/<name>.png"
```

`$FLAVOR_ARGS` comes from the repo (e.g. `--flavor sandbox --dart-define-from-file=.env`) or from `.claude/screenshots.json`. There is no simple way to drive navigation, so capture the landing screen — unless the project has an `integration_test/` directory, in which case `flutter drive` can reach a specific screen. Afterwards kill the `flutter run` process, and `xcrun simctl shutdown "$UDID"` only if `WAS_BOOTED` was 0.

**Expo — prefer the Android emulator over the iOS simulator**

Default to Android for Expo projects, and only fall back to iOS if the repo has no Android setup. The reason is navigation: **an iOS simulator cannot be tapped from here.** `xcrun simctl` has no tap command, System Events `click at` fails with `-25204`, `cliclick` and `idb` are generally not installed, pyobjc's `Quartz` is absent from system python, and keyboard events do not reach the simulator. iOS also gates `simctl openurl` on a custom scheme behind an "Open in …?" confirmation that itself needs a tap. So on iOS you can only ever photograph the launch screen — useless whenever the changed screen sits behind a tab or a push.

`adb` solves all of that, and Android debug builds are signed with the debug keystore, so none of the Apple code-signing risk applies.

```bash
ADB="$HOME/Library/Android/sdk/platform-tools/adb"
EMU="$HOME/Library/Android/sdk/emulator/emulator"
AVD=$("$EMU" -list-avds | head -1)            # or .claude/screenshots.json -> avd
WAS_RUNNING=$("$ADB" devices | grep -c emulator-)

[ "$WAS_RUNNING" = "0" ] && { "$EMU" -avd "$AVD" -no-snapshot-save > "$SHOT_DIR/emu.log" 2>&1 & }
"$ADB" wait-for-device
# Wait for the launcher, not just the device: booted != ready.
until [ "$("$ADB" shell getprop sys.boot_completed 2>/dev/null | tr -d '\r')" = "1" ]; do :; done

EXPO_PUBLIC_ENV=dev npx expo run:android > "$SHOT_DIR/run.log" 2>&1 &
```

**First check whether you can skip that build entirely.** A dev-client build loads its JS bundle from Metro, so when the app is already installed *and* the diff touches no native code, starting Metro alone delivers the new code in about a second instead of the many minutes a cold Gradle build costs:

```bash
"$ADB" shell pm list packages | grep -q "$APP_ID" && echo "installed — try the Metro path first"
```

It applies when the diff is only `.ts`/`.tsx`/`.js`/JSON/asset changes with no new native dependency (nothing added to `package.json` that ships native code, no `app.config.ts` plugin change, no `expo-doctor`-relevant install). If in doubt, or if the app then shows a stale screen, fall back to the full `expo run:android`.

```bash
"$ADB" reverse tcp:8081 tcp:8081                       # let the emulator reach the host Metro
EXPO_PUBLIC_ENV=dev npx expo start --dev-client --port 8081 > "$SHOT_DIR/metro.log" 2>&1 &
until curl -sf -o /dev/null http://localhost:8081/status; do :; done
"$ADB" shell am force-stop "$APP_ID"
"$ADB" shell am start -W -a android.intent.action.VIEW \
  -d "exp+<scheme>://expo-development-client/?url=http%3A%2F%2Flocalhost%3A8081" "$APP_ID"
until grep -q "Android Bundled" "$SHOT_DIR/metro.log"; do :; done
```

Tear down with `"$ADB" reverse --remove tcp:8081` and by killing Metro.

A cold Gradle build, when you do need one, takes many minutes — say so before you start. Poll the log for the install rather than sleeping, then drive the app:

```bash
# 1. Deep-link straight to a route (expo-router strips group segments: (tabs)/(results)/results -> /results).
#    No confirmation dialog on Android.
"$ADB" shell am start -W -a android.intent.action.VIEW -d "<scheme>:///<route>" "$APP_ID"

# 2. Or navigate by text, which beats hardcoded coordinates. Dump the hierarchy and tap the node's centre:
"$ADB" shell uiautomator dump /sdcard/ui.xml >/dev/null
"$ADB" pull /sdcard/ui.xml "$SHOT_DIR/ui.xml" >/dev/null
python3 - "$SHOT_DIR/ui.xml" "Resultat" <<'PY'
import re, sys, xml.etree.ElementTree as ET
tree, want = ET.parse(sys.argv[1]), sys.argv[2]
for n in tree.iter('node'):
    if want in (n.get('text') or '') or want in (n.get('content-desc') or ''):
        x1, y1, x2, y2 = map(int, re.findall(r'-?\d+', n.get('bounds')))
        print((x1 + x2) // 2, (y1 + y2) // 2); break
else:
    sys.exit(f"no node matching {want!r}")
PY
"$ADB" shell input tap "$CX" "$CY"

# 3. Scroll a long screen into view if needed, then capture.
"$ADB" shell input swipe 540 1600 540 600 300
"$ADB" exec-out screencap -p > "$SHOT_DIR/<name>-android.png"
```

`$APP_ID` and `<scheme>` come from `app.config.ts` (`android.package`, `scheme`) or the override file. Verify each tap landed by re-dumping or re-capturing before moving on — a tap that missed is silent. Afterwards kill the `expo run:android` process, and `"$ADB" emu kill` only if `WAS_RUNNING` was 0.

**Never destroy local state to get a better picture.** A dev device holds the user's own data, and a sparse-looking chart is not a reason to reach for a "seed" or "reset" control — those routinely wipe a table first. Capture what is there and say in the PR that the device holds little data, so a reviewer does not read thin data as a bug. If seeded data is genuinely needed, ask first.

If the app shows expo-dev-client's first-launch developer menu, dismiss it once with
`"$ADB" shell am broadcast -a "$APP_ID.DEV_MENU"` or by tapping "Continue" via the text lookup above.

**iOS fallback**

Only when Android is unavailable. Use the project's own script so env prefixes and flavors are respected (`pnpm ios`, i.e. `EXPO_PUBLIC_ENV=dev expo run:ios`). Note that `expo run:ios` skips prebuild when `/ios` already exists, so an env prefix reaches only the JS bundle — if native identifiers look stale, say so rather than re-running prebuild yourself. **Check `PRODUCT_BUNDLE_IDENTIFIER` and `DEVELOPMENT_TEAM` in `ios/*.xcodeproj/project.pbxproj` before building** if the repo pins a team. Capture with `xcrun simctl io <udid> screenshot`, and state plainly that only the launch screen could be reached.

If the target screen needs navigation on iOS and you have no tap tool, do not fake it: capture nothing, and say in the PR which screen was missed and why. Maestro (`curl -fsSL https://get.maestro.mobile.dev | bash`) drives both platforms by text and is the upgrade path — suggest it rather than installing it unasked.

Name each file after the screen and platform, e.g. `results-android.png`, `fund-web.png`.

### 2e. When capture fails

Warn and carry on. Missing simulator, failed build, dev server that will not boot, occupied port, unresolvable dynamic route — all of these are non-fatal. Report in one or two lines which screens you captured and which you could not, then continue to step 3. **Never abandon the commit because a screenshot failed.**

## 3. Analyze and group changes into logical commits

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
- Screenshots from step 2 live outside the repo and are **never** part of any commit.

Briefly tell the user the grouping you chose before committing.

## 4. Commit each group

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

## 5. Push

Push the current branch to origin:
```
git push -u origin HEAD
```

## 6. Upload the screenshots to the `pr-screenshots` branch

Skip this step entirely if step 2 produced no PNGs.

The images must appear in the PR without ever entering the feature branch diff, so they go on a dedicated branch written through the API — the working tree is never touched.

```bash
REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
SLUG=$(git branch --show-current | tr '/' '-')
STAMP=$(date +%Y%m%d-%H%M%S)
DEST="screenshots/$SLUG/$STAMP"
```

If `refs/heads/pr-screenshots` does not exist, create it as a root commit with no parents:

```bash
if ! gh api "repos/$REPO/git/refs/heads/pr-screenshots" >/dev/null 2>&1; then
  BLOB=$(gh api -X POST "repos/$REPO/git/blobs" \
    -f content="Screenshots attached to pull requests. Not part of the product." \
    -f encoding=utf-8 --jq .sha)
  TREE=$(gh api -X POST "repos/$REPO/git/trees" --input - --jq .sha <<JSON
{"tree":[{"path":"README.md","mode":"100644","type":"blob","sha":"$BLOB"}]}
JSON
)
  COMMIT=$(gh api -X POST "repos/$REPO/git/commits" --input - --jq .sha <<JSON
{"message":"chore: initialise screenshot branch","tree":"$TREE","parents":[]}
JSON
)
  gh api -X POST "repos/$REPO/git/refs" -f ref=refs/heads/pr-screenshots -f sha="$COMMIT"
fi
```

Then upload each PNG:

```bash
for f in "$SHOT_DIR"/*.png; do
  NAME=$(basename "$f")
  B64=$(base64 -i "$f" | tr -d '\n')
  gh api -X PUT "repos/$REPO/contents/$DEST/$NAME" --input - >/dev/null <<JSON
{"message":"chore: screenshots for $SLUG","content":"$B64","branch":"pr-screenshots"}
JSON
done
```

If a path already exists the API returns 422 — re-request it with `?ref=pr-screenshots`, take the returned `sha`, and include it in the payload to overwrite.

Each image's URL is then:

```
https://github.com/<REPO>/raw/pr-screenshots/<DEST>/<NAME>
```

Finally, **delete the local copies** so nothing is left on disk:

```bash
rm -rf "$SHOT_DIR"
```

## 7. Create or update the Pull Request

First check whether this branch already has an open PR:

```
gh pr list --head "$(git branch --show-current)" --state open --json number,url,baseRefName
```

The PR description must cover **all** the commits on this branch (not just the ones from this run), targeting the `dev` branch. Write a description that:
- Has a title that follows the **PR title** standard below
- Includes a "## Summary" section with 2-4 bullet points explaining what and why
- Includes a "## Changes" section listing key changes (one bullet per logical commit reads well)
- Includes a "## UI changes" section **if and only if** step 6 uploaded screenshots
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

### The "## UI changes" section

**Never embed a screenshot at full width.** A plain `![](url)` is scaled by GitHub to the full content width (~830px), so one phone screenshot renders around 830×1850 and a handful of them bury the description under a huge scroll. Use `<img width>` instead, which GitHub honours in a PR body — setting `width` alone preserves the aspect ratio, so the source PNG stays full-size for anyone who opens it.

Widths that work: **180** for a phone screenshot (roughly 1:2.2, so ~400px tall), **320** for desktop web. Four 180px columns is the practical maximum before the table overflows.

Wrap every `<img>` in an `<a href>` to the same URL. Without it a thumbnail is a dead end — with it, clicking opens full size.

Put multiple captures of one screen in a single `<table>` row so they can be compared side by side, with the variant in a header cell. Stacking them vertically forces the reviewer to scroll between things they are trying to compare:

```markdown
## UI changes

One line on how these were captured (platform, device, locale, variant), and what the reviewer should look at. Click any image for full size.

<table>
<tr>
<td align="center"><b>V</b> — week</td>
<td align="center"><b>ÅR</b> — year</td>
</tr>
<tr>
<td><a href="$B/results-week-android.png"><img width="180" alt="Results, week view" src="$B/results-week-android.png"></a></td>
<td><a href="$B/results-year-android.png"><img width="180" alt="Results, year view" src="$B/results-year-android.png"></a></td>
</tr>
</table>

<details><summary>Full-size screenshot links</summary>

- [results-week-android.png]($B/results-week-android.png)
- [results-year-android.png]($B/results-year-android.png)

</details>
```

`$B` is `https://github.com/<REPO>/raw/pr-screenshots/<DEST>`. Substitute it — never leave a placeholder in the body.

Always repeat the images as plain links in `<details>`. These repos are private, so a raw URL only renders for a logged-in viewer, and that matters more once the embeds are thumbnails. **Do not verify those URLs with unauthenticated `curl` — a private repo returns 404 and it looks like a broken link.** Check with `gh api "repos/$REPO/contents/$DEST/<name>?ref=pr-screenshots" --jq .size` instead.

Say what each image demonstrates, not just which screen it is — "the line now spans the unlogged days, which previously drew no line" earns its place in a way that "the results screen" does not. If a UI change was detected but a screen could not be captured, add a one-line note saying which, so the omission is visible rather than silent.

**When updating an existing PR**, splice this one section rather than retyping the body: fetch it with `gh pr view <n> --json body --jq .body`, replace from `## UI changes` up to the next `##`/`###` heading, and write it back with `gh pr edit <n> --body-file`. Then confirm the text outside the section is byte-identical — GitHub may append one trailing newline, which is the only diff you should accept.

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

Then review the full set of commits now on the branch (`git log --oneline origin/dev..HEAD`) and rewrite the description so it reflects **everything** the PR now contains — fold the new commits into the existing Summary/Changes rather than appending a disjointed section. Keep the existing title unless the scope materially changed. **Replace** any existing "## UI changes" section wholesale with the new one — never append a second copy. Update with:

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

Return the PR URL when done, and note which screens were captured.
