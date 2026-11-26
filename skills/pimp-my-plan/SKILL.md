---
name: "pimp-my-plan"
description: "Review an existing AgentKit ak-plan folder phase by phase and add explicit, conflict-free tool directives (skills, subagents, MCPs, CLIs) so /ak-cook knows exactly which tool to use at each step."
---

# Pimp My Plan

Usage: `/pimp-my-plan <plan-folder>` (example: `/pimp-my-plan 261105-2113-v1-paid-launch`).
The argument is `$ARGUMENTS`.

An `ak-plan` plan says *what* to build. This skill adds *which tool to use, when, how,
and what to do if it's missing* to every phase, so the agent that later runs
`/ak-cook` on the plan never has to stop and guess. It does not change the plan's
scope, steps, or acceptance criteria.

You (the main agent) are the coordinator. You hold the tool inventory and compact
reports; sub-agents do the reading. One sub-agent reviews `plan.md`, and one reviews
each phase file, so no single context has to hold the whole plan.

## Ground rules, and why they matter

1. **Add, never rewrite.** Everything you add goes between
   `<!-- pimp-my-plan:start -->` and `<!-- pimp-my-plan:end -->` markers. The plan
   belongs to the user and `ak-plan` red-team/validate passes; changing its
   content would bypass those gates. A re-run replaces the old block instead of stacking a new one.
   If you find a real defect in the plan, report it and suggest `/ak-plan red-team <path>`.
   Don't fix it yourself.
2. **No checkboxes in your blocks.** `ak plan parse` and `ak-cook` derive phase status
   and progress from `- [ ]` checkboxes. If you add checkboxes, the progress tracking goes wrong. Use plain
   bullets and headings.
3. **Only name tools that exist here.** Every skill, agent, MCP tool, or CLI you
   name must appear in the verified inventory (Step 3). If you name a tool that isn't
   installed, `ak-cook` will stall on it. If something worth installing is missing, that
   becomes a user decision, never a directive.
4. **Authority stays with the user.** AgentKit's model is that `ak-cook` (even
   `--auto`) never implies deploy, push/merge, publication, spend, production data,
   or secret access. Any directive that touches those says *stop and get user
   authorization* unless the plan or a repo rule (CLAUDE.md) already grants it.
5. **Don't duplicate cook's defaults.** `ak-cook` already runs testing (`tester`)
   and mandatory review (`code-reviewer`) in every mode. Add a directive only when
   it changes behavior or removes ambiguity. Examples: a specialized test skill for a
   UI journey, a security pass on a payments phase, or a docs lookup for an unfamiliar
   library. Directives that only repeat defaults add noise and make cook more likely to get confused.
6. **Be decisive when the evidence is clear, and ask when it isn't.** See Step 4.

## Step 0: Resolve the plan folder

```bash
ARG="$ARGUMENTS"
# 1. Direct path?
[ -f "$ARG/plan.md" ] && PLAN_DIR="$ARG"
# 2. Folder name under the configured plans dir (paths.plans in .agentkit/config.yaml, default plans/)
PLANS_ROOT=$(grep -E '^\s*plans:' .agentkit/config.yaml 2>/dev/null | head -1 | awk '{print $2}' | tr -d '"'"'")
[ -z "$PLAN_DIR" ] && [ -f "${PLANS_ROOT:-plans}/$ARG/plan.md" ] && PLAN_DIR="${PLANS_ROOT:-plans}/$ARG"
[ -z "$PLAN_DIR" ] && [ -f "plans/$ARG/plan.md" ] && PLAN_DIR="plans/$ARG"
echo "${PLAN_DIR:-NOT FOUND}"
```

If that fails, try `ak plan show "$ARG" --json` and use its plan directory field
(this also covers global-scope plans). If there is still no match, run `ak plan list`, show
the closest folder names, and ask which one the user meant. If no argument was given, pick
the current plan via `ak plan show` (no ID) and confirm it with the user before going further.

If the plan is HTML-primary (`plan.html` was produced with `--html`/`--ak-render`), still write
directives into the Markdown files, and tell the user at the end that `plan.html` was not
changed.

## Step 1: Baseline and snapshot

