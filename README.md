# AutoReason — Claude Code Plugin

Iterative quality optimization for subjective work through adversarial multi-agent debate.

Instead of asking one LLM to "make this better," AutoReason pits isolated agents against
each other in a convergence loop until nothing can beat the current version.

Based on [@shannholmberg's AutoReason method](https://x.com/shannholmberg).

## Install

**Local testing:**
```bash
claude --plugin-dir ./autoreason
```

**From the marketplace:**
```bash
claude plugin marketplace add jackbarrie1341/claude-plugins
claude plugin install autoreason@jackbarrie
```

## Usage

Invoke directly:
```
/autoreason:run Write a cold outreach email for a SaaS product targeting CFOs
```

Or just describe your task and mention quality — the skill auto-triggers when it detects
you want something optimized to the highest standard with no objective metric.

Check or change model configuration:
```
/autoreason:config
/autoreason:config max
/autoreason:config budget
```

## What Happens

1. **Author** writes a draft (sees only the task — fresh context, no bias)
2. **Strawman** attacks it across 11 dimensions with severity ratings (CRITICAL/MAJOR/MINOR — problems only, no fixes)
3. Three candidates generated: **A** (original), **B** (rewrite), **AB** (synthesis — generated once, shared across judges)
4. **3 blind judges** do per-criterion analysis then rank with Latin square shuffled labels (Borda count)
5. Winner determined with multi-signal convergence:
   - **Incumbent advantage** (round 3+): challenger needs ≥2 Borda margin
   - **Streak**: incumbent wins/holds twice → converged
   - **Stability plateau**: margin ≤1 for 2 consecutive rounds → converged
   - **Critique floor**: ≤4 problems or 0 critical → converged

Every agent runs in isolation with fresh context. No shared state between roles.

## Model Configuration

**Defaults:**

| Agent | Model | Why |
|---|---|---|
| Author | sonnet | Quality ceiling set by the iterative loop, not the first draft |
| Strawman | sonnet | Severity-rated critique across 11 dimensions |
| Rewriter | sonnet | Prioritizes CRITICAL/MAJOR fixes; avoids overcorrection |
| Synthesizer | sonnet | Merges best elements with length discipline |
| Judge | **opus** | Per-criterion analysis before ranking — most critical role |

**Override with environment variables:**

```bash
# Max mode — all agents use Opus (highest quality, ~3x cost)
export AUTOREASON_MODE=max

# Budget mode — all agents use Haiku
export CLAUDE_CODE_SUBAGENT_MODEL=haiku

# Custom per-agent overrides
export AUTOREASON_MODEL_JUDGE=sonnet    # downgrade judge to save cost
export AUTOREASON_MODEL_AUTHOR=opus     # upgrade author for better first drafts
```

**Override priority:** `AUTOREASON_MODE=max` > per-agent env vars > `CLAUDE_CODE_SUBAGENT_MODEL` > agent frontmatter defaults.

Run `/autoreason:config` to see your current resolved configuration.

## Cost

**Default configuration:** ~4 Sonnet + 3 Opus calls per round.
**Max mode (all Opus):** ~8 Opus calls per round.
**Budget mode (all Haiku):** ~8 Haiku calls per round.

Typical convergence in 2-3 rounds.

## Plugin Structure

```
autoreason/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── agents/
│   ├── author.md            # Writes drafts (model: sonnet)
│   ├── strawman.md          # Adversarial critic (model: sonnet)
│   ├── rewriter.md          # Rewrites addressing critique (model: sonnet)
│   ├── synthesizer.md       # Merges best of two versions (model: sonnet)
│   └── judge.md             # Blind ranked evaluation (model: opus)
├── skills/
│   ├── run/
│   │   └── SKILL.md         # Orchestration logic
│   └── config/
│       └── SKILL.md         # Model configuration display
└── README.md
```

## Skill vs Plugin — Why This Is a Plugin

A **skill** is a single SKILL.md file that teaches Claude how to do something. It can't
bundle agents.

A **plugin** is a self-contained package with a manifest that can include skills, agents,
hooks, MCP servers, and settings. AutoReason needs 5 dedicated agents alongside 2 skills —
that's a plugin. Plugins also get namespacing, marketplace distribution, and clean
install/update via `/plugin install`.
