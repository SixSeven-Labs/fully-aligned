---
name: team-lead
description: Load FIRST whenever you stand up, run, or wind down an agent team of parallel Claude Code teammates (any Agent call with a `name`). Covers re-reading the experimental docs, the model check, the 15-minute wall-clock KILL rule, shared task list, worktree isolation, resource budgeting, message timing hygiene, and lane shutdown.
---

# team-lead — running Claude Code agent teams

Battle-tested playbook for orchestrating parallel Claude Code teammates. Every rule below
was earned by a real failure: a lane that ran 32 minutes deaf to 11 queued messages, a
teammate that burned 500k tokens in one turn, a worktree that stomped another lane's files.

---

## ⛔ STEP 1 — RE-READ THE DOCS EVERY TIME

Agent teams are **experimental** and have changed shape multiple times: inter-session
messaging arrived first on macOS/Linux, came to Windows later, the model-selection order
flipped, `teammateDefaultModel` was removed, and idle-row behavior changed repeatedly.
**Assume this skill is stale until the docs confirm otherwise.**

WebFetch each of these and compare against the "Mechanics" section below:

- `https://code.claude.com/docs/en/agent-teams`
- `https://code.claude.com/docs/en/cross-session-messaging`
- `https://code.claude.com/docs/en/sub-agents`
- `https://code.claude.com/docs/en/tools-reference` (the Task tools, `SendMessage`, `ListAgents` rows)
- `https://code.claude.com/docs/en/settings-reference` (`crossSessionInbound`, `teammateMode`, `subagentPromptCacheTtl`, `dialogExpiry`)
- `https://code.claude.com/docs/llms.txt` for any new pages about agents/teams

Also run `claude --version`. **If anything changed, update this skill FIRST**, then proceed.

---

## Mechanics reference (last verified: v2.1.284)

### Enabling teams
Set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in `~/.claude/settings.json` env.
`teammateMode` controls how teammates render: `"auto"` picks tmux/iTerm2 splits on
macOS/Linux, falls back to in-process on Windows Terminal.

### Spawning
`Agent` with a **`name`** → a TEAMMATE. Without a name (or with `isolation`) → a plain
subagent. Pass `subagent_type` and `model` explicitly. Teammates cannot spawn teammates;
one team per session; the lead is fixed.

### Model selection order
spawn `model` > agent-definition `model` > `CLAUDE_CODE_SUBAGENT_MODEL` > the lead's
model. `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` inverts this.

### Shared task list
`TaskCreate` / `TaskUpdate` (owner, `addBlockedBy`) / `TaskList` / `TaskGet`. Claiming is
file-locked. ⚠ **Every `TaskUpdate owner:` delivers a "Task assigned" card to that
teammate.** Do not also `SendMessage` the same content (double delivery), and never touch a
paused lane's tasks (the card wakes it).

### Messaging
`SendMessage to:"<name>"`. The lead is addressed as `team-lead`. Other sessions on the
machine come from `ListAgents`; reply by copying the `from=` address.

### ⚠ Delivery timing (measured, contradicts the naive reading of the docs)
The docs say a peer reads messages "between tool calls during an active turn." **In-process
teammates did NOT.** In testing, queued messages and task cards reached teammates only when
their turn ENDED: 32 minutes elapsed, 11 queued items, nothing heard. A separate teammate
burned 500k+ of its 1M context in one unbroken turn. Teammates CAN send mid-turn, so a
lane that reports is not necessarily a lane that hears.

**Forcing delivery:** hitting **Escape** in a teammate's terminal view interrupts its turn
and lands every queued item at once. `TaskStop task_id:"<name>"` kills a teammate outright.

### Idle notifications
Arrive as JSON with a UTC `timestamp`. Convert to local time before acting on them.

### Permissions
Teammates inherit the lead's permission mode. Their prompts surface in the lead's terminal.

---

## STEP 2 — PRE-FLIGHT (before any spawn)