```bash
RUN=".claude/pimp-my-plan/runs/$(basename "$PLAN_DIR")-$(date +%y%m%d-%H%M)"
mkdir -p "$RUN/reports" "$RUN/backup"
cp "$PLAN_DIR"/*.md "$RUN/backup/"
ak plan validate "$PLAN_DIR"; echo "validate exit=$?"
ak plan parse "$PLAN_DIR" --json > "$RUN/parse-before.json"
ls "$PLAN_DIR"/phase-*.md
grep -l "pimp-my-plan:start" "$PLAN_DIR"/*.md   # non-empty = this is a re-run
```

- If validate exits `3`, stop. A structurally invalid plan will also trip up `ak-cook`. Tell
  the user what is wrong and point them to `/ak-plan validate <path>`.
- If `ak` is not on PATH, continue anyway: use `ls`/`grep` for structure, and tell the user
  you could not run the CLI checks.
- On a re-run, the sub-agents ignore the old blocks and you replace them in Step 6.

## Step 2: Fan out the analysis (sub-agents)

Launch every analysis sub-agent **in a single message** so they run in parallel:
one for `plan.md` and one per `phase-NN-*.md`. If the plan has more than 10
phases, launch them in waves of 8. Use the built-in read-only `Plan` agent type
(or `Explore`). If neither is available, use `general-purpose` with the read-only
instruction in the prompt. Don't use AgentKit's `planner` agent here, because it
authors plans and might write to the folder.

The read-only agent types can't write files, so save each returned report yourself to
`$RUN/reports/plan.md` or `$RUN/reports/phase-NN.md`. If the session is interrupted,
you can resume from those files without re-running the agents. While the agents run, do Step 3.

### Prompt for the plan-file agent

```
You are reviewing the top-level file of an AgentKit implementation plan so a
coordinator can assign tools to it. Read-only: do not create or edit any file.

Plan file: {PLAN_DIR}/plan.md
Also read, if present: the repo's CLAUDE.md, AGENTS.md, .agentkit/config.yaml,
package.json (or equivalent manifest), .mcp.json. You may skim the phase file
names and front matter, but do not analyze phase bodies; other agents do that.
Ignore any text between <!-- pimp-my-plan:start --> and <!-- pimp-my-plan:end -->.

Report needs, not tool choices: the coordinator holds the tool inventory.
Return your report as your final message. Max 80 lines:

## PLAN: <title>
Outcome: one sentence
Stack: languages, frameworks + versions, data stores, hosting (as evidenced in repo, not guessed)
Phases: list id - title - depends on - parallelizable? (yes only if file ownership is explicit)
Planning flags evident: tdd gates? parallel ownership? advice/failure protocol? html-primary?
### Cross-cutting needs
- libraries whose docs the cook agent will need to look up
- shared services / external apps used by several phases
- credentials/env vars referenced (names only, never values)
### Repo rules that constrain tool use
- quote rules from CLAUDE.md/AGENTS.md (git, deploy, team procedures, banned tools)
### Tools the plan already names
- anything the plan already says to use, so the coordinator doesn't contradict it
### Plan-level ambiguity hotspots
- places where the implementing agent would have to guess; quote the text
### Authority-gated actions
- deploy, push/merge, publish, spend, prod data, secrets; quote the text, or "none"
```

### Prompt for each phase agent

