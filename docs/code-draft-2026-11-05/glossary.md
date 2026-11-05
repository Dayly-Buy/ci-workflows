# Glossary (ci-workflows code draft, 2026-11-05)

- **AAB (Android App Bundle):** the publishing format Google Play requires for new apps; Play
  builds per-device APKs from it. Produced by `bundleRelease`, not by this workflow.
- **APK:** an installable Android package. This workflow builds only a *debug* APK
  (`assembleDebug`).
- **actionlint:** static checker for GitHub Actions workflow files; with shellcheck it also lints
  `run:` scripts.
- **artifact (Actions):** a file uploaded by a run with `actions/upload-artifact`, downloadable by
  anyone with read access to the repo for the retention period.
- **artipacked:** zizmor's name for credentials persisted by `actions/checkout` into
  `.git/config`, which later steps (or uploaded folders) can read.
- **caller / called (reusable) workflow:** a workflow in another repo that says
  `uses: owner/repo/.github/workflows/x.yml@ref` is the caller; this repo's workflow is the
  called one and runs with the caller's event, token and passed secrets.
- **Corepack:** Node's shim that provides the Yarn/pnpm version named in `packageManager`.
- **`cancel-in-progress` / concurrency group:** runs sharing a group key; a new run cancels the
  older one when set to true.
- **debug signing:** Android signs debug builds with a debug key auto-generated on the build
  machine; Play rejects debug-signed and debuggable builds.
- **DEC-xxx / AUD-xxx:** decision and audit identifiers from the owner's master book, cited in this
  repo's comments and commit messages (not defined in this repo).
- **EAS (Expo Application Services):** Expo's hosted build (`eas build`) and submit
  (`eas submit`) service; caller comments say Play builds will move there (DEC-020).
- **Expo prebuild:** `expo prebuild` generates native `android/`/`ios/` projects from
  `app.json`/`app.config.*`.
- **Firestore emulator / rules tests:** local Firestore used to test `firestore.rules`; needs Java
  (21 for current firebase-tools).
- **`GITHUB_TOKEN`:** the per-job token GitHub issues; its scope is set by `permissions:`; it
  expires when the job ends.
- **IST:** India Standard Time, `Asia/Kolkata`, UTC+05:30, no daylight saving.
- **lockfile:** `package-lock.json`, `npm-shrinkwrap.json`, `yarn.lock` or `pnpm-lock.yaml`;
  pins exact dependency versions.
- **OIDC:** GitHub-issued identity token (`id-token: write`) that cloud providers accept instead of
  stored keys; not used here.
- **pinning (by SHA vs tag):** `uses: x@<40-hex commit>` is immutable; `@v4` is a tag the publisher
  can move.
- **`pull_request` vs `pull_request_target`:** the first runs fork PRs without secrets and with a
  read-only token; the second runs in the base repo's context with secrets and is dangerous if it
  checks out PR code. Callers use the first.
- **quality gate (Sonar):** Sonar's pass/fail verdict on a scan; the scan step does not wait for it
  unless asked.
- **SARIF:** standard format for static-analysis results that GitHub code scanning displays.
- **Snyk:** commercial dependency-vulnerability scanner; `snyk test` fails on vulnerabilities at or
  above `--severity-threshold`.
- **SonarCloud / SonarQube:** hosted / self-hosted code-quality and static-analysis service.
- **template injection:** `${{ … }}` expanded into `run:` script text before the shell runs, so a
  value containing shell syntax becomes code.
- **upload key / app signing key:** with Play App Signing, the developer signs uploads with an
  upload key; Google re-signs with the app signing key.
- **`versionCode`:** Android integer that must increase with every Play upload.
- **`workflow_call`:** the trigger that makes a workflow reusable.
- **zizmor:** static security auditor for GitHub Actions workflows.
