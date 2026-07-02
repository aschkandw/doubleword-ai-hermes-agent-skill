# Doubleword AI Hermes Agent Skill

This repository contains the `doubleword` skill for Hermes/OpenClaw agents.

The skill teaches agents how to use the Doubleword CLI and OpenAI-compatible API
for cost-aware inference across realtime, async, and 24-hour batch tiers.

## Structure

```text
skills/
  doubleword/
    SKILL.md
    references/
      cli-recipes.md
      models-and-pricing.md
```

The main skill file is intentionally concise for progressive disclosure:

- Level 0: skill list metadata from `SKILL.md`
- Level 1: full operating procedure in `skills/doubleword/SKILL.md`
- Level 2: detailed command and model references under
  `skills/doubleword/references/`

References are included because model/pricing tables and command recipes are
large, detail-heavy, and more likely to change than the core skill procedure.
