# M11 Shared-Topology Probe Report

Date: 2026-05-10

This document records the probe run that tested true cross-pipeline GPU
sharing for the M11 MILES integration. It is intentionally an RCA report,
not a merge proposal.

## Summary

M11.2's green dual-pipeline smoke validates two concurrent pipelines on
disjoint GPU pools:

```text
P1: train=[0], infer=[0,1]
P2: train=[2], infer=[2,3]
```

That is useful as a dual-driver and per-pipeline isolation test, but it does
not exercise RLix's core shared-GPU arbitration path because the two pipelines
never contend for the same physical GPU.

The shared-topology probe changed the topology to make both pipelines claim
the same two-GPU pool:

```text
P1: train=[0], infer=[0,1]
P2: train=[0], infer=[0,1]
```

Result: FAIL, `EXIT_CODE=125`.

Both pipelines initialized. One pipeline progressed through training. The
other pipeline stalled at `rollout_id=0` with `collected=0/8` until the smoke
watchdog terminated the run.

## Branches And Artifacts

| Repo | Branch | Commit | Purpose |
|---|---|---:|---|
| `miles` | `howard/m11-shared-dual-topology-probe` | `4f6fddaf7185a974b6f55d02fb1aba8c2dd3e96b` | Probe-only shared topology driver + RCA instrumentation |
| `rlix` | `howard/m11-configurable-free-gpu-threshold` | `c194b117bcd6c49a00e9c3220caaf8f73ab749a5` | RLix branch used during the probe run |

Local artifact bundle:

```text
~/Downloads/shared_probe_artifacts_2/
```

Included files:

```text
key_events.txt
miles_branch.txt
miles_commit.txt
ps_after_timeout.txt
rlix_branch.txt
rlix_commit.txt
run.log
```

The `miles` branch is explicitly probe-only and is not intended for merge.
The probe added a `--dual-topology=shared` / `MILES_DUAL_TOPOLOGY=shared`
mode plus temporary instrumentation. Default `disjoint` behavior remains
unchanged.

## Probe Setup

Shared probe command shape:

```bash
cd /root
MILES_DUAL_TOPOLOGY=shared \
SCRIPT=/root/rlix/scripts/run_smoke_dual.sh \
SILENCE_LIMIT=900 \
RUN_LIMIT=3600 \
bash /root/rlix/scripts/run_smoke_with_watchdog.sh
```

Probe instrumentation enabled automatically for shared topology:

```text
MILES_DEBUG_SHARED_PROBE=1
MILES_ROUTER_TEST_HOOKS=1
```

Important instrumentation added in `miles`:

| File | Probe-only evidence |
|---|---|
| `examples/rlix/run_miles_dual.py` | Threads `pipeline_id` / `pipeline_index` into actor args and enables probe env vars for shared topology |
| `miles/utils/rlix_train_loop.py` | Adds per-pipeline loop prefixes such as `[loop mp2 pipeline_id=...]` |
| `examples/fully_async/fully_async_rollout.py` | On rollout starvation, prints worker task counters plus router admission state |
| `miles/router/router.py` | Adds router diagnostic state and full proxy exception logging |

## Outcome

Final exit:

```text
EXIT_CODE=125
```

Stable stuck snapshot from `key_events.txt` / `run.log`:

```text
[shared-probe] no-progress snapshot:
  pipeline_index=1
  pipeline_id=miles_cfdf043e31ca
  exp_name=miles_dual_mp1
  rollout_id=0
  collected=0/8
  completed_buffer=0
  worker={
    running=True,
    thread_alive=True,
    active_tasks=8,
    started_groups=8,
    completed_callbacks=0,
    fatal_callbacks=0,
    task_errors=0,
    last_error=None,
    output_queue_size=0
  }
  router={
    status=200,
    candidate_count=2,
    enabled_workers=[
      http://172.17.0.2:16000,
      http://172.17.0.2:16003
    ],
    dead_workers=[],
    worker_request_counts={
      http://172.17.0.2:16000: 16,
      http://172.17.0.2:16003: 16
    },
    worker_failure_counts={
      http://172.17.0.2:16000: 0,
      http://172.17.0.2:16003: 0
    }
  }
```

