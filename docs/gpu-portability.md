# GPU portability

NInfer targets exactly one GPU: an NVIDIA GeForce RTX 5090, compiled for `sm_120a`. This document
records what that targeting actually consists of, what the device code genuinely requires, and what
it would take to run on a different architecture. Hopper (H200, `sm_90a`) is worked through in
detail because it is the case where the answer is least obvious.

The headline result is counterintuitive: the engine emits no Blackwell-exclusive instructions. The
`sm_120a` target is a policy choice and a tuning baseline, not an instruction-set dependency.

## Two gates, not one

Retargeting is refused in two independent places. Both are policy assertions rather than technical
constraints, and both must be relaxed.

**Build gate.** `CMakeLists.txt:9-13` raises `FATAL_ERROR` unless `CMAKE_CUDA_ARCHITECTURES` is
exactly the string `120a`. It sits before `project(... LANGUAGES CUDA)` so it fires before compiler
detection. Corroborated at `README.md:69` and `.clangd:7`.

**Runtime gate.** `src/targets/qwen3_6/impl/runtime/layouts_impl.h:428-430`:

```cpp
if (device.sm() != 120) {
    throw std::invalid_argument("Qwen3.6 family runtime requires compute capability 12.0");
}
```

`DeviceContext::sm()` is `props.major * 10 + props.minor` (`src/core/device.cu:117`). This is exact
equality, a whitelist of one, not a minimum-capability floor, so it rejects every other
architecture including future ones. Every Engine construction throws before a kernel launches.

## What the device code actually requires

The complete inline-asm surface of the engine, which is the only place an architecture dependency
could hide:

| Instruction | Site | Introduced |
|---|---|---|
| `ldmatrix.sync.aligned.m8n8.x2/.x4[.trans].shared.b16` | `src/ops/common/mma.cuh:7-30` | sm_75 |
| `mma.sync.aligned.m16n8k16.row.col.f32.bf16.bf16.f32` | `src/ops/common/mma.cuh:36` | sm_80 |
| `mma.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32` | `src/ops/common/mma.cuh:45` | sm_80 |
| `mma.sync.aligned.m16n8k32.row.col.s32.s8.s8.s32` | `src/ops/common/mma.cuh:53` | sm_80 |
| `mma.sync.aligned.m16n8k8.row.col.f32.tf32.tf32.f32` | `src/ops/common/mma.cuh:62` | sm_80 |
| `cp.async.ca/.cg` plus commit and wait | `src/ops/common/memory.cuh:40-61` | sm_80 |
| `cvt.rn.bf16x2.f32` | `src/ops/common/math.cuh` | sm_80 |

A repository-wide search for `wgmma`, `tcgen05`, `cp.async.bulk`, `mbarrier`, `cluster`,
`setmaxnreg`, `elect.sync`, the FP8 and FP4 conversion types, and the wider sub-byte MMA shapes
returns no hits across `src`, `include`, `bench`, `tests`, and `tools`.

This matters because consumer Blackwell (`sm_120`) is not datacenter Blackwell (`sm_100`). It
programs its tensor cores with the `mma.sync` family, the same one Ampere and Ada use. It has
neither `wgmma` (Hopper) nor `tcgen05` (datacenter Blackwell). Everything above is `sm_80` era or
older, so any architecture from Ampere onward can execute it.

**The quantized GEMMs do not change this.** `Q4G64_F16S`, `Q5G64_F16S`, `Q6G64_F16S`, and
`W8G32_F16S` are storage formats, not MMA datatypes. Weights are decoded to BF16 in shared memory
and fed to the standard `m16n8k16` BF16 instruction. See `src/ops/linear/q4/q4_rowsplit_gemm_mma.cuh`
(the header comment states this, and the `mma_bf16` calls are in the main loop) and
`src/ops/linear/w8/w8_rowsplit_gemm_mma.cuh`, where a dequantizing lambda produces `__nv_bfloat162`
values that the same instruction consumes. The narrowest tensor-core datatype anywhere in the engine
is 8-bit, used only in the INT8 KV attention kernels, never in a weight GEMM.

Shared memory is not a constraint above Ampere either. The tightest kernel pins 90.5 KiB
(`static_assert` in `src/ops/kernel/gqa_attention_prefill_i8.cuh`, opted in through
`cudaFuncSetAttribute` in `src/ops/launcher/gqa_attention_prefill.cu`). Hopper allows 227 KB per SM
against `sm_120`'s roughly 100 KB, so the existing tiling is conservative there rather than
oversized.

## Hopper (H200): what would actually break

Beyond the two gates, one genuine defect and several silent risks.

### The cooperative-launch residency budget is a real defect

`src/ops/gdn_gating_proj/bf16/bf16_gdn_gating_proj_plan.cpp:130` and `:138-140` hardcode the
device-wide resident-CTA budget as 340 or 680. The comment derives those numbers explicitly from
the RTX 5090's 170 SMs. The value is used as an admission predicate before a cooperative launch.

The grid itself is derived from problem dimensions, not from the SM count, so on a 132-SM device
the predicate admits grids the driver will refuse. The launch uses `cudaLaunchKernelEx` with the
cooperative attribute, and the device side performs a grid-wide sync. A refused launch returns
`cudaErrorCooperativeLaunchTooLarge`, which `cuda_check` at `src/core/device.cu:38-43` turns into
`std::abort()`. The failure is therefore a loud crash rather than silent corruption, which is the
better of the two outcomes but still fatal.

