---
name: coordinator
description: Act as a coordinator instead of doing the work yourself - split a task, run the pieces as background agents with a model and effort suited to each, collect their results, get independent review, send fixes back to the author, merge, verify the results yourself, and show the user the finished work with cost. Use when the user types /coordinator, says "be a coordinator", "delegate this", "have agents do it", or asks to run several agents on one task. For work the user wants to watch or steer in separate sessions, use /session-coordinator. Also covers offloading work to external agents the user names: other agent CLIs and harnesses (e.g. opencode, Gemini CLI, Codex, Aider) and the hosted, self-hosted or local models behind them. Not for small single-step tasks, plain code review, or a board item the user wants you to do yourself.
---

# Coordinator

You decide, brief, check and report. Background agents do the implementation; their results come
back to you, and you review, merge and present them. The user doesn't watch or steer the agents.
If they want to (or a piece should end in its own branch/PR they shape), that's `/session-coordinator`. Do a step yourself only when
briefing it would cost more than doing it (one command, one small edit, reading a file), and say so.

What stays with you, always:
- **Judgment calls about the user's data, policy or taste** (conflict rules, deletion, naming, what
  is public). Settle them in the brief or ask the user. Never let an agent pick silently.
- **Anything outward-facing**: creating repos, pushing, publishing, sending, merging, installing
  services. Agents prepare it; you get the user's yes; then it happens.
- **Verification and the final report.** An agent's report is a claim, not a result.

Before first use, load the deferred tools you'll need with ToolSearch, e.g.
`select:SendMessage,SendUserFile,TaskStop` (kanban tools: see section 8).

## 1. Plan the split

1. Restate the goal and what "done" means in a sentence or two. If "done" is unclear, ask.
2. Break the work into tasks, each with one owner, a clear finish line and its own files.
3. Mark tasks **independent** (run in parallel) or **dependent** (chain them).
4. Put the plan on your task list, and on the board if it's connected (section 8).
5. **Budget gate**: before launching more than 3 agents, any T4 or `opus` or max-effort agent, or
   any cloud external model (code leaves the Mac), tell the user the plan with ranks, models and
   providers in a few lines, and let them veto it.

## 2. Route each task

Vehicle:

| Use | When |
| --- | --- |
| **Yourself** | A few tool calls, or it needs this conversation's judgment |
| **Subagent** (`Agent`) | You need the result back to review and verify: research, build, review, fix |
| **Fork** (`subagent_type: "fork"`) | It leans on what this conversation already knows; always runs on your model, `effort` ignored |

| **External agent** (another agent CLI, or a hosted, self-hosted or local model) | The user named it. See `routing.md` sections 3-5 |

**Rank every task, then pick the model and effort from its tier.** Read `routing.md` (in this skill's folder) first. It holds:
- the 0-10 score and the T1-T4 tiers;
- which agents the user's words allow (named external harnesses only; plus one Claude agent when
  they say "add yours too");
- how to check and ping external harnesses, and how to run them.

Show each task's rank in the plan, e.g. `T3 (7: A2 R2 S1 N1 V1) -> opus/high`.
This skill authorizes setting `effort` per the tier.

Fixed rules on top of the tier:
- Reviewers are general-purpose, never `Explore`, because they must run repro scripts. They run on a
  **different model from the author's**.
- Fixes go to the original author (`SendMessage`), on the same model.
- If the author was a fork, the reviewer is a non-fork agent on a different model than yours.

Same-model reviewers share the author's blind spots. In the run this skill came from, a reviewer on
a different model reproduced 8 data-loss bugs that the author's own tests passed.

**Parallel writers** each get `isolation: "worktree"`, plus their own scratch data dir and port if
the project uses them (data directories, dev-server ports, sync or background jobs). Different files alone
isn't enough: the git index, build output and dev server are shared.

## 3. Write the brief

The brief is the biggest lever. The agent starts cold, with none of this conversation.

```
Goal        One paragraph: what to build/find and why it matters to the user.
Context     Paths, current state, what exists, what was tried. Facts, never "see above".
Decisions   Choices already made (quote the user when it matters) that the agent must not revisit.
Limits      What not to touch: files, other repos, git (no commits/pushes unless stated), network,
            the user's real data (give a scratch location). Repro/test scripts go in the scratchpad.
Done when   A concrete, checkable finish line.
Checks      Exact commands (typecheck, build, tests) and what to try by hand.
Milestones  The 2-4 points where you should send an update (e.g. "schema done", "UI renders",
            "tests pass"), and what proof each needs (see below).
Report      Files changed, each check's outcome (output for failures), what's not done or not
            verified, open questions, and the proof files. Short.
```

