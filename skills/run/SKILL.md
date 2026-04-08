---
name: run
description: >
  Iterative quality optimization for subjective work through adversarial multi-agent debate.
  Use this skill whenever the user wants to produce the highest-quality version of something
  that has no objective metric — writing, arguments, marketing copy, essays, emails, pitches,
  blog posts, landing page copy, product descriptions, cover letters, speeches, or any creative
  or persuasive text. Also trigger when the user says "autoreason", "make this as good as
  possible", "optimize this", "iterate on this until it's great", "red-team this", or
  "debate-refine this". If the user wants something polished to a very high standard and
  there's no numeric score to optimize, this is the skill to use.
argument-hint: "Describe what you want written (e.g., 'a cold outreach email targeting CFOs')"
allowed-tools:
  - Bash
  - AskUserQuestion
  - Read
  - Write
---

# AutoReason

Iterative quality optimization through adversarial multi-agent debate.
Replaces numeric fitness functions with structured argument between isolated agents.

## Why This Exists

LLMs are sycophantic when asked to "improve," overly critical when asked to find flaws,
and overly compromising when asked to merge. AutoReason fixes this by separating every role
into isolated agents with NO shared context — the same way science uses peer review where
math can use proofs.

## Plugin Agents

This plugin provides five dedicated subagents:

| Agent | Plugin Name | Default Model | Role |
|---|---|---|---|
| Author | `autoreason:author` | sonnet | Writes drafts from task prompt only |
| Strawman | `autoreason:strawman` | sonnet | Attacks drafts adversarially (problems only, no fixes) |
| Rewriter | `autoreason:rewriter` | sonnet | Rewrites from scratch using original + critique |
| Synthesizer | `autoreason:synthesizer` | sonnet | Merges best of two versions |
| Judge | `autoreason:judge` | opus | Blind ranked evaluation of three candidates |

## Configuration

Settings are stored in `~/.autoreason.json` and persist across sessions. Environment
variables override file settings for one-off changes. On first run, the skill prompts
the user to choose model configuration and max rounds, then saves to the config file.

**Settings:**
- **Models** — which model each agent uses (default: sonnet for generators, opus for judges)
- **Max rounds** — maximum debate rounds before stopping (default: 5)

**Override hierarchy (highest priority first):**

For models:
1. `AUTOREASON_MODE=max` env var → all agents use `opus`
2. Per-agent env vars: `AUTOREASON_MODEL_AUTHOR`, `AUTOREASON_MODEL_STRAWMAN`,
   `AUTOREASON_MODEL_REWRITER`, `AUTOREASON_MODEL_SYNTHESIZER`, `AUTOREASON_MODEL_JUDGE`
3. `CLAUDE_CODE_SUBAGENT_MODEL` — Claude Code's built-in global override
4. Config file `~/.autoreason.json`
5. Agent frontmatter default (sonnet for most, opus for judge)

For max rounds:
1. `AUTOREASON_MAX_ROUNDS` env var
2. Config file `~/.autoreason.json`
3. Default: 5

When spawning each agent, if an override applies, pass the `model` parameter to the
Agent tool call. Resolve configuration ONCE at Step 0 and report to the user.

## The Algorithm

```
ANCHOR: Original task prompt (immutable, seen by all agents)

LOOP:
  1. AUTHOR        → sees task only → produces Draft A
  2. STRAWMAN      → sees task + A → finds all problems (no fixes)
  3. GENERATE THREE CANDIDATES:
     A  = original draft (unchanged)
     B  = REWRITER sees task + A + critique → full rewrite
     AB = SYNTHESIZER sees task + A + B → merges best of both
  4. BLIND JUDGE PANEL (3 judges in parallel):
     - Latin square shuffle: A, B, AB into "Option 1/2/3" per judge
     - Each judge ranks independently, no labels or author info
     - Borda count: 1st=2pts, 2nd=1pt, 3rd=0pts
  5. CONVERGENCE:
     - Winner becomes new A
     - If A (incumbent) wins → streak++
     - If B or AB wins → streak = 0
     - streak == 2 → STOP (converged)
     - round == max_rounds → STOP (max reached)
     - Otherwise → back to step 2
```

## Execution Instructions

### Step 0: Setup — Anchor and Configuration

**0a. Resolve configuration (models and max rounds).**

