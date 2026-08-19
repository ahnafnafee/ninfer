# Windows port

NInfer currently builds and runs on 64-bit Linux. This document records what a native Windows build
requires, what the alternatives are, and which parts of the work are done.

The porting surface is small. Of 597 first-party sources, only four ever reach for POSIX
interfaces, and three of those are already handled. Outside `third_party/` there are no GCC or Clang
host-compiler extensions and no use of the C++20 features that MSVC and nvcc handle poorly. The
cost is concentrated in one class rewrite, dependency discovery, and toolchain configuration.

## Status

| Area | State |
|---|---|
| `ninfer_text` static-linkage define | landed |
| Media-root containment guard | landed |
| Serve process-id and log stream mode | landed |
| Load-progress TTY detection | not started |
| Toolchain prerequisites | not started |
| Dependency discovery (FFmpeg, libcurl) | not started |
| MSVC compile flags and presets | not started |
| Artifact reader (`MappedFile`) | not started |
| Media acquire Winsock headers | not started |
| Serve shutdown signalling | not started |
| CLI console and stream modes | not started |
| Runtime DLL deployment | not started |

## Choosing a route

| | Native MSVC | WSL2 | Docker Desktop with WSL2 |
|---|---|---|---|
| Source changes | six blockers, one rewrite | none | none |
| Time to first run | weeks | hours | under an hour |
| Builds tests, benchmarks, tools | after extra work | yes | no, disabled in the Dockerfile |
| Produces an executable for Windows | yes | no | no |
| GPU access | direct, WDDM | paravirtualized | paravirtualized |
| Pinned host memory | full | documented as limited | documented as limited |
| Unbuffered artifact read | must be reimplemented on Win32 | works on ext4, about 4 GiB/s | works on ext4 |
| Performance against published results | unmeasured | measured at or above baseline | not separately measured |

Native MSVC is the right route when a Windows executable is the deliverable, when Windows-side
profiling is needed, or when the engine has to be embedded in a Windows application. Otherwise WSL2
reproduces the Linux build exactly and is the only route that builds the full tree including tests,
benchmarks, and tools.

Whichever route is chosen, building under WSL2 first is worthwhile: it produces a known-good
baseline on the same GPU to measure the native port against.

### WSL2 and Docker notes

The artifact reader opens the model with `O_DIRECT` and has no buffered fallback, so the `.ninfer`
file must live on the WSL ext4 filesystem rather than on a mounted Windows drive. Whether a Windows
drive mount rejects the flag or silently ignores it has not been tested, and is worth confirming
before publishing instructions.

Docker Desktop's WSL2 backend includes the NVIDIA container runtime, so `--gpus` works without
installing the container toolkit separately. The `Dockerfile` disables `BUILD_TESTING`,
`NINFER_BUILD_BENCHMARKS`, and `NINFER_BUILD_TOOLS`, so it produces the two applications and
nothing else. Docker Desktop's per-distro WSL integration is off by default; without it the `docker`
CLI inside a WSL distro is the Windows binary and cannot bind-mount that distro's paths, so either
enable the integration or keep the artifact in a Docker volume.

### Measured WSL2 results

Both registered targets were run under WSL2 on Ubuntu 24.04 against an RTX 5090 with the desktop
attached, using the committed `examples/cli/` fixtures and the `docs/performance.md` server
settings: max context 262,144, prefill chunk 1,024, INT8 group-64 KV, CUDA Graph on, prefix reuse
off, temperature 0.6 / top-p 0.95 / top-k 20 / presence penalty 1.0. Each phase discards a warm-up.
Metrics come from the server's own request JSONL phase timings using the formulas that document
specifies. Both artifacts were SHA-256 verified against the README before running, and both passed
the `text_smoke_zh` correctness oracle with the expected exact output.

**Qwen3.6-35B-A3B**

| Fixture | Metric | WSL2 | Published | Ratio |
|---|---|---|---|---|
| `long_niah_8k` (7,680 tok, MTP0) | prefill | 16,203-16,222 tok/s | 15,544.3 | 104% |
| `long_niah_8k` (7,680 tok, MTP0) | decode | 296.8-303.6 tok/s | 271.1 | 110-112% |
| `long_niah_256k` (260,096 tok, MTP0) | prefill | 5,137.6 tok/s | 5,157.1 | 99.6% |
| `long_niah_256k` (260,096 tok, MTP0) | decode | 194.5 tok/s | 188.2 | 103% |
| `scenario_structured_jsonl` (MTP3) | decode | 762.0 tok/s | 714.3 | 107% |
| `scenario_structured_jsonl` (MTP3) | acceptance | 95.6% | 87.7% | above |
| `long_decode_aime26_01` (MTP3) | decode | 657.3 tok/s | 584.0-695.1 | within range |
| `long_decode_aime26_01` (MTP3) | acceptance | 79.5% | 72.4-83.3% | within range |

