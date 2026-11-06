---
name: session-coordinator
description: Coordinate work across separate Claude Code sessions that the user can open, watch and steer - split a task, start one session per piece (each in its own worktree, with a model and effort suited to it), follow their reports, get each branch reviewed, and bring the results together with the user's approval. Use when the user types /session-coordinator or asks for the work to run in sessions they can peek into or steer. For hidden background agents whose results you review and merge yourself, use /coordinator instead. Also covers offloading work to external agents the user names: other agent CLIs and harnesses (e.g. opencode, Gemini CLI, Codex, Aider) and the hosted, self-hosted or local models behind them.
---

# Session coordinator

Same job as `/coordinator` (decide, brief, check, report; don't implement), but every piece of work
runs in its **own session**: a real Claude Code session the user sees in the sidebar, can open,
read, steer or stop. The user is a co-driver. Plan for them stepping into any session at any time.

How it differs from `/coordinator`:

| | `/coordinator` | `/session-coordinator` |
| --- | --- | --- |
| Workers | Background subagents (`Agent`) | Sessions (`start_session`) |
| User can watch and steer | No | Yes, at any time |
| Results come back | Full report, this turn | A note (first ~400 chars of the session's report) when it ends a turn |
| Milestone updates | `SendMessage` to `"main"`, queued for your next turn | `send_message` to `"parent"`; arrives as a cross-session message. The user also sees it live in the session |
| Work lands on | Your checkout, or worktrees you merge | Each session's own branch (or PR) |
| Merging | You merge, then show the user | Only when the user tells you to |
| Best for | Fast, contained work you want finished and reviewed in one go | Bigger or longer pieces, work the user wants to shape, anything ending in its own PR |

Small steps (one command, one edit, reading a file) you still do yourself. A quick lookup you need
for planning can go to a subagent; the work itself goes to sessions.

Judgment calls and outward-facing actions stay with you, as in `/coordinator`. A session may push,
open a PR or merge only if the brief says the user already approved that exact step.

Before first use, load the deferred tools with ToolSearch, e.g.
`select:mcp__ccd_session__start_session,mcp__ccd_session_mgmt__list_sessions,mcp__ccd_session_mgmt__get_session,SendMessage,mcp__kanban__create_item,mcp__kanban__report_progress,mcp__kanban__claim_item`.

## 1. Plan the split

1. Restate the goal and what "done" means. If it's unclear, ask.
2. Split into pieces that can each run in their own session: one owner, a clear finish line, their
   own files, nothing that needs another piece's unmerged changes. Pieces that depend on each other
   run one after another: start the second when the first's branch is merged (with the user's yes).
3. Rank each piece (`../coordinator/routing.md`). Then show the user the plan in a few lines: each session's or
   terminal's title, what it does, rank, model, effort, and provider for external models.
   **Budget gate**: with more than 3 sessions, or any on your top model at high effort, wait for
   their go-ahead before starting.
4. Put the plan on your task list, one step per session, with the step's `session` set to its name.

## 2. Start the sessions

One `start_session` call per piece, in one message, after a bold title line and one sentence each.
If you start two or more, call `mark_chapter` first with a short batch title.

- `initiation: "user_asked"` (the user asked for sessions by invoking this skill).
- `context: "fresh"` with a full `prompt` (the brief, section 3) and `background` (goal so far,
  decisions, how this piece fits; at least 80 characters). Use `"fork"` only when the piece truly
  needs this whole conversation and the conversation is short.
- `use_worktree: true` (fresh sessions only) for anything that writes code, so each session gets its own branch.
  `false` only for read-only pieces (research, review of an existing branch).
- `name`: a short handle ([a-z0-9-]); `branch_name`: a slug for the branch.
- `model` and `effort`: **from the task's rank and tier** in `../coordinator/routing.md` (the coordinator skill's routing file; read it first). Cap them at
  yours, since higher values are ignored. A review session runs on **a different model from the
  author's**.
- **External agents** the user named (other agent CLIs, hosted or local models): `../coordinator/routing.md` decides
  whether Claude sessions join them, and covers how to check and run them. Each one runs in its own
  **Terminal tab** in its own worktree, so the user can watch it and type into it like a session.
  Link the tab. They can't message you: they write milestones to `<proof>/<task>/progress.md`.

- `wake_when_done`: leave it unset. The app continues you when sessions end a turn anyway; `true`
  only also wakes you if the session never gets its task.

Link every session you mention as `[its title](#local_<id>)`, using the id from the tool result.

## 3. Write the brief

A fresh session has none of this conversation. The user may also read the brief to understand the
session, so write it for both.