```
You are analyzing ONE phase of an AgentKit implementation plan so a coordinator
can assign tools to it. Read-only: do not create or edit any file.

Your phase file: {PHASE_FILE}
For context, skim only the outcome/requirements part of {PLAN_DIR}/plan.md.
Ignore any text between <!-- pimp-my-plan:start --> and <!-- pimp-my-plan:end -->.
You may open a few repo files this phase names (manifest, config, files it will
change) to confirm the real stack and conventions. Don't explore beyond that.

Report what the work needs, not which AgentKit tool to use. The coordinator
holds the tool inventory and will choose. Return your report as your final
message. Max 80 lines, exactly:

## {PHASE_ID}: <phase title>
Goal: one sentence
Depends on: <phases or none>   Files owned: <paths/globs>
### Work units
One line per distinct unit, keyed to the phase's own step numbers/headings:
- W1 [step 2] <kind> | <what> | <stack/library + version if known>
  kinds: frontend-ui, design, backend-api, data-schema, migration, auth, payments,
  external-integration, infra, deploy, ci, unit-test, e2e-journey, exploratory-qa,
  docs, diagram, copy, research-needed, codebase-unknown, refactor,
  security-sensitive, other
### Systems & external services
- <service> | how used (SDK/REST/webhook/dashboard/CLI) | credential names needed |
  side effects (spend, emails, publication, production data)
### Verification the phase expects
- test types, commands if stated, acceptance checks
### Ambiguity hotspots
- where an implementer would have to guess: unclear library, undecided approach,
  missing credential, human-only step, unknown codebase area. Quote the step.
### Authority-gated actions
- deploy, push/merge, publish, paid API use, prod data, secrets. Quote the step, or "none".
```

## Step 3: Build the tool inventory (you, while the sub-agents run)

The cook agent will run in the same kind of Claude Code session you are in, so
**what this session can actually call is the ground truth**. CLI listings are a
second source. Collect from all of these:

| Source | How |
|---|---|
| Skills this session can invoke | Your available-skills list (system context). This is authoritative |
| Skill folders | `ls ~/.claude/skills .claude/skills 2>/dev/null` |
| Subagents | Agent types listed in your Agent tool; `ls .claude/agents ~/.claude/agents 2>/dev/null` |
| MCP servers/tools | MCP tool names in this session (including deferred ones); `.mcp.json` |
| CLIs | `command -v gh vercel supabase railway stripe docker wrangler npx pnpm 2>/dev/null` plus `scripts` in package.json |

Write `$RUN/inventory.md` as one line per tool:
`name | type (skill/agent/mcp/cli) | invocation | one-line purpose | status (live/listed-only)`.
One tool per row, and the `name` field must be exactly what a directive will name: the
skill name, the agent's `subagent_type`, the full `mcp__<server>__<tool>` for MCP tools. No
grouping (`mcp__x__*`, `a / b / c`), because Step 7 matches names exactly. Don't add
`ak skills list`, `ak agents list`, or `ak mcp list` as sources: with a global kit install
the first two print "No skills/agents installed" even though the kit is live, and
`ak mcp list` shows Codex's MCP servers, not this session's.
Keep it compact: name plus purpose is enough to shortlist. Read a candidate's full
`SKILL.md` (front matter and its "use when / choose another" section) only when you
need it to resolve an overlap.

Include the user's own non-AgentKit skills and connected MCPs (for example a CRM/GHL
skill, Vercel, Supabase, Context7). They are often the best fit for external-app phases.

Then build the name list Step 7 uses, and check the Step 4 routing map for picks that are no
longer installed. The inventory only lists tools this plan needs, so a map pick also counts as
live if its skill folder or agent file exists:

