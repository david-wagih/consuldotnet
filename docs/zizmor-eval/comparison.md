# A vs B: Grafana reusable-zizmor (no Prometheus) vs. zizmor-action

| | A — Grafana reusable (minus bench) | B — `zizmorcore/zizmor-action` |
|---|---|---|
| Form | reusable **workflow** | composite **action** |
| Runs zizmor via | `uvx zizmor` (native) | **Docker image** (needs container runtime) |
| Maintained by | us, once forked into `G-Research/shared-workflows` | upstream |

## The findings are equal

Both shell out to the *same zizmor binary* with the same `--min-severity` /
`--min-confidence` / `--config`. Same version → identical SARIF.

```
zizmor 1.25.2 on .github/workflows/  (min-severity low, min-confidence low)
  → 86 findings: 18 medium, 8 high   (3 ignored, 57 suppressed)
```

So "better output" is the wrong axis. Decide on **what each does with the
output**.

## What each does with the findings

| Capability | A | B | Winner |
|---|:---:|:---:|---|
| SARIF → Code Scanning | ✅ | ✅ (default) | tie |
| **Build gating** (`fail-severity`) | ✅ never/any/…/high | ❌ never fails on findings, only internal errors | **A** |
| **PR-comment fallback** (no-GHAS/private) | ✅ + hides stale | ❌ | **A** |
| Inline PR annotations | via Code Scanning | ✅ (cap 10, *excludes* SARIF upload) | tie |
| **Central org config** (OIDC) | ✅ one shared baseline | ❌ per-repo | **A** |
| **Config policy gate** | ✅ blocks repos disabling audits | ❌ | **A** |
| `persona` / online-offline toggle | via `extra-args` | ✅ first-class inputs | **B** |
| Version pinning | ✅ pinned by default | ✅ but `latest` default ⚠️ | **A** |
| Hardening defaults | ✅ baked in | you write the caller | **A** |
| Runtime dep | `uv`, no Docker | **Docker required** | **A** |
| Cost to adopt | **high** (host + rewire org refs) | **low** (~10 lines) | **B** |

## The three decisive points

1. **Gating.** zizmor-action *cannot fail the build on findings* by design — only
   on internal error. Want a red ❌ on a high-severity finding? You bolt on your
   own SARIF threshold step. A gives `fail-severity` for free.
2. **Org platform vs. single repo.** A's real value is the central-config fetch +
   policy gate (one baseline everywhere; repos can't silently disable audits). B
   has none of that. Dead weight for one repo; the whole point for an org.
3. **Cost to adopt.** B is ~10 lines, upstream-maintained. A only pays off after
   you fork it into `G-Research/shared-workflows` and rewire every `grafana` →
   `G-Research` reference (2 OIDC fallbacks, org gates, the validator).

## Adaptation costs found while testing A on this fork

Running A on consuldotnet surfaced two breakages that don't show up in
Grafana's own setup — concrete evidence of the "ADAPT, don't lift" cost:

1. **`delete-vulnerable-branch` startup_failure.** That job declares
   `contents: write`; GitHub validates a reusable workflow's declared job
   permissions at startup *even for jobs that `if:`-skip*, so a caller granting
   only `contents: read` gets a hard `startup_failure`. Fixed by trimming to the
   `job-workflow-ref` + `analysis` minimal slice (also drops the Prometheus job).
2. **Missing central config crash.** The "Set up Zizmor configuration" step
   passed `--config <temp>/zizmor.yml` whenever a ref SHA existed, without
   checking the file downloaded. Grafana's repo always ships `.github/zizmor.yml`
   so it never 404s; consuldotnet has none → fetch 404 → `--config` points at a
   missing file → `zizmor` errors out. Fixed with a `[ -f ... ]` guard so it
   falls back to zizmor's default config.

B (zizmor-action) hit neither — it has no central-config machinery to adapt.

## Recommendation

```
Single repo, working this week         →  B
Org-wide rollout + enforced config     →  A  (drop bench + delete-vulnerable-branch)
```

- **consuldotnet alone → B.** Identical findings, minimal setup, upstream-owned,
  Code Scanning out of the box. Close the no-gating gap later with a small
  SARIF-threshold step if you want blocking PRs.
- **G-Research-wide → A.** Central config, policy gate, hardening, and built-in
  gating are exactly the controls B lacks — worth the one-time hosting cost. See
  [grafana-takeaways.md](grafana-takeaways.md) for what to keep.

> Don't pick on findings — they're equal. Pick **A for the governance layer**,
> **B for the scanner**.

## Reproduce

```bash
zizmor --offline --min-severity low --min-confidence low .github/workflows/
# B inputs:   https://github.com/zizmorcore/zizmor-action/blob/main/action.yml
# A workflow: .github/workflows/grafana-reusable-zizmor.yml
```
