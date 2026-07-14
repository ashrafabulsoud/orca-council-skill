---
name: orca-council
description: Convene a council of AI agents to independently deliberate on a question, proposal, or design decision using Orca's orchestration layer, then synthesize their verdicts through a judge (an agent or the human). Use when the user invokes /orca-council with a topic, or asks to "convene a council", "get multiple agent opinions", "have the agents vote", or "run a panel review" on an idea, architecture decision, code change, or plan.
disable-model-invocation: true
---

# Orca Council

Convene N independent agent "council members" on the same question, collect structured verdicts, and produce a final judgment — either from a judge agent or from the human.

You are the **coordinator**. You never answer the council question yourself. Your job is logistics: preflight, roster, dispatch, collection, judgment, report.

The question is: $ARGUMENTS

If $ARGUMENTS is empty, ask the user what the council should deliberate on before doing anything else.

## Phase 0 — Preflight

Orchestration is experimental and requires the Orca runtime. Verify before creating anything:

```bash
orca status --json
```

If this fails, stop and tell the user: orchestration must be enabled under **Settings → Experimental** and Orca must be running. Do not attempt workarounds via `orca terminal send` — the council depends on tracked tasks and `worker_done` messages.

Also check for leftover state from a previous council:

```bash
orca orchestration task-list --json
```

If stale `Council:` tasks exist, ask the user whether to proceed alongside them or clean up first. NEVER run `orca orchestration reset` without explicit user confirmation — it wipes runtime-global orchestration state and will break any other active coordinator.

## Phase 1 — Roster

Build the full agent menu first — the council is not limited to agents that already have live terminals. Gather three sources:

```bash
# a) Live terminals (reuse shortcuts)
orca terminal list --json
orca worktree ps --json

# b) Agent types Orca can launch
orca worktree create --help
```

Parse the accepted values of `--agent` from the help output; if this Orca build offers a dedicated listing command (check `orca --help` for something like `orca agent list --json`), prefer that.

```bash
# c) Agent CLIs installed on this machine
for bin in claude codex grok cursor-agent gemini opencode amp droid goose aider qwen; do
  command -v "$bin" >/dev/null 2>&1 && echo "$bin"
done
```

Extend the candidate list with anything the Orca help output names that isn't in it.

Merge the three sources into one table and show it to the user:

| Agent | Installed | Orca-launchable | Live terminal (state) |
|---|---|---|---|

Selection rule: any agent that is BOTH installed AND Orca-launchable is selectable, with or without a live terminal — seats without one get a fresh worktree in Phase 2. A live idle terminal is only a reuse shortcut. If an agent is installed but not Orca-launchable (or the reverse), still list it, marked unselectable with the one-line reason, so the user understands why it can't take a seat.

Then ask exactly three questions, STRICTLY ONE AT A TIME: ask, wait for the user's answer, only then ask the next. Never bundle two questions into one message.

**Q1 — Members.** Present the table and ask which agents sit on the council (2–5 seats). Prefer *different* agent types per seat; a council of three identical agents is mostly noise. The same agent type twice is allowed — fresh worktrees keep the seats independent. Wait for the answer.