**Qwen3.6-27B**

| Fixture | Metric | WSL2 | Published | Ratio |
|---|---|---|---|---|
| `long_niah_8k` (7,680 tok, MTP0) | prefill | 3,272-3,298 tok/s | 3,218.1 | 102% |
| `long_niah_8k` (7,680 tok, MTP0) | decode | 79.8-79.9 tok/s | 77.6 | 103% |
| `long_niah_256k` (260,096 tok, MTP0) | prefill | 1,631.4 tok/s | 1,614.8 | 101% |
| `long_niah_256k` (260,096 tok, MTP0) | decode | 56.8 tok/s | 54.8 | 104% |
| `scenario_structured_jsonl` (MTP3) | decode | 202.5 tok/s | 193.0 | 105% |
| `scenario_structured_jsonl` (MTP3) | acceptance | 96.4% | 88.7% | above |
| `long_decode_aime26_01` (MTP3) | decode | 175.2 tok/s | 161.9-175.4 | top of range |
| `long_decode_aime26_01` (MTP3) | acceptance | 79.4% | 73.4-78.8% | slightly above |

The full 262,144-token context fits on a 32 GiB card with the desktop attached, for both targets:

| Target | Mode | Weights | Sequence (KV payload) | Workspace | Total reserved |
|---|---|---:|---:|---:|---:|
| 35B-A3B | MTP0 | 19.59 GiB | 2.70 (2.58) GiB | 0.12 GiB | 22.41 GiB |
| 35B-A3B | MTP3 | 20.56 GiB | 3.14 (2.84) GiB | 0.12 GiB | 23.82 GiB |
| 27B | MTP0 | 15.25 GiB | 8.55 (8.25) GiB | 0.17 GiB | 23.97 GiB |
| 27B | MTP3 | 16.01 GiB | 9.50 (8.77) GiB | 0.17 GiB | 25.67 GiB |

The 27B carries lighter weights but a KV cache more than three times larger, because it has more
full-attention layers and more KV heads than the sparse target. See
[GPU portability](gpu-portability.md) for the per-token KV derivation. Artifact load ran at 3.3 to
4.1 GiB/s through the unbuffered read path on ext4, so server start including weight upload was
about 5 seconds in every configuration.

Read these as evidence that WSL2 imposes no visible penalty at these fixtures, not as a
recalibration of the published numbers. The runs used fewer seeds than the published five-seed
protocol and report per-run values rather than mean and deviation; the tested revision and driver
differ from those the performance document records; and the structured fixture reaches its output
cap rather than stopping naturally, which likely inflates its acceptance figure. The margins above
100% are therefore not attributable to WSL2 being faster than bare Linux.

None of this constrains the native Windows question. WSL2's paravirtualized GPU path is not the
Windows display driver model, so the residency question in the open-questions section below remains
entirely untested.

## Toolchain prerequisites

The CUDA version gate at `CMakeLists.txt:37-41` runs before dependency discovery, so an unsatisfied
toolkit floor is the first failure a Windows configure will report.

nvcc gates its host compiler on `_MSC_VER` in `crt/host_config.h`. Read that header from the
installed toolkit rather than inferring the bound from a published support table: the tables list
Visual Studio product versions, while the header enforces a numeric toolset range, and the two do
not correspond in an obvious way. When more than one toolset is installed, whichever developer
environment was initialized last decides which `cl.exe` nvcc selects, so pin it explicitly with
`vcvarsall.bat -vcvars_ver=<toolset>` or `CMAKE_CUDA_HOST_COMPILER`.

nvcc on Windows accepts only `cl.exe` as a host compiler. clang-cl would mean abandoning nvcc for
Clang's own CUDA support, which needs independent validation for the target architecture and does
not uniformly cover the inline PTX in `src/ops/common/`. MinGW and MSYS2 are out for the same
reason.

Use the Ninja generator, not the Visual Studio generator. Four things break under MSBuild:

