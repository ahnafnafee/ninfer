# tools/ AGENTS

Rules for work inside `tools/`. The repository root `AGENTS.md` applies first; these rules refine
behavior for maintainer tooling.

## Governing objective

Deliver the requested tool change or diagnostic result. Tools are project-owner workflows, not
public products. They must be correct and usable, but they do not carry backward-compatibility,
SDK, or distribution obligations.

## Scope control

- Tool implementation changes: modify the affected tool, update its README if behavior changed,
  and run it against the relevant artifact to confirm it still works.
- Diagnostics and investigations: use the appropriate tool to gather evidence. Report findings
  back to the caller. Do not turn a diagnostic run into an unrequested tool refactor.
- Conversion, reference, and parity tools are target-private. Do not generalize a 27B-only path
  into a family abstraction unless the task explicitly requires multi-target support.
- Do not add test infrastructure, CI hooks, or benchmark harnesses unless the task calls for them.

## Task routing

| Task | Location |
|---|---|
| Artifact conversion | `convert/<target>/` |
| `.ninfer` framing inspection | `artifact/inspect.py` |
| Python numerical reference | `reference/<target>/` |
| C++ activation dump | `qwen3_6_27b_dump/` (built with `-DNINFER_BUILD_TOOLS=ON`) |
| Parity comparison | `parity/<target>/` |
| Benchmark matrix | `bench/run_ninfer_bench_matrix.py` |
| Serving smoke tests | `smoke/serve_contract.py` |
| Local server switching | `ninfer-switch` |

## Python conventions

- Use `python3` explicitly (Python 3.11). Do not install or upgrade dependencies unless the task
  requires it.
- Modules under `tools/` use `__init__.py`. Run with `python3 -m tools.<path>` when a module path
  is documented, or `python3 tools/<script>` for standalone scripts.
- Target-private converters and references live under `convert/<target>/` and
  `reference/<target>/`. They may import shared helpers from `artifact/` but never import from
  another target's private directory.
- Numeric comparisons in parity tools use the oracle rules from the root `AGENTS.md`: one
  independent naive FP32/FP64 oracle per operator, exact comparison for transforms and codecs.

## Artifact and checkpoint access

- Product artifacts live at `out/<target>.ninfer` after build or download.
- Source checkpoints are local prerequisites; do not download them unless the task requires it.
- Large profiler outputs and generated reports live under `profiles/` and are git-ignored.

## ninfer-switch

The `ninfer-switch` bash script is a local-development convenience for WSL environments. It is not
a public CLI, not installed by CMake, and not part of the product distribution. It assumes:

- `~/bin/ninfer-serve` is available;
- model artifacts reside under `~/models/`;
- `LD_LIBRARY_PATH=~/lib` resolves CUDA runtime dependencies.

Do not generalize it into a cross-platform or CMake-installed tool.

## Build

Target-private C++ diagnostics require `-DNINFER_BUILD_TOOLS=ON`:

```bash
cmake -S . -B build -DNINFER_BUILD_TOOLS=ON
cmake --build build --parallel --target ninfer-qwen3_6_27b-dump
```

Python tools have no build step. Their dependencies are listed in per-tool `requirements.txt` files
when needed.

## Documentation

Update `tools/README.md` when adding a new tool or changing an existing tool's interface. Keep the
task index table current. Remove entries for deleted tools.