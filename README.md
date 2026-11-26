# Claude Skills

Free, ready-to-use skills for [Claude Code](https://docs.claude.com/en/docs/claude-code/overview)
and [claude.ai](https://claude.ai). Each one came out of real client and engineering work,
and has been fixed after it went wrong in practice.

A skill is a folder with a `SKILL.md` file. It teaches Claude how to do one job well: what to
do, in what order, what to avoid, and when to stop and ask. Install it once and call it by
name, for example `/pimp-my-plan`.

More skills will be added over time. Watch or star the repo to see new ones.

## The skills

| Skill | What it does | Needs |
|---|---|---|
| [pimp-my-plan](skills/pimp-my-plan/) | Reviews an implementation plan phase by phase and writes down exactly which skill, subagent, MCP tool, or CLI to use at each step, so the agent that builds it never has to guess. | Claude Code, [AgentKit](https://agentkit.best) (`ak` CLI, `ak-plan`, `ak-cook`) |
| [zoom-gather-analytics](skills/zoom-gather-analytics/) | On a Zoom call where a client has granted Remote Control, drives the client's own browser to export their analytics (CRM, LinkedIn, Facebook, Instagram, TikTok, Google Business Profile, Search Console, GA4, newsletter) and uploads each file to your upload survey. | [Claude in Chrome](https://claude.com/chrome), Zoom web client, an upload survey of your own |

Each skill's folder has its own README with the full details: purpose, how it works,
setup, usage, and safety rules.

## Install

### Claude Code

Copy a skill folder into your skills directory. User level makes it available in every
project; project level keeps it to one repository.

```bash
git clone https://github.com/TheOnlyDustin/claude-skills.git
cp -R claude-skills/skills/pimp-my-plan ~/.claude/skills/        # every project
# or
cp -R claude-skills/skills/pimp-my-plan your-project/.claude/skills/   # one project
```

Start a new Claude Code session, then run `/pimp-my-plan`. Claude also picks a skill
on its own when your request matches the skill's description.

### claude.ai (and the Claude in Chrome extension)

Zip the skill folder, so the zip holds the folder itself rather than only its contents:

```bash
cd claude-skills/skills && zip -r zoom-gather-analytics.zip zoom-gather-analytics
```

Then upload it in **Settings → Capabilities → Skills**. Skills uploaded to your account
also sync to Claude Code on the computers you sign in from.

## Customizing

Skills are plain Markdown. Open `SKILL.md`, change what doesn't fit how you work, and
reinstall. A skill's README lists anything you must set before first use, such as the
survey link in `zoom-gather-analytics`.

## Contributing

Issues and pull requests are welcome: bug reports from real runs, clearer wording, or
fixes for platforms whose menus have moved. Please describe what happened and what you
expected, and keep changes focused on one skill.

## License

[MIT](LICENSE). Use, change, and share these skills freely, including commercially. Keep
the copyright notice.
