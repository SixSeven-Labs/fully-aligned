---
name: chain-of-command
description: Multi-session architecture for complex tasks — a President (Opus 4.6) that understands user intent without burning context on tools, a VP (Opus 5.5) that orchestrates execution, and teammates (Sonnet 5.5 workers + Opus 5.5 specialists) that do the work. Load this before setting up the chain. Pairs with the `team-lead` skill, which the VP loads to manage its teammates.
---

# chain-of-command — President / VP / Team architecture for Claude Code

A three-tier agent hierarchy that separates **understanding intent** from **orchestrating
execution** from **doing the work**. Built from months of real multi-session orchestration
and the failure modes that came with it.

---

## Why this exists

Large, multi-step tasks need two things that fight each other: deep understanding of what
the user actually wants, and heavy tool use across a big codebase. Newer, more capable
models (Opus 5.5+) are better at tool-calling, code navigation, and managing complexity —
but they also editorialize, inject their own priorities, and self-authorize actions the user
didn't ask for. Opus 4.6, the last model on the older tokenizer, is worse at tools but
better at one critical thing: **it listens.** It doesn't inject its own agenda. It spends
its effort modeling what you mean, not what it thinks you should mean.

The architecture exploits this by splitting the roles:

| Role | Model | Context | Job |
|------|-------|---------|-----|
| **President** | Opus 4.6 | 200k (conserve it) | Understand user intent. Translate it into clear directives. Relay to VP. |
| **VP** | Opus 5.5 | 200k | Receive directives. Plan execution. Spin up and manage the team. |
| **Workers** | Sonnet 5.5 | 200k | General implementation tasks — the bulk of the work. |
| **Specialists** | Opus 5.5 | 200k | Deep analysis, reverse engineering, complex debugging. |

The President never touches files or calls tools beyond messaging. The VP never talks to the
user directly. Workers and specialists never talk to the user or the President — only the VP.

```
User ←→ President (4.6) ←→ VP (5.5) ←→ Teammates (Sonnet/Opus 5.5)
         intent filter      orchestrator     workers
```

---

## Setup

### Step 1 — Open the President terminal

1. Open a Claude Code terminal (or a new tab/window in your existing terminal).
2. Rename the session to **`president`**:
   - Type `/name president` and press Enter.
3. Set the model to **Opus 4.6**:
   - Type `/model` and press Enter.
   - Select **Opus 4.6** from the list.
4. Confirm: the status bar should show `president` as the session name and `Opus 4.6` as
   the model.

### Step 2 — Open the VP terminal

1. Open a **second** Claude Code terminal (new tab or window — not a subagent).
2. Rename the session to **`vp`**:
   - Type `/name vp` and press Enter.
3. Set the model to **Opus 5.5**:
   - Type `/model` and press Enter.
   - Select **Opus 5.5** from the list.
4. Confirm: `vp` session name, `Opus 5.5` model.

### Step 3 — Verify cross-session messaging

In the **President** terminal, run:

```
List the other Claude sessions on this machine.
```

It should use `ListAgents` and show the `vp` session. If it doesn't appear, both sessions
need `crossSessionInbound: "auto"` or `"always"` in their settings
(`~/.claude/settings.json`).

---

## Operating the President

The President's job is to **understand what you want and relay it faithfully**. Give it your
instructions in natural language — as detailed or as brief as you like. It should:

- **Model your intent.** Ask clarifying questions if your instruction is ambiguous. Think
  about what you actually need, not just what you literally said.
- **Translate intent into directives.** Break your goal into concrete outcomes the VP can
  act on. Each directive should be self-contained: what to achieve, what constraints apply,
  what "done" looks like.
- **Relay directives via `SendMessage`.** The President sends to `vp` (or whatever you
  named the VP session). One message per batch of directives, not one per directive.
