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
---

# AutoReason

Iterative quality optimization through adversarial multi-agent debate.
Replaces numeric fitness functions with structured argument between isolated agents.

## Why This Exists

LLMs are sycophantic when asked to "improve," overly critical when asked to find flaws,
and overly compromising when asked to merge. AutoReason fixes this by separating every role
into isolated agents with NO shared context — the same way science uses peer review where
math can use proofs.

## Plugin Agents (all run on model: sonnet)

This plugin provides five dedicated subagents. Reference them by their plugin-scoped names:

| Agent | Plugin Name | Role |
|---|---|---|
| Author | `autoreason:author` | Writes drafts from task prompt only |
| Strawman | `autoreason:strawman` | Attacks drafts adversarially (problems only, no fixes) |
| Rewriter | `autoreason:rewriter` | Rewrites from scratch using original + critique |
| Synthesizer | `autoreason:synthesizer` | Merges best of two versions |
| Judge | `autoreason:judge` | Blind ranked evaluation of three candidates |

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
     - Shuffle A, B, AB into randomized "Option 1/2/3" per judge
     - Each judge ranks independently, no labels or author info
     - Borda count: 1st=2pts, 2nd=1pt, 3rd=0pts
  5. CONVERGENCE:
     - Winner becomes new A
     - If A (incumbent) wins → streak++
     - If B or AB wins → streak = 0
     - streak == 2 → STOP (converged)
     - Otherwise → back to step 2
```

## Execution Instructions

### Step 0: Capture the Anchor

Ask the user what they want created, or use what they've already described. Distill it into
a clear, self-contained task prompt.

**Pre-flight checklist** — before confirming the anchor, verify it includes or you have inferred:
- **Target audience**: Who will read this?
- **Desired tone**: Formal, casual, persuasive, technical, etc.
- **Approximate length/format**: Email, blog post, landing page, paragraph count, word count, etc.
- **Key points to cover**: What must be included?
- **Constraints**: Anything to avoid, brand voice requirements, etc.

If any are missing and would materially affect quality, ask the user before proceeding.

Confirm the anchor with the user. This is the ANCHOR — it never changes across rounds.

### Step 1: Author A

Spawn the `autoreason:author` subagent. Pass it ONLY the anchor prompt:

```
Complete this task to the highest possible standard:

TASK:
{anchor}
```

Save the result as Draft A.

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

### Step 3: Generate Three Candidates

**Candidate A** = the current Draft A, unchanged.

**Candidate B** — spawn `autoreason:rewriter`:
```
Rewrite this from scratch, addressing the critique.

TASK:
{anchor}

ORIGINAL DRAFT:
{draft_A}

CRITIQUE:
{critique}
```

**Candidate AB** — spawn `autoreason:synthesizer` (AFTER B completes, since AB needs B):
```
Synthesize the best of both versions.

TASK:
{anchor}

VERSION 1:
{draft_A}

VERSION 2:
{draft_B}
```

### Step 4: Blind Judge Panel

**CRITICAL: Invoke all three judge subagents in a SINGLE turn so they run in parallel.**
Do NOT spawn one, wait for its result, then spawn the next. Issue all three Agent tool
calls in the same response. Claude Code will execute them concurrently.

Before spawning, decide on three different label shuffles. Each judge must see candidates
in a different order to eliminate positional bias. Example:

- Judge 1 mapping: Option 1=B, Option 2=AB, Option 3=A
- Judge 2 mapping: Option 1=A, Option 2=B, Option 3=AB
- Judge 3 mapping: Option 1=AB, Option 2=A, Option 3=B

Spawn all three `autoreason:judge` subagents simultaneously, each with its own shuffle:

```
[Judge 1 prompt]
Rank these three versions of the same piece.

TASK:
{anchor}

OPTION 1:
{candidate per Judge 1's shuffle}

OPTION 2:
{candidate per Judge 1's shuffle}

OPTION 3:
{candidate per Judge 1's shuffle}
```

```
[Judge 2 prompt — different ordering]
...same format, different candidate-to-option mapping...
```

```
[Judge 3 prompt — different ordering again]
...same format, different candidate-to-option mapping...
```

Record which shuffle you used for each judge so you can de-shuffle the votes in Step 5.

### Step 5: Tally Votes

Parse each judge's ranking output. De-shuffle labels to map back to A, B, AB.
Borda scoring: 1st place = 2 points, 2nd = 1, 3rd = 0.
Candidate with highest total wins. Break ties by preferring the incumbent (A).

Report to the user after each round:
- Round number
- Borda scores for A, B, AB
- Winner and whether streak incremented

### Step 6: Convergence Check

- If A (the incumbent) wins → increment streak
- If B or AB wins → reset streak to 0; winner becomes new Draft A
- If streak reaches 2 → STOP. Output final version.
- If round count reaches 5 → STOP. Output best from final round + note non-convergence.
- Otherwise → loop back to Step 2.

### Step 7: Present Final Result

Show the user the converged winning version. Also provide a brief summary:
- Total rounds to convergence
- Borda scores from each round
- One-line description of what changed each round

## Handling Malformed Output

**Judge output**: After each judge returns, validate the output matches the expected format
(`RANK 1: Option [1-3]`, `RANK 2: Option [1-3]`, `RANK 3: Option [1-3]` with all three
options present and distinct). If a judge's output is unparseable or invalid:
- Discard that judge's vote and proceed with the remaining judges (2 judges is still
  sufficient for Borda scoring).
- If 2 or more judges return malformed output, re-run the entire judge panel once.
- Log a warning to the user when any judge output is discarded.

**Author/rewriter/synthesizer output**: If an agent's output begins with preamble
("Here's the draft:", "Sure!", "I'll write...", etc.), strip the preamble before passing
the content to downstream agents. Only the actual piece should flow through the pipeline.

## Critical Rules

1. **Every role is a separate subagent invocation.** Never combine roles in one context.
2. **Judges never see labels like "original", "rewrite", or "synthesis."** Always "Option 1/2/3."
3. **Shuffle order is DIFFERENT for each judge** to eliminate positional bias.
4. **The strawman never suggests fixes.** Problems only.
5. **The synthesizer sees A and B but NOT the critique.** It works from outputs only.
6. **Maximum 5 rounds.** Stop and present best if no convergence.
7. **All spawned agents use `model: sonnet`** via their agent definitions to save cost.
8. **When constructing prompts for subagents, copy the template EXACTLY as specified.** Do NOT add additional context, framing, commentary, or instructions beyond what each template specifies. The isolation of each agent depends on this.
