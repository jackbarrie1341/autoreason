---
name: config
description: >
  Display and configure AutoReason model settings. Use when the user says
  "autoreason config", "autoreason settings", "autoreason models", "configure autoreason",
  or wants to change which models the autoreason agents use.
argument-hint: "Optional: 'max' to show max mode setup, or 'budget' for budget mode"
allowed-tools: Bash
---

# AutoReason Configuration

Display the current model configuration and help the user set overrides.

## Instructions

### 1. Read Current Configuration

Run this Bash command to check all relevant environment variables:

```bash
echo "=== AutoReason Model Configuration ===" && echo "" && echo "AUTOREASON_MODE=${AUTOREASON_MODE:-unset}" && echo "AUTOREASON_MODEL_AUTHOR=${AUTOREASON_MODEL_AUTHOR:-unset}" && echo "AUTOREASON_MODEL_STRAWMAN=${AUTOREASON_MODEL_STRAWMAN:-unset}" && echo "AUTOREASON_MODEL_REWRITER=${AUTOREASON_MODEL_REWRITER:-unset}" && echo "AUTOREASON_MODEL_SYNTHESIZER=${AUTOREASON_MODEL_SYNTHESIZER:-unset}" && echo "AUTOREASON_MODEL_JUDGE=${AUTOREASON_MODEL_JUDGE:-unset}" && echo "CLAUDE_CODE_SUBAGENT_MODEL=${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"
```

### 2. Resolve and Display Effective Configuration

Apply the override hierarchy (highest priority first):

1. `AUTOREASON_MODE=max` → all agents use `opus`
2. Per-agent env vars (`AUTOREASON_MODEL_*`) → override specific agents
3. `CLAUDE_CODE_SUBAGENT_MODEL` → override all agents globally
4. Agent frontmatter defaults (see table below)

**Default configuration** (no env vars set):

| Agent | Default Model | Effort | Rationale |
|---|---|---|---|
| Author | sonnet | high | Iterative loop compensates; quality ceiling set by debate |
| Strawman | sonnet | high | Structured dimensions guide critique effectively |
| Rewriter | sonnet | high | Parallel capability to author |
| Synthesizer | sonnet | high | Well-defined merge task |
| Judge | opus | max | Most critical role — bad judging breaks the loop |

Display the resolved model for each agent based on current env vars.

### 3. Show Configuration Options

Based on the user's argument (if any) or current state, show the relevant setup:

**If user said "max" or wants maximum quality:**
```
# Max mode — all agents use Opus (highest quality, ~3x cost)
export AUTOREASON_MODE=max

# To make persistent, add to your shell profile:
echo 'export AUTOREASON_MODE=max' >> ~/.zshrc
```

**If user said "budget" or wants lowest cost:**
```
# Budget mode — all agents use Haiku
export CLAUDE_CODE_SUBAGENT_MODEL=haiku

# To make persistent:
echo 'export CLAUDE_CODE_SUBAGENT_MODEL=haiku' >> ~/.zshrc
```

**If user wants custom per-agent configuration:**
```
# Override individual agents (valid values: opus, sonnet, haiku)
export AUTOREASON_MODEL_AUTHOR=opus
export AUTOREASON_MODEL_STRAWMAN=sonnet
export AUTOREASON_MODEL_REWRITER=sonnet
export AUTOREASON_MODEL_SYNTHESIZER=sonnet
export AUTOREASON_MODEL_JUDGE=opus

# To make persistent, add the exports to ~/.zshrc or ~/.bashrc
```

**To reset to defaults:**
```
unset AUTOREASON_MODE AUTOREASON_MODEL_AUTHOR AUTOREASON_MODEL_STRAWMAN AUTOREASON_MODEL_REWRITER AUTOREASON_MODEL_SYNTHESIZER AUTOREASON_MODEL_JUDGE CLAUDE_CODE_SUBAGENT_MODEL
```

### 4. Cost Estimates

Show approximate cost per round based on resolved configuration:

- **Default** (4 sonnet + 3 opus): ~7 Sonnet calls + 3 Opus calls per round
- **Max mode** (all opus): ~8 Opus calls per round
- **Budget mode** (all haiku): ~8 Haiku calls per round
- **All sonnet**: ~8 Sonnet calls per round

Typical convergence: 2-3 rounds.