- the device-link serialization guard at `CMakeLists.txt:32-35` is conditional on the Ninja
  generator and goes inert, allowing concurrent device links across the relocatable objects;
- the `CMAKE_BUILD_TYPE` default at `CMakeLists.txt:28-30` is ignored by multi-config generators,
  so a build with no explicit configuration produces Debug output;
- `CMAKE_EXPORT_COMPILE_COMMANDS` at `CMakeLists.txt:27` is ignored, breaking the `.clangd`
  workflow;
- the build tree layout diverges from the paths in the project README.

Debug configuration is impractical regardless. `_ITERATOR_DEBUG_LEVEL` defaults to 2 under MSVC
Debug, is ABI-affecting and must match across every dependency, and slows the double-precision CPU
oracles in the Op tests by roughly an order of magnitude. Use `RelWithDebInfo` as the debuggable
configuration.

## Remaining work

Ordered by dependency. Each stage assumes the previous one has landed.

### Dependencies

`CMakeLists.txt:59-64` discovers FFmpeg and libcurl exclusively through `pkg-config`, which is not
present on a stock Windows installation. Line 59 sits outside every conditional and
`ninfer_media_decode` is created unconditionally at `src/CMakeLists.txt:184-187`, so disabling the
optional build options does not avoid it. Two consumption sites hardcode the imported target names:
`src/CMakeLists.txt:187` and `src/CMakeLists.txt:194`.

libcurl is the straightforward half: `find_package(CURL REQUIRED)` yields an imported target from
either CMake's bundled finder or a package config, and that branch can replace the pkg-config path
on both platforms. FFmpeg has no upstream CMake finder, so the Windows path yields variables rather
than an imported target and `src/CMakeLists.txt:187` has to change shape. An interface library that
both platforms populate keeps the consumption site free of platform branching.

Two traps are worth knowing before provisioning:

**The declared FFmpeg minimums are lower than the code requires, on every platform.**
`src/media/decode/decode.cpp:354-356` uses `av_packet_side_data_get()` and reads
`AVCodecParameters::coded_side_data`. Both postdate the declared `libavcodec>=60` floor, which
admits releases that will not compile. Tighten the four minimums to the release that actually
introduced these APIs so a mis-provisioned environment fails at configure with a legible message
rather than at compile with an undeclared identifier. The existing container build is unaffected
because its base image ships a new enough FFmpeg.

**Package-manager feature selection can silently remove the PNG decoder.** FFmpeg's PNG decoder has
a hard zlib dependency, and at least one common Windows package configuration disables both
autodetection and zlib unless zlib is requested explicitly. Every committed image example is PNG,
and `src/media/decode/decode.cpp:171-172` queries the decoder registry without pre-validating the
result, so the failure surfaces at request time and reads as a vision defect rather than a
packaging one. Request zlib explicitly and verify the feature set against the exact package
baseline being pinned.

Two further consequences of Windows dependency delivery:

- There is no `install()` rule anywhere in the tree, and NInfer is documented as running from its
  build tree. On Linux shared libraries resolve through rpath. On Windows the executable needs the
  CUDA runtime, the FFmpeg libraries, libcurl, and their transitive dependencies co-located or
  discoverable, or it fails before `main()`. A post-build copy of `$<TARGET_RUNTIME_DLLS:...>` is
  the cheap fix. Note that this expands to nothing for dependencies arriving through a pkg-config
  imported target, since those record import libraries rather than runtime binaries. That is an
  independent reason to prefer package configs.
- Use a dynamic-runtime configuration. CMake's default MSVC runtime setting is the DLL variant and
  propagates to nvcc; mixing static-runtime dependencies into it produces either a link failure or
  heap corruption across the module boundary. The latter is a live hazard here because the decoder
  receives FFmpeg-allocated buffers and the acquisition path receives libcurl-allocated buffers.

### Build configuration

Three compile settings are required. Justification, not habit:

- `/utf-8`. `tests/targets/qwen3_6/test_frontend.cpp:64` and `:525` carry raw UTF-8 inside narrow
  string literals in a file with no byte-order mark. Without the flag MSVC decodes them using the
  active code page and re-encodes into the execution character set, silently altering tokenizer
  fixture data; the test then fails with a token mismatch that reads as a tokenizer regression.
  Sources under `src/` use hex escapes for the same purpose and are unaffected in substance, but
  carry non-ASCII in comments widely enough that omitting the flag buries real diagnostics under
  code-page warnings. Keeping hex escapes as the rule for CUDA sources remains worthwhile, because
  the flag only affects the host front end while nvcc's device front end reads the file
  independently.
