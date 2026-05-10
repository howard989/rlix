# M11 Feature Summary — MILES under RLix

This is the short feature-level entry point for the M11 MILES integration. It summarizes what landed across the two Zhenyu M11 branches without requiring reviewers to read the full port plan first.

## Branches

| Repo | Branch | Reference HEAD | Role |
|---|---|---:|---|
| `rlix` | `rlops/rlix:zhenyu/miles-mvp-e2e` | `62a20d9` | RLix control plane, scheduler integration, MilesPipeline, MilesCoordinator, model-update service, smoke scripts |
| `miles` | `rlops/miles:zhenyu/m11-mvp-test` | `6126e01` | MILES runtime hooks, SGLang/Megatron adapters, RLix drivers, placement provider, validation |


## Source Documents

| Document | Purpose |
|---|---|
| `plans/miles-port-unified-plan.md` | Full design plan. Deep dive for each F1-F12 feature, ROLL comparison, invariants, and deferred scope. |
| `docs/m11-implementation-guide.md` | Post-implementation walkthrough. Maps F1-F12 to current files, bugs found during M11.1/M11.2, fixes, and smoke evidence. |
| `plans/m11-e2e-test-log.md` | M11.1 single-pipeline smoke history. |
| `plans/m11-2-dual-pipeline-log.md` | M11.2 dual-pipeline smoke history. |
| `plans/m11-review.review-report.md` | Code-review findings and follow-up recommendations. |

## Goal

M11 makes MILES run under RLix's GPU time-share control plane.

The core problem is partial overlap: rollout/inference and train both need GPUs, and the train GPU set can overlap the inference GPU set. RLix arbitrates those GPUs. MILES supplies the SGLang + Megatron runtime hooks needed to sleep inference engines, train, refresh weights, and resume generation.

M11 has two green validation targets:

| Target | Meaning | Status |
|---|---|---|
| M11.1 | One MilesPipeline, partial-overlap topology, Qwen2.5-0.5B GRPO, `--num-rollout 2` | Green in existing logs |
| M11.2 | Two concurrent MilesPipelines on disjoint pools, each running its own GRPO loop | Green in existing logs |

Production hardening, actor recovery, 3+ pipeline contention, and fine-grained fault tolerance are deferred beyond M11.

## Feature Map

| Feature | Name | What it provides | Main files |
|---|---|---|---|
| F1 | SGLang sleep/wake | Releases SGLang inference VRAM so Megatron train can reuse overlap GPUs, then reloads engines later. | `miles/miles/ray/rollout.py`, `miles/miles/backends/sglang_utils/sglang_engine.py`, `miles/miles/backends/megatron_utils/actor.py` |
| F2 | Selective engine sleep/wake | Shrinks only engines mapped to overlap GPUs; non-overlap engines can stay active. | `miles/miles/ray/rollout.py`, `rlix/pipeline/miles_pipeline.py` |
| F3 | Generation routing around sleeping engines | Prevents dispatch to inactive engines and handles empty/changed active-engine sets. | `miles/miles/router/router.py`, `miles/miles/rollout/generate_hub/multi_turn.py`, `miles/miles/router/middleware_hub/radix_tree_middleware.py` |
| F4 | Training-side weight cache | Builds a CPU bucket cache from Megatron train weights after init/train so SGLang engines can refresh without live train GPU residency. | `miles/miles/backends/megatron_utils/update_weight/cpu_bucket_cache.py`, `miles/miles/ray/actor_group.py`, `rlix/pipeline/miles_pipeline.py` |
| F5+F6 | Weight refresh and version accounting | Syncs CPU-cached weights to active/expanded engines as one atomic unit and publishes engine weight versions exactly once. | `rlix/pipeline/miles_model_update_service.py`, `rlix/pipeline/miles_coordinator.py` |
| F7 | Per-pipeline Ray namespace isolation | Places each pipeline's coordinator and actors in a deterministic namespace/name scheme so scheduler RPCs resolve correctly. | `rlix/protocol/types.py`, `rlix/scheduler/scheduler.py`, `miles/examples/rlix/run_miles_rlix.py`, `miles/examples/rlix/run_miles_dual.py` |
| F8 | Pipeline registration lifecycle | Provides RLix-mode drivers: allocate/register/admit pipeline, build coordinator/pipeline actors, initialize train+infer, then run train loops. | `rlix/orchestrator/orchestrator.py`, `rlix/pipeline/miles_pipeline.py`, `miles/examples/rlix/run_miles_rlix.py`, `miles/examples/rlix/run_miles_dual.py` |
| F9 | Progress reporting | Reports rollout progress from MILES to the scheduler so gap-ratio planning has live cadence. | `rlix/pipeline/miles_coordinator.py`, `rlix/pipeline/miles_hooks.py`, `miles/miles/utils/tracking_utils.py`, `miles/miles/utils/wandb_utils.py` |
| F10 | RLix topology validation | Fails fast on unsupported RLix-mode configs instead of silently producing wrong placement or mid-run OOMs. | `miles/miles/utils/rlix_validation.py`, `miles/examples/rlix/run_miles_rlix.py` |
| F11 | `RLIX_CONTROL_PLANE=rlix` gate | Keeps RLix-only behavior behind one env flag so standalone MILES behavior remains unchanged. | `miles/miles/utils/rlix_validation.py`, `miles/miles/ray/rollout.py`, `miles/examples/rlix/run_miles_rlix.py` |
| F12 | Shared placement-group adapter | Replaces standalone placement assumptions with RLix/ROLL-derived placement objects for train workers and rollout engines. | `miles/miles/ray/placement_provider.py`, `rlix/pipeline/miles_pipeline.py`, `miles/miles/ray/actor_group.py` |

