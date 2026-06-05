# Decision guide: pick by your requirements

Two ways to run zizmor (both proven to produce the **identical 27-finding SARIF**
on this repo — see [comparison.md](comparison.md)). This page helps you pick
based on *what you need*, not on findings.

- **A** = Grafana `reusable-zizmor.yml` (trimmed: `job-workflow-ref` + `analysis`)
- **B** = upstream `zizmorcore/zizmor-action`

## 30-second decision

```
Is this for ONE / a few repos you own?
  └─ yes → B   (10 lines, upstream-maintained, worked first try)
  └─ no, it's an ORG-WIDE rollout with a shared security baseline?
       └─ yes → A   (central config + policy gate + gating — accept the setup cost)

Special overrides:
  • You MUST fail PRs on findings out of the box        → A
  • Runners have no Docker                              → A  (B needs Docker)
  • You want it running THIS week with ~zero upkeep     → B
  • You need per-repo persona/online toggles as inputs  → B
```

## Map your requirement → the option

Tick the rows that matter to you; whichever column has more ticks for *your*
rows is your pick.

| If you need… | A (Grafana) | B (zizmor-action) |
|---|:---:|:---:|
| Identical zizmor findings | ✅ | ✅ |
| Upload to GitHub Code Scanning | ✅ | ✅ |
| **Fail the build on findings** (severity gate) out of the box | ✅ | ❌ (needs a custom SARIF step) |
| **One shared config baseline across many repos** | ✅ | ❌ |
| **Policy gate** (repos can't silently disable audits) | ✅ | ❌ |
| PR-comment fallback for private repos without GHAS | ✅ | ❌ |
| Hardened defaults baked in (pinned, `permissions:{}`, harden-runner) | ✅ | you write them |
| Run without Docker on the runner | ✅ | ❌ (Docker required) |
| Pin the zizmor version by default | ✅ | ⚠️ defaults to `latest` |
| First-class `persona` / `online-audits` inputs | ⚠️ via `extra-args` | ✅ |
| **Lowest setup effort** (~10 lines, no host repo) | ❌ | ✅ |
| **Lowest maintenance** (upstream owns updates) | ❌ | ✅ |
| Works on a non-Grafana repo with no changes | ❌ (needed 3 fixes) | ✅ (0 fixes) |

## Weighted scorecard (adjust weights to taste)

Score 1 = better. Re-weight the left column for your context and re-add.

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

## What it costs to actually adopt

| | A (Grafana) | B (zizmor-action) |
|---|---|---|
| Files to add per repo | 1 caller | 1 caller |
| One-time platform work | host in `G-Research/shared-workflows`, rewire `grafana → G-Research` refs (OIDC fallbacks, org gates), rebuild/drop the config validator | none |
| Proven fixes needed to run here | **3** (see [comparison.md](comparison.md)) | **0** |
| Who maintains updates | you | upstream |

## Recommendation for consuldotnet

Given it's a single public repo: **start with B**. It produced the same findings
with zero fixes and uploads to Code Scanning out of the box. Its only gap —
no build-gating — is closable later with a small SARIF-threshold step *if* you
decide you want blocking PRs.

Revisit **A** only if/when G-Research wants a centrally-governed rollout across
many repos, where A's shared-config + policy + gating machinery earns back its
setup cost.

> Decide on the rows above that match your requirements — the findings are equal,
> so the right answer is whichever column wins *your* weighted needs.
