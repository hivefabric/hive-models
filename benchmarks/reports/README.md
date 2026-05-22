# Benchmark reports

This directory holds the JSON reports produced by `hive-bench` runs.

## Latest run — 2026-05-21 (with LFM2)

Run against the local docker stack with **four** model families:

* `qwen2.5:0.5b` — comb-node-demo (Ollama, 397 MB)
* `gemma:2b` — comb-node-demo-2 (Ollama, ~1.5 GB)
* `lfm2.5-thinking:1.2b` — comb-node-demo-3 (Ollama, 731 MB) — **new, Liquid AI**
* `llama3.2:3b` — queen-comb-demo (Ollama, ~2 GB, queen orchestrator)

### math-basics — 19/24

| Target | Pass | Avg latency |
|---|---|---|
| **lfm2.5-thinking:1.2b** | **6 / 6** | 11.4 s |
| gemma:2b | 5 / 6 | 3.2 s |
| qwen2.5:0.5b | 3 / 6 | 0.85 s |
| sum_agent (qwen) | 3 / 3 | 3.2 s |
| multiply_agent (gemma) | 2 / 3 | 2.4 s |

LFM2.5 ran the table on arithmetic — perfect 6/6, ahead of every other model. Cost: roughly 13× the latency of qwen, since it's a chain-of-thought model.

### freeform-qa — 9/15

| Target | Pass | Avg latency |
|---|---|---|
| gemma:2b | 3 / 5 | 2.9 s |
| lfm2.5-thinking:1.2b | 3 / 5 | 27.9 s |
| qwen2.5:0.5b | 3 / 5 | 2.1 s |

Three-way tie at 3/5. The remaining failures are grader-tightness issues: all three models return `299,792,458 m/s` for the speed-of-light prompt (grader expected `299792` without comma); refusal phrasings vary.

### Note on LFM2.5-thinking and the agent path

`lfm2.5-thinking:1.2b` is a **thinking** model: it returns a chain-of-thought via Ollama's OpenAI-compat layer in a separate `encrypted_content` field that the current `slm` agent handler does not unwrap. With the existing "Reply with ONLY the integer" prompts, the visible `output_text` ends up empty because the answer lands in the thinking trace.

**Direct-inference benchmarks work cleanly** (the chat-completions content extraction picks up the visible answer). Agent dispatch needs either (a) a thinking-aware extractor in `slm.rs`, or (b) a non-thinking lfm2 variant. Filed as a known gap in `models/lfm2.5-thinking-1.2b.yaml`.

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
