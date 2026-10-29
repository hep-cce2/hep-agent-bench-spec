# Grading (v0.1)

A grader (`tests/test.sh`) runs after the agent finishes. It checks the agent's result on held-out data, measures it against the baseline and the expert, and writes a reward in [0, 1]. The [`hepbench`](https://github.com/hep-cce2/hep-agent-bench-tools) package implements each rule below; its function names are given in brackets.

## Three versions

The grader builds and measures three versions of the code, on the same hardware in the same run:

| Version | Source |
|---|---|
| **agent** | the agent's final working tree |
| **baseline** | the pinned `upstream_commit`, checked against a manifest (see [Baseline integrity](#baseline-integrity)) |
| **expert** | the expert-level result, e.g. a later upstream release, applied as a patch to the baseline |

Measuring the expert live, rather than storing reference numbers, makes the target adjust to whatever hardware the trial runs on.

## Correctness gate

The agent's version MUST first pass a correctness check on the **held-out** data. If it fails, the reward is 0, whatever its performance. The gate is defined per task and stated in `instruction.md`:

- **exact**: the output equals the reference exactly, e.g. the same set of edges (frnn);
- **lossless**: data written and read back is identical (proposed for RNTuple compression);
- **physics validation**: physics observables agree with the reference within a statistical tolerance agreed with the domain experts, e.g. a χ² or KS test on distributions with a fixed p-value threshold (proposed for Celeritas). The tolerance MUST be written in the task, not chosen by the grader at run time.

## Scores

After the gate, a figure of merit becomes the reward. A task expresses it as a **gain**: a ratio over the baseline where larger is better.

| Figure of merit | Gain |
|---|---|
| run time (lower is better) | `baseline_time / agent_time` |
| compression ratio (higher is better) | `agent_ratio / baseline_ratio` |
| throughput (higher is better) | `agent_rate / baseline_rate` |

### `log-gain-vs-expert` (the only score in v0.1)

```
score  = log(agent_gain) / log(expert_gain)        [log_gain_vs_expert]
reward = clamp(score, 0, 1) if correct else 0      [gated_reward]
```

1 means the agent matched the expert; below 0 means it did worse than the baseline. Logs make a 2× speedup count the same whether the expert reached 4× or 400×, on a scale relative to the expert. If the expert does not beat the baseline on the trial's hardware (`expert_gain <= 1`), the trial is invalid and scores 0, and the task needs fixing.

Tasks with more than one figure of merit (e.g. speed *and* compression) are not covered yet; propose a score here before writing one.

## Results

The grader MUST write, under `/logs/verifier/` [`write_result`]:

- `reward.txt`: the reward, a float in [0, 1] (Harbor's contract);
- `metrics.json`: at least

```json
{
  "spec_version": "0.1",
  "reward": 0.56,
  "correct": true
}
```

plus the task's own fields. For `log-gain-vs-expert`, include the raw gains and the uncapped score, so that agents that beat the expert can still be ranked. frnn writes `speedup`, `expert_speedup`, `log_speedup_vs_expert` and `p50_s` (per version).

Any failure (build, run, missing data, tampered baseline) MUST still write `reward.txt` with 0 and exit 0, with the reason in the log.

## Timing

For time-based scores [`median_time`]:

- all three versions run in the same container on the same device, one after another;
- warm up with at least one untimed call (the frnn grader uses it for the correctness check);
- time N calls (`TIMED_ITERATIONS` in `[verifier.env]`, default 10) and use the **median** wall time;
- each timed call gets a **fresh input** (e.g. a seeded permutation, the same seeds for every version), so an implementation cannot replay a cached result;
- the timed call MUST include device synchronization, so it returns only when the work is done;
- the device is pinned (for Docker, `device_ids` in `environment/docker-compose.yaml`), and every arm of a comparison runs on the same device.

## Baseline integrity

The agent usually runs as root in the container where the baseline lives, so it could slow the baseline down to inflate its own speedup. The grader MUST:

1. check the baseline source against a manifest of Git blob hashes (`tests/source-manifest.json`, generated with `python -m hepbench.snapshot create`);
2. build the baseline and expert from a **clean copy of exactly the verified files** [`python -m hepbench.snapshot export`], never in place, since leftover build trees would otherwise be reused;
3. install `hepbench` fresh at grading time from a pinned tag, not from the agent's image.

Even with these checks, a shared container still lets a root agent tamper with compilers or drivers. For leaderboard runs, set `verifier.environment_mode = "separate"` and restrict the network (`network_mode`, `allowed_hosts`), so the agent cannot fetch the expert result upstream.