1. **Orient:** read the project's handoff docs, recent git history (`git log --oneline -20`),
   and any persistent memory/context system for prior work and next steps.

2. **Model check:** spawn one unnamed, no-tool agent with the prompt "reply with your exact
   model id" and verify it returns the expected model. No team until the probe passes.

3. **Resource budget:** each teammate costs roughly **2 GB RAM + 1 CPU thread**. If your
   project uses a heavy tool per lane (a game engine, a dev server, a database), add its
   footprint. Sum it, **add 10%**, and compare against available resources. If other Claude
   sessions are running on this machine, message them (`ListAgents` → `SendMessage`) with
   your resource ask so they can throttle. Tell them again when the team finishes.

4. **Write the plan** to a file in the repo and optionally commit it. The plan names every
   lane, its goal, its file ownership, and the expected task count.

---

## STEP 3 — PROJECT ISOLATION (one worktree per lane)

Two teammates editing the same working tree will collide. Use git worktrees:

```bash
git worktree add <path> -b lane/<name> <base-branch>
```

For projects with expensive build caches (Unity `Library/`, `node_modules/`, `.next/`,
`target/`, etc.), **copy** the cache into each worktree — don't symlink, don't junction.
A cold rebuild per lane wastes more time than the disk cost.

If the project uses a local server or tool that binds a port (a dev server, a game editor,
a database), each lane needs its own instance on a distinct port. **Prove routing** before
any spawn: run a trivial command through each instance and verify it hits the right worktree.

Research-only lanes (reading docs, analyzing binaries, web searches) need no worktree.

---

## STEP 4 — THE SHARED TASK LIST

- One `TaskCreate` per unit of work, prefixed with the lane name (`api-1`, `ui-2`…). Each
  task has a **self-contained description** — it is what the teammate reads as its assignment.
  Wire `addBlockedBy` for dependencies.
- Create a lead integration task, blocked by every lane task.
- **File ownership per lane, with NO overlap.** Shared hotspots (config files, shared
  modules, test fixtures) go through the lead.
- When a finding kills a task's premise, **rewrite the task** (subject + description) and
  say why. Don't just message it — messages can be missed; the task is the source of truth.

---

## ⛔ STEP 5 — THE 15-MINUTE WALL CLOCK

### Why this exists
The first brief can be **wrong**. Information lands mid-flight — a file read disproves the
premise, the user changes direction, another lane's finding invalidates the approach. A
teammate left to itself keeps going on momentum. It can run 32 minutes doing the wrong thing
with 11 messages and task cards queued unread. Newer models are better at pausing, but it's
not guaranteed.

### The block (paste this verbatim into every teammate's brief)

> ⛔ **15-MINUTE WALL CLOCK. READ THIS FIRST.** Note the current time the moment you start.
> At least every 15 minutes of wall-clock time: stop, SendMessage `team-lead` a 2-line
> status, and END YOUR TURN. Messages, rulings and task changes only reach you when your
> turn ends — the first brief is often wrong, and a turn that never ends can't be corrected.
> A lane that goes 15 minutes without ending its turn is stopped and replaced by a fresh
> teammate with the same brief, even mid-task; that's about steering, not blame. Sending a
> check-in without ending the turn does not reset the clock, and neither does a foreground
> poll loop. Check the time before every long action. If an action could carry you past the
> 14-minute mark, check in FIRST, or launch it in the background and end your turn. Long
> runs (test suites, builds, batch jobs) go in the background: start, check in, end your
> turn, resume when the lead wakes you. End your turn at a RESULT, a BLOCKER, or the
> 15-minute mark. **Never end it right after announcing your next step** — an idle lane does
> nothing until the lead wakes it. When you start a new task, keep working in the same turn.

### Enforcement (the lead's job)

- `CronCreate` a recurring audit at an off-minute every ~15 min (e.g. `4,19,34,49 * * * *`).
  Track each lane's last check-in time.