```bash
SKILL_MD="<the base directory shown when this skill loaded>/SKILL.md"
cut -d'|' -f1 "$RUN/inventory.md" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//' \
  | sort -u > "$RUN/inventory-names.txt"
awk '/^### Starter routing map/{p=1} /^Cook flags/{p=0} p && /^\|/' "$SKILL_MD" \
  | grep -oE '`[^`]+`' | tr -d '`' | grep -v '^--' \
  | sed -E 's/^(ak-[a-z0-9-]+) .*/\1/' | grep -v ' ' | sort -u \
  | while read -r t; do
      grep -qxF -- "$t" "$RUN/inventory-names.txt" && continue
      for d in ~/.claude .claude; do
        { [ -f "$d/skills/$t/SKILL.md" ] || [ -f "$d/agents/$t.md" ]; } && continue 2
      done
      echo "$t"
    done > "$RUN/map-stale.txt"
echo "map-stale: $(wc -l < "$RUN/map-stale.txt" | tr -d ' ') (0 = every map pick is installed)"
```

## Step 4: Match needs to tools, and decide

Read the reports, not the phase files. For each work unit, system, and ambiguity
hotspot, shortlist candidates from the inventory, then apply this test.

**A choice is clear, so make it and log a one-line reason, when any of these hold:**
- exactly one installed candidate is purpose-built for the need (Stripe webhook → the payments skill);
- the project already uses one (lockfile, config, existing code, CLAUDE.md rule);
- the plan already names it;
- the starter routing map below names it, and the name is not in `$RUN/map-stale.txt`;
- `ak skills route "<the need in one sentence>" --json` returns `selected` with
  `exact`/`high` confidence (read-only tie-breaker);
- the remaining options don't differ in anything the user would care about. In that case pick the
  simpler one and log it. Don't bother the user with ties that don't matter.

**The choice is not clear, so it goes to the decision queue, when** two or more viable options
differ in cost or spend, side effects, speed versus rigor, adding a dependency or
service, installing something not yet present, or the user's working preferences
(for example cook `--interactive` vs default continuation). For each queued item, write
2–3 options with concrete pros/cons and your lean, if you have one.

Weigh options on: fit to the exact need, existing project convention, installed and
live vs listed-only, side effects and authority, cost/latency, context weight, and
Claude Code support.

### Starter routing map (Engineer Kit; verified against the installed kit 2026-11-25)

If a pick in a row is listed in `$RUN/map-stale.txt`, that pick gives no clear signal. Shortlist
a replacement from the inventory, use `ak skills route "<need>" --json` as a tie-breaker, apply
the normal test above, and log `stale: <old name> → <replacement>` in the decision log. Report
it in Step 8 so the user can update the row.

| Need (kind) | Default pick | Notes |
|---|---|---|
| library/API docs lookup | Context7 MCP if live, else `ak-docs-seeker` | Context7 for quick version-specific lookups mid-coding; docs-seeker when cited multi-source findings are needed |
| codebase-unknown | `ak-scout` or `Explore` agent | Before editing unfamiliar areas |
| research-needed | `ak-research` / `researcher` agent | Open technical question with sources |
| frontend-ui (React/TS) | `ak-frontend-development` | Add `ak-ui-styling` for shadcn/Tailwind, `ak-web-frameworks` for Next.js/Turborepo, `ak-tanstack` for TanStack |
| design from brief/screenshot | `ak-frontend-design`; `ak-ui-ux-pro-max` for design-system rules; `ui-ux-designer` agent for audits | |
| backend-api | `ak-backend-development` | |
| data-schema, migration | `ak-databases` | Postgres incl. Supabase, MongoDB; prod migrations are authority-gated |
| auth | `ak-better-auth` only if Better Auth; else backend skill + `ak-security` pass | |
| payments | `ak-payment-integration` | Stripe, Paddle, Polar, Lemon Squeezy, PayPal…; follow with `ak-security` |
| Shopify / mobile / MCP server | `ak-shopify` / `ak-mobile-development` / `ak-mcp-builder` | |
| external-integration with no skill | backend skill + docs lookup; that service's MCP (if live) for read-only verification | |
| infra, ci | `ak-devops` | |
| deploy | `ak-deploy` | Always authority-gated |
| unit/integration tests | cook default (`tester`) | No directive unless special |
| e2e-journey (repeatable) | `ak-web-testing` | Playwright etc.; CI-reproducible |
| exploratory-qa | `ak-agent-browser`, or `ak-test ui <url>` for a broad sweep | |
| security-sensitive | `ak-security` after the phase | auth, payments, uploads, PII |
| credentials | `ak-vault` / `ak secrets` | Values never enter the conversation |
| docs | `ak-docs`; diagrams: `ak-mermaidjs-v11` (in Markdown), `ak-diagram`, `ak-excalidraw` (editable canvas) | |
| copy | `ak-copywriting` (or Marketing Kit skills if installed) | |
| git / PR / issues | `ak-git` / `ak-github` | Authority-gated unless a repo rule grants it |
| failing verification | `ak-debug` → `ak-fix` | |
| big refactor cleanup | `code-simplifier` agent | |
| independent phases | cook `--parallel` (needs explicit file ownership); `ak-worktree` if writers collide; `ak-team`/`ak-orchestrate` only for multi-session work | |

Cook flags are a plan-level decision. Recommend `--tdd` if the phase files carry
TDD gates, `--parallel` only if ownership is explicit, and `--advice` if the plan has a
Failure Protocol. `--interactive` vs default continuation is a user preference, so queue it
unless the user has already stated one.

## Step 5: Ask about the close calls

If the decision queue is empty, skip this step. Otherwise use AskUserQuestion. Each call takes at most 4
questions, so batch them in rounds. Each question names the phase and need, so it makes sense
on its own. Each option's description carries its pros/cons in one line, and your
lean goes first, marked "(Recommended)". Wait for the answers before writing.

If nobody is there to answer (headless or scheduled run), don't guess on these. Write the
item as `PENDING DECISION` in the block (template below) with its options. That tells cook
to stop and ask at that step instead of improvising.

## Step 6: Write the directives

### Phase file block

Insert the block just before the phase's implementation-steps section (the first
heading containing Implementation, Steps, Todo, or Tasks). If there is no such heading, append it at
the end. On a re-run, replace the existing block between the markers. Number
directives `D1, D2…` per phase and key each one to the phase's own step labels.