```
Goal        What to build/find and why it matters to the user.
Context     Paths, current state, what exists, what was tried. Facts, never "see above".
Decisions   Choices already made (quote the user when it matters). Don't revisit them.
Limits      Files and areas it owns; what it must not touch. Commit to your own branch; no push,
            PR or merge unless stated. Use scratch data (a copy of any data directory, sync or
            background jobs off, a free port), never the user's real data.
Done when   A concrete, checkable finish line.
Checks      Exact commands and what to try by hand (including the UI in a browser).
Report      End your turn with: branch name, files changed, each check's outcome, what's not done
            or not verified, open questions. Put the most important line first: only the first
            ~400 characters reach the coordinator automatically.
Milestones  The 2-4 points to send the coordinator an update: load mcp__ccd_session_mgmt__send_message
            and send to session_id "parent". One or two lines, plus proof: the passing check's
            output, or for UI the URL and steps (you show screenshots in your own session; the
            coordinator opens the URL itself). Image files go in <absolute proof folder>: use
            `<coordinator scratchpad>/proof/<session name>/`. Sessions on this Mac can write there.
            If the send fails, put the milestones at the top of your final report.
Steering    The user may step in and redirect you. Their instructions win over this brief; mention
            any change of direction in your report.
```

On a question the brief doesn't cover: the session should ask in its own session (the user can see
and answer it there), or end its turn with the question at the top of its report.

## 4. While sessions run

- **Don't poll and don't predict.** The app continues you with a `<side-sessions-event>` when sessions
  end their turn. Until then, answer the user or do your own small steps.
- A milestone update arrives: check its proof yourself (Read the file, or open the URL in the browser
  pane), then pass it to the user in a line (image via
  `SendUserFile` when worth seeing).
- When a note arrives: it's a claim. Look at the result (`get_session`, the branch's diff and log)
  before telling the user something is done.
- **Follow-ups go to the session that has the context**: `SendMessage` with `to` set to its
  `local_...` id. Never start a new session for a fix to an existing one. "Delivered" or "queued"
  means it arrived, not that it has been acted on; wait for the next note.
- If the user drove a session themselves, read what changed before sending it anything; don't undo
  their steering.
- A session waiting on the user's answer: tell the user in one line, with its link. Don't answer for
  them.
- Find your sessions with `list_sessions` `{"linked": true}`, never a search.
- On a turn started by a side-sessions event you can't start new sessions (except a step the user
  just asked for); propose them instead and let the user start them.

## 5. Review

For each code-writing session, before calling its piece done:
- Start a **review session** on a different model (or a subagent here, if it's small), pointed at the
  author's branch by its full ref (`refs/heads/...`) and the original goal, not the author's report.
- Require reproduction: each finding needs steps or a script. Unreproduced concerns are "possible".
- The reviewer reads and tests; it doesn't fix.
- You triage, then send the confirmed findings to the **author session** with `SendMessage`. The user
  may also read the review and steer the fix themselves.

## 6. Bring it together

Each session's work stays on its own branch. Nothing is in your checkout until it's merged.

- When a step needs the pieces together (testing it all, a final build), tell the user, name each
  branch by its full ref, and **ask whether to merge them here**. Merge only if the user themselves
  says so, not because a session asked. Never push those branches.
- After merging: run all the checks on the merged result and look at the UI yourself. A pass in each
  worktree doesn't count.
- If the user doesn't want them merged, say where each piece is (branch, PR) and leave it.
- Once the user is happy, you may offer to archive the finished sessions, and do it only on their yes.

## 7. Working from the kanban board

When the `kanban` MCP tools resolve, the board shows the sessions' work too:

1. Take the item the user named (or `next_item`, with `include_todo: true` for Todo items) and
   `claim_item` it with the plan.
2. Give each session its **own item**: `create_item` with `Parent: #N` and the brief in the body,
   unassigned. Tell the session to `claim_item` it first and `report_progress` on that item only
   (with `branch` when it has one, `blocked` + question when stuck, `needs_review` when done).
   Every session acts as the same board actor (`KANBAN_AGENT`), so a session must never touch the
   parent item.
3. You own the parent: post a summary as sessions finish; `needs_review` when everything is in.
   Never mark items done; the human approves.

## 8. Report to the user

Lead with the outcome, then:
- **Each session**: link, what it did, its branch or PR, its state (done, waiting on you, running).
- **What you verified yourself** versus what's from session reports.
- **Not done, not verified, caveats.**
- **What needs the user**: merges, pushes, PRs, answers to waiting sessions.
- **Cost**: sessions, models, effort, and reported usage or duration where available
  ("not available" rather than a guess).

The most common failure is a brief that leans on this conversation ("as discussed", "the findings
above"). Every brief must stand on its own, for the session and for the user reading it.
