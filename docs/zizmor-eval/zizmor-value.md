# What zizmor catches on this repo

Why run zizmor at all — and what commit #632 ("pin GitHub Actions to commit
SHAs") did and didn't fix. This is the *finding set both options share*.

| | |
|---|---|
| Baseline commit | `b16bd3f` |
| Fix commit | `12b1da4` — *pin GitHub Actions references to commit SHAs (#632)* |
| Auditor | zizmor 1.25.2, scope `.github/workflows/*.yml` |

## Headline

```
findings:  116  ───────────►  67     (after pinning)
high:       44  ───────────►   4
```

Commit #632 fixes **1 of 6** finding categories — but fixes that one completely.

| Category | Severity | Fixed by #632? |
|---|:---:|:---:|
| `unpinned-uses` | 🔴 High | ✅ fully (all 44 high) |
| `dangerous-triggers` | 🔴 High | ❌ |
| `template-injection` | 🔴 High | ❌ |
| `excessive-permissions` | 🟡 Medium | ❌ |
| `cache-poisoning` | 🟡 Medium | ❌ |
| `artipacked` | ⚪ Low | ❌ |

## What's left (~23 findings, needs design decisions)

| Category | Where |
|---|---|
| `dangerous-triggers` | `pull_request_target` (formatter.yml), `workflow_run` (nudge.yml) |
| `template-injection` | `${{ … }}` in `run:` steps |
| `excessive-permissions` | jobs on default `GITHUB_TOKEN` perms |
| `cache-poisoning` | cache restore on privileged path |
| `artipacked` | checkout without `persist-credentials: false` |

These need changing triggers / scoping tokens — not a mechanical rewrite.

## Takeaway

#632 is a correct, complete fix *for pinning* — but pinning is 1 of 6 things
zizmor checks. The auditor's real value is the **~23 remaining findings** a
single-purpose commit can't address. That's the case for running zizmor in CI
(whichever wrapper) on every PR.

## Reproduce

```bash
zizmor .github/workflows/                       # audit current state
git diff b16bd3f 12b1da4 -- .github/workflows/  # what #632 changed
```
