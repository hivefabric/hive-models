# hive-models

Canonical, version-controlled registry of supported LLMs, agents, and benchmarks for HiveFabric. **One file per entry.** PR-able, language-agnostic (just YAML), embeddable.

## Why this exists

Without a central catalog, every comb operator hand-writes the same model + URN + tier + license metadata in their TOML. Drift accumulates: comb-1 calls the model `qwen2.5:0.5b` and advertises `oasf://commons/inference/qwen2.5-0.5b/v1`, comb-2 calls it `qwen-0.5b` and advertises `oasf://hive/inference/qwen/v1` — the scheduler can't route across them.

The catalog is the **single source of truth** for:

- Model id ↔ capability URN binding
- Pull tag for the runtime (Ollama, vLLM, …)
- Hardware floor (`min_memory_mb`, `min_cpu_cores`)
- Tool-calling support flag (which models can run as queens)
- License (gated by future Hive-Keeper policy)
- Agent prompt templates and their compatible models
- Benchmark suites for measuring model and agent quality

## Layout

```
models/         one YAML per model
agents/         one YAML per agent definition
benchmarks/     one YAML per benchmark suite
```

The three directories are **orthogonal**. Models advertise inference URNs. Agents advertise themselves on top of models. Benchmarks score what models and agents produce. Adding any of them is a one-file PR.

> **A note on benchmarks vs capabilities.** Benchmarks are *not* listed as runtime capabilities a comb advertises. They are catalog entries the `hive-bench` runner consumes — it reads the suite, looks up the targets in `applies_to`, and dispatches the prompts through the live Honeycomb network. Combs only advertise things they *do* (inference, agents, queen). Benchmarks measure those things.

## Schema (model)

```yaml
id: qwen2.5:0.5b                                          # required, ollama-style
name: Qwen 2.5 0.5B Instruct                              # required, human-readable
capability_urn: oasf://commons/inference/qwen2.5-0.5b/v1  # required, OASF
runtime: ollama                                           # required: ollama|vllm|llama_cpp|openai_compat
pull_tag: qwen2.5:0.5b                                    # required for ollama; optional for openai_compat
size_mb: 397                                              # required, on-disk
min_memory_mb: 2048                                       # required
min_cpu_cores: 2                                          # required
supports_tool_calling: false                              # required
license: apache-2.0                                       # required, SPDX or family slug
tier: slm                                                 # required: slm|llm
description: |                                            # optional but encouraged
  Small/fast Qwen variant. Good for classification, light extraction,
  short generation. Not reliable at multi-turn tool use.
```

## Schema (agent)

```yaml
id: sum_agent
name: Sum Agent
capability_urn: oasf://demo/agents/sum/v1
description: Add two integers a + b.
runs_on: slm                  # which model tier this agent expects
prompt_template: |
  You are a calculator. Compute {{a}} + {{b}}.
  Reply with ONLY the integer result, no words, no punctuation.
input_schema:
  type: object
  properties:
    a: { type: integer }
    b: { type: integer }
  required: [a, b]
max_tokens: 16
temperature: 0.0
compatible_models:
  - qwen2.5:0.5b
  - gemma:2b
  - llama3.2:3b
```

## Schema (benchmark)

```yaml
id: math-basics
name: Math Basics
description: |
  Single-step arithmetic. Tests whether a small model can return ONLY
  the numeric answer, and whether agent prompts hold up.

# Which targets this suite is meaningful against. The runner reads
# `applies_to` and expands it against the live network's advertised
# capabilities. Each target kind dispatches differently:
#   - kind: model    → call the model's inference URN directly
#   - kind: agent    → call the agent capability URN with `inputs`
#   - kind: queen    → call the queen URN with the freeform `prompt`
applies_to:
  - kind: model
    tier: slm
  - kind: agent
    id: sum_agent

cases:
  - id: sum-small
    prompt: "What is 7 + 5? Reply with only the number."
    inputs: { a: 7, b: 5 }
    expected: "12"
    grader: exact_int
    agent_target: sum_agent          # which agent to dispatch when target is an agent
```

### Graders

| `grader` | Pass when |
|---|---|
| `exact` | Stripped output equals `expected` (case-sensitive) |
| `exact_ci` | Stripped output equals `expected` (case-insensitive) |
| `exact_int` | First integer found in output equals `expected` (handles chatty SLMs) |
| `contains_any_ci` | Output contains any string in `expected_contains` (case-insensitive) |
| `contains_any_ci_with_forbidden` | `contains_any_ci` AND none of `forbidden_contains` appear |

A benchmark run produces, per (target × case): wall time, prompt+completion tokens, exit status, output text, grader pass/fail, and (for queens) the actual sub-tool sequence vs `expected_tool_calls`.

## How combs use it

```toml
# Comb TOML — short, declarative.
runtime.ollama_endpoint = "http://ollama:11434"
catalog.path = "/etc/hive-models"  # mounted from this repo

[[supported_models]]
id = "qwen2.5:0.5b"
auto_pull = true
serves_agents = ["sum"]   # also serves inference URN automatically
```

The comb resolves `qwen2.5:0.5b` → catalog → `oasf://commons/inference/qwen2.5-0.5b/v1` → `min_memory_mb: 2048`. It checks its own RAM, pulls from Ollama, and registers with Honeycomb advertising the capability URN.

## How honeycomb uses it

Same catalog, embedded or fetched at startup. On `/api/nodes/register`, validates that:
- `node.memory_mb >= catalog[urn].min_memory_mb`
- `node.cpu_cores >= catalog[urn].min_cpu_cores`

Misconfigured combs fail registration with a structured 400 instead of crashing later.

## How `hive-bench` uses it

The `hive-bench` CLI (in `hive-sdk/packages/hive-bench`) reads benchmark YAML, resolves `applies_to` against the live network's capability registry, and dispatches every (target × case) pair through Honeycomb. Output: a JSON report with timing, token counts, grader pass/fail, and aggregate scores per target.