- `/bigobj`. `src/serve/http_server.cpp` combines the vendored HTTP and JSON headers in one
  translation unit, and the JSON header is included by nineteen files. The flag has no codegen
  effect.
- `NOMINMAX` and `WIN32_LEAN_AND_MEAN`. Not yet required, because nothing first-party includes
  Windows headers today and the vendored HTTP header defines `NOMINMAX` itself before including
  Winsock. They become mandatory the moment the artifact reader and media acquisition changes land,
  given the number of `std::min` and `std::max` call sites. Set them at project scope rather than
  relying on include order.

Forward the first two to nvcc's host pass with `-Xcompiler`. `/Zc:preprocessor` is worth applying
prophylactically but was not required by the toolkit tested here, so it should not be budgeted as a
blocker.

`CMakePresets.json` does not exist and is the highest-leverage ergonomics addition: the generator,
toolchain file, triplet, runtime library, and CUDA host compiler all belong there rather than in a
command line that has to be retyped correctly. `.gitignore` already lists `CMakeUserPresets.json`,
so presets were considered previously.

### Milestone: compile the Op library

```
cmake --build build --target ninfer_ops
```

This compiles the CUDA translation units and device-links them with no POSIX porting required. It
validates the entire CUDA and MSVC toolchain end to end, and surfaces the compile-flag and
designated-initializer-ordering issues, before any Win32 code is written. Do this before starting
the reader rewrite.

Compile-only validation needs no GPU. The architecture is fixed before `project()` specifically so
CMake never injects a native-architecture probe; CMake's CUDA ABI and feature detection compiles and
links a test binary but never runs it; and the container build already compiles the full default
target set without GPU access. A GPU-less Windows CI runner can therefore assert configure, compile,
device link, and inspection of the emitted fatbin. It cannot assert anything that launches a kernel,
which is all of `tests/` and `bench/`; that needs a self-hosted runner.

### MSVC conformance

The sweep result is mostly an absence, which is worth recording as a finding rather than silence.
There are no GCC or Clang extensions outside `third_party/`: no `__attribute__`, no `__builtin_*`,
no statement expressions, no `typeof`, no `__int128`, no `#pragma GCC`. There are no concepts, no
ranges, no `std::format`, no coroutines, no modules, no three-way comparison. There are no bare
`long` declarations, so the data-model difference between Linux and Windows cannot bite. The inline
PTX all sits inside device functions consumed by nvcc's own front end, so the host compiler never
sees it.

What does need attention:

- **Designated initializers**, used widely. Supported under `/std:c++20`. The action item is
  negative: never fall back to `/std:c++17` as a workaround for something else, because MSVC does
  not accept them as a C++17 extension the way GCC does. C++20 also requires designators in
  declaration order where GCC permits reordering; any offenders appear as clean errors on the first
  compile.
- **Template lambdas** invoked through explicit `operator()` template arguments, in both host and
  device code. This exercises MSVC's newer lambda processor, which `/std:c++20` does not enable on
  its own across all toolset versions. Pin a toolset new enough that `/permissive-` implies
  `/Zc:lambda`, or pass `/Zc:lambda` explicitly. Without it the errors are parse failures that are
  easy to misattribute to nvcc.
- **`ssize_t`** at `src/artifact/reader.cpp:228` and `:232`. Not declared by the MSVC runtime. Use a
  fixed-width signed type. Do not add a project-wide alias: the vendored HTTP header already
  declares one at global scope on Windows, and a mismatched second declaration is a hard error.
- **Missing direct standard includes** that libstdc++ currently satisfies transitively. Every
  instance is a compile error, never a behavior change. Fix iteratively as the compiler reports
  them.

On floating point, `tests/CMakeLists.txt:31-38` applies contraction and fast-math suppression only
for GNU and Clang, leaving MSVC unguarded. The practical gap is small, because the MSVC default
model already disables contraction, but stating it explicitly is cheap. Do not reach for
`/fp:strict`, which also changes exception and rounding-environment semantics at a real cost. Note
that clang-cl reports as Clang and would match the existing branch, receiving GNU-spelled flags.

### Artifact reader

