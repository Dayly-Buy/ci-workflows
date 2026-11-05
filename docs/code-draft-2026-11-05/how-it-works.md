# ci-workflows: how it works (code draft, 2026-11-05)

Written from the code at `main` = `c2cfff3` (2026-10-12), read in full. All citations are
`path:line` against that commit. "Unverified" marks anything the code alone does not settle.
Findings referenced as `DRAFT-CI-NNN` are in [findings.md](findings.md); the caller contract is in
[contracts.md](contracts.md).

## 1. What the repository is

The repository tracks exactly two files (`git ls-files`):

| File | Lines | What it is |
|---|---|---|
| `README.md` | 1 | The heading `# ci-workflows` and nothing else (`README.md:1`). No usage notes, no list of inputs, no versioning policy. |
| `.github/workflows/react-native-ci.yml` | 358 | The only workflow: a **reusable** workflow named "Reusable React Native CI" (`.github/workflows/react-native-ci.yml:1`), triggered only by `workflow_call` (`:3-4`). |

There are **no** composite actions (`action.yml`), no scripts, no other workflows, no
`CODEOWNERS`, no `LICENSE`, no `dependabot.yml`, and no workflow that checks this repository's
own workflows. The remote has a single branch, `main`, and **no tags or releases**
(`git ls-remote --tags --heads origin` lists only `refs/heads/main`). The GitHub API reports
`main` as `"protected": false` (via the `list_branches` tool; rulesets were not checked; see
[open-questions.md](open-questions.md)).

### 1.1 Is the earlier description right?

The earlier notes said: "shared, reusable GitHub Actions workflows called by the other DaylyBuy
repos (tests, lint, builds, possibly deploys and APK releases)". From the code:

- **Shared and reusable: yes.** It is one `workflow_call` workflow (`:4`).
- **Tests and lint: yes**, if the caller turns them on (`run_lint`, `run_tests`, both default
  `false`, `:21-31`).
- **Builds: partly.** An optional job builds an Android **debug** APK (`:275-358`). There is no
  web build, no Firebase Functions build, and no iOS build.
- **Deploys: no.** Nothing in this repo deploys anything. There is no Firebase, Play, EAS,
  hosting or Functions deploy step anywhere in the file.
- **APK releases: no.** The only APK is `assembleDebug` (`:352`), uploaded as a workflow artifact
  (`:354-358`). No release build, no signing, no AAB, no upload to Google Play.
- **Also present, not mentioned in the notes:** SonarQube/SonarCloud scan (`:199-228`), Snyk
  dependency scan (`:230-273`), and Firestore security-rules tests with the emulator
  (`:158-196`).
- **Called by (from a read-only GitHub code search of the org, not by reading those repos):**
  six repos have `.github/workflows/ci.yml` with
  `uses: dayly-buy/ci-workflows/.github/workflows/react-native-ci.yml@main`:
  `dayly-buy-market`, `magic-link-customer`, `daylybuy-admin`, `DaylyBuy`,
  `daylybuy-partner-app`, `magic-link-market`. `dayly-buy-market-admin` has its own `ci.yml` that
  does **not** call this workflow. No caller was found for `daylybuy-website` (unverified: the
  search may not cover every repo). What each caller passes is summarised in
  [contracts.md §5](contracts.md#5-who-calls-it-and-what-they-pass).
- The name says "React Native", but two of the six callers are Firebase Functions backends that
  use it with `working_directory: functions`, and one (`daylybuy-admin`) is a React web panel.

## 2. Trigger, inputs, secrets, permissions, concurrency

### 2.1 Trigger

`on: workflow_call` only (`:3-4`). It cannot be run directly, on push, on a schedule or by
`workflow_dispatch`. It therefore runs **with the caller's event, the caller's `github.*`
context, the caller's `GITHUB_TOKEN`, and only the secrets the caller passes**.

### 2.2 Inputs (`:5-49`)

| Input | Type | Default | Lines | Used at |
|---|---|---|---|---|
| `node_version` | string | `"20"` | `:6-10` | `actions/setup-node` in `quality` (`:106`) and `android-build` (`:303`) |
| `working_directory` | string | `"."` | `:12-19` | `working-directory:` of `quality` steps (`:77,116,134,148,165,195`) and the Snyk run step (`:260`); also **interpolated into script text** at `:93,142,156,272` (DRAFT-CI-013). Not used by `sonar` or `android-build`. |
| `run_lint` | boolean | `false` | `:21-25` | `if:` of the lint step (`:132`) |
| `run_tests` | boolean | `false` | `:27-31` | `if:` of the test step (`:146`) and the four Firestore-rules steps (`:164,174,186,193`) |
| `run_sonar` | boolean | `false` | `:33-37` | `if:` of job `sonar` (`:203`) |
| `run_snyk` | boolean | `false` | `:39-43` | `if:` of job `snyk` (`:234`) |
| `run_android_build` | boolean | `false` | `:45-49` | `if:` of job `android-build` (`:279`) |

The input descriptions say "SonarCloud analysis" (`:34`) and "Build Android debug APK (Expo
prebuild + Gradle)" (`:46`).

