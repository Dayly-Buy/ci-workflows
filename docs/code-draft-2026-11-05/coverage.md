# Coverage (ci-workflows code draft, 2026-11-05)

## 1. Tracked files

`git ls-files` at `main` = `c2cfff3f71615f856854c2a25c1b67d9bced2874` (branch
`audit-code-draft-2026-11-05` was cut from it):

| File | Lines | Read in full | Described in how-it-works | Findings |
|---|---|---|---|---|
| `.github/workflows/react-native-ci.yml` | 358 | yes | §2–§6 | DRAFT-CI-001 … 036 |
| `README.md` | 1 | yes | §1 | DRAFT-CI-027 |

Total: 2 of 2 tracked files read. No `CLAUDE.md` or `AGENTS.md` exists in the repo.

Also read, for context only: the full history (`git log -p`, 10 commits from `a81832c` to
`c2cfff3`), summarised in how-it-works §7.

## 2. Commands run and outputs (trimmed)

All read-only. No workflow was triggered, no tag pushed, no live DaylyBuy system called.

### 2.1 Repository state

```
$ git ls-files
.github/workflows/react-native-ci.yml
README.md

$ git rev-parse HEAD origin/main
c2cfff3f71615f856854c2a25c1b67d9bced2874   (both)

$ git ls-remote --tags --heads origin
c2cfff3f71615f856854c2a25c1b67d9bced2874  refs/heads/main      # no tags

GitHub MCP list_branches(Dayly-Buy/ci-workflows)
[{"name":"main","sha":"c2cfff3…","protected":false}]
```

### 2.2 Secret scan of the whole history

```
$ git log --all -p | grep -nEi 'ghp_|github_pat_|AKIA|BEGIN (RSA|EC|OPENSSH|PRIVATE)|private_key|-----BEGIN|keystore|\.jks|google-services|service.?account|sqp_|squ_|api[_-]?key|password'
(no matches)
```

No committed secret values found. Secret *names* referenced: `SONAR_TOKEN`, `SNYK_TOKEN`,
`GITHUB_TOKEN`.

### 2.3 actionlint 1.7.7 (built from source with `go install`) + shellcheck 0.11.0 (`pip install shellcheck-py`)

```
$ actionlint -no-color -shellcheck=/usr/local/bin/shellcheck .github/workflows/react-native-ci.yml
(no output)
actionlint+shellcheck exit=0
```

No syntax, expression, type or shellcheck errors. (A first call with a wrong flag, `-color=never`,
printed usage and exited 2; it is not a result.)

### 2.4 zizmor 1.32.0 (`pip install zizmor`), offline

```
$ zizmor --offline .github/workflows/react-native-ci.yml
26 findings (1 suppressed, 12 unsafe fixes): 4 informational, 0 low, 4 medium, 17 high

$ zizmor --offline --persona=auditor .github/workflows/react-native-ci.yml
26 findings (12 unsafe fixes): 4 informational, 1 low, 4 medium, 17 high
```

Grouped (auditor persona):

| zizmor audit | Level | Lines | Draft finding |
|---|---|---|---|
| `artipacked` (×4) | warning, low confidence | 71, 219, 247, 282 | DRAFT-CI-014 |
| `template-injection` on `inputs.working_directory` (×4) | error, high confidence | 93, 142, 156, 272 | DRAFT-CI-013 |
| `template-injection` on `steps.pm.outputs.manager` (×4) | info, low confidence | 119, 121, 314, 316 | DRAFT-CI-013 (values are fixed words) |
| `unpinned-uses` (×13, blanket policy) | error, high confidence | 72, 104, 175, 187, 220, 225, 248, 283, 301, 329, 340, 345, 355 | DRAFT-CI-006 (225, 340, 345), DRAFT-CI-015 (rest) |
| `undocumented-permissions` | help | 59 | DRAFT-CI-017 |

Not run: zizmor's online audits (`impostor-commit`, `known-vulnerable-actions`,
`ref-confusion`, `stale-action-refs`), because they need a GitHub token to be handed to the tool.

### 2.5 Snyk action pin, checked by hand

```
$ git ls-remote https://github.com/snyk/actions 'refs/tags/v1*'
9adf32b1121593767fc3c057af55b55db032dc04  refs/tags/v1
9adf32b1121593767fc3c057af55b55db032dc04  refs/tags/v1.0.0
$ git ls-remote https://github.com/snyk/actions refs/heads/master
11b673cfb4b0dc0d860a1323caa199d54950dcce  refs/heads/master
```

The pinned SHA at `W:255` is the commit the `v1.0.0` tag names (so not an impostor commit).
`setup/action.yml` and `setup/setup_snyk.sh` at that commit were fetched into the scratchpad and
read: `snyk-version` defaults to `latest`; the binary and its `.sha256` come from
`downloads.snyk.io` (fallback `static.snyk.io`). See DRAFT-CI-031.

### 2.6 Callers, by read-only GitHub code search

`gh api -X GET search/code -f q='<term> org:Dayly-Buy path:.github/workflows' ` with the
text-match header, for: `react-native-ci.yml`, `ci-workflows`, `secrets: inherit`,
`pull_request_target`, `working_directory`, `run_android_build`, `run_sonar`, `run_snyk`,
`node_version`, `SONAR_TOKEN`, `permissions`, `on: push`, `pull_request`, `branches`,
`concurrency`, `TZ`.

```
react-native-ci.yml → 9 hits: 6 caller ci.yml files (dayly-buy-market, magic-link-customer,
  daylybuy-admin, DaylyBuy, daylybuy-partner-app, magic-link-market) + 3 master-book docs
  (dayly-masterbook; deliberately not read: this draft is code-only)
uses: dayly-buy/ci-workflows/.github/workflows/react-native-ci.yml@main   (all 6)
secrets: inherit → 0     pull_request_target → 0     concurrency (callers) → 0     TZ → 0
working_directory: functions → magic-link-customer, magic-link-market
SONAR_TOKEN/SNYK_TOKEN passed → dayly-buy-market only
run_sonar/run_snyk/run_android_build: false → all 6
node_version "22" → the two backends; "20" → the other four
permissions: contents: read, pull-requests: read → all 6
```

Only search fragments were seen, not whole files; the table in contracts.md §5 marks the rest
unverified.

## 3. What could not be read or run

- The callers' full workflow files, their branch protection, and any deploy workflows (outside
  this repo; only search fragments).
- Repository/org rulesets, Actions settings (allowed actions, artifact retention) and the
  collaborator list of this repo.
- Any actual workflow run or log (not triggered by design; run history not read).
- zizmor online audits (§2.4).
- The master book (`dayly-masterbook`): visible in search results but intentionally not opened.
