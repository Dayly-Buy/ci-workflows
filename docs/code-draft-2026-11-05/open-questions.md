# Open questions (ci-workflows code draft, 2026-11-05)

Things the code in this repo cannot settle. Each says what would settle it.

1. **Who can push to `main` here, and are there rulesets?** `list_branches` reports
   `protected: false`; org/repo *rulesets* were not checked. Settle: repo Settings → Rules, and
   the collaborator/team list. (DRAFT-CI-001)
2. **Are the six caller repos private?** Affects how much a malicious change here exposes (private
   source via the read token) and who can download the APK artifact. Settle: repo visibility.
3. **Do callers' deploy workflows wait for this CI?** `daylybuy-admin` mentions
   `firebase-hosting-merge.yml` on `main`; the backends' deploy method is unknown here. Settle:
   read those workflows and branch protection required checks. (DRAFT-CI-008)
4. **Are required status checks configured in callers?** If not, every check here is advisory.
   Settle: callers' branch protection.
5. **Do any callers use Yarn or pnpm, or Yarn Berry?** Decides whether DRAFT-CI-009 / -032 bite.
   Settle: lockfiles and `packageManager` in each caller.
6. **Do the Core's tests set `TZ` themselves** (e.g. in `jest.config` or the `test` script)? If
   they pin one zone, CI's UTC default hides the other. Settle: `functions/package.json` and test
   config in `magic-link-customer`. (DRAFT-CI-003)
7. **What does each `test:rules` script run, and is `firebase-tools` a pinned devDependency?**
   (DRAFT-CI-033)
8. **Is there a `sonar-project.properties` in any caller, and which Sonar edition is meant
   (SonarCloud per the input description at `W:34`)?** (DRAFT-CI-007, -016)
9. **Which release path is decided — EAS (DEC-020) or a GitHub Actions release workflow?** And
   where do the upload keystore and Play service account live today (owner's machine per caller
   comments)? (DRAFT-CI-004)
10. **What is the repo/org artifact retention setting?** (DRAFT-CI-022)
11. **Does the org restrict allowed actions** (Settings → Actions → "Allow select actions")? That
    would mitigate DRAFT-CI-006/-015.
12. **How long do `quality` runs take on a cold cache for the Core?** (DRAFT-CI-029)
13. **Is `daylybuy-website` meant to use this workflow?** No caller found by search.
14. **Exact date GitHub stops running node20-runtime actions.** (DRAFT-CI-019)
15. **Did the zizmor online audits (impostor-commit, known-vulnerable-actions, ref-confusion) find
    anything?** Not run: they need a GitHub token, which this session does not give to third-party
    tools. The Snyk SHA was checked by hand instead (coverage.md).
