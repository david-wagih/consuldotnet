# zizmor evaluation

Picking how to run [zizmor] (GitHub Actions static analysis) on this repo, and
on G-Research repos generally. Two options on the table:

- **A — Grafana's `reusable-zizmor.yml`** (with the Prometheus/Grafana-Bench job
  stripped) — a full reusable *workflow*.
- **B — [`zizmorcore/zizmor-action`]** — the upstream composite *action*.

[zizmor]: https://docs.zizmor.sh/
[`zizmorcore/zizmor-action`]: https://github.com/zizmorcore/zizmor-action

## TL;DR

Both wrap the **same zizmor binary → identical findings**. Proven on live runs:
both uploaded the **same 27 SARIF results** (same rule/file/line set) despite A
running zizmor `1.24.1` and B `1.25.2`. The choice is about the machinery
*around* the findings, not the findings — and A needed **3 fixes** to run on a
non-Grafana repo while B needed **0**.

| You want… | Choose |
|---|---|
| This one repo working fast, low upkeep | **B** (zizmor-action) |
| An org-wide platform with enforced shared config + gating | **A** (Grafana, bench stripped) |

B's one real gap is **no build-gating on findings**; A's cost is **hosting +
rewiring** it for G-Research. Full reasoning in [comparison.md](comparison.md).

## Docs

| Doc | What it answers |
|---|---|
| [comparison.md](comparison.md) | A vs B — the decision matrix and recommendation |
| [grafana-takeaways.md](grafana-takeaways.md) | What's worth lifting from Grafana's workflow (TAKE / ADAPT / DROP) + inputs |
| [zizmor-value.md](zizmor-value.md) | What zizmor actually catches on this repo (and what commit #632 did/didn't fix) |
| [testing.md](testing.md) | How to run both options in a PR **on this fork** |

## Test it

Two caller workflows let you run both options on a real PR against your fork:

- `.github/workflows/zizmor-grafana-test.yml` → option A
- `.github/workflows/zizmor-action-test.yml` → option B

See [testing.md](testing.md) for the PR flow (PRs target `david-wagih/consuldotnet`,
not upstream).
