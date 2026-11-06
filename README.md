# Agent Coordinator

Two Claude Code skills that make Claude act as a **coordinator**. Claude plans the work, ranks
each piece, and hands the pieces to the right agents. It gets the results reviewed by a different
model, checks them itself, and reports back with proof and cost.

| Skill | Workers | You can watch and steer | Results |
| --- | --- | --- | --- |
| `coordinator` | Background subagents (and external agent CLIs) | No; you get milestone updates with proof | Claude reviews, merges and shows you the finished work |
| `session-coordinator` | Separate Claude Code sessions (and external CLIs in Terminal tabs) | Yes; open any session and redirect it | Each piece lands on its own branch, merged when you say so |

## What they do

- **Rank every task.** Each piece gets a 0-10 score on five questions: how clear the spec is, how
  risky a mistake would be, how big it is, how new it is to the codebase, and how hard it is to
  prove it works. The score gives a tier (T1-T4), and the tier picks the model, effort level and
  how much review the piece gets.
- **Write self-contained briefs.** Agents start cold, so every brief has the same parts: goal,
  context, decisions already made, limits, a "done when" line, the checks to run, milestones, and
  what to report.
- **Review with a different model**, which has to reproduce each bug it reports. Fixes go back to
  the agent that wrote the code.
- **Milestone updates with proof** while agents work: check output, screenshots, or a URL the
  coordinator opens itself.
- **Verify before reporting.** Claude re-runs the checks on the merged result, looks at the UI,
  and reads the diff for anything outside the brief.
- **Keep judgment calls and outward-facing steps with you.** Pushing, publishing and merging all
  wait for your yes.

### External agents

Ask for other agents by name ("use opencode with Mistral", "have Gemma review it"). Claude then
finds the tool, learns how to run it from its `--help`, looks up the exact model name, and checks
it with a one-line test prompt. Then it delegates, giving each writer its own git worktree.

Which agents it uses depends on what you say:

| You say | Claude uses |
| --- | --- |
| Nothing about other agents | Claude agents only |
| "Use X" | Only X for the work. Claude still coordinates and verifies |
| "Use X, and add yours too" | X plus one Claude agent |
| "Only X" | X for everything, including review |

Hosted models get your code, so the plan names the provider for each task. Local models keep your
code on your machine.

### Kanban board (optional)

The skills can work with an [Open Agentic Kanban](https://github.com/takasurazeem/open-agentic-kanban)
board from any repo:
- Each board project lists the repos it belongs to.
- In a repo, the coordinator looks up that repo's project from its git remote. If none is linked,
  it asks whether to create one or link the repo to an existing project.
- It takes work from that project's queue, gives each session its own item, and posts progress.
- It never marks items Done; you approve them.

To make the board available in every repo, register its MCP server for your user rather than one
project:

```bash
claude mcp add --scope user kanban -e KANBAN_URL=http://localhost:5173 -e KANBAN_AGENT=claude -- node /path/to/open-agentic-kanban/server/mcp.mjs
```

## Install

As a plugin:

```text
/plugin marketplace add takasurazeem/agent-coordinator
/plugin install agent-coordinator@agent-coordinator
```

Plugin skills are namespaced: `/agent-coordinator:coordinator` and
`/agent-coordinator:session-coordinator`. Asking Claude to "be a coordinator" or to "run this in
sessions I can steer" also triggers them.

Or copy the skills in, for the short names `/coordinator` and `/session-coordinator`:

```bash
git clone https://github.com/takasurazeem/agent-coordinator
cp -R agent-coordinator/skills/coordinator agent-coordinator/skills/session-coordinator ~/.claude/skills/
```

Keep both folders side by side: `session-coordinator` reads `../coordinator/routing.md`.

## Files

```
skills/coordinator/SKILL.md          background-agent coordinator
skills/coordinator/routing.md        task ranking, agent choice, external harnesses (shared)
skills/session-coordinator/SKILL.md  session-based coordinator
```

## License

MIT
