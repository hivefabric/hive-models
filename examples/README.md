# examples/

Prompt templates and usage patterns — **not** first-class catalog entries.

## prompt-templates/

These YAML files show how to craft effective prompts for common developer tasks using
the model capabilities in `../models/`. They are reference material, not catalog entries.

The HiveFabric catalog is model capabilities only. The *what* (code review, test generation,
summarisation) lives in the caller's prompt — the platform routes to a model that can handle
it. There are no pre-baked task agents in production.

**Use these files to:**
- Copy prompt templates into your own orchestrator
- Understand the expected input/output shape for a given task pattern
- Seed a queen's tool catalogue

**Do NOT use these files as capability URNs.** The URNs in these files (`oasf://commons/dev/...`)
are illustrative and not registered in the active catalog.

## Model recommendations per task

| Task | Recommended model | Notes |
|---|---|---|
| Code review | `qwen2.5-coder:7b` or `qwen2.5:7b` | 7B+ for reliable structured output |
| Test generation | `qwen2.5-coder:7b` or `llama3.1:8b` | Code-trained models preferred |
| Documentation | `llama3.2:3b` or `qwen2.5:3b` | 3B sufficient for docstrings |
| Summarisation | `qwen2.5:1.5b` or `smollm2:1.7b` | 1.5B+ for coherent summaries |
| Classification | `qwen2.5:0.5b` or `qwen2.5:1.5b` | Tiny models excel at exact-label output |
