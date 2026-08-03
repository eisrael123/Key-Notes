# Claude Code CLI cheat sheet

A crash course for using Claude Code from the terminal panel inside an IDE, written for a
first-time user. Covers basic input, slash commands, plan mode, subagents, forking/rewinding a
conversation, context management, and autonomous/background work.

## Basic input

| Action | Keystroke |
|---|---|
| Submit a message | `Enter` |
| Newline without submitting | `Shift+Enter` (or `\` then `Enter` in some terminals) |
| Cycle through your previous prompts | `Up` / `Down` arrow (at an empty/start-of-line prompt) |
| Interrupt Claude mid-response or mid-tool-call | `Esc` |
| Rewind — jump back to an earlier point in the conversation and continue from there | `Esc` `Esc` (double-tap) |
| Cancel current input line | `Ctrl+C` |
| Exit the session | `Ctrl+D`, or `Ctrl+C` twice, or type `/exit` |
| Reference a file for context | `@` then start typing a filename — fuzzy autocomplete pops up |
| Open slash-command menu | `/` at the start of the line — arrow keys to navigate, `Enter` to pick |
| Cycle permission mode (normal → auto-accept edits → plan mode) | `Shift+Tab` |

## Slash commands worth knowing early

- `/help` — the real reference, more authoritative than any cheat sheet
- `/clear` — wipes the conversation and starts fresh (context reset, not project memory)
- `/compact` — manually summarizes the conversation to free up context space instead of waiting for it to auto-trigger
- `/resume` — pick up a previous session where you left off
- `/init` — scans the codebase and writes/updates `CLAUDE.md` (rerun after big structural changes)
- `/model` — switch which model you're talking to
- `/agents` — see/manage subagents
- `/permissions` — control what Claude's allowed to do without asking
- `/add-dir` — give Claude access to another directory beyond the project root

## Plan Mode

A read-only mode: Claude can look around (search, read files) but can't edit anything or run
mutating commands. It ends by presenting a plan, which you approve, reject, or send back for
revision. Good for anything risky or nontrivial where you want to see the approach before code
gets touched. Enter with `Shift+Tab` (cycles into it), or just ask in plain English.

## Subagents

A subagent is a separate Claude instance delegated a specific task — searching, reading,
multi-step investigation — so the legwork doesn't fill up the main conversation. It reports back
a summary; the raw work isn't shown unless asked. Mostly invoked automatically when a task is a
good fit (e.g. "find every place X is computed" gets handed to a fast read-only search agent),
but can be requested explicitly too. `/agents` shows what's available/configured.

Mental model: main conversation = you and Claude talking and deciding things; subagent = an
errand sent out with a report that comes back.

## Forking a conversation

The rewind feature (`Esc` `Esc`) is what enables this: jump back to an earlier message, and if a
different message is sent from there, that's effectively a branch — the original path isn't
destroyed, a new one just starts from that checkpoint. `/resume` gets back to earlier
sessions/branches later.

## Context

"Context" = everything in the current conversation window Claude can see and reason over.

1. **It fills up.** As it nears the limit, older messages get auto-summarized behind the scenes
   to keep going. `/compact` does this manually; `/clear` nukes it entirely for a clean slate.
2. **`CLAUDE.md` is not context, it's memory.** A project's `CLAUDE.md` (and Claude's own
   separate memory files) get reloaded fresh every session regardless of context resets — the
   right place for durable facts about the project (conventions, gotchas, "always do X") versus
   one-off details of a single task.

## Autonomous / background work

- **Auto-accept edit mode** (`Shift+Tab` once) — Claude stops asking permission for file edits,
  still asks for riskier stuff.
- **Background tasks** — long-running jobs (a subagent, or a shell command) can run in the
  background while other work continues in the foreground; notification arrives on completion.
- **`/loop`** — repeat a task on an interval or self-paced (e.g. "check on this pipeline run
  every 10 minutes").
- **Scheduled agents** — cron-style, for work that should run on its own on a schedule,
  independent of an open session.

## Notes specific to this project

- `CLAUDE.md` in this repo already has real content (conventions, directory roots, schema
  notes) — keep it updated as decisions get made, since that's what saves future sessions from
  relearning already-established context.
- For "where in the codebase does X happen" questions on a large file like `rnaseq.py`, hand it
  to a search subagent rather than grepping live — faster, keeps the main window clean.
- Before anything that touches the pipeline's output schema or triggers an expensive rerun, Plan
  Mode is worth the extra step — cheap insurance against a wrong edit costing a multi-hour rerun.
