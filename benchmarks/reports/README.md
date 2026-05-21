# Benchmark reports

This directory holds the JSON reports produced by `hive-bench` runs.

## Latest run

Run on **2026-05-21** against the local docker stack:

* `qwen2.5:0.5b` — comb-node-demo (Ollama)
* `gemma:2b` — comb-node-demo-2 (Ollama)
* `llama3.2:3b` — queen-comb-demo (Ollama, queen orchestrator)

| Suite | Pass / Total | Notes |
|---|---|---|
| `math-basics` | 15 / 18 | qwen and gemma each missed one product (`13×17` and `0×999`); `multiply_agent` returned 151 for `13×17` — real model regressions |
| `freeform-qa` | 7 / 10 | Both models returned `299,792,458 m/s` for speed-of-light (grader expected `299792` without comma); gemma's refusal phrasing didn't match the configured `expected_contains` set — both grader-tightness issues |
| `queen-decompose` | 2 / 4 | `compound-add-mul`: queen only called `multiply_agent` (model shortcut the decomposition); `greeting-no-tools`: queen called `ask_inference` for a greeting (over-eager dispatch) |

The two `queen-decompose` failures are exactly the kind of regression a benchmark is supposed to catch — both are issues with the decomposition prompt or model choice, not the runner.

## How to reproduce

```bash
cd hive-sdk/packages/hive-bench
cargo build
./target/debug/hive-bench \
  --catalog ../../../hive-models \
  --bench all \
  --report /tmp/all.json
```

The runner reads the catalog, queries `/api/nodes` for live URN advertisements, dispatches every (target × case) pair through Honeycomb, polls until terminal, and grades the outputs. See `hive-models/README.md` for the schema and `hive-sdk/packages/hive-bench/src/lib.rs` for the runner internals.