Run a single Bash command to check the config file and environment variables:

```bash
echo "=== Config File ===" && cat ~/.autoreason.json 2>/dev/null || echo "(none)" && echo "" && echo "=== Env Overrides ===" && echo "AUTOREASON_MODE=${AUTOREASON_MODE:-unset}" && echo "AUTOREASON_MAX_ROUNDS=${AUTOREASON_MAX_ROUNDS:-unset}" && echo "AUTOREASON_MODEL_AUTHOR=${AUTOREASON_MODEL_AUTHOR:-unset}" && echo "AUTOREASON_MODEL_STRAWMAN=${AUTOREASON_MODEL_STRAWMAN:-unset}" && echo "AUTOREASON_MODEL_REWRITER=${AUTOREASON_MODEL_REWRITER:-unset}" && echo "AUTOREASON_MODEL_SYNTHESIZER=${AUTOREASON_MODEL_SYNTHESIZER:-unset}" && echo "AUTOREASON_MODEL_JUDGE=${AUTOREASON_MODEL_JUDGE:-unset}" && echo "CLAUDE_CODE_SUBAGENT_MODEL=${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"
```

**If no config file exists (`~/.autoreason.json`)**, this is the first run. Ask the user
two questions using AskUserQuestion before proceeding:

1. **Models:** "Which model configuration? **default** (sonnet + opus judges, recommended),
   **max** (all opus), or **budget** (all haiku)?"
2. **Max rounds:** "Max rounds before stopping? (default: 5, typical convergence: 2-3)"

Save their answers to `~/.autoreason.json` using the Write tool:
```json
{
  "models": {
    "mode": "{mode or null}",
    "author": null,
    "strawman": null,
    "rewriter": null,
    "synthesizer": null,
    "judge": null
  },
  "max_rounds": {N}
}
```

**Resolve effective configuration using this hierarchy (highest priority first):**

For models:
1. `AUTOREASON_MODE` env var → all agents use that model
2. Per-agent env vars (`AUTOREASON_MODEL_*`)
3. `CLAUDE_CODE_SUBAGENT_MODEL` env var
4. Config file values from `~/.autoreason.json` (`models.mode` or per-agent overrides)
5. Agent frontmatter defaults (sonnet for most, opus for judge)

For max rounds:
1. `AUTOREASON_MAX_ROUNDS` env var
2. Config file `max_rounds` from `~/.autoreason.json`
3. Default: 5

Note: config file `models.mode` values map as follows:
- `"max"` → all agents use opus
- `"budget"` → all agents use haiku
- `null` → use per-agent values or frontmatter defaults

Store the resolved config as state for use throughout the run.

Report to the user:

```
Models: author={model}, strawman={model}, rewriter={model}, synthesizer={model}, judge={model}
Max rounds: {max_rounds}
```

**0b. Capture and validate the anchor.**

Extract or ask for the task. Then validate it against these requirements:

<anchor-validation>
REQUIRED (ask if missing):
- What type of piece? (email, essay, landing page, etc.)
- Who is the audience?
- What is the goal? (persuade, inform, sell, etc.)

RECOMMENDED (infer if not stated, confirm with user):
- Tone/register (formal, casual, technical, etc.)
- Approximate length or format constraints
- Key points or arguments that must be included
- Things to avoid (competitors, certain claims, etc.)

OPTIONAL (use defaults if not stated):
- Brand voice reference
- Specific call-to-action
- Voice sample — if the user provides an example of their own writing or a style to
  match, capture it verbatim in the anchor as voice_sample
</anchor-validation>

Once validated, present the complete anchor to the user for confirmation:

```
ANCHOR:
Type: {type}
Audience: {audience}
Goal: {goal}
Tone: {tone}
Length: {length}
Key points: {points}
Constraints: {constraints}
Voice sample: {voice_sample or "none provided"}
Brief: {the full task description}
```

After user confirms, this is IMMUTABLE for the rest of the run.

**0c. Initialize state.**

```
round = 0
streak = 0
current_draft = null
history = []
```

### Step 1: Author A

Spawn the `autoreason:author` subagent. Pass it ONLY the anchor prompt:

```
Complete this task to the highest possible standard:

TASK:
{anchor}
```

If the anchor includes a voice_sample, append this block after the TASK block:

