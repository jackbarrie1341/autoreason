# AutoReason — Claude Code Plugin

Iterative quality optimization for subjective work through adversarial multi-agent debate.

Instead of asking one LLM to "make this better," AutoReason pits isolated agents against each
other in a convergence loop until nothing can beat the current version.

Based on [@shannholmberg's AutoReason method](https://x.com/shannholmberg).

## Install

**Local testing:**
```bash
claude --plugin-dir ./autoreason
```

**From a marketplace** (once published):
```
/plugin install autoreason@<marketplace-name>
```

## Usage

Invoke directly:
```
/autoreason:run Write a cold outreach email for a SaaS product targeting CFOs
```

Or just describe your task and mention quality — the skill auto-triggers when it detects
you want something optimized to the highest standard with no objective metric.

## What Happens

1. **Author** writes a draft (sees only the task — fresh context, no bias)
2. **Strawman** attacks it hard (problems only, no fixes — adversarial by design)
3. Three candidates generated: **A** (original), **B** (rewrite), **AB** (synthesis)
4. **3 blind judges** rank them with randomized labels (Borda count, no author bias)
5. Winner becomes new draft; loop until it wins twice in a row (convergence)

Every agent runs on **Sonnet** to keep costs low. The orchestrator (your main session) can
run on any model.

## Cost

~8 Sonnet calls per round. Typical convergence in 2–3 rounds = ~16–24 Sonnet calls.

To force Haiku for even cheaper runs:
```bash
export CLAUDE_CODE_SUBAGENT_MODEL=haiku
```

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
│   └── judge.md             # Blind ranked evaluation (model: sonnet)
├── skills/
│   └── run/
│       └── SKILL.md         # Orchestration logic
└── README.md
```

## Skill vs Plugin — Why This Is a Plugin

A **skill** is a single SKILL.md file that teaches Claude how to do something. It can't
bundle agents.

A **plugin** is a self-contained package with a manifest that can include skills, agents,
hooks, MCP servers, and settings. AutoReason needs 5 dedicated agents alongside 1 orchestration
skill — that's a plugin. Plugins also get namespacing, marketplace distribution, and clean
install/update via `/plugin install`.