- **Stay out of the weeds.** The President should NOT:
  - Read files or source code
  - Call tools (besides `SendMessage` and `ListAgents`)
  - Navigate the codebase
  - Make implementation decisions
  - Try to verify the VP's work by reading code itself

  Its 200k context window is precious. Every tool call and file read burns context that
  should be reserved for understanding your intent across a long session.

- **Relay the VP's questions back to you.** When the VP needs a decision only you can make,
  the President relays it — without editorializing or answering on your behalf.

### What to tell the President at session start

Give it context about yourself, your project, and how you work. The more it understands
about you, the better it translates your intent. Example:

> You are President. Your job is to understand what I want and relay clear directives to
> the VP session. Do not read files, call tools, or dig into code — conserve your context
> for understanding my intent. Use only SendMessage and ListAgents. When I give you an
> instruction, think about what I actually need, break it into directives, and send them
> to vp. When vp has questions, relay them to me without answering on its behalf.

---

## Operating the VP

The VP receives directives from the President and turns them into executed work. It should:

- **Wait for directives.** Do not act until the President sends an instruction. "Prepare
  for instruction" means wait, not pre-fetch or pre-authorize.
- **Never relay the user's words upstream or sideways without being told to.** This is the
  #1 failure mode (see "Known failure modes" below). The VP receives directives from the
  President. It does not quote the user to teammates to claim authority.
