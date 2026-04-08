---
name: config
description: >
  Display and configure AutoReason settings (models and max rounds). Use when the user says
  "autoreason config", "autoreason settings", "autoreason models", "configure autoreason",
  or wants to change which models the autoreason agents use or how many rounds to allow.
argument-hint: "Optional: 'max' for max quality, 'budget' for low cost, or 'reset' to clear saved config"
allowed-tools:
  - Bash
  - AskUserQuestion
  - Read
  - Write
---

# AutoReason Configuration

Display the current configuration, help the user set overrides, and persist settings
to `~/.autoreason.json` so they survive across sessions.

## Config File

Settings are stored in `~/.autoreason.json`. This file is created automatically when the
user first configures autoreason (or on first run of the plugin). Environment variables
still override file settings for one-off changes.

**File format:**
```json
{
  "models": {
    "mode": null,
    "author": null,
    "strawman": null,
    "rewriter": null,
    "synthesizer": null,
    "judge": null
  },
  "max_rounds": 5
}
```

A `null` value means "use default." Only non-null values override defaults.

## Instructions

### 1. Read Current Configuration

First, check for a saved config file and environment variables:

```bash
echo "=== Saved Config ===" && cat ~/.autoreason.json 2>/dev/null || echo "(no config file)" && echo "" && echo "=== Environment Overrides ===" && echo "AUTOREASON_MODE=${AUTOREASON_MODE:-unset}" && echo "AUTOREASON_MAX_ROUNDS=${AUTOREASON_MAX_ROUNDS:-unset}" && echo "AUTOREASON_MODEL_AUTHOR=${AUTOREASON_MODEL_AUTHOR:-unset}" && echo "AUTOREASON_MODEL_STRAWMAN=${AUTOREASON_MODEL_STRAWMAN:-unset}" && echo "AUTOREASON_MODEL_REWRITER=${AUTOREASON_MODEL_REWRITER:-unset}" && echo "AUTOREASON_MODEL_SYNTHESIZER=${AUTOREASON_MODEL_SYNTHESIZER:-unset}" && echo "AUTOREASON_MODEL_JUDGE=${AUTOREASON_MODEL_JUDGE:-unset}" && echo "CLAUDE_CODE_SUBAGENT_MODEL=${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"
```

### 2. Resolve and Display Effective Configuration

Apply the override hierarchy (highest priority first):

**For models:**
1. `AUTOREASON_MODE=max` env var → all agents use `opus`
2. Per-agent env vars (`AUTOREASON_MODEL_*`) → override specific agents
3. `CLAUDE_CODE_SUBAGENT_MODEL` → override all agents globally
4. Config file `models.mode` or per-agent values from `~/.autoreason.json`
5. Agent frontmatter defaults (see table below)

**For max rounds:**
1. `AUTOREASON_MAX_ROUNDS` env var
2. Config file `max_rounds` from `~/.autoreason.json`
3. Default: 5

**Default configuration** (no config file, no env vars):

| Agent | Default Model | Effort | Rationale |
|---|---|---|---|
| Author | sonnet | high | Iterative loop compensates; quality ceiling set by debate |
| Strawman | sonnet | high | Structured dimensions guide critique effectively |
| Rewriter | sonnet | high | Parallel capability to author |
| Synthesizer | sonnet | high | Well-defined merge task |
| Judge | opus | max | Most critical role — bad judging breaks the loop |

| Setting | Default |
|---|---|
| Max rounds | 5 |

Display the resolved model for each agent and the effective max rounds.

### 3. Interactive Configuration

If the user wants to change settings, ask them two questions using AskUserQuestion:

**Question 1 — Models:**
> Which model configuration do you want?
> - **default** — sonnet for generators, opus for judges (recommended)
> - **max** — opus for all agents (highest quality, ~3x cost)
> - **budget** — haiku for all agents (lowest cost)
> - **custom** — choose per agent

If they pick "custom," ask which model (opus/sonnet/haiku) for each agent.

**Question 2 — Max rounds:**
> How many rounds maximum before stopping? (default: 5, typical convergence: 2-3 rounds)

### 4. Save Configuration

After the user answers, write the config to `~/.autoreason.json` using the Write tool.

Example for default models with 4 max rounds:
```json
{
  "models": {
    "mode": null,
    "author": null,
    "strawman": null,
    "rewriter": null,
    "synthesizer": null,
    "judge": null
  },
  "max_rounds": 4
}
```

Example for max mode:
```json
{
  "models": {
    "mode": "max",
    "author": null,
    "strawman": null,
    "rewriter": null,
    "synthesizer": null,
    "judge": null
  },
  "max_rounds": 5
}
```

Confirm to the user: `"Settings saved to ~/.autoreason.json."`

### 5. Presets

**If user said "max":** Set `models.mode` to `"max"`, keep current max_rounds, save.

**If user said "budget":** Set `models.mode` to `"budget"`, keep current max_rounds, save.
Note: budget mode maps to setting `CLAUDE_CODE_SUBAGENT_MODEL=haiku` at runtime since
there's no built-in "budget" mode. When the run skill reads `mode: "budget"`, it should
resolve all agents to haiku.

**If user said "reset":** Delete `~/.autoreason.json` and confirm.

### 6. Cost Estimates

Show approximate cost per round based on resolved configuration:

- **Default** (4 sonnet + 3 opus): ~7 Sonnet calls + 3 Opus calls per round
- **Max mode** (all opus): ~8 Opus calls per round
- **Budget mode** (all haiku): ~8 Haiku calls per round
- **All sonnet**: ~8 Sonnet calls per round

Typical convergence: 2-3 rounds.