This is the only genuine rewrite in the port, and it unblocks the widest surface: `ninfer_artifact`
is unconditional and feeds `ninfer_engine`, which gates every target.

The work is confined to one class, `MappedFile` at `src/artifact/reader.cpp:173-247`, plus the
includes at `:18-21`. The rest of that file, and all of `materializer.cpp`, `binder.cpp`, and
`typed_binding.cpp`, is portable.

| Construct | Windows replacement |
|---|---|
| POSIX I/O headers | Windows headers. The MSVC `fcntl.h` and `sys/stat.h` compatibility headers are a dead end: no unbuffered flag, no mapping |
| `open` with read-only, close-on-exec, and direct flags | `CreateFileW`. `path::c_str()` becomes correct and idiomatic here, since it yields wide characters. Close-on-exec is free: handles are non-inheritable with null security attributes |
| `struct stat` and `fstat` | `GetFileSizeEx`, whose signed 64-bit result preserves the existing negative-size sanity check |
| `mmap` read-only private | `CreateFileMappingW` with read-only page protection, then `MapViewOfFile` with a zero size meaning the whole file |
| `MAP_FAILED` comparison | Both mapping calls return null on failure, not a negative sentinel |
| the zero-size guard | Keep it. It becomes load-bearing: mapping a zero-length file fails on Windows where `mmap` succeeds |
| `munmap` and `close` | Unmap the view, then close each handle. Two different failure sentinels now exist in one class, which is an easy source of a leaked-handle defect |
| `pread` with interrupt retry | `ReadFile` with an overlapped structure carrying the 64-bit absolute offset. This is documented as valid on a handle not opened for overlapped I/O and blocks until complete. The interrupt-retry loop becomes dead code |

Use two handles, one buffered for the mapping and one unbuffered for bulk reads, because the
unbuffered flag is handle-wide on Windows whereas the POSIX direct flag is a property of the read
call only. Both paths here are read-only and nothing writes the file, so this is a cleanliness and
cache-behavior decision rather than a correctness requirement: it lets metadata stay cached while
the multi-gigabyte payload bypasses the cache.

Two semantics must be preserved deliberately.

**Short reads past end of file occur on every model load.** `src/artifact/materializer.cpp:194-196`
rounds each request length up to the direct-I/O alignment, so the final chunk of the final span
reads past the logical end of the file. Neither registered artifact is aligned to that boundary.
Windows splits this into two cases that must both be treated as short reads: a read starting before
the end and extending past it succeeds with a partial count, while a read starting at or beyond the
end fails with an end-of-file status. Mapping the second to a throw fails every load on its final
chunk. The caller's existing short-count check at `materializer.cpp:199-201` already tolerates a
partial result correctly.

**Error category changes.** The current code pairs the C error number with the generic category.
Windows status codes are not error numbers and must be paired with the system category, or the
message text is meaningless.

What does not change: the direct-I/O alignment constant and the runtime assertion at
`src/artifact/reader.cpp:222-226` are exactly the right check on Windows, because unbuffered I/O
requires the offset, the byte count, and the destination address to be sector-size multiples, and
the existing constant is a multiple of the sector sizes in use. Destination buffers come from pinned
host allocations and are page-aligned. Optional hardening: query the volume sector size at open and
fail with a clear message if it exceeds the constant, rather than letting every read fail with an
opaque parameter error.

One behavior change to decide consciously: opening with a restrictive share mode makes the artifact
undeletable while the reader is alive, where Linux imposes no such lock. In practice the reader is a
function-local in `src/targets/registry.cpp:122` destroyed once weights are resident, so the lock
spans the load only, not the server lifetime. Add delete sharing if hot-swapping the artifact during
startup matters.

Finally, a throughput note that is not a correctness matter. The load loop is serial: the pinned
slots pipeline the host-to-device copy against the next read, but the read itself is blocking and
single-threaded, so at most one file read is ever in flight. A synchronous Windows port preserves
that. Overlapped reads, one per slot, would match the slot count. Do not simplify by dropping
unbuffered I/O: a buffered read of the full artifact pushes its entire size into the system standby
list, which is exactly the pressure the current design exists to avoid.

### Media acquire

`ninfer_media_acquire` is forced on by `CMakeLists.txt:43-46` whenever applications, tests, or tools
are built, and is a transitive dependency of both the serving library and the prompt-input library.