### 2.3 Secrets (`:51-55`) — names only

| Secret | Required | Lines | Read at |
|---|---|---|---|
| `SONAR_TOKEN` | no | `:52-53` | env of "Fail if SONAR_TOKEN is missing" (`:212`) and of "Sonar scan" (`:227`) |
| `SNYK_TOKEN` | no | `:54-55` | env of "Fail if SNYK_TOKEN is missing" (`:240`) and of "Run Snyk" (`:262`) |
| `GITHUB_TOKEN` (automatic, not declared) | — | — | env of "Sonar scan" as `GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` (`:228`); also implicitly used by every `actions/checkout` and persisted into `.git/config` (DRAFT-CI-014) |

No secret value is committed in this repository. A search of both tracked files and of the full
git history found no tokens, keys, keystores, service-account JSON or `google-services.json`
(see [coverage.md](coverage.md)). The workflow never asks for a keystore, a Play service account,
a Firebase token, an Expo token, or `google-services.json`.

Both secrets are declared `required: false`, so a caller may omit them; the job that needs one
then fails at its own "missing" check (`:214-217`, `:242-245`).

### 2.4 Permissions (`:57-59`)

Workflow-level `permissions: contents: read, pull-requests: read`. No job overrides it. In a
called workflow these can only narrow what the caller grants; a caller that grants less than
this (e.g. only `contents: read`) makes the call fail at start-up (GitHub refuses a nested job
that asks for more than the caller allows). All six callers grant exactly these two
(caller code search). No `id-token: write`, so **no OIDC** is used anywhere.

### 2.5 Concurrency (`:61-63`)

`group: ${{ github.workflow }}-${{ github.ref }}`, `cancel-in-progress: true`. Inside a called
workflow, `github.workflow` is the **caller's** workflow name (all callers name theirs `CI`,
per search), and `github.ref` is the caller's ref (`refs/heads/<branch>` for push,
`refs/pull/<n>/merge` for pull requests). So a newer push to the same branch cancels the older
run, including on `main` (DRAFT-CI-011).

## 3. Jobs

```
quality ──┬──> sonar          (if run_sonar)
          ├──> snyk           (if run_snyk)
          └──> android-build  (if run_android_build)
```

`quality` always runs. The other three each `needs: [quality]` (`:202,233,278`), so they run
only if `quality` succeeded, and are reported as **skipped** when their input is false.

### 3.1 Job `quality` (`:66-196`)

`runs-on: ubuntu-latest` (`:68`), `timeout-minutes: 20` (`:69`). No job-level `env`, no
`defaults`, no outputs.

