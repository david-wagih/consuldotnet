# What to lift from Grafana's reusable-zizmor

If we go with **option A**, this is what's worth keeping. Verdicts: **TAKE**
(lift as-is), **ADAPT** (rewire for G-Research), **DROP** (Grafana-only).

| Capability | What it does | For G-Research |
|---|---|---|
| zizmor scan + SARIF → Code Scanning | inline PR annotations where reviewers look | **TAKE** (free on public repos w/ GHAS) |
| PR-comment fallback | posts results when SARIF upload unavailable (private/no-GHAS) | **TAKE** |
| Severity/confidence knobs | `min-severity`, `min-confidence`, `fail-severity` | **TAKE** — the gradual-onboarding path |
| Exit-code → severity mapping | separates "crashed" (1) from "found issues" (10–14) | **TAKE** |
| Dual run (sarif + plain) | machine output + readable logs | **TAKE** |
| OIDC `job-workflow-ref` config fetch | pulls one central `.github/zizmor.yml` baseline | **ADAPT** — host in `G-Research/shared-workflows`, change fallback ref + org |
| Config precedence + policy gate | repo-local wins, else default; validator blocks disabling key audits | **ADAPT** — keep precedence; validator is custom Grafana tooling, rebuild or drop |
| Hardening defaults | `permissions: {}`, SHA-pins, harden-runner, `persist-credentials:false` | **TAKE** — copy verbatim |
| `grafana-bench` metrics | Vault → Prometheus `zizmor_*` metrics | **DROP** — needs Grafana infra |
| `delete-vulnerable-branch` | Slack + auto-delete branch on dangerous-triggers | **DROP** for OSS (deletes contributors' branches) |

## Inputs (the reusable workflow)

| Input | Default | Note |
|---|---|---|
| `min-severity` | `low` | show findings ≥ this |
| `min-confidence` | `low` | show findings ≥ this confidence |
| `fail-severity` | `high` | fail build ≥ this (`never` = non-blocking) |
| `runs-on` | `ubuntu-latest` | |
| `always-use-default-config` | `false` | ignore repo-local config when `true` |
| `extra-args` | `""` | passthrough, e.g. `--offline`, `--persona auditor` |
| `send-bench-metrics` | `true` | **set `false`** to skip Prometheus job |
| `auto-delete-dangerous-branches` | `false` | leave off |

## OSS gotchas we inherit

- **Fork PRs have no OIDC token** and no `security-events: write` → the central
  config fetch and SARIF upload degrade (hence the fallbacks). consuldotnet takes
  external PRs, so this path must be tested.
- **Code Scanning is free on public repos**, so SARIF upload works here; the
  PR-comment fallback is mainly for *private* G-Research repos.

## Minimal slice for G-Research

Keep jobs `job-workflow-ref` + `analysis` only. Substitute:

| Grafana | G-Research |
|---|---|
| `grafana/shared-workflows/.github/workflows/reusable-zizmor.yml` | `G-Research/shared-workflows/...` |
| fallback `grafana/shared-workflows@main` (2 spots in OIDC script) | `G-Research/shared-workflows@main` |
| `github.repository_owner == 'grafana'` gates | `== 'G-Research'` |
| central Grafana `.github/zizmor.yml` | our own baseline — triage real findings, don't suppress |
| `grafana/.../validate-zizmor-config` | drop, or build a G-Research policy action |

Drop `grafana-bench` and `delete-vulnerable-branch` on day one.