Replace the POSIX networking headers at `src/product/media_acquire/acquire.cpp:5-7` with the
Winsock 2 headers, ordered so Winsock precedes any Windows header, and link the sockets library.
`NOMINMAX` matters in this file specifically, because it uses `numeric_limits::max()` and
`std::min` nearby. Everything the file calls survives the header swap with matching signatures:
address resolution and release, address-to-text conversion, byte-order conversion, and every struct
field it touches. The address-classification guard ports intact, because the address-family test
macros and union-member accessors it depends on all exist on Windows. One wrinkle: the error-string
function is a per-thread-buffer macro on Windows and is not thread-safe, so prefer formatting the
numeric code.

Explicit Winsock initialization is not required. `curl_global_init` with the default flag set
includes socket initialization, runs inside the existing one-time initialization at
`acquire.cpp:173-178`, and the only resolver call site is downstream of it. Adding an explicit call
is defensible hardening against a future reordering, nothing more.

One Windows-specific concern that the compile fix does not address: `weakly_canonical` and
`relative` do not normalize NTFS alternate data streams, short names, trailing dots or spaces,
reserved device names, or extended-length prefixes. Since the containment guard governs
caller-supplied input, the Windows port widens the bypass surface even with the guard behaving
correctly. This deserves a dedicated test matrix before the media path accepts untrusted input on
Windows.

### Serve

`src/serve/http_server.cpp` needs no changes. The vendored HTTP library's Windows branches are
complete and correctly gated, it links the sockets library itself, it defines `NOMINMAX` before
including Winsock, and it manages its own socket-subsystem lifecycle. No TLS backend is configured
anywhere in the project, so none of the related link dependencies apply.

Two items remain:

- `apps/serve/main.cpp:47-48` registers signal handlers that Windows does not deliver. The
  termination signal is defined by the runtime but never raised by the operating system, so a normal
  process-stop request never reaches the graceful shutdown path and the server is terminated with
  requests in flight. The interrupt signal does fire on console break, but the runtime dispatches it
  on a separate thread rather than interrupting the main one; the existing atomic server pointer
  makes that safe. Add a console control handler covering the close, logoff, and shutdown events,
  keeping the signal registration as a fallback. Two constraints: those handlers run under an
  operating-system deadline of a few seconds before the process is killed regardless, so the stop
  path must be fast, and running as a service requires a service control handler instead. One free
  improvement: there is no broken-pipe signal on Windows, so a client disconnecting mid-stream
  cannot terminate the process.
- Streaming disconnect detection at `src/serve/http_server.cpp:380-387` and `:611-616` polls the
  vendored library's socket-liveness check so a vanished client cancels generation during prefill.
  That check has an error-number fast path that is inert on Windows, so control always falls through
  to a peek-based probe, which still reports correctly in the common case. There is no source fix
  short of forking the vendored header. Treat it as a required manual acceptance test: start a
  streaming request, terminate the client during prefill, and confirm the generation worker is
  cancelled rather than running to completion.

### CLI and console

None of these block the build, but all three affect whether the port looks correct.

- `apps/cli/main.cpp:117-132` writes model output to standard output in text mode, so newlines are
  translated on the way to a file. The CLI documentation advertises redirection as the supported
  capture mechanism, and any golden-file or hash comparison against Linux output will fail. Set the
  underlying descriptors to binary mode at the top of `main`, before any output.
- Raw UTF-8 model output goes to the console with no code-page configuration, so non-Latin output is
  garbled interactively. Display only; redirected output is correct once binary mode is set.
- Command-line arguments arrive in the active code page, so a non-ASCII artifact or messages path is
  lossily converted before reaching `std::filesystem::path`.
- `src/product/load_progress/load_progress.cpp:3,63` includes `<unistd.h>` and calls
  `::isatty(STDERR_FILENO)` to decide whether the weight-load progress line redraws in place or
  prints one record per update. Neither the header nor the constant exists under MSVC; the
  equivalent is `_isatty(_fileno(stderr))` from `<io.h>`. This is the only POSIX include outside the
  three sites already handled, and it is a compile error rather than a behavior difference.

An application manifest declaring UTF-8 as the active code page fixes all three with no code change,
and also defuses a latent hazard in the reader error paths at `src/artifact/reader.cpp:178`, `:186`,
and `:201`: the narrow-string conversion used to build those messages is specified to throw on
unrepresentable characters, which would mean throwing while constructing an exception message.

### Runtime hardening

