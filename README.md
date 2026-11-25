# seyfert-skills

Agent skills for building and migrating Discord bots with [Seyfert](https://www.seyfert.dev). The skills are packaged as a Claude Code plugin marketplace. Each skill is plain Markdown and works with any agent that reads `SKILL.md` files.

## Skills

| Skill | What it does |
|---|---|
| [`discordjs-to-seyfert`](plugins/seyfert/skills/discordjs-to-seyfert/SKILL.md) | Migrates a discord.js v14 bot to Seyfert v6. It inventories the bot, stops at mandatory checkpoints for human decisions, records them in `SEYFERT_MIGRATION.md`, then migrates phase by phase with a typecheck gate after each phase. |

## Install

**Claude Code plugin**

```
/plugin marketplace add zeing/seyfert-skills
/plugin install seyfert@seyfert-skills
```

**Any agent, one command** ([`skills` CLI](https://github.com/vercel-labs/skills), installs into Codex, Cursor, Gemini CLI, Copilot, OpenCode and many more):

```bash
npx skills add zeing/seyfert-skills            # pick agents interactively
npx skills add zeing/seyfert-skills -a codex -a cursor -a gemini-cli -y
npx skills add zeing/seyfert-skills -g         # user-level instead of the project
```

`skills.sh.json` points the CLI at `plugins/seyfert/skills/`.

**Manual install for other agents (Codex, GitHub Copilot, Cursor, Gemini CLI, …)**

The skill uses the open [Agent Skills](https://agentskills.io) format: a folder with a `SKILL.md` (YAML frontmatter + Markdown) plus `references/` and `assets/`. Copy or symlink the whole folder, not just `SKILL.md`. The references are loaded on demand.

```bash
SKILL="$PWD/plugins/seyfert/skills/discordjs-to-seyfert"

# Shared cross-agent location, read by several tools (project-level, inside the bot repo)
mkdir -p .agents/skills && ln -s "$SKILL" .agents/skills/discordjs-to-seyfert

# Claude Code (personal)
ln -s "$SKILL" ~/.claude/skills/discordjs-to-seyfert
```

Skill folders differ per tool and change between releases. Check your tool's docs for the current path. Commonly cited locations:

| Tool | Project | Personal |
|---|---|---|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| GitHub Copilot / VS Code | `.github/skills/`, `.agents/skills/` | `~/.copilot/skills/` |
| Cursor | `.cursor/skills/`, `.agents/skills/` | — |
| OpenAI Codex | `.agents/skills/` (older docs: `.codex/skills/`) | `~/.agents/skills/` or `~/.codex/skills/` |
| Gemini CLI | `.gemini/skills/`, `.agents/skills/` | `~/.gemini/skills/` |

**Agents without skill support**: point them at the file from the repo's `AGENTS.md` (or the tool's rules file):

```markdown
## discord.js → Seyfert migration
When asked to migrate this bot from discord.js to Seyfert, read and follow
`<path>/discordjs-to-seyfert/SKILL.md`. Open files under its `references/` when SKILL.md points to them.
Stop at every checkpoint (C1–C11) and wait for my answer.
```

The `.claude-plugin/` manifests only matter to Claude Code. Other tools ignore them.

**Tested with**: Claude Code (a real migration, juice-box-bot). Other agents should work through the standard format but are not tested yet. Reports are welcome.

## Usage

In the bot repository, ask: *"Migrate this bot from discord.js to Seyfert"*. The skill will:

1. inventory discord.js usage (read-only)
2. ask checkpoint questions: where startup code goes, whether the application id is shared (it never bulk-overwrites a shared one), prefix arg parsing, cooldowns and permissions, collector vs `ComponentCommand`, cache policy, REST and webhooks, logger, Seyfert plugins
3. write `SEYFERT_MIGRATION.md` in your repo
4. migrate on a branch, phase by phase

**Headless runs** (no one to answer, e.g. `codex exec`, CI): the agent stops after writing `SEYFERT_MIGRATION.md` with recommended answers marked `PENDING`. Fill in the Decisions table, set each row to `ANSWERED`, and run it again. The second run migrates.

## Scope and verification

- Targets **Seyfert 6.x** from **discord.js 14.x**, for a single-process gateway bot.
- Not covered: HTTP interactions mode, sharding and workers, i18n, voice. The skill points you to the Seyfert docs for these.
- The API mappings were checked against `seyfert@6.0.0` type definitions and source. The reference snippets typecheck under TypeScript 7.0 with Bun 1.4.
- The skill tells the agent to re-verify the mappings against the installed version.

## Contributing

Corrections are welcome, especially for API changes in newer Seyfert releases. Each claim in `references/` should be checkable against `node_modules/seyfert/lib` or the Seyfert docs.

## License

[MIT](LICENSE)
