# Testing both options on this fork

Goal: run **A** and **B** on a real PR and compare. All PRs must target **your
fork** (`david-wagih/consuldotnet`), never upstream `G-Research/consuldotnet`.

## Workflows

| Option | Caller workflow | Trigger |
|---|---|---|
| A — Grafana reusable (no bench) | `.github/workflows/zizmor-grafana-test.yml` | PR touching `.github/**`, or manual `workflow_dispatch` |
| B — zizmor-action | `.github/workflows/zizmor-action-test.yml` | same |

Both are **non-blocking** during the eval (A uses `fail-severity: never`; B never
fails on findings by design), so neither will red-X the PR.

## Open the PR on YOUR fork (not upstream)

The default repo is already set to the fork:

```bash
gh repo set-default david-wagih/consuldotnet   # already done
```

Then:

```bash
git checkout -b zizmor-eval
# make a trivial change under .github/ so the paths filter triggers, e.g. edit a comment
git add -A && git commit -m "test: trigger zizmor eval workflows"
git push -u origin zizmor-eval

# IMPORTANT: --repo points the PR at your fork, not the upstream parent
gh pr create --repo david-wagih/consuldotnet --base master --head zizmor-eval \
  --title "zizmor eval" --body "Compare Grafana reusable vs zizmor-action"
```

Without `--repo david-wagih/consuldotnet`, `gh pr create` defaults to the
**upstream** parent (`G-Research/consuldotnet`) because this is a fork. Always
pass it (or rely on the `set-default` above).

> Tip: you can also skip the PR entirely and run each workflow from the Actions
> tab via **Run workflow** (`workflow_dispatch`).

## What to compare on the run

| Look at | A | B |
|---|---|---|
| Findings count/severity | identical | identical |
| Code Scanning upload | ✅ | ✅ |
| Readable findings in logs | "Run zizmor with plain output" step | action console output |
| PR comment when GHAS off | ✅ fallback | — |
| Build pass/fail behavior | controlled by `fail-severity` | always passes on findings |
| Wall-clock / setup | native `uv` | Docker pull + run |

Findings will match (same binary). Judge the **surfacing, gating, and config**
behavior — that's the actual decision (see [comparison.md](comparison.md)).

## Cleanup

These are eval-only. To stop running them, delete the two
`.github/workflows/zizmor-*-test.yml` files (and this folder) once you've chosen.