None of these block a Windows build, and all are latent on Linux too. Windows adds a second
toolchain in which the assumptions can drift, which is why they are worth closing as part of this
work.

- **No cooperative-launch capability guard.**
  `src/ops/gdn_gating_proj/bf16/bf16_gdn_gating_proj_kernels.cu:285-290` issues a cooperative launch
  unconditionally, with no query of the corresponding device attribute anywhere in the tree. The
  failure path is an abort, not an exception, and it occurs inside graph capture. Measurement on a
  target card shows cooperative launch is supported under the ordinary display driver mode, so this
  is defensive hardening rather than a Windows blocker; the widely repeated claim that it requires a
  compute-only driver mode is stale. A non-cooperative route already exists in the plan table.
- **Hardcoded occupancy constants.**
  `src/ops/gdn_gating_proj/bf16/bf16_gdn_gating_proj_plan.cpp:126-141` hardcodes a per-thread
  register count and a device-wide resident-block limit measured on one Linux build. If the Windows
  assembler allocates one more register per thread and occupancy drops, the launched grid exceeds
  what is co-resident and the grid-wide synchronization hangs rather than returning an error.
  Replace the literals with a runtime occupancy query multiplied by the device multiprocessor count,
  and compare verbose assembler output across platforms before trusting anything else.
- **The graph-allowance check.** `src/targets/qwen3_6/impl/runtime/program_impl.h:420-427` throws if
  the free-memory delta across graph capture exceeds a planned allowance. The measurement is a
  system-wide free-memory delta sampled across a multi-second capture loop, so on a desktop any
  other process moving video memory during that window can trip it with a message naming a cause
  that did not occur.
- **Abort on CUDA error.** `src/core/device.cu:38-43` aborts on every CUDA error. Combined with a
  release build carrying no debug symbols, a Windows failure produces a fault dialog with no
  symbolized stack, which is strictly worse than the Linux behavior of a diagnostic line plus a core
  file. Converting this to a throw would let the existing HTTP exception handler return a server
  error instead of killing the process with clients connected. Independently, emit debug information
  in release builds.

### Tests, benchmarks, and tools

Enabling `BUILD_TESTING` forces the serving and media-acquisition libraries on, so tests add no
independent porting surface. They only remove the option of deferring the application work. The
shared CUDA test harness and the benchmark harness are fully portable, using only the CUDA runtime
and the standard library.

Four specific items:

- `tests/test_request_log.cpp:14` uses the same process-id pattern the serving library did.
- `tests/test_row_split_pack_golden.cpp:26-37` builds a POSIX single-quoted command string and runs
  it through `std::system`. On Windows that invokes the command interpreter, where the single quote
  is not a quoting character. Registering the script step as its own test with an argument vector
  removes the problem on both platforms.
- `tests/targets/qwen3_6/test_frontend.cpp:145-149` and `:168-172` hardcode absolute paths under one
  developer's home directory with no skip guard, unlike the opt-in artifact tests that correctly use
  an environment variable and a skip return code. Already broken on any other machine, so Windows
  does not make it meaningfully worse. Several Python modules hardcode the same paths at module
  scope.
- The reference-implementation dependency requires Triton, which has no official Windows support.
  Run the Python parity tooling under WSL2, or confirm whether the guarded import has a pure-PyTorch
  alternative.

## Open questions

These are not resolved and should not be presented as if they were.

1. **Video-memory residency under the Windows display driver model.** The largest unknown, and the
   one that decides whether the native route is viable for large-context work. The video memory
   manager can demote a process's allocations to system memory. `src/targets/registry.cpp:42-56`
   performs a one-shot free-memory admission check at load time and nothing re-verifies residency
   afterwards; there is no residency hint, prefetch, or watchdog anywhere in the tree. If demotion
   occurs the failure mode is not an error, it is a silent throughput collapse with no diagnostic.
   Measure this before publishing any Windows performance figure. Note that the simpler concern,
   whether desktop video-memory overhead leaves room for the resident footprint at all, was measured
   and is not a problem.
2. **Whether the device-link object survives archive-member selection.** `src/CMakeLists.txt:9-14`
   enables separable compilation and device-symbol resolution on static libraries whose consumers
   contain no CUDA sources, so no device link occurs at the executable. The Microsoft linker pulls an
   object from an archive only when a referenced symbol demands it, and a device-link object whose
   only contribution is fatbin registration through static initializers has no such symbol. The
   failure would be a clean build followed by an invalid-device-function error at the first kernel
   launch, which a compile-only CI would not catch. This is the first thing to check after the build
   goes green.
