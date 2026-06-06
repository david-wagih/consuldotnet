# zizmor evaluation — full findings & comparison

How should we run [zizmor] (GitHub Actions static analysis) on consuldotnet, and
on G-Research repos generally? This is the complete write-up: what zizmor finds,
the two ways to run it, a live head-to-head, the version↔SHA validation question,
and a requirements-driven decision guide.

[zizmor]: https://docs.zizmor.sh/

> **Bottom line:** the two options produce **identical findings** (proven: 27 SARIF
> results, same rule/file/line, on a live run). The choice is about the machinery
> *around* the findings. **Single repo → use zizmor-action (B).** **Org-wide
> platform → use the Grafana reusable workflow (A)**, accepting its adaptation cost.

---

## Contents

1. [The two options](#1-the-two-options)
2. [Why zizmor at all — what it catches](#2-why-zizmor-at-all--what-it-catches)
3. [Live head-to-head: the findings are identical](#3-live-head-to-head-the-findings-are-identical)
4. [Capability comparison](#4-capability-comparison)
5. [Adaptation costs (3 fixes for A, 0 for B)](#5-adaptation-costs-3-fixes-for-a-0-for-b)
6. [Version ↔ commit-SHA validation (uses GH_TOKEN)](#6-version--commit-sha-validation-uses-gh_token)
7. [Decision guide — pick by your requirements](#7-decision-guide--pick-by-your-requirements)
8. [Recommendation](#8-recommendation)
9. [Reproduce](#9-reproduce)

---

## 1. The two options

- **A — Grafana `reusable-zizmor.yml`** — a reusable *workflow*. For the eval we
  use the **trimmed minimal slice**: jobs `job-workflow-ref` + `analysis` only
  (the Prometheus/Grafana-Bench job and the `delete-vulnerable-branch` job
  removed).
- **B — [`zizmorcore/zizmor-action`]** — the upstream composite *action*
  (pinned `v0.5.6`).

[`zizmorcore/zizmor-action`]: https://github.com/zizmorcore/zizmor-action

| | A — Grafana reusable (trimmed) | B — `zizmorcore/zizmor-action` |
|---|---|---|
| Form | reusable **workflow** | composite **action** |
| Runs zizmor via | `uvx zizmor` (native) | **Docker image** (needs container runtime) |
| Maintained by | us, once forked into `G-Research/shared-workflows` | upstream |

Both were run on real PRs against the fork: **PR #2** (A) and **PR #3** (B).

---

## 2. Why zizmor at all — what it catches

zizmor checks six+ classes of GitHub Actions risk. A prior look at consuldotnet's
own history makes the case: commit **#632** ("pin GitHub Actions to commit SHAs")
fixed **1 of 6** categories — completely, but only one.

```
findings:  116  ───────────►  67     (after #632 pinning)
high:       44  ───────────►   4
```

| Category | Severity | Fixed by #632? |
|---|:---:|:---:|
| `unpinned-uses` | 🔴 High | ✅ fully (all 44 high) |
| `dangerous-triggers` | 🔴 High | ❌ |
| `template-injection` | 🔴 High | ❌ |
| `excessive-permissions` | 🟡 Medium | ❌ |
| `cache-poisoning` | 🟡 Medium | ❌ |
| `artipacked` | ⚪ Low | ❌ |

The remaining findings (dangerous triggers, template injection, excessive
permissions, …) need design decisions, not a mechanical rewrite — which is the
ongoing value of running zizmor in CI, whichever wrapper.

---

## 3. Live head-to-head: the findings are identical

Both wrappers shell out to the *same zizmor binary*, so the output is the same.
We ran both on the fork and compared the uploaded SARIF. **27 results each,
byte-for-byte the same `(rule, file, line)` set — despite A running zizmor
`1.24.1` and B running `1.25.2`:**

| Rule | A (Grafana, v1.24.1) | B (zizmor-action, v1.25.2) |
|---|---:|---:|
| `excessive-permissions` | 12 | 12 |
| `artipacked` | 8 | 8 |
| `template-injection` | 4 | 4 |
| `dangerous-triggers` | 2 | 2 |
| `dependabot-cooldown` | 1 | 1 |
| **TOTAL (SARIF results)** | **27** | **27** |

Set diff: `only-in-A: []`, `only-in-B: []` — equal.

Notes on the numbers:
- A's logs also print `123 findings`. That's the raw plain-text count;
  `123 − 93 suppressed − 3 ignored = 27` SARIF results uploaded to Code Scanning.
- CI runs **online audits** (token present), so counts are higher than a local
  `--offline` run (which reports `86 findings: 18 medium, 8 high`).
- **Both upload to Code Scanning.** Analyses attach to the run's ref. PR runs
  attach to `refs/pull/N/merge`; the Security → Code scanning page defaults to
  `master` (empty), so to see them, run on a branch ref (`workflow_dispatch`) or
  filter the page by branch.

**Conclusion: "better output" is the wrong axis — decide on what each does *with*
the output.**

---

## 4. Capability comparison

| Capability | A (Grafana) | B (zizmor-action) | Winner |
|---|:---:|:---:|---|
| Identical zizmor findings | ✅ | ✅ | tie |
| SARIF → Code Scanning | ✅ | ✅ (default) | tie |
| **Build gating** (`fail-severity`) | ✅ never/any/…/high | ❌ never fails on findings, only internal errors | **A** |
| **PR-comment fallback** (no-GHAS/private) | ✅ + hides stale | ❌ | **A** |
| Inline PR annotations | via Code Scanning | ✅ (cap 10, *excludes* SARIF upload) | tie |
| **Central org config** (OIDC) | ✅ one shared baseline | ❌ per-repo | **A** |
| **Config policy gate** (repos can't disable audits) | ✅ | ❌ | **A** |
| Readable findings summary in logs | ✅ dual sarif+plain | ❌ (SARIF only) | **A** |
| `persona` / `online-audits` toggles | ⚠️ via `extra-args` | ✅ first-class inputs | **B** |
| Version pinning | ✅ pinned by default | ⚠️ `latest` default | **A** |
| Hardening defaults baked in | ✅ | you write the caller | **A** |
| Runtime dependency | `uv`, no Docker | **Docker required** | **A** |
| Setup effort | **high** (host + rewire org refs) | **low** (~10 lines) | **B** |
| Ongoing maintenance | you | upstream | **B** |
| Ran here with no changes | ❌ (3 fixes) | ✅ (0 fixes) | **B** |

The three decisive differences:

1. **Gating.** zizmor-action *cannot fail the build on findings* by design — only
   on internal error. A gives `fail-severity` for free. For a repo taking external
   PRs, blocking-on-severity is often the point.
2. **Org platform vs. single repo.** A's real value is the central-config fetch +
   policy gate (one baseline everywhere; repos can't silently disable audits). B
   has none of it — dead weight for one repo, the whole point for an org.
3. **Cost to adopt.** B is ~10 lines, upstream-maintained. A only pays off after
   you fork it into `G-Research/shared-workflows` and rewire every `grafana →
   G-Research` reference (OIDC fallbacks, org gates, the config validator).

---

## 5. Adaptation costs (3 fixes for A, 0 for B)

Running A on consuldotnet surfaced **three** breakages that never show up in
Grafana's own setup — concrete evidence of the "ADAPT, don't lift" cost. B needed
**zero** fixes and passed on the first run.

1. **`github` context in a `workflow_call` input default.** `github-token`
   defaulted to `${{ github.token }}`, which GitHub rejects in reusable-workflow
   input defaults → `startup_failure`. Fixed by defaulting to `""` (consumers fall
   back to `${{ inputs.github-token || github.token }}`).
2. **`delete-vulnerable-branch` startup_failure.** That job declares
   `contents: write`; GitHub validates a reusable workflow's declared job
   permissions at startup *even for `if:`-skipped jobs*, so a caller granting only
   `contents: read` gets a hard `startup_failure`. Fixed by trimming to the
   `job-workflow-ref` + `analysis` minimal slice (also drops the Prometheus job).
3. **Missing central config → zizmor crash.** The "Set up Zizmor configuration"
   step passed `--config <temp>/zizmor.yml` whenever a ref SHA existed, without
   checking the file actually downloaded. Grafana's repo always ships
   `.github/zizmor.yml` so it never 404s; consuldotnet has none → fetch 404 →
   `--config` points at a missing file → `zizmor` errors out (exit 1). Fixed with a
   `[ -f ... ]` guard so it falls back to zizmor's default config.

**Net: A needed 3 fixes to run on a non-Grafana repo; B needed 0.**

---

## 6. Version ↔ commit-SHA validation (uses `GH_TOKEN`)

**Question:** how does zizmor validate that a pinned commit SHA matches its claimed
version, and does it use a GitHub token? **Answer: yes** — zizmor uses the GitHub
API (via `GH_TOKEN`) to resolve a tag/version to its real SHA and check it against
your pin. Three **online** (token-required) audits do this:

| Audit | Validates | Online? | Needs token |
|---|---|:---:|:---:|
| **`ref-version-mismatch`** | the `# vX.Y.Z` comment matches the pinned SHA | ✅ | ✅ |
| **`impostor-commit`** | the SHA genuinely belongs to the action's repo (not a fork impostor) | ✅ | ✅ |
| **`stale-action-refs`** | the SHA points to an actual release tag (`--pedantic`) | ✅ | ✅ |

### Live proof

A test pin with a **correct SHA but a lying comment** (`actions/checkout` SHA
`11bd719…`, which is really `v4.2.2`, commented `# v9.9.9`):

| Run | `ref-version-mismatch`? | Total findings |
|---|---|---|
| `--offline` | ❌ skipped | 4 |
| **no token**, not offline | ❌ skipped (can't reach API) | 4 |
| **`GH_TOKEN=…`** (online) | ✅ **fired** | 5 |

The exact finding (note zizmor resolved the SHA's true tag via the API):

```
8 | - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v9.9.9
  |   ---------------------------------------------------------------   ^^^^^^ points to unknown ref
  |   |
  |   is pointed to by tag v4.2.2
  = note: audit confidence → High
  = note: this finding has an auto-fix
```

Correcting the comment to `# v4.2.2` clears the finding (back to 4) — confirming
it's a genuine match check, not "any comment fails."

### Reproduce the version↔SHA check

```bash
mkdir -p /tmp/zz-test/.github/workflows
cat > /tmp/zz-test/.github/workflows/test.yml <<'YAML'
on: push
permissions: {}
jobs:
  t:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v9.9.9
YAML
cd /tmp/zz-test
zizmor --offline --pedantic .github/workflows/test.yml      # no mismatch finding
GH_TOKEN="$(gh auth token)" zizmor --pedantic .github/workflows/test.yml  # ref-version-mismatch fires
```

### Relevance to the two options

Because these are **online** audits, they only run when zizmor has a token. Both
wrappers provide one by default: **B** has `online-audits: true`; **A** passes
`GH_TOKEN` and runs online unless you add `--offline`. So **both catch
version↔SHA mismatches out of the box** — this is not a differentiator between
them, but it is a reason to keep online audits enabled.

---

## 7. Decision guide — pick by your requirements

### 30-second decision

```
Is this for ONE / a few repos you own?
  └─ yes → B   (10 lines, upstream-maintained, worked first try)
  └─ no, it's an ORG-WIDE rollout with a shared security baseline?
       └─ yes → A   (central config + policy gate + gating — accept the setup cost)

Overrides:
  • You MUST fail PRs on findings out of the box     → A
  • Runners have no Docker                            → A  (B needs Docker)
  • You want it running this week with ~zero upkeep   → B
  • You need per-repo persona/online toggles as inputs → B
```

### Map your requirement → the option

Tick the rows that matter to *you*; the column with more ticks on your rows wins.

| If you need… | A (Grafana) | B (zizmor-action) |
|---|:---:|:---:|
| Identical zizmor findings | ✅ | ✅ |
| Upload to GitHub Code Scanning | ✅ | ✅ |
| **Fail the build on findings** out of the box | ✅ | ❌ |
| **One shared config baseline across many repos** | ✅ | ❌ |
| **Policy gate** (repos can't silently disable audits) | ✅ | ❌ |
| PR-comment fallback for private/no-GHAS repos | ✅ | ❌ |
| Hardened defaults baked in | ✅ | you write them |
| Run without Docker on the runner | ✅ | ❌ |
| Pin the zizmor version by default | ✅ | ⚠️ `latest` |
| First-class `persona` / `online-audits` inputs | ⚠️ via `extra-args` | ✅ |
| **Lowest setup effort** | ❌ | ✅ |
| **Lowest maintenance** | ❌ | ✅ |
| Works on a non-Grafana repo unchanged | ❌ (3 fixes) | ✅ |

### Weighted scorecard (re-weight to taste)

| Requirement | Weight | A | B |
|---|:---:|:---:|:---:|
| Finding quality / coverage | ★★★ | tie | tie |
| Build gating built-in | ★★★ | **A** | |
| Org-wide shared config + policy | ★★★ | **A** | |
| Setup effort | ★★ | | **B** |
| Ongoing maintenance | ★★ | | **B** |
| No infra dependency (Docker) | ★ | **A** | |
| Per-repo flexibility (persona/online) | ★ | | **B** |
| Private-repo / no-GHAS coverage | ★ | **A** | |

- **Single-repo / small team** → setup + maintenance dominate → **B**.
- **Platform / security-team-owned, many repos** → gating + shared config +
  policy dominate → **A**.

---

## 8. Recommendation

- **For consuldotnet alone → B (`zizmorcore/zizmor-action`).** Identical findings,
  zero fixes to run, upstream-maintained, Code Scanning out of the box. Its one
  real gap — no build-gating — is closable later with a small SARIF-threshold step
  if you want blocking PRs.
- **For a G-Research-wide platform → A (Grafana reusable, trimmed).** The
  central-config fetch, policy gate, hardening defaults, and built-in
  `fail-severity` gating are exactly the org-level controls B lacks — worth the
  one-time cost of hosting it in `G-Research/shared-workflows` and the 3
  adaptation fixes. Drop `grafana-bench` (Prometheus) and `delete-vulnerable-branch`.

> Don't pick on findings — they're equal. Pick **A for the governance layer**,
> **B for the scanner**.

---

## 9. Reproduce

```bash
# the shared finding set both wrappers produce (offline):
zizmor --offline --min-severity low --min-confidence low .github/workflows/

# online audits (version↔SHA, impostor-commit, vulnerable actions) need a token:
GH_TOKEN="$(gh auth token)" zizmor --pedantic .github/workflows/

# what commit #632 changed:
git diff b16bd3f 12b1da4 -- .github/workflows/
```

- Option A workflow + caller live on branch `zizmor-eval-grafana` (PR #2).
- Option B caller lives on branch `zizmor-eval-action` (PR #3).
- zizmor-action inputs: <https://github.com/zizmorcore/zizmor-action/blob/main/action.yml>
- zizmor audits reference: <https://docs.zizmor.sh/audits/>
