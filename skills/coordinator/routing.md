# Routing: rank each task, pick who runs it

Shared by `/coordinator` and `/session-coordinator`. Read this before dispatching anything.

## 1. Rank every task

Score each task 0-2 on five questions and add them up (0-10):

| Question | 0 | 1 | 2 |
| --- | --- | --- | --- |
| **Ambiguity**: is the spec clear? | Exact steps | Goal clear, approach open | Goal itself fuzzy, needs design |
| **Risk**: what breaks if it's wrong? | Nothing (read-only, scratch) | Code that tests catch | User data, security, concurrency, money, public output |
| **Size**: how much changes? | One file or a lookup | A few files | Many files or a new subsystem |
| **Novelty**: has this codebase done it before? | Pattern exists to copy | Partly | New territory |
| **Verification**: how hard is it to prove it works? | One command | Tests plus a manual check | Needs reproduction, races, real devices |

Score to tier:

| Score | Tier | Claude model | Effort | Review |
| --- | --- | --- | --- | --- |
| 0-2 | **T1** trivial | `haiku` (`Explore` for read-only) | low | None. You spot-check |
| 3-4 | **T2** routine | `sonnet` | medium | Only if it touches data or security |
| 5-7 | **T3** substantial | `sonnet`, or `opus` if Ambiguity is 2 | high | Independent, different model |
| 8-10 | **T4** critical | `opus` | high (max only if the user agrees) | Independent, different model, with reproduction |

Rules:
- **Risk 2 sets a floor of T3**, whatever the total.
- Use `fable` only when the user asks for it.
- In the plan, show each task as `T3 (7: A2 R2 S1 N1 V1) -> opus/high`, so the user can disagree
  with the ranking rather than the model.
- In a session, `model` and `effort` can't exceed yours; a higher value is ignored, so cap them.
- Re-rank after a review that found real bugs: the task was harder than scored.

## 2. Which agents may you use?

The user's words decide. Read their request for harnesses or models they named.

| The user said | You use |
| --- | --- |
| Nothing about agents | Claude agents only |
| Named external ones ("use Muse Spark and Gemma") **plus** "you too" / "add your agent" / "and Claude" | The named ones **plus one** Claude agent, routed by tier |
| Named external ones and nothing about Claude | **Only the named ones** for the work. Don't add Claude agents. You still coordinate, rank, brief, verify and report |
| "Only X" / "don't use Claude" | Only X, even for review (a second X model if there is one; otherwise you review, and say so) |
| A harness without a model ("use opencode") | That harness; pick its model by tier (section 4) |

If a named model or harness isn't found (section 3), say so in one line, and ask before you
substitute anything. Never quietly fall back to Claude when the user excluded it.

## 3. Learn the harness the user named

An external agent is any agent CLI or local model the user asks for: a coding harness (opencode,
pi, Gemini CLI, Qwen Code, Codex, Aider, Goose, ...) or a model reached through one (a hosted model,
a self-hosted or local one served by Ollama, LM Studio, llama.cpp, vLLM, ...). Don't assume which
ones exist on this machine, and don't assume their flags. Learn each one once per conversation.

1. **Is it installed?** `command -v <cli>`. If it isn't, tell the user. Don't install it unless they
   ask.
2. **Read its help.** Run `<cli> --help`, plus `<cli> <subcommand> --help` for the run or prompt
   subcommand. Write down six things:

   | You need | Typical spellings |
   | --- | --- |
   | Non-interactive run (prompt in, answer out, exit) | `run <msg>`, `-p/--prompt`, `--print`, `exec`, a positional prompt |
   | Model choice | `-m/--model provider/model` |
   | Its model list | `models`, `--list-models`, `ls`, or the provider's own list |
   | Permission mode | `--auto`, `--yolo`, `--approval-mode`, a plan or read-only mode, tool allow-lists |
   | Prompt from a file | `-f/--file`, `@file`, stdin |
   | Resume a run for fixes | `-s/--session`, `--continue`, `--resume` |

   Also note a machine-readable output mode if it has one (`--format json`, `--mode json`).
3. **Resolve the model name.** If the user said a model ("Muse Spark", "Mistral", "Gemma"), find its
   exact id in the harness's model list. Match loosely, then confirm. If several match, pick the
   newest or ask. If none match, ask rather than substituting.
4. **Ping it.** Send a one-line prompt ("Reply with exactly the word: pong") with the model you'll
   use, and judge by the **output**, not the exit code. Some harnesses exit 0 on a connection error
   or an expired key.
5. **If the ping fails**, tell the user exactly what failed ("<harness>: API key expired",
   "<model>: connection error, its server isn't running") and ask how to proceed.
   - Never fix credentials, rotate keys, or edit the harness's config yourself.
   - Never start, install or download model servers without the user's yes.

Local and self-hosted models answer only while their server is up.

## 4. Picking an external model by tier

You can't measure these models, so don't invent rankings.
- Use the user's preference if they gave one.
- Otherwise go by the provider's own naming:
  - T1-T2: the `flash`, `mini`, `lite` or `free` variants, or local models.
  - T3-T4: the `pro`, `max` or `large` variants.
- Reviews still need a different model from the author's.
- **Data leaves the machine** with any hosted model. Local models keep it on the machine. Name the
  provider for each task in the plan, so the user sees where their code goes. Never send secrets,
  `.env` files or private data to a hosted model.

## 5. Running external agents

External harnesses run through Bash, not the `Agent` tool:
- They can't use `SendMessage`.
- They don't see this conversation.
- Their permissions are their own, so be careful.

1. **Isolate.** Each writer gets its own git worktree, created with
   `git worktree add <scratchpad>/wt/<task> -b coord/<task>`, and runs with that as its cwd. It also
   gets its own scratch data and port if the project has them. Read-only tasks can run in the
   checkout, in the harness's read-only or plan mode if it has one.
2. **Brief.** Use the same brief template as for Claude agents, written to
   `<scratchpad>/proof/<task>/brief.md` and passed with the harness's prompt-from-file option.
3. **Auto-approve only inside an isolated worktree with scratch data.** Use the harness's mode that
   auto-approves edits, not its approve-everything mode unless the user asks. Never auto-approve in
   the user's checkout.
4. **Run in the background** and capture all output:

   ```bash
   cd <worktree> && <cli> <run-flag> --model <id> <prompt-from-file brief.md> > <proof>/<task>/out.log 2>&1
   ```

   Use `run_in_background: true` in `/coordinator`. In `/session-coordinator`, run it in a Terminal
   tab instead (`mcp__terminal__open_terminal_tab` / `run_in_terminal`) so the user can watch and
   type into it, and link the tab.
5. **Milestones.** These agents can't message you. The brief tells them to append milestone lines
   to `<proof>/<task>/progress.md` (time, what's done, proof paths), and to finish with a `DONE` or
   `BLOCKED: <question>` line. Read that file when you check in. Don't poll in a tight loop; if you
   need to wait, use `Monitor` with an until-condition on `DONE|BLOCKED`.
6. **Review and merge** exactly as for Claude agents:
   - A different model reviews, and it must reproduce each finding.
   - Fixes go back to the same harness and model, resuming its run to keep its context.
   - Checks run on the merged result.
   - Afterwards remove the worktrees you created (`git worktree remove`), and keep the branches until
     the user is done with them.