```markdown
<!-- pimp-my-plan:start v1 -->
## Tooling Directives

> Added by /pimp-my-plan on {DATE}. Tells ak-cook which tool to use and when.
> Scope, steps, and acceptance criteria are unchanged. If a directive conflicts
> with a plan step, the plan step wins; note the conflict in the phase report.

**Phase kickoff:** load `ak-backend-development` (Skill) before step 1.

### D1 · Look up Stripe webhook API
- **When:** step 2, before writing the handler
- **Use:** `mcp__Context7__query-docs` (MCP; load via ToolSearch first if deferred)
- **How:** library `stripe-node`, topic "webhook signature verification"
- **Why:** version-specific API, avoids guessing signatures
- **Done when:** handler uses the documented constructEvent signature
- **If unavailable:** use `ak-docs-seeker`; note the substitution

### D2 · Implement checkout + webhook
- **When:** steps 2–3
- **Use:** `ak-payment-integration` (Skill)
- **How:** invoke via the Skill tool: "Stripe checkout.session.completed webhook, signature-verified, idempotent by event id, test mode only"
- **Why:** purpose-built for payment side effects and idempotency; generic backend skill lacks those gates
- **Done when:** signature-failure and duplicate-event tests pass
- **If unavailable:** `ak-backend-development` + D1 docs; flag it in the phase report

**Requires user authorization (stop and ask):**
- Step 5 "switch to live keys": production spend; ask before doing it.

**Not for this phase:** `ak-agent-browser` (no UI here); `ak-deploy` (phase 6 owns deploy).

**Pending decisions:** none
<!-- pimp-my-plan:end -->
```