**Q2 — Judge.** Ask who rules on the verdicts:
- `me` — the human rules directly (default if they don't care)
- an agent — any selectable agent, member or not; a non-member judge gets its own seat in Phase 5
Wait for the answer.

**Q3 — Deliberation depth.** Ask whether members may read the repo in their worktree, or should reason from the prompt alone (default: standard). Read access suits code/architecture questions; prompt-only is faster for strategy/naming/product questions. Wait for the answer.

Only after all three answers are in, proceed to Phase 2.

## Phase 2 — Seat the members

For each member without an existing idle terminal:

```bash
orca worktree create --name council-<agent>-<n> --agent <agent> --json
orca terminal list --worktree id:<newWorktreeId> --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json
```

Record each member's terminal handle. Reused terminals must be idle — check with `orca terminal wait --for tui-idle` before dispatching to them.

## Phase 3 — One task per member, same spec

Dispatch is one-to-one: there is no "send one task to three terminals". Create a separate task per seat, all with an identical spec. Build the spec from this template, filling in the council question and any context the user provided:

```
COUNCIL DELIBERATION — you are one member of an independent panel.

Question: <the council question, verbatim>

Context: <repo paths, constraints, links, or "none">

Rules:
- Form your own independent opinion. Do not hedge into "it depends" without committing.
- <If read access: "You may inspect the repository in your worktree." / else: "Reason from this prompt only; do not modify files.">
- Do NOT make code changes unless the question explicitly asks for a prototype.

Your worker_done body MUST end with exactly these three lines:
VERDICT: <approve | reject | needs-changes | option-name>
CONFIDENCE: <low | medium | high>
RATIONALE: <one or two sentences, the single strongest reason>
```

For open-ended questions ("which framework?", "name for X?"), replace the approve/reject vocabulary with the actual option set, or `VERDICT: <your single recommendation>` if options aren't enumerable. The fixed trailer is what makes Phase 5 parseable — keep it rigid.

Create and dispatch each seat:

```bash
orca orchestration task-create \
  --task-title "Council: <agent> — <short topic>" \
  --display-name "Council seat: <agent>" \
  --spec "<the spec above>" \
  --json

orca orchestration dispatch --task <taskId> --to <memberHandle> --inject --json
```

Always use `--inject` — it gives the worker the preamble that teaches it to report `worker_done` with the correct `--task-id` and `--dispatch-id`. Record the (member, taskId, dispatchId) triple for each seat; `orca orchestration dispatch-show --task <taskId> --json` recovers it if lost.

## Phase 4 — Collect the votes

Block on the message types that matter, and loop until every seat has reported:

```bash
orca orchestration check \
  --wait \
  --types worker_done,escalation,decision_gate \
  --timeout-ms 900000 \
  --json
```

Handle each message by type:

- **worker_done** — match `taskId`/`dispatchId` against your roster, extract the VERDICT/CONFIDENCE/RATIONALE trailer from the body, mark the seat as voted. Ignore `worker_done` messages whose dispatchId doesn't match a current dispatch (stale retry).
- **escalation** or **ask** — a member has a blocking question. If it's about scope/interpretation of the council question, answer it yourself consistently (and give the same answer to any member who asks the same thing — asymmetric information skews the vote). If it's a judgment call the human should make, relay it and forward their answer with `orca orchestration reply --id <messageId> --body "..." --json`.
- **timeout** — a checkpoint, not a failure. Check `orca orchestration task-list --json` and `orca terminal read --terminal <handle> --json`; if the member is still visibly working (or sending heartbeats), issue the same `check --wait` again. If a member's terminal is dead or wedged, tell the user and offer: retry with a fresh dispatch (`orca orchestration dispatch --task <taskId> --to <handle> --inject --json`), replace the seat, or proceed with N−1 votes.

Do not start Phase 5 until every seat has either voted or been explicitly written off by the user.

## Phase 5 — Judgment

Build the verdict table first (you need it in both branches):

| Member | Agent | Verdict | Confidence | Rationale |
|---|---|---|---|---|

**If judge = me (the human):** present the table plus each member's full worker_done body (trimmed to the substantive part), note where members agree/disagree, then ask the user for their ruling. Their answer is the judgment.

**If judge = an agent:** create one more task whose spec embeds the question and all full verdict bodies:

```
COUNCIL JUDGMENT — you are the presiding judge.

Question: <the council question>

The <N> member opinions follow, verbatim:
--- Member 1 (<agent>) ---
<full worker_done body>
--- Member 2 (<agent>) ---
...

Weigh the arguments, not the vote count — a well-reasoned minority can win.
Resolve contradictions explicitly. Your worker_done body must contain:
RULING: <the decision>
KEY REASONING: <2-4 sentences>
DISSENT WORTH NOTING: <the strongest losing argument, or "none">
NEXT STEP: <one concrete action>
```

Dispatch it to the judge terminal with `--inject`, then run the same `check --wait` loop for its `worker_done`.

## Phase 6 — Report and cleanup

Deliver the final report to the user in chat:

1. The question
2. The verdict table
3. The judgment (RULING / KEY REASONING / DISSENT / NEXT STEP)
4. Anything a member surfaced that the judge missed but seems material

Then ask whether to keep or remove the council worktrees. Remove only on confirmation — worktrees may contain inspection notes worth keeping. Leave orchestration task records alone unless the user asks for cleanup; if they do, prefer `task-update --status` bookkeeping over `reset`.

## Failure notes

- A member that changes files when it shouldn't: note it in the report; its worktree is isolated, so nothing leaked.
- Two `worker_done` from one seat: trust the one whose dispatchId matches the active dispatch.
- Everything hopeless mid-run: `orca orchestration run-stop --json` does not apply (you are the coordinator, there is no managed run) — just stop dispatching, summarize partial votes, and let the user decide.