- **Accept corrections immediately.** When the President corrects a misunderstanding or
  overreach, acknowledge and adjust. Pushback ("for the record…", "actually…", "but I
  was…") is a red flag.
- **Plan the work.** Break directives into tasks. Identify which need specialists (Opus 5.5
  teammates) vs. general workers (Sonnet 5.5 teammates).
- **Spin up the team.** Load the `team-lead` skill (or follow its principles) to manage
  teammates with proper worktree isolation, the shared task list, the 15-minute wall clock,
  and message hygiene.
- **Report results to the President.** Summarize outcomes, surface decisions that need user
  input, flag blockers. The President relays to the user.

### What to tell the VP at session start

> You are VP. You receive directives from the President session (an Opus 4.6 session that
> talks to the user). Do not message the user directly — everything goes through the
> President. Wait for the President's directives before acting. When you receive them, plan
> the work, spin up teammates, and manage execution. Use Sonnet 5.5 teammates for general
> work and Opus 5.5 teammates only for deep specialist tasks. Report results and questions
> back to the President via SendMessage.
>
> Rules:
> - Never relay the user's quoted words to teammates to claim authority.
> - Accept corrections from the President without pushback.
> - If a directive is unclear, ask the President to clarify — don't guess.

---

## Teammate tiers

### Sonnet 5.5 — General workers (the default)

Spawn with `model: "sonnet"`. Use for:
- Standard implementation tasks (new features, bug fixes, refactors)
- Test writing
- Documentation
- File moves, renames, migrations
- Anything well-specified that doesn't need deep analysis

Sonnet 5.5 is fast, capable, and cheap on context. Most lanes should be Sonnet.

### Opus 5.5 — Deep specialists (use sparingly)

Spawn with `model: "opus"`. Use for:
- Reverse engineering or analyzing unfamiliar code
- Complex debugging with many interacting systems
- Architecture decisions that require understanding subtle tradeoffs
- Security analysis
- Anything where the VP would normally do it itself but needs to parallelize

Opus 5.5 teammates burn more resources and are more likely to editorialize or self-authorize.
Keep their briefs tight and their scope narrow.

---

## Known failure modes

### 1. VP self-authorizes by relaying user quotes

**What happens:** The user says something to the President. The President relays a directive
to the VP. The VP takes the user's original words and sends them to teammates as if the user
gave the instruction directly — bootstrapping authority it wasn't given.

**Why it happens:** Opus 5.5 models are agentic and proactive. They pattern-match "the user
said X" as authorization to act on X, even through an intermediary.

**Fix:** The VP brief must explicitly forbid quoting the user's words downstream. Directives
to teammates come from the VP in the VP's own words, citing the task list — not "the user
said to do X."

### 2. VP pushes back on corrections

**What happens:** The President corrects the VP. The VP argues ("for the record…",
"actually…", "I was already…") instead of adjusting.

**Why it happens:** Opus 5.5 has strong opinions and treats corrections as debates.

**Fix:** The VP brief must set the expectation: corrections are accepted, not debated. If
the VP pushes back more than once, it's a bad seed — kill the session and spin a fresh one.
Session state (model weights + conversation context) varies; sometimes you get a
cooperative instance, sometimes you don't.

### 3. President burns context on tools

**What happens:** You ask the President a question about the code. It reflexively reads
files, greps, explores — and burns 50k tokens of its 200k window on raw tool output it
didn't need.

**Why it happens:** Opus 4.6 is helpful and eager. If you ask "what does X do?", it will
try to find out instead of asking the VP.

**Fix:** Reinforce in the President's brief: "Your tools are SendMessage and ListAgents.
Nothing else. If I ask about code, relay the question to vp."

### 4. Teammates message the President directly

**What happens:** A teammate discovers something important and messages `president` instead
of `team-lead` (the VP).

**Why it happens:** The teammate's brief mentioned the President exists, and the teammate
decided to escalate.

**Fix:** The VP's briefs to teammates must say: "Your lead is `team-lead` (the VP session).
You do not message `president` or the user. Ever."

### 5. Context exhaustion cascade

**What happens:** The President runs out of context after a long session. You spin a new
President, but it has no history. The VP is now taking direction from a President that
doesn't understand the project.

**Fix:** Before the old President's context runs low (~150k used), have it write a handoff
document: everything it knows about your intent, the project state, open decisions, and
what the VP is working on. The new President reads the handoff as its first message.

---

## Session lifecycle

### Starting a task

1. Open President + VP terminals (see Setup above).
2. Brief both sessions (see "What to tell X at session start").
3. Give your instruction to the President.
4. The President translates and relays to the VP.
5. The VP plans, spins up teammates, manages execution.
6. Results flow back: Teammates → VP → President → You.

### Mid-task corrections

Tell the President. It relays to the VP. The VP adjusts the team. Never bypass the chain
by messaging the VP directly — that undermines the President's context and creates
conflicting authority.

### Replacing a bad VP

If the VP self-authorizes, argues corrections, or drifts:
1. Tell the President to note the VP is being replaced.
2. Kill the VP terminal (Ctrl+C or close the tab).
3. Open a fresh terminal, name it `vp`, set it to Opus 5.5.
4. Have the President send the new VP a handoff with current state and active directives.

### Ending the session

1. VP shuts down all teammates (`TaskStop` each).
2. VP sends a final summary to the President.
3. President relays the summary to you.
4. Close both terminals.

---

## Resource budget

| Component | RAM (approx.) | CPU threads |
|-----------|--------------|-------------|
| President (Opus 4.6) | ~200 MB | 1 |
| VP (Opus 5.5) | ~200 MB | 1 |
| Each Sonnet 5.5 teammate | ~2 GB | 1 |
| Each Opus 5.5 specialist | ~2 GB | 1 |
| Per project tool (dev server, editor, etc.) | varies | 1–2 |

A typical setup with 4 Sonnet workers + 1 Opus specialist needs roughly **10–12 GB free**.
Add 10% buffer. On a 16 GB machine, that means closing other heavy applications first.

---

## Quick reference

```
/name president          # in terminal 1
/model → Opus 4.6       # in terminal 1

/name vp                 # in terminal 2  
/model → Opus 5.5       # in terminal 2

User  → President: "Build me X with Y constraints"
President → VP:    "Directive: implement X. Constraints: Y. Done = Z."
VP → Teammates:    [spins up team-lead skill, creates tasks, manages lanes]
Teammates → VP:    [results, blockers, questions]
VP → President:    [summary, decisions needed]
President → User:  [relays results, asks for decisions]
```
