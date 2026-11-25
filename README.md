# seyfert-skills

Agent skills for building and migrating Discord bots with [Seyfert](https://www.seyfert.dev). The skills are packaged as a Claude Code plugin marketplace. Each skill is plain Markdown and works with any agent that reads `SKILL.md` files.

## Skills

| Skill | What it does |
|---|---|
| [`discordjs-to-seyfert`](plugins/seyfert/skills/discordjs-to-seyfert/SKILL.md) | Migrates a discord.js v14 bot to Seyfert v6. It inventories the bot, stops at mandatory checkpoints for human decisions, records them in `SEYFERT_MIGRATION.md`, then migrates phase by phase with a typecheck gate after each phase. |

## Install

**Claude Code plugin**

```
/plugin marketplace add <owner>/seyfert-skills
/plugin install seyfert@seyfert-skills
```

**Manual (any agent)**: copy or symlink the skill directory into your agent's skills folder:

```bash
ln -s "$PWD/plugins/seyfert/skills/discordjs-to-seyfert" ~/.claude/skills/discordjs-to-seyfert
```

## Usage

In the bot repository, ask: *"Migrate this bot from discord.js to Seyfert"*. The skill will:

1. inventory discord.js usage (read-only)
2. ask checkpoint questions: where startup code goes, whether the application id is shared (it never bulk-overwrites a shared one), prefix arg parsing, cooldowns and permissions, collector vs `ComponentCommand`, cache policy, REST and webhooks
3. write `SEYFERT_MIGRATION.md` in your repo
4. migrate on a branch, phase by phase

## Scope and verification

- Targets **Seyfert 6.x** from **discord.js 14.x**, for a single-process gateway bot.
- Not covered: HTTP interactions mode, sharding and workers, i18n, voice. The skill points you to the Seyfert docs for these.
- The API mappings were checked against `seyfert@6.0.0` type definitions and source. The reference snippets typecheck under TypeScript 7.0 with Bun 1.4.
- The skill tells the agent to re-verify the mappings against the installed version.

## Contributing

Corrections are welcome, especially for API changes in newer Seyfert releases. Each claim in `references/` should be checkable against `node_modules/seyfert/lib` or the Seyfert docs.

## License

[MIT](LICENSE)
