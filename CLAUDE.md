# CLAUDE.md — hive-models

## What this is

The HiveFabric capability catalog. YAML files that describe what models, agents, and benchmarks are available in the hive. Consumed by:
- **Honeycomb** at startup for hardware-floor validation (`HONEYCOMB_MODEL_CATALOG_PATH`)
- **hive-bench** for benchmark suite execution
- **Comb nodes** to know which models to pull and advertise

This repo contains data, not code. Any service with `Catalog::from_dir(path)` can consume it.

## Directory structure

```
models/         Model capability entries (19 entries)
agents/         Demo agents for benchmarks (2 entries: sum, multiply)
benchmarks/     Benchmark suite YAML + benchmark reports
examples/
  prompt-templates/  Reference prompt patterns (NOT registered capabilities)
```

## Design decision — models only, no task agents

**Production capabilities are model URNs only.** The catalog does not contain pre-baked task agents like `code_review_agent` or `summarise_agent`. The reason:

> The *what* (code review, test generation, summarisation) lives in the caller's prompt. HiveFabric routes to a model that can handle it. There is no special-cased handler.

The `examples/prompt-templates/` directory has reference prompt patterns for common tasks, but these are NOT registered capability URNs.

**Exceptions:** `sum_agent` and `multiply_agent` in `agents/` are benchmark fixtures — they exist because the math-basics benchmark requires exact integer output and the prompt engineering is load-bearing. They use the `oasf://demo/` namespace, not `oasf://commons/`.

## Capability URN format

```
oasf://{namespace}/{domain}/{model-id}/v{version}

Examples:
  oasf://commons/inference/qwen2.5-7b/v1
  oasf://commons/inference/qwen2.5-coder-7b/v1
  oasf://demo/agents/sum/v1
  oasf://commons/openclaw/bridge/v1
```

## Model tiers

| Tier | Size | RAM | Use case |
|---|---|---|---|
| Tiny | 135M–360M | 1–2 GB | Classification, routing, phones |
| Small SLM | 0.5B–3B | 3–6 GB | Extraction, summarisation, tool-calling |
| Medium LLM | 7B–14B | 10–18 GB | Code, math, reasoning, long context |
| Large LLM | 70B+ | 48 GB+ | Frontier-quality, approaches GPT-4 |

## Adding a new model

1. Create `models/{model-id}.yaml` following the schema in `README.md`
2. Required fields: `id`, `name`, `capability_urn`, `runtime`, `pull_tag`, `size_mb`, `min_memory_mb`, `min_cpu_cores`, `supports_tool_calling`, `license`, `tier`
3. Commit and push to `main`
4. Combs will discover it when they reload their catalog

## How to test

The catalog is validated at parse time. Test with:
```bash
cargo test -p hive-models  # validates all YAML entries
```

## What's not done

- Gemini and Bedrock model entries (Phase 2.4)
- vLLM / TensorRT runtime entries (currently all Ollama)
- Per-model benchmark results embedded in YAML (roadmap)
- Catalog versioning / semver for model entries