```
VOICE REFERENCE (match this writing's rhythm, vocabulary level, and register — don't copy its content):
{voice_sample}
```

If a model override applies for the author, pass the `model` parameter to the Agent tool.

Save the result as Draft A. Strip any preamble if present (see Handling Malformed Output).

Report: `"Draft A complete ({word_count} words). Sending to critic..."`

### Step 2: Strawman Critique

Spawn the `autoreason:strawman` subagent. Pass it the task AND Draft A:

```
Attack this draft. Problems only, no fixes.

TASK:
{anchor}

DRAFT:
{draft_A}
```

Save the critique.

Report: `"Critique received ({N} problems identified). Generating candidates..."`

### Step 3: Generate Three Candidates

**CRITICAL: Generate candidate AB exactly ONCE.** The synthesizer runs a single time,
producing one Candidate AB. This same AB text is sent to all three judges. Do NOT
regenerate AB for each judge — that would create three different synthesis candidates
and invalidate the judging.

**Execution order:**

1. **Candidate A** = current Draft A (no agent call needed)

2. **Candidate B** — spawn `autoreason:rewriter`:
```
Rewrite this from scratch, addressing the critique.

TASK:
{anchor}

ORIGINAL DRAFT:
{draft_A}

CRITIQUE:
{critique}
```

If the anchor includes a voice_sample, append the VOICE REFERENCE block (same as Step 1) after the CRITIQUE block.

3. **Wait for B to complete.**

4. **Candidate AB** — spawn `autoreason:synthesizer` (needs both A and B):
```
Synthesize the best of both versions.

TASK:
{anchor}

VERSION 1:
{draft_A}

VERSION 2:
{draft_B}
```

If the anchor includes a voice_sample, append the VOICE REFERENCE block (same as Step 1) after the VERSION 2 block.

5. **Wait for AB to complete.**

Strip any preamble from B and AB if present.

Report: `"Three candidates ready (A: {words}, B: {words}, AB: {words}). Sending to judges..."`

### Step 4: Blind Judge Panel

**CRITICAL: Invoke all three judge subagents in a SINGLE turn so they run in parallel.**
Do NOT spawn one, wait for its result, then spawn the next. Issue all three Agent tool
calls in the same response.

**Latin square shuffle** — use these EXACT three mappings. Each candidate appears exactly
once in each position across the three judges, providing maximum position-bias cancellation:

| Judge | Option 1 | Option 2 | Option 3 |
|---|---|---|---|
| Judge 1 | B | AB | A |
| Judge 2 | A | B | AB |
| Judge 3 | AB | A | B |

Spawn all three `autoreason:judge` subagents simultaneously, each with its own mapping:

```
[Judge 1 prompt]
Rank these three versions of the same piece.

TASK:
{anchor}

OPTION 1:
{candidate_B}

OPTION 2:
{candidate_AB}

OPTION 3:
{candidate_A}
```

```
[Judge 2 prompt]
Rank these three versions of the same piece.

TASK:
{anchor}

OPTION 1:
{candidate_A}

OPTION 2:
{candidate_B}

OPTION 3:
{candidate_AB}
```

```
[Judge 3 prompt]
Rank these three versions of the same piece.

TASK:
{anchor}

OPTION 1:
{candidate_AB}

OPTION 2:
{candidate_A}

OPTION 3:
{candidate_B}
```

**De-shuffle mapping** (use this to convert judge output back to candidates):

```
Judge 1: Option 1=B, Option 2=AB, Option 3=A
Judge 2: Option 1=A, Option 2=B,  Option 3=AB
Judge 3: Option 1=AB, Option 2=A, Option 3=B
```

Record this mapping BEFORE spawning judges. After judges return, apply it to convert
"Option N" rankings back to A/B/AB.

Report: `"Judges returned. {N}/3 valid rankings."`

### Step 5: Tally Votes (Borda Count)

Parse each judge's ranking output (see Judge Output Parsing below). De-shuffle labels
to map back to A, B, AB using the mapping from Step 4.

**Borda scoring:**
- 1st place: 2 points
- 2nd place: 1 point
- 3rd place: 0 points
- Maximum possible score per candidate: 6 (all three judges rank it first)
- Minimum possible score: 0 (all three judges rank it last)

