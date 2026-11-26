# pimp-my-plan

Turns a plan that says *what* to build into one that also says *which tool to use* at
every step.

## Why it exists

An AgentKit `ak-plan` plan describes phases,
steps, and acceptance criteria. When `ak-cook` later builds it, the agent still has to pick
tools on the fly from a catalog of a hundred or more skills, subagents, MCP tools, and
CLIs. It guesses, picks two that compete, calls a tool that isn't installed, or does
something that needs your sign-off without asking.

`pimp-my-plan` makes those choices up front, from the tools your session can actually
call, and writes them into the plan. You decide the close calls once, before any code is
written.

## What it does

1. **Finds the plan** by path or folder name, snapshots every file to a backup, and runs
   `ak plan validate` and `ak plan parse` as a baseline.
2. **Reads the plan in parallel.** One read-only subagent per phase, plus one for
   `plan.md`, reports what each step *needs* (frontend UI, payments, a migration, an e2e
   test, docs lookup for a library…), not which tool to use. No single context has to
   hold the whole plan.
3. **Builds a live tool inventory** from what the current session can call: skills,
   subagent types, MCP tools (including deferred ones), and CLIs on your PATH. It then
   checks its own built-in routing map against what's installed and flags entries that
   have gone stale.
4. **Matches needs to tools.** Clear choices are made and logged with a one-line reason:
   one purpose-built tool, a tool the project already uses, a tool the plan names, or an
   explicit routing rule. Real trade-offs (cost, side effects, speed vs. rigor, a new
   dependency) go into a decision queue.
5. **Asks you about the close calls** with the options, pros and cons, and a
   recommendation. Headless runs mark them `PENDING DECISION` so `ak-cook` stops there
   instead of improvising.
6. **Writes a "Tooling Directives" block** into each phase file and a "Tooling Map" into
   `plan.md`. Each directive says when, which tool, how to invoke it exactly, why, what
   done looks like, and the fallback if the tool is missing. Deploys, pushes, spend, and
   production data are marked "stop and get user authorization".
7. **Verifies its own work.** It checks that `ak plan parse` output is unchanged, that the
   markers are balanced, and that every tool it named is in the inventory (whole-name
   match). It then runs a fresh "cook dry-run" subagent that hunts for vague triggers,
   competing tools, and missing fallbacks.
8. **Reports** what it decided, what you decided, setup to do before cooking, and the
   exact `/ak-cook` command to run next.

## Guarantees

- **Add, never rewrite.** Everything goes between `<!-- pimp-my-plan:start -->` and
  `<!-- pimp-my-plan:end -->` markers. Scope, steps, and acceptance criteria are unchanged,
  and a re-run replaces its old blocks instead of stacking new ones.
- **No checkboxes** in its blocks, so AgentKit's progress tracking stays correct.
- **Only installed tools** are named. Anything worth installing becomes a question for
  you, never a directive.
- **Full undo**: every run keeps a backup in `.claude/pimp-my-plan/runs/<plan>-<time>/`.

## Requirements

- Claude Code
- AgentKit Engineer Kit: the `ak` CLI and the `ak-plan` / `ak-cook` skills

## Usage

```text
/pimp-my-plan 261105-2113-v1-paid-launch
/pimp-my-plan plans/261105-2113-v1-paid-launch
```

The intended flow is back to back: `/ak-plan` → `/pimp-my-plan` → `/ak-cook`.

## Customizing

The "Starter routing map" table in `SKILL.md` holds the default pick for each kind of need,
and the rules for choosing between overlapping tools. Edit it to match your stack. If a
pick stops being installed, the skill reports it in its final summary and chooses a
replacement from your live inventory, so the table tells you when it needs updating.
