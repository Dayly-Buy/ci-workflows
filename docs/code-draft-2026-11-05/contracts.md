# ci-workflows: caller contracts (code draft, 2026-11-05)

There is one reusable workflow, `.github/workflows/react-native-ci.yml` (`W` below). Code at
`main` = `c2cfff3`.

## 1. How to call it

```yaml
jobs:
  ci:
    uses: dayly-buy/ci-workflows/.github/workflows/react-native-ci.yml@<ref>
    with: { ... }        # section 2
    secrets: { ... }     # section 3, names only
```

Requirements on the caller that the YAML imposes but does not document:

- **Permissions:** the caller must grant at least `contents: read` and `pull-requests: read`
  (`W:57-59`); a caller that grants less fails at start-up, because a called workflow cannot
  request more than its caller allows.
- **Ref:** there are no tags or releases to pin to; every known caller uses `@main`
  (DRAFT-CI-001).
- **Repository layout:** `package.json` (and normally a lockfile) in `working_directory`; for the
  Android build, an Expo app at the repo root with `app.json`/`app.config.*` (`W:322-326`).
- **Scripts:** `lint`, `test`, `test:rules` in that `package.json` if the matching checks are
  wanted (`W:137,151,169`). Missing scripts only warn (DRAFT-CI-002).
- **Rules tests:** whatever `test:rules` uses to start the Firestore emulator (e.g.
  `firebase-tools`) must be in the caller's dependencies; the workflow provides only Java 21 and a
  cache folder (`W:173-196`).
- **Sonar:** host/organisation/project key must come from the caller repo (e.g.
  `sonar-project.properties`); none is passed (`W:224-228`).

## 2. Inputs

| Name | Type | Required | Default | Effect | Lines |
|---|---|---|---|---|---|
| `node_version` | string | no | `"20"` | Node for `quality` and `android-build` | `W:6-10` |
| `working_directory` | string | no | `"."` | Where `quality` and Snyk run; ignored by `sonar` and `android-build` | `W:12-19` |
| `run_lint` | boolean | no | `false` | Run `npm run lint` if script exists, else warn | `W:21-25` |
| `run_tests` | boolean | no | `false` | Run `npm test` if script exists, else warn; also gates `test:rules` | `W:27-31` |
| `run_sonar` | boolean | no | `false` | Run job `sonar`; requires `SONAR_TOKEN` | `W:33-37` |
| `run_snyk` | boolean | no | `false` | Run job `snyk`; requires `SNYK_TOKEN` | `W:39-43` |
| `run_android_build` | boolean | no | `false` | Run job `android-build` (debug APK) | `W:45-49` |

## 3. Secrets (names only)

| Name | Declared required | Actually required when | Used by | Lines |
|---|---|---|---|---|
| `SONAR_TOKEN` | no | `run_sonar: true` (job fails without it) | `sonar` | `W:52-53`, `:212`, `:227` |
| `SNYK_TOKEN` | no | `run_snyk: true` (job fails without it) | `snyk` | `W:54-55`, `:240`, `:262` |
| `GITHUB_TOKEN` | — (automatic) | always | checkouts; Sonar step env | `W:228` |

Secrets the workflow does **not** accept, which a release path would need: Android upload
keystore and its passwords, Play service-account JSON, `google-services.json`, Expo/EAS token,
Firebase deploy credentials (DRAFT-CI-004, DRAFT-CI-005).

`secrets: inherit` is not needed and no caller uses it (caller search: 0 hits).

## 4. Outputs and artifacts

- **`workflow_call` outputs:** none (DRAFT-CI-035).
- **Job outputs (internal only):** `quality.steps.pm.outputs.manager|lockfile` (`W:91-101`),
  `quality.steps.rules.outputs.has` (`W:170`), `android-build.steps.pm.outputs.manager`
  (`W:290-298`). None is exposed to the caller.
- **Artifacts:** `app-debug-apk` → `android/app/build/outputs/apk/debug/app-debug.apk`, only when
  `run_android_build: true` (`W:354-358`).
- **Check names seen by the caller:** `<caller job id> / quality`, `/ sonar`, `/ snyk`,
  `/ android-build` (the `name:` values at `W:67,200,231,276`). Disabled jobs appear as skipped.
- **Caches written in the caller's repo:** setup-node package cache (`W:103-108`, `W:300-304`),
  `firebase-emulators-<os>` (`W:185-190`), Gradle user home via setup-gradle (`W:344-345`).

## 5. Who calls it and what they pass

From a read-only GitHub code search of `org:Dayly-Buy path:.github/workflows` (fragments of each
caller's `.github/workflows/ci.yml`; the full files were not read, so anything not shown in the
fragments is unverified).

| Caller repo | Caller job | Ref | `working_directory` | lint | tests | sonar | snyk | android | node | secrets passed | caller triggers | caller permissions |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `dayly-buy-market` | `rn-ci` | `@main` | default | true | true | false | false | false | "20" | `SONAR_TOKEN`, `SNYK_TOKEN` | push `**`, PR→main | contents, pull-requests: read |
| `magic-link-customer` | `backend-ci` | `@main` | `functions` | true | true | false | false | false | "22" | none seen | push `**`, PR→main | same |
| `magic-link-market` | `backend-ci` | `@main` | `functions` | true | true | false | false | false | "22" | none seen | push `**`, PR→main | same |
| `daylybuy-admin` | `ci` | `@main` | default | true | true | false | false | false | "20" | none seen | push `**`, PR→main | same |
| `DaylyBuy` | `rn-ci` | `@main` | default | true | true | false | false | false | "20" | none seen | push `**`, PR→main | same |
| `daylybuy-partner-app` | `rn-ci` | `@main` | default | true | true | false | false | false | "20" | none seen | push `**`, PR→main | same |

Not callers (search): `dayly-buy-market-admin` (own `ci.yml`), `daylydine-app`,
`daylydine-backend`, `daylydine-admin` (own CI). `daylybuy-website`: no CI workflow found by the
search (unverified).

No caller uses `pull_request_target`, `workflow_run` or `secrets: inherit` (search: 0 hits each).