SGLang continued to emit `Prefill batch`, `Decode batch`, and many
`POST /generate 200 OK` lines during the stall. The stuck pipeline's rollout
worker never received completed callbacks.

## What This Rules Out

The probe narrows the failure surface:

| Candidate failure | Ruled out? | Evidence |
|---|---:|---|
| Ray placement rejected the second pipeline | Yes | Both pipelines initialized; the run reached rollout/train runtime |
| Router had zero active workers | Yes | `candidate_count=2`, `enabled_workers` has both workers, `dead_workers=[]` |
| Fully async rollout worker never started | Yes | `thread_alive=True`, `started_groups=8`, `active_tasks=8` |
| Immediate task exception / fatal scheduler preempt surfaced | Yes | `task_errors=0`, `fatal_callbacks=0`, `last_error=None` |
| Request path returned cleanly to rollout worker | No | `completed_callbacks=0`, `output_queue_size=0` |

## Interpretation

The shared run fails at runtime arbitration, not at placement or admission.

The stuck pipeline successfully starts eight rollout generation groups. Those
requests enter the router, and the router still believes two workers are live.
However, the router's `worker_request_counts` remain stuck at `16/16`, which
means the router still has long-running upstream `/generate` requests in
flight. The rollout worker sees zero completed callbacks, so those requests
never return to the fully-async collection path.

In short:

```text
generate_and_rm_group -> router -> SGLang stays in-flight indefinitely
```

The current M11.2 path supports per-pipeline partial overlap, but it does not
provide a cross-pipeline lease/preempt/cancel protocol for true shared GPU
contention.

## Root Cause Class

Missing cross-pipeline runtime arbitration.

True shared GPU scheduling needs a pipeline-level lease contract:

1. A pipeline entering train requests a GPU lease from the scheduler.
2. The lease holder must force peer pipelines on the same physical GPUs to:
   - stop admitting new rollout requests,
   - abort or drain in-flight generation,
   - offload/sleep the affected SGLang engines,
   - publish a state that the training pipeline can wait on.
3. After train releases the lease, the scheduler can wake/sync the previously
   paused peer engines and reopen admission.

M11.2 has pieces for single-pipeline partial overlap:

```text
current pipeline shrink -> train -> sync -> expand
```

It does not yet have the cross-pipeline version:

```text
pipeline A train lease -> pipeline B infer preempt/drain/sleep -> A train
-> A release -> B wake/sync/resume
```

## Relationship To Existing M11.2 PASS

The M11.2 dual-pipeline PASS remains valid for the scope it actually covered:

```text
two pipelines, disjoint pools, independent per-pipeline partial overlap
```

It should not be cited as evidence that RLix can time-share the same physical
GPU pool across multiple MILES pipelines.

Recommended wording for M11.2:

```text
M11.2 validates the disjoint-pool dual driver and per-pipeline namespace /
placement isolation. It does not validate true cross-pipeline shared-GPU
runtime arbitration.
```

## Recommended Next Steps

### Short term

1. Keep the M11.2 disjoint result, but document the scope explicitly.
2. Do not merge the `miles` shared-topology probe branch as a product feature.
3. Use this report as the evidence packet for deciding whether shared-GPU
   arbitration belongs in M11.3.

### M11.3 design work

The likely design items are:

| Area | Required work |
|---|---|
| Scheduler | Pipeline-level GPU lease, fairness, and explicit preempt/release events |
| Coordinator | Cross-pipeline shrink/expand orchestration, not only same-pipeline hooks |
| RolloutManager/router | Bounded preempt/cancel semantics for in-flight `/generate` requests |
| SGLang engine lifecycle | Reliable drain/abort/sleep/wake under peer-pipeline contention |
| Observability | First-class counters for in-flight generate, disabled admission, dead workers, and lease owner |

The immediate fix is not a small constant or local routing change. The probe
shows the system needs an explicit cross-pipeline lease/preempt/cancel design
before shared pools can be made reliable.

## One-line Conclusion

M11.2 disjoint dual-pipeline works, but true shared-GPU dual-pipeline does not:
the shared probe fails with `EXIT_CODE=125` because one pipeline's rollout
requests remain in-flight indefinitely while the router still has live workers,
which demonstrates missing cross-pipeline runtime arbitration.