**Tie-breaking:**
1. Prefer the incumbent (A) — biases toward convergence, which is desirable
2. If tie is between B and AB (neither is incumbent): prefer AB (synthesis incorporates both)

**Confidence indicator** based on judge agreement:
- UNANIMOUS: All 3 judges ranked the same candidate first (strong signal)
- MAJORITY: 2 of 3 judges agree on the top candidate (normal signal)
- SPLIT: All 3 judges ranked different candidates first (weak signal — result driven by 2nd-place votes)

Report to the user:

```
ROUND {N} RESULTS:
  A (incumbent): {score}/6
  B (rewrite):   {score}/6
  AB (synthesis): {score}/6
  Winner: {winner} {" (tie-break: incumbent advantage)" if applicable}
  Confidence: {UNANIMOUS|MAJORITY|SPLIT}
  Streak: {streak}/2
```

### Step 6: Convergence Check

- If A (the incumbent) wins → increment streak
- If B or AB wins → reset streak to 0; winner becomes new Draft A
- If streak reaches 2 → STOP. Output final version.
- If round count reaches max_rounds → STOP. Output best from final round + note non-convergence.
- If confidence is SPLIT on the final convergence round → note that the user may want to run an additional round.
- Otherwise → loop back to Step 2.

Report: `"Streak: {streak}/2. {Converged!|Continuing to round {N+1}...}"`

### Step 7: Present Final Result

Show the user the converged winning version. Also provide a brief summary:

```
AUTOREASON COMPLETE
Rounds: {total}
Convergence: {Yes (streak=2) | No (max rounds reached)}

Round history:
  Round 1: Winner={winner}, Confidence={level}, Scores: A={s}, B={s}, AB={s}
  Round 2: Winner={winner}, Confidence={level}, Scores: A={s}, B={s}, AB={s}
  ...

Final version below:
---
{final_text}
```

## Judge Output Parsing

After each judge returns, validate the output:

<parsing-rules>
1. EXPECTED FORMAT:
   RANK 1: Option [1-3] — [reason]
   RANK 2: Option [1-3] — [reason]
   RANK 3: Option [1-3] — [reason]

2. VALIDATION:
   - All three ranks present (RANK 1, RANK 2, RANK 3)
   - All three option numbers present and distinct
   - Option numbers are in {1, 2, 3}
   - No rank is repeated

3. RECOVERY (in order of preference):
   a. If format is close but has minor deviations (extra whitespace, "option" lowercase,
      em-dash vs en-dash), normalize and accept.
   b. If the judge provided a clear ranking but in a different format (e.g., prose with
      unambiguous ordering), extract the ranking if unambiguous.
   c. If the ranking is ambiguous or contradictory, discard this judge's vote.

4. MINIMUM QUORUM: At least 2 valid judge rankings required.
   - If only 1 valid ranking: re-run the entire judge panel once.
   - If re-run also fails: use the single valid ranking (Borda still works with 1 judge,
     though less robust).

5. LOG: Report to user which judges were valid and which were discarded, with reason.
</parsing-rules>

## Handling Malformed Output

**Author/rewriter/synthesizer output**: If an agent's output begins with preamble
("Here's the draft:", "Sure!", "I'll write...", "Here is the...", etc.), strip the
preamble before passing the content to downstream agents. Only the actual piece should
flow through the pipeline.

## Critical Rules

1. **Every role is a separate subagent invocation.** Never combine roles in one context.
2. **Judges never see labels like "original", "rewrite", or "synthesis."** Always "Option 1/2/3."
3. **Latin square shuffle** is FIXED — use the exact mapping specified in Step 4. Each candidate appears in each position exactly once across the three judges.
4. **The strawman never suggests fixes.** Problems only.
5. **The synthesizer sees A and B but NOT the critique.** It works from outputs only.
6. **Generate candidate AB exactly ONCE per round.** The same AB text goes to all three judges.
7. **Maximum rounds = max_rounds (from config).** Stop and present best if no convergence.
8. **Agents use their frontmatter model by default.** Override via the Agent tool's `model` parameter when environment variables specify different models (see Model Configuration).
9. **When constructing prompts for subagents, copy the template EXACTLY as specified.** Do NOT add additional context, framing, commentary, or instructions beyond what each template specifies. The isolation of each agent depends on this.