## Features in Detail

### F1 — SGLang sleep/wake

MILES must release inference-engine VRAM before train can safely wake on overlap GPUs. F1 wires SGLang's `release_memory_occupation` / wake path into RLix-mode rollout management.

The M11 implementation forces SGLang memory saver on in RLix mode when `--offload-rollout` is used. It also standardizes the TMS hook mode used by smoke runs through `MILES_TMS_HOOK_MODE=torch`, avoiding the broader LD_PRELOAD path that was unstable on the validated CUDA 12.9/Blackwell stack.

Primary MILES files:
- `miles/miles/backends/sglang_utils/sglang_engine.py`
- `miles/miles/ray/rollout.py`
- `miles/miles/backends/megatron_utils/actor.py`

### F2 — Selective engine sleep/wake

F2 is the partial-overlap mechanism. Instead of sleeping the entire rollout side, MILES tracks each rollout engine and sleeps only the engines whose GPUs must be handed to training.

The runtime state model is explicit: engines move through states such as `active`, `disabling`, `offloaded`, and `loading`. Shrink uses abort/drain/sleep ordering so SGLang is not asked to release memory while it is still processing a request.

M11.2 also fixed the physical-GPU-to-local-engine-index conversion for the second pipeline. Pipeline 2 may use physical GPUs `[2,3]`, but its local RolloutManager engine indices are still `[0,1]`.

Primary files:
- `miles/miles/ray/rollout.py`
- `rlix/pipeline/miles_pipeline.py`

### F3 — Generation routing around sleeping engines

F3 prevents generation from being dispatched to engines that RLix has shrunk or temporarily disabled. The router keeps active/disabled/dead/preempted worker state and notifies waiting dispatchers when the active set changes.

The happy-path M11 smoke mostly exercises the admission and active-set plumbing. More adversarial preempt/retry behavior is intentionally deferred; the M11 contract is that the router should not silently route to sleeping engines.

Primary files:
- `miles/miles/router/router.py`
- `miles/miles/rollout/generate_hub/multi_turn.py`
- `miles/miles/router/middleware_hub/radix_tree_middleware.py`

### F4 — Training-side weight caching

F4 creates the handoff point between Megatron training and SGLang inference. After initialization and after train steps, the cache-owner Megatron actor builds a CPU bucket cache of model weights.

This lets train offload or release GPU residency while later sync operations read from the CPU cache. The key lifecycle is build cache, publish the ready step, offload train, then let the model-update service read the cache for receiver-side refresh.

Primary files:
- `miles/miles/backends/megatron_utils/update_weight/cpu_bucket_cache.py`
- `miles/miles/ray/actor_group.py`
- `rlix/pipeline/miles_pipeline.py`

### F5+F6 — Weight refresh and version accounting

F5+F6 define the receiver-side sync path. `MilesModelUpdateService.sync_selected_workers(...)` is the atomic unit that moves cached weights to selected SGLang engines, finalizes the update, and publishes the new weight version.

Two paths use the same sync contract:
- Active in-flight refresh for engines already serving.
- Expand sync for engines that were offloaded and need current weights before routing opens.

M11 keeps version publication centralized: the rollout manager gets one version update per completed sync, not per bucket.

Primary files:
- `rlix/pipeline/miles_model_update_service.py`
- `rlix/pipeline/miles_coordinator.py`

### F7 — Per-pipeline Ray namespace isolation

F7 makes actor naming deterministic and collision-free. Each pipeline gets its own Ray namespace derived from the pipeline id, and the scheduler resolves the pipeline coordinator through the canonical coordinator name in that namespace.

This is mandatory for M11.2. Without it, two pipelines collide or the scheduler cannot find the correct coordinator when it needs to resize inference.

Primary files:
- `rlix/protocol/types.py`
- `rlix/scheduler/scheduler.py`
- `miles/examples/rlix/run_miles_rlix.py`
- `miles/examples/rlix/run_miles_dual.py`

### F8 — Pipeline registration lifecycle

F8 is the driver-facing lifecycle. The driver initializes RLix, allocates a pipeline id, registers train/infer cluster mappings, admits the pipeline, starts a `MilesCoordinator`, creates a `MilesPipeline`, initializes train and infer, then runs the RL training loop.