| # | Step | Lines | Condition | What it does |
|---|---|---|---|---|
| 1 | Checkout | `:71-72` | always | `actions/checkout@v4` (tag, not SHA). Default `fetch-depth: 1`, default `persist-credentials: true`. Checks out the caller's repo at the caller's `github.sha`. |
| 2 | Detect package manager (`id: pm`) | `:74-101` | always | Bash, `set -euo pipefail` (`:79`), in `working_directory` (`:77`). Picks the first of `package-lock.json` → npm, `npm-shrinkwrap.json` → npm, `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, else npm with no lockfile (`:80-90`). Writes `manager` (`:91`). Builds `lockfile` relative to the repo root: reads `${{ inputs.working_directory }}` **by template interpolation** (`:93`), strips one trailing `/` (`:94`), and writes empty, `<lock>` or `<wd>/<lock>` (`:95-101`). Outputs: `manager`, `lockfile`. |
| 3 | Setup Node.js | `:103-108` | always | `actions/setup-node@v4` with `node-version: inputs.node_version`, `cache: <manager>`, `cache-dependency-path: <lockfile>`. With `lockfile` empty, setup-node falls back to searching the repo root for a lockfile and fails if none exists (DRAFT-CI-010). For `pnpm`, setup-node needs the `pnpm` binary already present; it is not yet (DRAFT-CI-009). |
| 4 | Enable Corepack | `:110-112` | `manager != 'npm'` | `corepack enable`. Runs **after** setup-node (ordering issue, DRAFT-CI-009). Default shell, no working directory. |
| 5 | Install dependencies | `:114-129` | always | Bash, `set -euo pipefail`, in `working_directory`. yarn → `yarn install --frozen-lockfile` (`:120`); pnpm → `pnpm install --frozen-lockfile` (`:122`); npm with a lockfile → `npm ci` (`:125`); npm without → `npm install` (`:127`, effectively unreachable, DRAFT-CI-010). The manager value is interpolated into the script (`:119,121`), but it can only be one of three fixed words written by step 2. Dependency lifecycle (`postinstall`) scripts run with the checked-out credentials on disk (DRAFT-CI-014). |
| 6 | Lint | `:131-143` | `inputs.run_lint` | Bash, `set -euo pipefail`. Uses `node -e` to read `./package.json` and test `scripts.lint` (`:137`). If present: `npm run lint` (`:140`) — always via npm, whatever the manager (DRAFT-CI-023). If absent: emits a `::warning` annotation "Lint did not run" and **the step succeeds** (`:142`, DRAFT-CI-002). |
| 7 | Tests | `:145-157` | `inputs.run_tests` | Same shape: checks `scripts.test` (`:151`); runs `npm test` (`:154`) or emits `::warning` "Tests did not run" and **succeeds** (`:156`). No `TZ` set, so tests run in the runner's default time zone (UTC on GitHub-hosted Ubuntu; DRAFT-CI-003). No coverage output is kept. |
| 8 | Detect Firestore rules tests (`id: rules`) | `:162-171` | `inputs.run_tests` | Bash, `set -euo pipefail`, in `working_directory`. Checks `scripts['test:rules']` and writes `has=true|false` (`:170`). Output: `has`. |
| 9 | Setup Java (Firestore emulator) | `:173-181` | `run_tests && rules.has == 'true'` | `actions/setup-java@v4`, Temurin, Java **21** (`:181`), with a comment that firebase-tools rejects Java < 21 (`:178-180`). |
| 10 | Cache the Firestore emulator | `:185-190` | same | `actions/cache@v4`, path `~/.cache/firebase/emulators`, key `firebase-emulators-<runner.os>` — a **constant** key with no version or hash (`:190`, DRAFT-CI-012). |
| 11 | Firestore rules tests | `:192-196` | same | `npm run test:rules` in `working_directory` (`:196`). Whatever starts the emulator (e.g. `firebase emulators:exec`) must come from the caller's own `test:rules` script and dependencies; this workflow installs no `firebase-tools` (unverified what the callers' scripts do; DRAFT-CI-033). |

Note the blank line at `:197-198`; no step is missing there.

If the lint step fails, the test and rules steps do not run (default `success()` condition).
If neither `run_lint` nor `run_tests` is set, the job is checkout + install only and passes
(DRAFT-CI-002).

### 3.2 Job `sonar` (`:199-228`)

`needs: [quality]` (`:202`), `if: inputs.run_sonar` (`:203`), `ubuntu-latest`,
`timeout-minutes: 20` (`:204`). Currently off in every caller (code search: all six pass
`run_sonar: false`).

| # | Step | Lines | What it does |
|---|---|---|---|
| 1 | Fail if SONAR_TOKEN is missing | `:209-217` | Bash (no `set -e`, not needed). `SONAR_TOKEN` comes in through `env` (`:211-212`), not pasted into the script — the comment at `:206-208` cites AUD-067 for that. Exits 1 with a message if empty. Runs before checkout. |
| 2 | Checkout (full history) | `:219-222` | `actions/checkout@v4`, `fetch-depth: 0`, default `persist-credentials: true`. |
| 3 | Sonar scan | `:224-228` | `SonarSource/sonarqube-scan-action@v6` — third-party, **tag-pinned** (DRAFT-CI-006). Env `SONAR_TOKEN` and `GITHUB_TOKEN` (`:226-228`). No `with:` inputs, no `SONAR_HOST_URL`, no `projectBaseDir`, no `args`. So host, organisation and project key must come from a `sonar-project.properties` in the caller repo (unverified whether any exists), the scan runs from the repo root (ignores `working_directory`), and no quality-gate wait is requested (DRAFT-CI-007). No test coverage is passed (tests ran in another job and saved nothing). |

### 3.3 Job `snyk` (`:230-273`)

`needs: [quality]` (`:233`), `if: inputs.run_snyk` (`:234`), `ubuntu-latest`,
`timeout-minutes: 15` (`:235`). Off in every caller.

| # | Step | Lines | What it does |
|---|---|---|---|
| 1 | Fail if SNYK_TOKEN is missing | `:237-245` | Same pattern as Sonar; token via `env` (`:239-240`). |
| 2 | Checkout | `:247-248` | `actions/checkout@v4`, defaults. |
| 3 | Setup Snyk CLI | `:254-255` | `snyk/actions/setup@9adf32b1121593767fc3c057af55b55db032dc04 # v1.0.0` — the only action pinned to a full SHA. Verified with `git ls-remote https://github.com/snyk/actions`: this SHA is what both `refs/tags/v1` and `refs/tags/v1.0.0` point to (the `master` branch has since moved on). Reading `setup/action.yml` at that commit: input `snyk-version` defaults to `latest`, and `setup/setup_snyk.sh` downloads the **latest** Snyk CLI binary from `downloads.snyk.io` (fallback `static.snyk.io`) and checks it against a `.sha256` file fetched from the same host. The workflow passes no `snyk-version` (DRAFT-CI-031). |
| 4 | Run Snyk (fail on >= medium) | `:258-273` | Bash, `set -euo pipefail`, in `working_directory` (`:260`), `SNYK_TOKEN` via env (`:261-262`). Prints `snyk --version` (`:265`), then for the first lockfile found in the order `package-lock.json`, `npm-shrinkwrap.json`, `yarn.lock`, `pnpm-lock.yaml` runs `snyk test --file=<lock> --severity-threshold=medium` and `exit 0` (`:266-271`). Because of `set -e`, a non-zero `snyk test` (vulns found, or error) fails the step before `exit 0`. If no lockfile: `::error` annotation and `exit 1` (`:272-273`). `working_directory` is interpolated into that annotation text (`:272`). No SARIF, no `snyk monitor`, no artifact. |