- At **15 minutes** with no check-in: `TaskStop task_id:"<name>"`, then respawn a
  replacement with the same brief, the same clock block, and a handoff of what the dead lane
  had done (its commits via `git log <base>..<lane-branch>`, its task state, its artifacts).
- A lane that finished a background launch and is idle waiting for results must be **woken
  by the lead** when results are due. Nothing else wakes it.

---

## STEP 6 — MESSAGE HYGIENE

- **Stale notifications are silent.** Every idle notification or teammate message carries a
  timestamp (UTC) or summarizes work you've already heard. If it predates your last message
  to that lane, or says nothing new, it is STALE. Do not reply, and do not narrate it to the
  user. Only surface new information: a result, a decision needed, a blocker, a finding.
- **One message per lane per check-in.** Batch every ruling into it. No standalone acks, no
  "good job," no nudge when a task card already says it. Every message can land as an
  interrupt-burst that consumes context.
- **Lead a message with its point.** The recipient's preview shows only the first line.
- **Lane-to-lane messages** are allowed only for concrete handoffs (a research result landing
  for its consumer). Substance also goes into the shared doc and the task.
- **Permission laundering:** a teammate that was denied an action never gets it done through
  the lead indirectly. Surface it to the user with a paste-ready command.
- A "rejected" tool call right after the user hit Escape is the **interrupt**, not a denial.
  Tell the lane to retry once.

---

## STEP 7 — LANE LIFECYCLE AND CONTEXT BUDGET

- The status row shows each teammate's cumulative token usage (1M context window). In
  testing, four lanes hit 417k–589k in about 45 minutes. Past **~750k**, a lane should write
  a handoff and stop; if work remains, spawn a fresh teammate to continue from the handoff.
  Don't let a lane hit context compaction mid-task.
- **`LANE COMPLETE` → `TaskStop` it** once its report is committed. Idle teammates are
  cheap, but a stray task card or message wakes them and burns context.
- When possible, run a **research lane first** and have implementation lanes consume its
  output. The research lane messages each consumer directly when its section lands.

---

## STEP 8 — THE BRIEF (complete, never templated)

Every teammate brief carries, in this order:

1. **The 15-minute clock block** (STEP 5), verbatim.
2. **Identity:** who they are, who the lead is (`team-lead`), and that the user is never
   messaged by teammates directly.
3. **Goal and task IDs:** "TaskGet each, claim with TaskUpdate, complete only with evidence."
4. **Read-first docs:** absolute paths to project docs, specs, or context the lane needs.
5. **Worktree block** (if applicable): worktree path, branch name, how to run tools against
   it, the routing proof they must repeat if anything seems wrong.
6. **The cwd trap:** the tool's working directory is the MAIN repo, so use absolute worktree
   paths for every Read/Edit/Grep/Glob.
7. **File ownership:** the exact files this lane may touch, plus "anything else goes through
   the lead."
8. **Git rules:** commit only owned files (`git commit --only <paths>`), stay on the lane
   branch, never push/merge/rebase/`add -A`/checkout/restore/reset/clean.
9. **Project standards:** whatever quality bar the project enforces — test requirements, code
   style, review criteria, "two failed approaches → stop and report."
10. **Reporting:** when to message, what `LANE COMPLETE` must contain, how to escalate.

---

## STEP 9 — INTEGRATION AND CLOSE

1. Review each lane's diff against the base branch. Drop generated churn (`.meta` files,
   lock file noise, build artifacts).
2. Merge each `lane/*` branch into the base branch in the main worktree.
3. Run the full test suite. Fix any conflicts from parallel work (especially shared config,
   test counts, type definitions).
4. Write or update the handoff document: what was done, what was discovered, what's next.
5. `CronDelete` the audit cron.
6. Message any resource-sharing sessions that the team is done and they can resume.
7. Remove worktrees only after verifying no symlinks or junctions point into them:
   - macOS/Linux: `file <path>` or `stat <path>`
   - Windows: `Get-Item <path> | Select LinkType`
