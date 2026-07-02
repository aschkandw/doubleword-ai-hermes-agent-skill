# Doubleword AI Hermes Agent Skill

This repository contains the `doubleword` skill for Hermes/OpenClaw agents.

The skill teaches agents how to use the Doubleword CLI and OpenAI-compatible API
for cost-aware inference across realtime, async, and 24-hour batch tiers.

See [`doubleword.skill.md`](./doubleword.skill.md) for the full skill definition,
including:

- required environment variables;
- Hermes-safe non-blocking batch scheduling;
- JSONL validation and upload guardrails;
- model and tier selection guidance;
- result retrieval and interrupted-download recovery.