### 3.4 Job `android-build` (`:275-358`)

`needs: [quality]` (`:278`), `if: inputs.run_android_build` (`:279`), `ubuntu-latest`,
`timeout-minutes: 45` (`:280`). Off in every caller; caller comments say "Off until Play builds
move to EAS (AUD-084, DEC-020)" and, for `DaylyBuy` and `daylybuy-partner-app`, "it cannot pass
here: prebuild needs google-services.json, which is not in the repo. Release APKs are built on
the owner's machine" (caller code search). This job **ignores `working_directory`** throughout:
every step runs at the repo root.

| # | Step | Lines | What it does |
|---|---|---|---|
| 1 | Checkout | `:282-283` | `actions/checkout@v4`, defaults. |
| 2 | Detect package manager (`id: pm`) | `:285-298` | A second, simpler copy of the detection (DRAFT-CI-021): npm if `package-lock.json` or `npm-shrinkwrap.json`, else yarn, else pnpm, else npm. Output `manager` only (no lockfile path). |
| 3 | Setup Node.js | `:300-304` | `actions/setup-node@v4`, `node_version`, `cache: <manager>`, no `cache-dependency-path`. |
| 4 | Enable Corepack | `:306-308` | if `manager != 'npm'`. After setup-node (same ordering as `quality`). |
| 5 | Install JS dependencies | `:310-320` | Bash, `set -euo pipefail`. yarn/pnpm with `--frozen-lockfile`; otherwise **`npm ci` unconditionally** (`:319`) — unlike `quality`, there is no `npm install` fallback (DRAFT-CI-020). |
| 6 | Expo prebuild | `:322-326` | `npx expo prebuild --platform android --non-interactive` (`:326`). Generates `android/` from `app.json`/`app.config.*`. No env vars or secrets are supplied, so a config that needs `google-services.json` or keys fails here (DRAFT-CI-005). |
| 7 | Setup Java (17) | `:328-332` | Temurin 17 — deliberately different from the 21 used for the emulator in `quality` (comment `:178-180`). |
| 8 | Setup Android SDK | `:339-342` | `android-actions/setup-android@v3` — third-party, tag-pinned (DRAFT-CI-006). `packages: platform-tools` only (`:342`); the comment at `:334-338` records that the action's default list included the retired `tools` package and failed (AUD-084). Gradle then downloads the platform and build-tools under the licences the action accepts. |
| 9 | Setup Gradle cache | `:344-345` | `gradle/actions/setup-gradle@v4` — tag-pinned, published by Gradle, not GitHub. Default caching of the Gradle user home. |
| 10 | Ensure gradlew is executable | `:347-348` | `chmod +x android/gradlew`. |
| 11 | Build Debug APK | `:350-352` | `./gradlew assembleDebug --no-daemon --stacktrace` in `android/`. Debug build type: debug-signed with the auto-generated debug keystore, `debuggable=true`. No release build, no AAB, no `versionCode` handling (DRAFT-CI-004). |
| 12 | Upload Debug APK artifact | `:354-358` | `actions/upload-artifact@v4`, name `app-debug-apk`, path `android/app/build/outputs/apk/debug/app-debug.apk`. No `retention-days` (repo/org default applies), default `if-no-files-found: warn` (DRAFT-CI-022). This is the workflow's only artifact. |