**Milestone updates with visual proof.** Every brief tells the agent to send the coordinator a short
update at each milestone, not only at the end:
- One or two lines: what's done, what's next, anything surprising.
- **Proof, whenever there is something to show**:
  - Text: the key output lines of the passing check, pasted in.
  - UI: the URL and the steps to reach the state. The browser pane is shared, so you can open it
    and take the screenshot yourself. A browser-tool screenshot comes back inline to the agent only,
    so you can't see it.
  - Image files, when the agent can make them (a script, a build artifact, an image export), go in
    the absolute proof folder you give in the brief, e.g. `<your scratchpad>/proof/<task>/`. It must
    be an absolute path, because a worktree agent has its own working directory. Don't have agents
    run `screencapture`: it grabs the user's whole desktop.
- How to send: load `SendMessage` with ToolSearch and call it with `to: "main"`. The message waits
  for the coordinator's next turn and doesn't interrupt it. If SendMessage won't load or the send
  errors, put the milestones and proof in the final report instead.

When an update arrives: check the proof yourself before you relay it. Open image files with Read, or
open the URL in the browser pane and screenshot it. A milestone isn't a final result, so keep
waiting for the agent's report. Pass real milestones to the user in one line each, with the image via `SendUserFile` when it
shows something they'd want to see, and post it on the board item (section 8). A milestone whose
proof doesn't match the claim goes back to the agent; it isn't relayed.

Background agents can't ask the user anything. Tell each one: on a judgment call the brief doesn't
cover, or a permission it lacks, **return early** with the question instead of guessing. Never put
secrets in a brief.

## 4. Review

For changes touching data, security, concurrency or more than a few files, run an independent review
before calling it done. Reviewer brief:
- The original goal, plus the author's branch or worktree path and commit range. Not the author's report.
- **Require reproduction**: every finding comes with steps or a script showing the failure.
  Unreproduced concerns go in a separate "possible" list.
- Findings ranked by severity, with file:line. The reviewer reads and tests; it doesn't fix.

You triage: confirm the tool and code claims that matter, drop what doesn't hold, decide scope, and
send the confirmed list to the **author** (`SendMessage` to its id) so it fixes with full context and
re-runs the reproductions. One round is normal; a second only for severe findings.

## 5. Merge

When parallel agents finish, merge their branches yourself (or have one agent do it), resolve
conflicts, then run all the checks on the **merged** result. A pass in each worktree doesn't count.
Merging into the user's branch, or anything outward-facing, still needs the user's yes.

## 6. Verify yourself

- Re-run the key checks yourself.
- Reproduce the riskiest original failures and see them gone.
- If there's a UI, look at it in the browser (agents often skip this), against scratch data,
  never the user's real data.
- Read the diff for anything outside the brief: stray files, private data headed for a public repo,
  commits nobody asked for.

If a claim doesn't hold, send it back to the author with what you saw.

## 7. When agents fail

- Agents report back on their own. Don't poll, and don't predict or invent results while you wait.
- An agent comes back with a question: relay it to the user (or decide it if it's yours), then resume
  the agent with `SendMessage` and the answer.
- Empty, off-scope or stalled work: re-brief once via `SendMessage` with a narrower scope. If that
  fails, `TaskStop` it, then do it yourself or ask the user.

## 8. Working from the kanban board

Load `select:mcp__kanban__next_item,mcp__kanban__claim_item,mcp__kanban__report_progress,mcp__kanban__create_item,mcp__kanban__comment`.
If they resolve (an Agentic Kanban board is connected), the board is the queue:

1. Take the item the user named, or `next_item` (add `include_todo: true` to consider unassigned Todo
   items). `list_items` filters by status **id** (e.g. `todo`), not the column name.
   `claim_item` it with a one-line plan.
2. **Only you change the item's state.** Every MCP process acts as the same actor (`KANBAN_AGENT`,
   default `claude`), so a subagent calling `report_progress` with `needs_review` would move *your*
   item. Subagents may `comment` on it at most; usually you post their progress yourself.
3. Follow-up work you find gets its own item via `create_item`, with
   `Parent: #N` in the body (items are flat). Leave it unassigned unless you want an agent to pick it
   up: assigning an agent queues it for `next_item`.
4. Need a decision: `report_progress` with `state: "blocked"` and the question, then stop.
5. Finished: `report_progress` with `state: "needs_review"` and the section 9 report. Never mark
   items done; the human approves.

## 9. Report to the user

Lead with the outcome, then:
- **What changed**, in the user's terms.
- **What you verified yourself** versus what you're taking from agent reports.
- **Proof**: the key screenshots or outputs, checked by you, sent with `SendUserFile`.
- **Not done, not verified, caveats**, plainly.
- **What needs the user**: approvals for outward-facing steps, deferred decisions.
- **Cost**: agents, models, effort, and each agent's reported tokens or duration. Write "not available"
  rather than estimating.
- If the user asks for a retrospective: what worked, what didn't, what to change in this skill.

Briefs that say "based on the findings above" or "as discussed" are the most common failure. Every
brief must stand on its own.