3. **Whether the Windows assembler produces the same register counts** the plan file hardcodes.
   Device code generation runs on nvcc's own front end and should be host-compiler independent, but
   that is an assumption, and a drop in resident blocks per multiprocessor turns a grid-wide
   synchronization into a hang with no error code.
4. **Cooperative launch inside graph capture on Windows.** The mixture-of-experts decode path does
   both at once. Each is independently confirmed supported; the combination is not covered by any
   source consulted.
5. **Whether unbuffered reads short-read cleanly at end of file** on both 512-byte-emulated and
   native 4K volume geometries. Every model load exercises this.
6. **Whether the direct-I/O flag on a Windows drive mount inside WSL** returns an error or is
   silently ignored. The ext4 case is settled: the artifact loads at roughly 4 GiB/s from the WSL
   filesystem, so the unbuffered path works there. The mounted-drive case is still untested, and it
   decides only whether the instructions must forbid model files on a Windows drive.
7. **Whether the pinned CMake version accepts the architecture feature suffix.** The build hard-errors
   on any other architecture value, so if the suffix misbehaves that guard must be relaxed before the
   problem can even be bisected. A related claim, that the suffix raises the shared-memory ceiling,
   does not hold: the documented difference is exposure of architecture-accelerated instructions and
   the absence of forward-compatibility guarantees. Since none of the inline PTX uses an
   architecture-exclusive instruction, the plain architecture number would very likely also work.

## Getting started

### Configure

Run from a developer command prompt with the toolset pinned to one inside the installed toolkit's
accepted range, then:

```
cmake -S . -B build -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_TOOLCHAIN_FILE=<package-manager toolchain file> ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL
cmake --build build --target ninfer_ops
```

When more than one toolset is installed, also set `CMAKE_CUDA_HOST_COMPILER` to the exact `cl.exe`
intended, because otherwise the last initialized developer environment decides silently.

### Compile settings

Add to the root `CMakeLists.txt` after the build-type default:

```cmake
if(MSVC)
  # /utf-8 is a correctness requirement, not cosmetics: several sources carry raw
  # UTF-8 inside narrow string literals, which MSVC otherwise re-encodes through the
  # active code page. /bigobj is needed for the translation units that combine the
  # vendored JSON and HTTP headers.
  add_compile_options(
    $<$<COMPILE_LANGUAGE:C,CXX>:/utf-8>
    $<$<COMPILE_LANGUAGE:C,CXX>:/bigobj>
    $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=/utf-8>
    $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=/bigobj>)
  add_compile_definitions(NOMINMAX WIN32_LEAN_AND_MEAN)
endif()
```

### Repository hygiene

Neither file exists today and both should land alongside the port.

`.gitattributes` normalizing line endings prevents a Windows clone with default conversion from
rewriting every text file, which matters for byte-comparison tests and for diff noise.

`.gitignore` currently lists object, shared-library, and archive extensions but omits every Windows
equivalent, including the import libraries, dynamic libraries, executables, and debug symbol files
that build-tree-local runtime deployment would place next to the binaries.

## Sequencing and effort

Person-days for one engineer familiar with C++ and CUDA but not this codebase. Compiling and running
correctly is separated from running fast, which is not estimable until the residency question is
measured.

| Workstream | Blocks | Effort |
|---|---|---|
| Toolkit install and toolset pinning | everything | 0.5 |
| Dependency discovery, FFmpeg floor, feature set | configure | 2 to 3 |
| Compile settings, presets, generator commitment | compile | 1 |
| Milestone: Op library compiles and device-links | validation | 0 |
| MSVC conformance sweep | compile | 2 to 3 |
| Artifact reader rewrite | everything downstream | 2 to 4 |
| Media acquire Winsock headers | serve, apps, tests, tools | 1 to 2 |
| Serve shutdown signalling | server lifecycle | 1 to 2 |
| CLI console and stream modes | usability | 1 |
| Runtime dependency deployment | first successful run | 0.5 to 1 |
| Milestone: both applications run | | cumulative 12 to 18 |
| Bring-up and debug slack | | 3 to 5 |
| Runtime hardening | trust | 2 to 4 |
| Tests green on Windows | test signal | 2 to 3 |
| Performance investigation | published results | blocked on residency |

The three landed changes are already counted inside the workstreams above.