## 4. Tooling versions in use

| Tool | Version in code | Where |
|---|---|---|
| Runner | `ubuntu-latest` (floating) | `:68,201,232,277` |
| Node.js | input, default `"20"`; callers pass `"20"` (4 repos) or `"22"` (2 backends) | `:10`, `:106`, `:303` |
| Java (emulator) | Temurin 21 | `:177,181` |
| Java (Gradle) | Temurin 17 | `:331-332` |
| Android SDK | `platform-tools` only; platform and build-tools pulled by Gradle | `:342` |
| Snyk CLI | `latest` at run time (action default) | `:255` + snyk/actions `setup/action.yml` |
| Sonar scanner | whatever `sonarqube-scan-action@v6` bundles at run time | `:225` |
| firebase-tools / emulator | not set here; from caller's dependencies; jar cached under a fixed key | `:185-196` |

## 5. Actions referenced

| Action | Ref | Publisher | Lines | Job holds secrets? |
|---|---|---|---|---|
| `actions/checkout` | `@v4` (tag) | GitHub | `:72,220,248,283` | sonar/snyk jobs: yes (token in env of other steps) |
| `actions/setup-node` | `@v4` | GitHub | `:104,301` | no |
| `actions/setup-java` | `@v4` | GitHub | `:175,329` | no |
| `actions/cache` | `@v4` | GitHub | `:187` | no |
| `actions/upload-artifact` | `@v4` | GitHub | `:355` | no |
| `SonarSource/sonarqube-scan-action` | `@v6` (tag) | SonarSource | `:225` | **yes**: `SONAR_TOKEN`, `GITHUB_TOKEN` in its env |
| `snyk/actions/setup` | `@9adf32b…` (SHA, = v1.0.0) | Snyk | `:255` | job holds `SNYK_TOKEN` (not in this step's env) |
| `android-actions/setup-android` | `@v3` (tag) | community org | `:340` | no secret; has job `GITHUB_TOKEN` via checkout |
| `gradle/actions/setup-gradle` | `@v4` (tag) | Gradle Inc. | `:345` | no secret; has job `GITHUB_TOKEN` via checkout |

## 6. What is absent (by design or not)

- No deploy (Firebase Hosting, Functions, Firestore rules deploy, EAS, Play) and no release.
- No `TZ` / time-zone matrix; no OS or Node matrix.
- No `workflow_call` **outputs** (contracts.md §4).
- No job summary (`$GITHUB_STEP_SUMMARY`), no test-report or coverage artifact.
- No `harden-runner`, no egress control.
- No self-test of this repository (no workflow on `push`/`pull_request` here, no actionlint).
- No `CODEOWNERS`, branch protection reported off, no tags to pin to.

## 7. History (from `git log`, for context only)

| Commit | Date | Summary |
|---|---|---|
| `a81832c` | 2025-12-14 | Initial commit (README). |
| `f623cbc` | 2025-12-14 | Workflow created (176 lines). |
| `2d6bec5` | 2025-12-15 | Sonar action v5 → v6; `android-build` job added. This commit left two `if` blocks without `fi` (shell syntax errors), fixed in the next commit. |
| `782c736` | 2025-12-15 | Detection/caching reworked, `fi` restored. |
| `a32791e` | 2025-12-15 | Timeouts added; Snyk switched from `snyk/actions/node@v1` to `snyk/actions/setup@master`; Java/Android SDK setup and Expo prebuild added. |
| `95657f7` | 2026-09-25 | `working_directory` input; missing-script messages become `::warning`. |
| `1f149cf` | 2026-09-26 | Firestore rules tests (Java 17). |
| `0e8cb24` | 2026-09-26 | Emulator Java 17 → 21. |
| `b940b58` | 2026-10-11 | Snyk setup pinned to SHA; secrets read via `env`; Snyk finds the real lockfile (AUD-067). |
| `c2cfff3` | 2026-10-12 | Android SDK step asks only for `platform-tools` (AUD-084). |

Every one of these went straight to `main`, which every caller runs (DRAFT-CI-001).