The fix is to compute the budget from `cudaOccupancyMaxActiveBlocksPerMultiprocessor` multiplied by
`props.multiProcessorCount`. Both are available already: `DeviceContext` stores `props`
(`src/core/device.h:17`) but exposes only `sm()` and `total_vram()`, so this needs one accessor plus
a per-kernel occupancy query.

Decode is unaffected because its grids are small enough to be resident regardless.

### Open risk: cooperative launch inside CUDA Graph capture

`src/core/decode_graph.cpp:61` begins stream capture, and the gating op is reachable from the text
decode path. Whether the driver validates cooperative grid size at capture, at instantiation, or at
replay determines whether an over-large captured node aborts loudly or deadlocks at the grid sync.
This cannot be settled by reading the repository and needs an experiment on the target device. It is
the one place a hang, rather than a crash, is plausible.

### Other SM-count assumptions

Persistent-grid and wave-sizing constants tuned against 170 SMs appear in the sparse MoE prefill
kernels, several launchers, and the `*_plan.cpp` dispatch tables. These degrade performance rather
than correctness, but the dispatch tables in particular were autotuned on one device and would need
re-running to select sensible schedules on another.

## Hopper: what it would and would not unlock

**Not longer context.** `src/targets/qwen3_6/impl/runtime/layouts_impl.h:395-396` rejects any
`max_context` above the variant's native capacity, and both variants set that to 262,144 tokens. The
request fails before allocation. Additional VRAM buys headroom, not reach.

**The real unlock is BF16 KV cache at full native context.** KV geometry is derived in
`src/core/kv_cache.cpp`, which shapes each K and V plane per layer and rounds the padded context up
to a multiple of 128. For the 27B target this works out to 64 KiB per token in BF16, so a full
262,144-token BF16 cache is 16 GiB. Added to the weights that configuration does not fit a 32 GB
card, which is why INT8 group-64 is the measured default. On a 141 GB card it fits comfortably.
Note that INT8 is slightly above half the BF16 cost rather than exactly half, because group-64
quantization adds an FP16 scale plane per K and per V. KV precision is already a runtime flag
(`--kv-dtype`), so this unlock costs no code.

**Bandwidth helps the dense target far more than the sparse one.** Batch-1 decode is a
memory-bandwidth-bound GEMV workload, so a card with substantially higher memory bandwidth raises
the dense 27B's decode ceiling roughly in proportion. The 35B-A3B is a mixture of experts that reads
only routed experts per step, so its weight traffic is a fraction of its resident set and it is much
less bandwidth-bound. Applying a single bandwidth ratio to both targets overstates the benefit for
the flagship model.

**Tensor peak stays largely unreachable.** `mma.sync` is warp-scoped and register-sourced. Reaching
Hopper's advertised tensor throughput requires `wgmma`, which is warpgroup-scoped and fed from
shared memory by TMA. The engine has neither, and stages through `cp.async`, which TMA supersedes.
The kernels would run correctly at roughly Ampere-class tensor utilization. Prefill, the
compute-bound phase, is where most of that gap would show.

**Most of the card would be idle.** The engine owns one resident sequence and runs one active
request, serialized behind a process-wide generation mutex (`src/runtime/engine/engine.cpp:121`).
With context capped and no second sequence, there is no consumer for the remaining capacity.
Batching is not a scheduler change: the K and V plane shape carries no batch, page, or block-table
dimension, so adding it means new cache geometry, new position plumbing, a rewrite of every
attention kernel, and rework of the graph capture machinery. `README.md` lists continuous batching,
multi-GPU execution, CPU/GPU offload, and distributed serving as not implemented.

## Lower-tier GPUs

For cards with substantially less memory the binding constraint is not the instruction set but
capacity. Measured resident weights are 19.59 GiB for the 35B-A3B at MTP0 and 20.56 GiB at MTP3,
before any KV cache, and the 27B artifact is 16.29 GiB. There is no offload path: `README.md` states
CPU/GPU offload is not implemented, and the weight arena is a device allocation established at load
time. A card that cannot hold the weights cannot run the model, independent of anything in this
document.

## Effort

| Workstream | Size | What it buys |
|---|---|---|
| Relax the build gate | hours | Compiles for the new architecture |
| Relax the runtime gate | hours | Engine constructs |
| Device-derived cooperative residency | 1 to 2 days | Removes the only known abort defect |
| Build and numerical validation | 1 to 2 weeks | Closes register-pressure and race questions static reading cannot |
| Resolve cooperative launch under graph capture | days | Removes the only plausible hang |
| SM-count-aware wave and persistent-grid constants | days | Recovers tail-wave losses, performance only |
| Re-run the autotuned dispatch tables | weeks, mechanical | Recovers schedule selection on a different SM count |
| Re-run the benchmark corpus | 1 to 2 weeks | Required before any performance claim |
| Hopper-native kernels (`wgmma` plus TMA, retile) | months | Access to the actual tensor peak. A different project |
| Continuous batching and paged KV | months | The only way to consume a large card |

Making it run is days. Making it fast is months. Making it worth a datacenter card is a different
engine.

## Caveats

The instruction inventory and the two gates were confirmed by direct reading. The memory and
bandwidth reasoning is derived from source plus measurements taken on an RTX 5090, not from runs on
other hardware, and nothing here has been executed on a non-Blackwell device. Treat the effort table
as an estimate to test rather than a plan to commit to, and note that the cheapest way to convert
the open questions above into data is to relax the two gates, build for the target, and measure.