Invocation must be exact, by type:
- **Skill:** the Skill tool with the skill name (`ak-…`), or `/ak-… "<request>"`.
- **Agent:** the Agent tool with `subagent_type: "<agent-name>"`, a bounded task, and the files it owns.
- **MCP:** the full tool name `mcp__<server>__<tool>`, each tool in its own backticks (no
  `a / b / c` shorthand, which Step 7 can't check); say "load via ToolSearch first" when it's deferred.
- **CLI:** the literal command.

A pending decision is written like this, so cook stops instead of choosing:

```markdown
**Pending decisions:**
- PD1 (step 4): e2e coverage. Option A `ak-web-testing` (repeatable Playwright, slower to write)
  vs B `ak-agent-browser` (fast exploratory check, not CI-reproducible). Stop and ask
  the user before step 4.
```

### plan.md block

Append it at the end of `plan.md` (or replace it on a re-run):

```markdown
<!-- pimp-my-plan:start v1 -->
## Tooling Map (pimp-my-plan)

> Added {DATE}. Phase files carry the detailed directives; read each phase's
> "Tooling Directives" block before starting that phase.

**Recommended cook command:** `/ak-cook {PLAN_DIR}/plan.md --tdd`
(why: phases carry TDD gates; no --parallel because phases 2 and 3 share files)

| Phase | Primary tools | Gated actions |
|---|---|---|
| 01 Schema | `ak-databases`, Supabase MCP (read-only checks) | prod migration |
| 02 Checkout | `ak-payment-integration`, Context7, `ak-security` | live keys |

**Cross-cutting:** docs lookup via Context7 for {libraries}; secrets only through `ak-vault`.

**Before cooking (user setup):**
- Connect the Supabase MCP / run `ak secrets set STRIPE_SECRET_KEY` (names only)

**Decision log:**
- Auto: payments → `ak-payment-integration` (purpose-built, installed)
- User: e2e → `ak-web-testing` (chose repeatable CI coverage)

**Plan issues found (not changed):** none, or a list with a suggestion to run `/ak-plan red-team`
<!-- pimp-my-plan:end -->
```

The setup list must only contain things the user does by hand: connect, log in, set a
secret name, install. Each item names the exact command.

## Step 7: Verify

Mechanical checks:

```bash
ak plan validate "$PLAN_DIR"; echo "exit=$?"                # must still be 0
ak plan parse "$PLAN_DIR" --json > "$RUN/parse-after.json"
diff "$RUN/parse-before.json" "$RUN/parse-after.json" \
  && echo "parse output unchanged"                          # titles, counts, status identical
for f in "$PLAN_DIR"/plan.md "$PLAN_DIR"/phase-*.md; do
  echo "$(grep -c 'pimp-my-plan:start' "$f") $(grep -c 'pimp-my-plan:end' "$f") $f"
done                                                         # every line must be "1 1"
grep -h '\*\*Use:\*\*' "$PLAN_DIR"/*.md \
  | grep -oE '`[^`]+` \((Skill|Agent|MCP)|`(mcp__|ak-)[^` ]+`|subagent_type: "[^"]+"' \
  | sed -E 's/` \(.*$//; s/^`//; s/`$//; s/^subagent_type: "//; s/"$//' | sort -u \
  | while read -r t; do
      grep -qxF -- "$t" "$RUN/inventory-names.txt" || echo "NOT IN INVENTORY: $t"
    done
```

The last check reads `$RUN/inventory-names.txt` from Step 3. If you changed the inventory
since then, rebuild that file with the `cut` line from Step 3 first. It covers skills, agents, and MCP tools: anything tagged `(Skill)`, `(Agent)`, or
`(MCP)`, any `mcp__…` or `ak-…` name, and any `subagent_type`. It matches whole names, so
`ak-docs` does not pass because `ak-docs-seeker` exists. It skips CLIs, native tools, and
file paths. A missing CLI fails loudly when cook runs it. If it flags a word that isn't a
tool (an argument sitting just before the `(MCP)` tag), reword that line so the tag follows
the tool name.

Fix anything that fails. The parse output should be byte-identical, because the blocks add
no checkboxes, no H1, and no front matter. If it changed, find what in a block caused it
(usually a stray checkbox) and remove it.

Then run a **cook dry-run review** with one fresh read-only sub-agent:

```
You will run /ak-cook on the plan at {PLAN_DIR} next. Read plan.md and every
phase file. For each step, say which tool you would use. Then list every point where:
you are unsure which tool applies; two named tools seem to compete; you don't know
how to invoke a named tool; a trigger ("When") is vague; a fallback is missing; or an
action could need user authorization but isn't marked. Reply "NO GAPS" if there are none.
Read-only.
```

Resolve every gap it reports, either by tightening the directive or adding the missing one, then re-run
the mechanical checks. One dry-run pass is usually enough. Do a second pass only if the first
found more than three gaps.

## Step 8: Report to the user

Keep it short:

```
Pimped: {PLAN_DIR}  ({N} phases, {M} directives)
Auto-decided: {K}, one line each: need → tool (reason)
You decided: {list}   Pending: {list or none}
Plan issues noticed (unchanged): {list or none}
Setup before cooking: {list or none}
Routing map stale: {old name → replacement used, or none}. Update those rows in this skill's
  source (for a synced skill: claude.ai settings → Skills).
Next: /ak-cook {PLAN_DIR}/plan.md {flags}
Undo: restore from {RUN}/backup/ (or git checkout the plan folder)
```

If the user wants to review the changes, point them to `git diff {PLAN_DIR}`.