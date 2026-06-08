# Codey Modifications

Date: 2026-06-08
Executor: Codex

Codey is an unofficial fork derived from OpenAI Codex CLI and distributed under
the Apache License, Version 2.0. This file records the main fork points and
high-level changes so downstream users can distinguish Codey-specific work from
upstream OpenAI Codex sources.

## Fork Points

- Upstream project: `openai/codex`
- Source fork branch: `bilisheep/tool-output-relevance-pruning`
- Split repository target: `bilisheep/codey`
- Split commit: `4acdf6e5374bdbaa06738555199dfb1187a37097`
- Included fork base commit: `f0d09b87ed50c13d9e9123e70a65048c0e7dae23`

## Main Changes

- Added optional `[tool_output_relevance_pruning]` configuration.
- Added the internal `trim_prompt_context` tool for replacing consumed historical
  tool outputs with compact traceable placeholders.
- Added context-history rewriting support and tests for pruned tool outputs.
- Added rollout trace event payloads for `tool_output_relevance_pruning.snapshot`.
- Documented isolated relevance-pruning configuration and release behavior.
- Added a Codey release workflow for multi-platform experimental binaries.
- Updated repository-facing branding, package metadata, notices, and release
  notes to identify this repository as an unofficial fork.

## Upstream Attribution

The upstream OpenAI Codex license and attribution notices are preserved in
[LICENSE](./LICENSE) and [NOTICE](./NOTICE). Codey-specific attribution is added
as an addendum and does not modify the upstream Apache-2.0 license terms.
