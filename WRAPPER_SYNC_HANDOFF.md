# Wrapper-Sync Automation — Handoff Doc

**Status as of:** 2026-06-12. Since 2026-06-04: composite-action refactor (tooling moved to `CleverTap/clevertap-wrapper-tooling`), recall pass + `flagged_for_review`, completion gate, headless skill invocation WORKING (verified in the 2026-06-11 stream traces). The 2026-06-11 RN run (Android 8.3.0 + iOS 7.7.1) failed post-sync iOS build: this prompt's own iOS reference code contained the non-existent `fetchInbox` selector and the Android sync copied it verbatim into `CleverTapReact.mm` (`fetchInboxWithCallback:` is the only real selector); Claude's attempts to verify against the cached native source were Bash-permission-denied (outside cwd). Fixed in tooling: corrected reference code, **mandatory source-verification via Read/Grep tools** ("verify-or-flag, never guess"), `OTHER_PLATFORM_SYNCING` coordination flag, `source_verified` output field. See `clevertap-flutter/WRAPPER_SYNC_HANDOFF.md` (the fresher doc) for current architecture.
**Owner:** @piyush-kukadiya
**Audience:** future Claude Code sessions starting fresh in this repo, and new developers picking up this project.

---

## Read this first if you're new (Claude or human)

This document is the single source of truth for "where we are right now" on the wrapper-sync automation project. If you're a fresh Claude session, read the whole thing top to bottom — it'll catch you up in ~10 minutes. If you're a new developer, read the **Overview** and **Architecture** sections, then jump to **What's pending** to see what you can pick up.

Two related docs exist alongside this one:

- **`NATIVE_RELEASE_SYNC_PLAN.md`** (same root directory) — the **design document** explaining what we're building and why. Cross-SDK reusable; Flutter/Unity teams will adapt from it. Read this if you want to understand the rationale behind design choices.
- **`piyush-kukadiya/clevertap-wrapper-tooling/TESTING.md`** — how to test the workflow without merging to a protected branch. Concrete fork-test procedure.

This doc (HANDOFF) tells you **what's done**, **what's pending**, and **how to pick up where we left off**. The other docs tell you **what we're building** and **how to test**.

---

## Overview — what this project is

When CleverTap releases new versions of its native SDKs (`clevertap-android-sdk`, `CleverTap-iOS-SDK`), every wrapper SDK (React Native, Flutter, Unity, Cordova) needs to catch up: version pins bumped, new public APIs surfaced through the bridge, any new permissions / minSdk bumps propagated. Today this is manual and easy to forget.

**The goal:** when a CleverTap maintainer wants to sync the RN wrapper with the latest native releases, they click ONE button in the wrapper repo's GitHub Actions UI. Everything else — diff, triage, surface new APIs, run lint + builds, open a structured PR — happens automatically. The only human step is reviewing the PR.

**Phase 1:** wire this up for the React Native SDK (in progress).
**Phase 2:** add Flutter, Unity, Cordova. Same tooling, same App, same pattern.

**Why "click a button" instead of fully auto?** Native releases sometimes ship Android-only or iOS-only (bug fixes). The maintainer is in the best position to say "both are out, sync now" — and a manual trigger means the same workflow handles "Android only", "iOS only", or "both" cleanly without complex auto-detect logic.

---

## Workflow structure (current — important to know before triggering)

The reusable workflow now has a deliberate ordering: **pre-sync builds → Claude sync → post-sync builds**. This protects Claude cost when the build pipeline itself is broken, and verifies Claude's edits actually compile.

```
Setup (token, checkouts, branch, Claude CLI install)
   ↓
Setup Java 17 (Android only)
   ↓
Write stub google-services.json (Android only — FCM plugin needs it)
   ↓
─── PRE-SYNC BUILD GATE ─────────────────────────────────
Android build (pre-sync)        runs on develop's current state
iOS build (pre-sync)            runs on develop's current state
   │
   │ If either fails → JOB STOPS HERE. $0 Claude cost spent.
   ↓
─── CLAUDE SYNC (skipped if skip_sync=true) ────────────
Sync Android                    Claude edits files
Sync iOS                        Claude edits files
   │
   │ If Sync fails → no commit/PR. Slack ping fires.
   ↓
─── POST-SYNC BUILD VERIFY (skipped if skip_sync=true) ─
Android build (post-sync)       on Claude-edited state
iOS build (post-sync)           on Claude-edited state
   │
   │ FAILURE HERE does NOT block PR opening (Option A).
   │ The PR opens with a `build-failed` label + warning banner
   │ in the body. Reviewer pulls the branch, fixes, pushes.
   ↓
Commit, push, open PR (gated by: Sync ran AND no Sync failed)
Cost report (always runs)
Slack ping on failure (gated on app-token success)
```

### Two flags worth knowing about

- **`skip_sync` (boolean, default false).** When `true`, skips Sync + post-sync builds. Pre-sync builds still run. Use to iterate on the build pipeline without paying Claude tokens. Each iteration costs ~5 min of CI time, $0 in Claude.
- **`model` (sonnet / opus / haiku, default sonnet).** Selects the lead reasoning model for Sync steps. Haiku is also used internally for cheap auxiliary calls regardless of choice; you can see the per-model split in the run's `claude-output-*.json`'s `modelUsage` block.
- **`release_name` (optional string).** Branch suffix override. Defaults to today's date. Pass a unique value to retry without colliding with a stuck `task/release_<date>` branch from a previous failed run.