M11.1 covers the single-pipeline driver. M11.2 adds the dual-pipeline driver, per-pipeline argument copies, disjoint GPU pool construction, per-pipeline rollout base ports, and concurrent train loops via `asyncio.gather`.

Primary files:
- `rlix/orchestrator/orchestrator.py`
- `rlix/pipeline/miles_pipeline.py`
- `miles/examples/rlix/run_miles_rlix.py`
- `miles/examples/rlix/run_miles_dual.py`

### F9 — Progress reporting

F9 reports rollout progress from MILES back to RLix. The scheduler uses this progress stream to make gap-ratio planning decisions with live cadence rather than only static assumptions.

The MILES side uses hook interfaces so standalone code can use no-op hooks while RLix mode forwards progress to `MilesCoordinator`.

Primary files:
- `rlix/pipeline/miles_coordinator.py`
- `rlix/pipeline/miles_hooks.py`
- `miles/miles/utils/tracking_utils.py`
- `miles/miles/utils/wandb_utils.py`

### F10 — RLix topology validation

F10 is the startup fail-fast layer. RLix mode has stricter requirements than standalone MILES: partial-overlap topology, supported transport, compatible parallelism, no unsupported async-save path, and other constraints must be checked before the system allocates actors and GPUs.

The design goal is simple: wrong topology should fail with a clear `ValueError`, not hang or OOM deep in a Ray actor.

Primary files:
- `miles/miles/utils/rlix_validation.py`
- `miles/examples/rlix/run_miles_rlix.py`

### F11 — `RLIX_CONTROL_PLANE=rlix` gate

F11 keeps RLix-specific behavior behind a single environment flag. When `RLIX_CONTROL_PLANE=rlix` is set, MILES enables the scheduler-aware branches. When it is not set, standalone MILES behavior is preserved.

This flag gates the sleep/wake override, routing changes, validation behavior, and RLix driver entry assertions.

Primary files:
- `miles/miles/utils/rlix_validation.py`
- `miles/miles/ray/rollout.py`
- `miles/examples/rlix/run_miles_rlix.py`

### F12 — Shared placement-group adapter

F12 adapts MILES worker placement to RLix/ROLL allocation. MILES standalone placement assumes it owns a simple rollout placement group; RLix mode receives scheduler-owned physical GPU mappings and converts them into worker placements.

`MilesPlacementProvider` is the bridge. It supplies train workers and rollout engines with placement metadata while preserving MILES expectations such as post-`CUDA_VISIBLE_DEVICES` local GPU ids and explicit `LOCAL_RANK=0` injection.

Primary files:
- `miles/miles/ray/placement_provider.py`
- `rlix/pipeline/miles_pipeline.py`
- `miles/miles/ray/actor_group.py`

## Repo Ownership View

| Area | RLix repo owns | MILES repo owns |
|---|---|---|
| Control plane | Orchestrator, scheduler, resource-manager integration, pipeline actor lifecycle | Driver calls into RLix from `examples/rlix/*` |
| Per-pipeline control | `MilesPipeline`, `MilesCoordinator`, scheduler resize/sync hooks | Runtime hooks called by pipeline/coordinator |
| Weight update | `MilesModelUpdateService`, sync orchestration, version publication | CPU bucket cache, Megatron cache-owner APIs, SGLang receiver APIs |
| Inference shrink/expand | Scheduler requests and coordinator decisions | RolloutManager engine state, SGLang sleep/wake, router admission |
| Placement | Pipeline passes scheduler mappings into provider | `MilesPlacementProvider`, train/rollout worker placement consumption |
| Validation | M11 docs and smoke scripts | RLix-mode startup validation and standalone guardrails |

## Verification Summary

| Run | What it validates | Evidence |
|---|---|---|
| M11.1 single smoke | One partial-overlap MilesPipeline can initialize, generate, train, sync weights, and shut down cleanly. | `plans/m11-e2e-test-log.md`, final attempt `EXIT_CODE=0` |
| M11.2 dual smoke | Two MilesPipelines can run concurrently on disjoint pools with separate coordinator/pipeline actors and SGLang port windows. | `plans/m11-2-dual-pipeline-log.md`, final attempt `EXIT_CODE=0` |
| Post-review dual rerun | Engine-index conversion fix does not regress M11.2. | `plans/m11-review.debug.md`, `EXIT_CODE=0`, no `KeyError`, no `engine_index` warnings |

## Deferred Work

The following are intentionally not part of the M11 feature baseline:

- 3+ pipeline contention and production scheduling hardening.
- Fine-grained fault tolerance after Ray actor crash, driver SIGKILL, host failure, or network partition.
- Full driver-level cleanup/recovery orchestration.
- Save/eval final-rollout offloaded-weight handling.
- Non-contiguous/custom ordered GPU mapping beyond the first-build contiguous topology.
- CUDA IPC optimized transport beyond the M11 `cpu_serialize` smoke path.

For details, see `docs/m11-implementation-guide.md` §6 and the review report under `plans/m11-review.review-report.md`.