### Option A behavior (post-sync build failure)

If the Claude sync succeeds but the post-sync build fails (Claude's edits compile-failed), the PR opens ANYWAY with a `build-failed` label and a warning banner in the PR body. Reviewer pulls the branch locally, fixes the build, pushes. The PR isn't gated on build success — the human-review gate is what catches issues.

### Intermediate changelog support

When the sync skips versions (e.g., pin is 8.1.0 and you sync to 8.3.0), the diff tool now extracts CHANGELOG entries for EVERY version strictly between old and new — not just the target. Both the target's entry and intermediates' entries land in the PR body so reviewers get the full release narrative without leaving the PR. Single-version bumps are unaffected (empty intermediates array). See `diff.json`'s `changelog` block:

```json
{
  "target_version": "8.3.0",
  "target_entry": "...",
  "intermediate_entries": [
    {"version": "8.2.0", "entry": "..."}
  ]
}
```

## System architecture at a glance

```
┌─────────────────────────────────────────────────────────────────────┐
│  Native SDK repos (unchanged — they release on their own schedule)  │
│                                                                     │
│  • CleverTap/clevertap-android-sdk    (tags: corev8.2.0, etc.)      │
│  • CleverTap/clevertap-ios-sdk        (tags: 7.7.0, etc.)           │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          │ (no automation here — releases happen normally)
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Maintainer clicks Run Workflow on:                                 │
│  CleverTap/clevertap-react-native → Actions → native-release-sync   │
│                                                                     │
│  Inputs: android_module + android_version, ios_module + ios_version │
│  (either, both, or release_name override)                           │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          │ workflow_dispatch event
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Reusable workflow in piyush-kukadiya/clevertap-wrapper-tooling     │
│  (.github/workflows/sync.yml, pinned to @v1)                        │
│                                                                     │
│  Steps:                                                             │
│   1. Check out tooling (this repo's scripts + prompts)              │
│   2. Sanity-check inputs                                            │
│   3. Mint short-lived token via the clevertap-wrapper-sync App      │
│   4. Check out wrapper repo at `develop`                            │
│   5. Create branch task/release_<release-name>                      │
│   6. Install Claude Code CLI                                        │
│   7. Run Claude headless with sync-orchestrator.md prompt:          │
│      - reads diff tool output                                       │
│      - walks triage decision tree                                   │
│      - applies surfaceable items via add-public-method recipe       │
│      - emits structured JSON log                                    │
│   8. Run lint + Android build + iOS Example build                   │
│      (each step has 3-retry self-heal loop driven by Claude)        │
│   9. Open PR on wrapper repo against `develop`                      │
│  10. Cost report + Slack ping on failure                            │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Pull request opens on CleverTap/clevertap-react-native             │
│  • Branch: task/release_<release-name>                              │
│  • Base: develop                                                    │
│  • Author: clevertap-wrapper-sync[bot]                              │
│  • Body: structured sync log (surfaced/skipped/deferred/cost)       │
│                                                                     │
│  Maintainer reviews. That's the only human step.                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The two repos involved

### 1. `CleverTap/clevertap-react-native` (this repo)

The React Native SDK itself. Hosts:

- `.claude/skills/` — six Claude Code skills (see "Skills" section below)
- `.github/workflows/native-release-sync.yml` — the dispatch button (small, just routes into the reusable workflow)
- `.github/CODEOWNERS` — default reviewer is @piyush-kukadiya
- Existing JS / Android / iOS bridge code (unchanged by this project, but it's what the automation reads and edits)
- `NATIVE_RELEASE_SYNC_PLAN.md` (root) — the design doc
- `WRAPPER_SYNC_HANDOFF.md` (this file) — handoff doc

### 2. `piyush-kukadiya/clevertap-wrapper-tooling` (public repo)

The **shared CI tooling**. Single place that holds the actual automation logic so RN, Flutter, Unity, etc. don't each maintain their own copy.

```
clevertap-wrapper-tooling/
├── README.md
├── TESTING.md                       fork-and-test procedure
├── tools/diff_native_api.py         the diff tool (Python, stdlib only)
├── .github/workflows/sync.yml       the reusable workflow
├── prompts/
│   ├── sync-orchestrator.md         Claude prompt for the sync skill (auto-apply mode)
│   ├── self-heal.md                 Claude prompt for code-level retry on lint/build failure
│   └── pr-description.md            Claude prompt to generate the PR body
└── scripts/
    ├── run-with-self-heal.sh        wraps a command in 3-retry loop
    ├── compute-cost.sh              sums tokens + posts soft-cap PR comment
    ├── open-combined-pr.sh          generates PR body + opens PR via gh CLI
    └── slack-notify.sh              POST to Slack webhook on non-recoverable failure
```

**Visibility:** PUBLIC. (Tried private; hit GitHub's "public repo cannot use private reusable workflow" restriction during fork testing. Made it public since nothing in it is secret.)

**Versioning:** tag `v1` is the moving "latest v1.x" pointer. Wrapper workflows pin to `@v1`. Switch to `@v1.0.0`-style pins for production once stable.

**Ownership note:** lives under `piyush-kukadiya` because CleverTap org owner was on leave when we started. Plan is to transfer to `CleverTap/clevertap-wrapper-tooling` later — at that point, update the `uses:` line in each wrapper repo's dispatch workflow.

---

## Skills built (Claude Code)

Six skills now live under `.claude/skills/` in this repo. They're organized as a funnel — broad to narrow:

| Skill | Purpose |
|---|---|
| `clevertap-react-native` | Broad overview — bridge architecture, feature map, where everything lives. Use when you don't know where to start. |
| `clevertap-react-native-android` | Android bridge deep dive — `CleverTapModuleImpl`, oldarch/newarch shims, event pipeline. |
| `clevertap-react-native-ios` | iOS bridge deep dive — `CleverTapReact.mm`, `RCT_EXPORT_METHOD`, AppDelegate integration, pending-event queue. |
| `clevertap-react-native-add-public-method` | Recipe for adding ONE new method end-to-end (TS spec → JS → Android impl + both shims → iOS .mm → Example app → docs). |
| `clevertap-react-native-sync-with-native-release` | Orchestrator for syncing with a new native release. Runs the diff tool, walks the triage decision tree, delegates to add-public-method. Supports `--auto-apply` mode for CI invocation. |
| `clevertap-react-native-backfill-missing-coverage` | For when the native SDK has a capability that RN never surfaced (e.g., multi-instance). Differs from add-public-method — it leads with JS API design before applying. |

All skills are auto-invokable: Claude picks the right one based on the task. They cross-reference each other so the right one surfaces for the right question.

---

## What's DONE ✅

### Code & tooling

- All six skills written and validated (frontmatter, link resolution, structural checks)
- Diff tool (`diff_native_api.py`) — 1132 lines, stdlib-only Python, verified against real Android `corev8.0.0 → corev8.1.0` (caught `+unmute` API and `minSdk 21→23` build-manifest change). Verified against iOS `7.5.0 → 7.6.0` (caught +46/-2 API delta).
- Diff tool covers four dimensions in one invocation: public API surface, build manifest (SDK levels, gradle/libs.versions.toml, AndroidManifest permissions, podspec), and the matching changelog entry as a sanity panel.
- Reusable workflow at `clevertap-wrapper-tooling/.github/workflows/sync.yml` — checks out tooling repo first, mints App token, runs sync per platform, lint + builds with self-heal, opens combined PR.
- Three Claude prompts (orchestrator, self-heal, PR description) with structured-output schemas.
- Four shell scripts (self-heal loop, cost compute, PR open, Slack notify) — all syntax-clean.
- Reference memories saved:
  - `feedback_untracked_tooling_files.md` — ignore tooling artifacts in git status
  - `feedback_cross_sdk_design_docs.md` — durable docs go in SDK repos
  - `reference_native_sdk_repos.md` — native SDK paths, tag conventions, cache locations
  - `feedback_plain_language_plans.md` — prefer plain language with code snippets

### GitHub setup (so far)

- `clevertap-wrapper-tooling` created at `piyush-kukadiya/clevertap-wrapper-tooling` (PUBLIC).
- `v1` tag points at latest main (force-updated as we iterated).
- Fork of the RN repo created at `piyush-kukadiya/clevertap-react-native` for fork-testing.
- Dispatch workflow + CODEOWNERS pushed to the fork's `task/setup-sync-automation` branch.
- Fork's default branch set to `task/setup-sync-automation` so the workflow_dispatch UI button appears.
- Smoke test passed end-to-end on the fork (the no-op variant with both modules `none`) — workflow loads correctly, sanity-check fails as designed, post-failure steps run cleanly.
- **First real Sync Android run succeeded.** With `android_module=core, android_version=8.1.0, model=sonnet`:
  - All steps up to and including "Sync Android" passed.
  - Claude actually executed, read the wrapper's pin (`8.1.0`), saw the requested version matched, and correctly exited with "no work to do" — emitting a valid structured JSON log to `claude-output-android.json`.
  - Cost report worked (~$0.05 — well under the $3 soft cap).
  - Lint step then failed because `yarn install` hadn't run on the fresh checkout (no `node_modules`); self-heal exhausted at 3 attempts. Fixed in the workflow afterward.
- The `clevertap-wrapper-sync` GitHub App created under `piyush-kukadiya` (with "Any account" visibility so it can be installed on CleverTap org repos later).
- App is installed on the fork; permissions: Contents R/W, Pull requests R/W, Actions R, Metadata R.
- All four required secrets added to the fork:
  - `CLEVERTAP_WRAPPER_SYNC_APP_ID` ✅
  - `CLEVERTAP_WRAPPER_SYNC_PRIVATE_KEY` ✅
  - `ANTHROPIC_API_KEY` ✅
  - `SLACK_WEBHOOK_URL` ⬜ (still pending — script gracefully skips when empty so not a blocker)

### Bug fixes made during iteration (chronological)

- Reusable workflow originally hardcoded `CleverTap/clevertap-${{ inputs.wrapper }}` for the checkout — broke fork testing. Switched to `${{ github.repository }}`.
- Tooling repo checkout was after sanity-check — caused post-failure scripts to be missing on disk. Moved tooling checkout to step 1.
- `compute-cost.sh` errored on missing claude-output JSONs — added graceful early-exit.
- Slack-notify step was firing even when nothing real had been attempted — gated on `steps.app-token.outcome == 'success'`.
- `working-directory: wrapper` on compute-cost step failed when the wrapper checkout was skipped. Removed (script uses absolute paths anyway).
- Model name `sonnet-4-6` was rejected by the Claude Code CLI. Switched to alias `sonnet`.
- Claude's first headless run in CI couldn't use any tools because the default permission mode requires interactive approval. Initially fixed with `--dangerously-skip-permissions`; then **tightened to `--allowed-tools` with a pinned allowlist** that pins `python3` to the exact diff-tool script path, restricts Bash to specific command prefixes, and excludes WebFetch/WebSearch entirely. Closes the `python3 -c` escape hatch.
- The `claude` invocation was redirecting stdout to a JSON file but stderr was lost — so when claude errored fast, the workflow showed no useful message. Added explicit stderr-to-file capture + grouped log dump on failure.
- `compute-cost.sh` was reading Claude's self-emitted fields (`tokens_used`, `cost_usd_estimate`) inside the result text — unreliable. Switched to the CLI envelope's authoritative `total_cost_usd` and `usage.{input,output}_tokens` fields.
- **Self-heal loop removed.** Original design wrapped lint/Android/iOS builds in a 3-retry Claude self-heal. In practice the loop attempted to "fix" things outside its scope (missing peer deps, missing tooling) and obscured root causes. Now commands run directly; first failure surfaces the real error.
- **Lint step removed entirely.** Wrapper repo's root lint script globs into `Example/` which has a broken `.eslintrc.js` referencing the deprecated `@react-native-community/eslint-config` (RN moved to `@react-native/eslint-config` in 0.74). Even on a fresh laptop clone, lint fails. The Android and iOS builds are the real correctness check anyway; lint added no value here.
- **`summarize-claude-output.py` added.** Replaces the old jq one-liner that looked at the wrong nesting level and always reported `0/0/0`. The script extracts the structured triage log from inside Claude's `.result` text (it lives in a ```json``` fenced block) and prints surfaced / skipped / deferred counts plus the per-platform cost.
- **`skip_sync` flag added.** Boolean workflow input that skips Sync + post-sync builds. Pre-sync builds still run. Lets us iterate on the build pipeline at $0 Claude cost.
- **Pre-sync build gate added.** Same build steps that used to come AFTER Sync now also run BEFORE Sync. If pre-sync fails, the job aborts — no Claude tokens spent on a broken pipeline.
- **Option A (open PR even on post-sync build failure)** implemented. Commit/Push/OpenPR steps no longer block on post-sync build outcome; they only block on Sync itself failing. PR gets a `build-failed` label and a warning banner in the body when post-sync failed. Reviewer pulls + fixes + pushes.
- **Cost cap lowered from $10 to $3 per run.** Earlier real runs cost ~$0.50–$1.20; $3 catches anomalies without triggering on normal runs.
- **`npm ci` → `npm install`** in the Example builds. Example's `package-lock.json` records the wrapper version (via `file:../` dep) at lockfile-generation time. The maintainer never regenerated it after bumping the parent — strict `npm ci` refused to proceed. `npm install --legacy-peer-deps` updates the lockfile in-place and tolerates the drift.
- **Stub `google-services.json` written before Android build.** Example's `android/app/build.gradle` applies the Google Services plugin (for FCM). The required file is gitignored. Wrote a syntactically valid stub at build time with `package_name: "com.reactnct"` matching the Example's applicationId.
- **iOS Podfile patched for SDWebImage modular headers.** First tried `use_modular_headers!` globally — broke ReactCommon (RN's internal pods already define their own modules; the global flag caused "redefinition of module ReactCommon"). Switched to per-target injection: `pod "SDWebImage", :modular_headers => true` inserted into each `target ... do ... end` block via awk (state-tracking so the `def` helper block isn't touched). Works without breaking RN.
- **iOS post-sync deletes `Podfile.lock`.** When Claude bumps `CleverTap-iOS-SDK` in the podspec, the existing Podfile.lock pins the old version and CocoaPods refuses to install (snapshot mismatch error). Deleting the lock forces re-resolution against the updated podspec; the fresh lock is committed with the auto-PR.
- **PR labels created via `gh label create --force` before opening the PR.** Fresh forks don't have labels like `auto-generated`; `gh pr create --label X` fails for missing labels. Script now ensure-creates each label first (idempotent — `--force` updates color if present). Plus a fallback: if `--label` still fails, open the PR without labels then add each via `gh pr edit --add-label` (each best-effort).
- **Bash hyphen-in-key trap.** First label-creation code used an associative array with keys like `auto-generated`. Under `set -u`, bash interprets `${arr[auto-generated]}` as `${VAR-default}` parameter expansion — looks for `$auto`, finds it unset, bails with "unbound variable: auto". Fix: switched to parallel arrays with integer indices.
- **Cross-platform coordination in orchestrator prompt.** iOS sync used to skip the iOS bridge for methods whose JS wrapper was added by Android sync earlier in the run — broken cross-platform contract. Prompt now explicitly says "your job is your platform's bridge code regardless of JS-layer state." Plus a reference implementation for the optional-callback pattern so Claude stops over-deferring overloads like `fetchInbox(callback)`.
- **Intermediate-changelog support.** When the sync skips native versions (pin 8.1.0 → target 8.3.0), the diff tool now extracts CHANGELOG entries for ALL versions strictly between. PR body renders all of them so reviewers see the full release narrative. Single-version bumps unaffected.
- **Upload post-sync APK + iOS .app as build artifacts.** Each full-sync run attaches the built APK and iOS sim `.app` to the workflow run's artifacts panel. Reviewers can download and inspect / side-load.

---

## What's PENDING 🟡

### Immediate (where we left off)

1. **Iterate on the Android pre-sync build until it's green.** As of last test, the build was failing at `:app:processDebugGoogleServices` — fixed by writing a stub `google-services.json`. Next failure will likely be SDK level, gradle plugin version, or kotlin compile errors. Trigger with `skip_sync=true` for each iteration:

   ```bash
   gh workflow run native-release-sync.yml \
     --repo piyush-kukadiya/clevertap-react-native \
     --ref task/setup-sync-automation \
     -f android_module=core -f android_version=8.2.0 \
     -f ios_module=none \
     -f skip_sync=true
   ```

   Each iteration: $0 Claude, ~5-10 min of CI runtime.

2. **Once Android pre-sync passes, add iOS to the smoke test.** Pass `-f ios_module=core -f ios_version=7.6.0 -f skip_sync=true`. iOS will probably surface its own setup issues (Pods, scheme name, xcodebuild flags).

3. **Once both pre-sync builds pass with skip_sync=true, do a real run.** Drop `skip_sync=true`. The workflow runs end-to-end: pre-sync → Claude Sync (real edits, ~$0.50-$2) → post-sync builds → PR opens.

4. **`SLACK_WEBHOOK_URL` secret** — still not added; the script silently skips when empty so it's not blocking, but add a placeholder webhook to exercise that path.

### Short-term (after the first fork test succeeds)

4. **Verify the structured output works.** Inspect `claude-output-android.json` in the workflow run's artifacts. Confirm `surfaced`, `skipped`, `deferred`, `build_propagated`, `tokens_used`, `cost_usd_estimate` are all populated.

5. **Iterate on the orchestrator prompt** based on what Claude actually does. The prompt at `clevertap-wrapper-tooling/prompts/sync-orchestrator.md` may need tightening — early-run behavior is the only way to see where it under- or over-surfaces.

6. **iOS Example app build is currently a placeholder xcodebuild line.** Verify the actual scheme name (currently assumes `CleverTapReactNativeExample`) matches what's in `Example/ios/`. Fix in `clevertap-wrapper-tooling/.github/workflows/sync.yml` if needed.

7. **Self-heal loop hasn't been tested with a real failure yet.** Introduce a deliberate lint error to confirm the 3-retry loop activates and recovers.

8. **Cost-cap PR comment hasn't been triggered.** Run something expensive enough to cross $3 (or temporarily lower `SOFT_CAP_USD` to e.g. $0.10 to test the path).

### Medium-term (graduating to production)

9. **Set up the same App + secrets on `CleverTap/clevertap-react-native`** — requires the CleverTap org owner (currently on leave) to install the App.

10. **Copy `.github/workflows/native-release-sync.yml` + `.github/CODEOWNERS` to the real repo via a PR.** Approval will be meaningful because the workflow has been proven on the fork.

11. **Make sure `develop` branch exists on `CleverTap/clevertap-react-native`.** It was restored after the team accidentally removed it; confirm it's still there before merging the dispatch workflow.

12. **Decide on App ownership transfer.** Currently owned by piyush-kukadiya. When CleverTap admin is back, optionally transfer to the CleverTap org. App ID stays the same; private key remains valid.

13. **Transfer `piyush-kukadiya/clevertap-wrapper-tooling` to `CleverTap/clevertap-wrapper-tooling`.** Update the `uses:` line in the wrapper repo's dispatch workflow afterward.

### Long-term (phase 2 and beyond)

14. **Add Flutter wrapper.** Follow the same pattern — install the App on `CleverTap/clevertap-flutter`, copy a Flutter-specific dispatch workflow, write Flutter-specific bridge skills (the mechanics differ from RN — Dart + MethodChannel vs TS + TurboModules).

15. **Same for Unity, Cordova.** Each wrapper needs its own per-platform skills but reuses the same reusable workflow + diff tool.

16. **Auto-merge for bug-fix-only PRs.** After a few months of clean runs, the team may opt-in to auto-merging PRs that are pure version-pin bumps (zero API changes). Requires confidence the system is reliable.

17. **Slash-command interactivity** (e.g., `/sync retry` in PR comments). Would require a webhook handler running on a server. Out of scope for now.

---

## Key decisions and why (so you don't relitigate)

| Decision | Rationale |
|---|---|
| Manual `workflow_dispatch` trigger (not on `repository_dispatch` from native) | Maintainer is in the best position to say "both natives are out". Auto-trigger would either delay (debounce) or fire twice (per-platform), both worse than one click. |
| One combined PR (not two per-platform PRs) | Reviewer sees the full release context in one place. Branch-name conflicts and CHANGELOG merge friction avoided. |
| `develop` as PR base (not `master`) | Team uses GitFlow-style integration; `develop` is the integration branch, `master` is the release branch. |
| Branch naming `task/release_<name>` | Matches the team's existing branch naming convention. |
| Soft cost cap at $3/run (not hard cap) | Don't fail mid-PR-creation. Report and let humans act on it. Started low; raise later if real-world runs need more. |
| Self-heal scope = code-level only, max 3 retries | Avoid runaway loops. Code-level issues (lint, simple build) are safe for Claude to fix; anything else needs humans. |
| Sonnet 4.6 as default model | Structured task; doesn't need Opus depth on every run. |
| Tooling repo PUBLIC | Required for fork testing (GitHub blocks public-calling-private). Nothing in tooling is secret. |
| GitHub App (not PAT) for auth | Cross-repo capable, short-lived tokens, bot identity in commit history, survives personnel changes. |
| "Any account" visibility on the App | Future-proofs for installing on CleverTap org repos later. |

---

## File inventory (where to look)

### In this repo (`CleverTap/clevertap-react-native`)

| Path | What it is |
|---|---|
| `.claude/skills/clevertap-react-native/` | Broad overview skill (broad/orient) |
| `.claude/skills/clevertap-react-native-android/` | Android bridge deep-dive skill |
| `.claude/skills/clevertap-react-native-ios/` | iOS bridge deep-dive skill |
| `.claude/skills/clevertap-react-native-add-public-method/` | Add-one-method recipe skill |
| `.claude/skills/clevertap-react-native-sync-with-native-release/` | Sync orchestrator skill (with auto-apply mode for CI) |
| `.claude/skills/clevertap-react-native-backfill-missing-coverage/` | Backfill-missing-API skill (multi-instance case) |
| `.github/workflows/native-release-sync.yml` | The dispatch button (workflow_dispatch entry) |
| `.github/CODEOWNERS` | Default reviewer for auto-PRs |
| `NATIVE_RELEASE_SYNC_PLAN.md` | The design doc |
| `WRAPPER_SYNC_HANDOFF.md` | This file |

### In the tooling repo (`piyush-kukadiya/clevertap-wrapper-tooling`)

| Path | What it is |
|---|---|
| `tools/diff_native_api.py` | The diff tool |
| `.github/workflows/sync.yml` | The reusable workflow |
| `prompts/sync-orchestrator.md` | Claude prompt for the sync skill in auto-apply mode |
| `prompts/self-heal.md` | Claude prompt for code-level retry on failure |
| `prompts/pr-description.md` | Claude prompt to generate the PR body |
| `scripts/run-with-self-heal.sh` | 3-retry loop wrapper |
| `scripts/compute-cost.sh` | Token sum + soft-cap PR comment |
| `scripts/open-combined-pr.sh` | PR body generation + gh pr create |
| `scripts/slack-notify.sh` | Slack failure webhook |
| `README.md` | Tooling repo overview |
| `TESTING.md` | Fork-test procedure |

### Local Claude memory (`~/.claude/projects/.../memory/`)

| File | What it is |
|---|---|
| `feedback_untracked_tooling_files.md` | Ignore tooling/IDE artifacts in `git status` |
| `feedback_cross_sdk_design_docs.md` | Cross-SDK design docs belong in the SDK repo root |
| `reference_native_sdk_repos.md` | Native SDK paths, tag conventions, cache locations |
| `feedback_plain_language_plans.md` | Plain language + code snippets preference |

---

## How to pick up where we left off

### If you're a fresh Claude session

1. **Read this document end-to-end.** It tells you what's done and what's next.
2. **Read `NATIVE_RELEASE_SYNC_PLAN.md`** for design rationale.
3. **Read `piyush-kukadiya/clevertap-wrapper-tooling/TESTING.md`** for the fork-test procedure.
4. **Check current state:**
   ```bash
   # What secrets are on the fork?
   gh secret list --repo piyush-kukadiya/clevertap-react-native

   # Has the workflow run? What was the most recent state?
   gh run list --repo piyush-kukadiya/clevertap-react-native \
       --workflow native-release-sync.yml --limit 3

   # Is the tooling repo still public?
   gh repo view piyush-kukadiya/clevertap-wrapper-tooling \
       --json visibility --jq '.visibility'

   # Is v1 tag still up to date with main?
   gh api /repos/piyush-kukadiya/clevertap-wrapper-tooling/git/refs/tags/v1 \
       --jq '.object.sha'
   ```
5. **Ask the user what they want to do next.** Likely candidates:
   - Trigger the first real fork test (item #3 in "Pending — Immediate")
   - Iterate on the orchestrator prompt
   - Set up a second wrapper (Flutter)
   - Graduate to the real CleverTap repo

### If you're a new developer

1. **Read this doc + `NATIVE_RELEASE_SYNC_PLAN.md`** to understand the system.
2. **Get GitHub access** to both `CleverTap/clevertap-react-native` (read at minimum) and your own fork (full).
3. **Get the Anthropic API key** from whoever owns the CleverTap-org Anthropic account (currently Vishvaas while Darshan is on leave).
4. **Walk the testing procedure** in `TESTING.md` on a fork — gives you hands-on familiarity with the whole pipeline.
5. **Pair with @piyush-kukadiya** for the first real run on the protected CleverTap repo.

### Useful commands

```bash
# Run the diff tool locally on a known version pair
python3 /Users/piyush.kukadiya/codebases/clevertap/clevertap-wrapper-tooling/tools/diff_native_api.py \
    --platform android --module core \
    --old-version 8.0.0 --new-version 8.1.0 \
    --local-path /Users/piyush.kukadiya/codebases/clevertap/clevertap-android-sdk

# Trigger the workflow on the fork (from terminal, instead of UI)
gh workflow run native-release-sync.yml \
    --repo piyush-kukadiya/clevertap-react-native \
    --ref task/setup-sync-automation \
    -f android_module=core -f android_version=8.1.0 \
    -f ios_module=none

# Watch a run after triggering it
gh run watch <run-id> --repo piyush-kukadiya/clevertap-react-native

# Download artifacts (the claude-output-*.json files)
gh run download <run-id> --repo piyush-kukadiya/clevertap-react-native
```

---

## Known issues / open questions

### Known issues

- The iOS Example app build step in the reusable workflow uses placeholder xcodebuild arguments. Real scheme name needs verification.
- The dispatch workflow on the fork pins to `@v1` which is a moving tag. Production should pin to a specific version (`@v1.0.0`) once stable.
- The sync skill's `--auto-apply` mode is documented in the SKILL.md but the actual Claude CLI flag passing is via prompt text (since headless Claude reads skills via context, not a CLI flag). The first real run showed Claude correctly picks up the orchestrator prompt and emits the structured JSON, so this approach works in practice.
- Claude in headless mode uses BOTH Sonnet and Haiku within a single run (Haiku for cheap auxiliary calls; Sonnet for the main reasoning). This is visible in the artifact's `modelUsage` block. Cost reporting sums all models correctly.

### Lessons learned during integration

A few gotchas worth knowing if you're setting this up elsewhere:

- **GitHub App private key — preserve newlines when pasting into secrets.** If you copy-paste the `.pem` via TextEdit / browser, line breaks can be munged and the App token-minting step fails with `Invalid keyData`. Use `cat <pem-file> | pbcopy` for a clean clipboard copy. Verify with `openssl rsa -in <pem-file> -noout -check`.
- **Public-calling-private reusable workflows are forbidden.** If the wrapper repo (caller) is public and the tooling repo (callee) is private, the workflow load fails with "workflow was not found" regardless of access settings. Either both public, or both private + access_level set appropriately. We chose public tooling.
- **Claude Code CLI `--model` accepts aliases, not versioned IDs.** Use `sonnet` / `opus` / `haiku` (or full IDs like `claude-sonnet-4-6`), but `sonnet-4-6` (no `claude-` prefix) fails silently.
- **Test with a version that differs from the current pin.** Claude correctly no-ops when the requested version matches the current pin — by design. To exercise the full pipeline, pick a version that actually changes something.
- **Headless Claude needs explicit tool permissions.** By default it asks interactively before using tools. In CI there's no human to approve, so every tool gets denied silently and Claude returns a "couldn't do anything" output. Use `--allowed-tools` with an explicit allowlist, NOT `--dangerously-skip-permissions` (which allows everything including network exfil paths). Pin `python3` to a specific script path to close the `python3 -c "import urllib..."` escape hatch.
- **Claude's structured output lives inside `.result` as a Markdown code block.** The CLI's `--output-format json` envelope wraps the response. Reliable cost/tokens come from the envelope's `total_cost_usd` and `usage.*`. The triage log Claude was instructed to emit lives inside `.result` as a fenced ```json``` block — needs extraction (`summarize-claude-output.py` does this).
- **`npm ci` fights `file:../` deps.** When the Example app depends on the wrapper via local path, the lockfile records the wrapper's version. If the wrapper has since bumped, `npm ci` refuses. Use `npm install --legacy-peer-deps` instead.
- **Google Services plugin requires `google-services.json` even when you don't need Firebase.** The file is gitignored in most projects. Write a stub at build time with the correct `package_name`.
- **Workflow_dispatch needs the workflow file on the default branch first.** When we set up the fork, we had to change the fork's default branch to `task/setup-sync-automation` so the "Run workflow" UI button would appear. Same applies for any new wrapper repo.
- **GitHub Actions step conditions like `!cancelled()`** are how you say "run even if a previous step failed, but not if the workflow was cancelled". Useful for the Option A pattern (open PR even after a failed build).

### Open questions for the team

- **App ownership long-term.** Stays with piyush-kukadiya, or transfer to CleverTap org? Transferring is one click but requires admin access.
- **Auto-merge policy for clean PRs.** Off for now; revisit after 2-3 months of clean runs.
- **Token budget governance.** Currently $3/run soft cap (intentionally conservative; raise if real runs consistently approach the ceiling). Monthly aggregate is bounded by release cadence but no formal ceiling. Should there be?
- **Failure-handling SLA.** When Slack pings about a failed run, who's on the hook to investigate? Currently informal (whoever's around).

---

## Useful workflow commands

```bash
# Build pipeline debug (zero Claude cost — iterate on build setup issues)
gh workflow run native-release-sync.yml \
  --repo piyush-kukadiya/clevertap-react-native \
  --ref task/setup-sync-automation \
  -f android_module=core -f android_version=8.2.0 \
  -f ios_module=none \
  -f skip_sync=true

# Full sync run (Claude actually does work, opens a PR)
gh workflow run native-release-sync.yml \
  --repo piyush-kukadiya/clevertap-react-native \
  --ref task/setup-sync-automation \
  -f android_module=core -f android_version=8.2.0 \
  -f ios_module=none

# Watch a triggered run
gh run list --repo piyush-kukadiya/clevertap-react-native --workflow native-release-sync.yml --limit 3
gh run watch <run-id> --repo piyush-kukadiya/clevertap-react-native

# Download Claude's output for inspection
gh run download <run-id> --repo piyush-kukadiya/clevertap-react-native

# Summarize a downloaded run's Claude output
python3 /path/to/clevertap-wrapper-tooling/scripts/summarize-claude-output.py \
        sync-outputs/claude-output-android.json
```

## Where the conversation left off (2026-06-04)

### What's working end-to-end

- App token minting (after PEM key was pasted correctly with newlines preserved)
- Claude Code CLI installs in the runner
- Sync Android runs with `--allowed-tools` allowlist; Claude actually edits files and emits a valid structured JSON output
- `summarize-claude-output.py` correctly reads the inside-result triage log; counts and per-platform cost display properly
- Cost report sums tokens + dollars across Android/iOS, reads envelope-level fields (`total_cost_usd`, `usage.*`) which are authoritative
- One `permission_denials` observed on `find ... | head -5` (allowlist correctly blocked composite piped command); Claude adapted and continued the run successfully
- Per-model breakdown in `modelUsage`: Sonnet (lead reasoning) + Haiku (cheap auxiliary calls). Both billed correctly.

### Where we're stuck right now

Pre-sync Android build, iterating. So far:

| Iteration | Failure | Fix shipped |
|---|---|---|
| 1 | `npm ci` rejected because Example's lockfile records wrapper@3.3.0 but parent is at 4.1.0 | Switched to `npm install --legacy-peer-deps` for Example installs |
| 2 | `:app:processDebugGoogleServices` failed: `google-services.json` missing | Added "Write stub google-services.json" step before Android build |
| 3 | TBD — waiting on the next test run |

After Android pre-sync turns green, do iOS pre-sync. Then full sync.

### Latest commits on the tooling repo (most recent first)

| Commit | Summary |
|---|---|
| `afbcae5` | feat - Include intermediate version changelogs in diff + PR body |
| `49df3df` | fix - Teach Claude per-platform bridge responsibility + overload handling |
| `c6b614f` | fix - Label loop crashed under set -u due to hyphen-in-key bash trap |
| `b438ad7` | fix - iOS lockfile regen + ensure-create PR labels |
| `9b2c9ea` | fix - Replace global use_modular_headers! with per-target SDWebImage patch |
| `dddb7c1` | fix - Patch Podfile to add use_modular_headers! before pod install (later reverted to per-target) |
| `ecf909e` | feat - Upload post-sync APK and iOS .app as build artifacts |
| `dbe57af` | fix - Write stub google-services.json before Android build |
| `433ca87` | fix - Use npm install in Example builds to tolerate stale lockfile |
| `b066a81` | feat - Option A: open PR even on post-sync build failure |
| `94f2d11` | feat - Add skip_sync flag + pre-sync build gate + post-sync verify |
| `0108156` | chore - Drop lint step + its install dependency |
| `4a43021` | fix - Drop self-heal loop, fix triage-log summary display |
| `f6986b5` | security - Replace --dangerously-skip-permissions with pinned --allowed-tools |
| `36f7caf` | fix - Unblock claude tool execution + npm peer deps + real cost reporting |
| `ea9bdeb` | chore - Lower cost soft cap from $10 to $3 per run |
| `55cb5cc` | fix - Correct lint/Android/iOS prereqs |

All re-tagged to `v1` (moving pointer) after each commit. `v1` currently points at `afbcae5`.

### Latest commit on the fork's task/setup-sync-automation branch

- `0905cca` — feat - Add skip_sync dispatch input

**Next concrete step:** trigger one more full end-to-end run (drop `skip_sync`) with the latest tooling. Verify:
1. iOS sync now adds the iOS bridge (`RCT_EXPORT_METHOD`) for `fetchInbox` even though the JS wrapper already exists from Android sync.
2. Both Sync steps surface the callback overload (`fetchInbox(callback)`) instead of deferring it — the optional-callback pattern is now in the prompt.
3. The PR body includes intermediate changelog entries if syncing across version skips.

Recommended trigger command for the verification run:

```bash
gh workflow run native-release-sync.yml \
  --repo piyush-kukadiya/clevertap-react-native \
  --ref task/setup-sync-automation \
  -f android_module=core -f android_version=8.2.0 \
  -f ios_module=core -f ios_version=7.7.0 \
  -f release_name=2026-06-05-verify
```

Branches/PRs are clean on the fork as of the last check. If a stuck branch appears, see "What you have to do if it bails" in the "How to pick up where we left off" section.

---

## Pointers / further reading

- **Design rationale:** `NATIVE_RELEASE_SYNC_PLAN.md` (this repo root)
- **Test procedure:** `https://github.com/piyush-kukadiya/clevertap-wrapper-tooling/blob/main/TESTING.md`
- **Tooling repo:** `https://github.com/piyush-kukadiya/clevertap-wrapper-tooling`
- **Fork (for testing):** `https://github.com/piyush-kukadiya/clevertap-react-native`
- **GitHub App settings:** the App `clevertap-wrapper-sync` lives under `piyush-kukadiya`'s personal account settings → Developer settings → GitHub Apps.
- **Anthropic API key** is the existing CleverTap-org key; owned by Darshan (on leave); Vishvaas can issue too.

---

_If something in this doc is outdated by the time you read it, update it as part of whatever change you're making. The doc is supposed to reflect "now" — let it._
