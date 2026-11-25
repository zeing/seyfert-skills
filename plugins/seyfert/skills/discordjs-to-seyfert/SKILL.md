---
name: discordjs-to-seyfert
description: Migrate a Discord bot from discord.js v14 to Seyfert v6 (https://www.seyfert.dev). Use when the user wants to move a bot off discord.js, port commands/events/components/collectors to Seyfert, or plan such a migration. Interactive - inventories the bot, stops at mandatory checkpoints for human decisions, records them in SEYFERT_MIGRATION.md, then migrates phase by phase with typecheck gates.
---

# discord.js → Seyfert migration

Target: **Seyfert v6** from **discord.js v14**, gateway bots (single process). Written against `seyfert@6.0.0`.

Out of scope for this skill (point the user at the docs instead): HTTP interactions mode (`config.http`), sharding / `WorkerClient`, i18n (`langs`), voice. If the bot uses any of these, say so at the checkpoint and ask how to proceed.

## Ground rules

1. **Never guess a checkpoint answer.** Each checkpoint below is a question for the human. Ask, wait, record the answer in `SEYFERT_MIGRATION.md`, then act.
2. **Verify API before use.** The mappings in `references/` were checked against Seyfert 6.0.0. Check the installed version first (`node_modules/seyfert/package.json`). If it is not 6.x, re-verify every API you touch against `node_modules/seyfert/lib/**/*.d.ts` or https://www.seyfert.dev/docs before writing code. The type definitions are the source of truth over the docs and over this skill.
3. **Preserve behavior.** A migration is not a refactor. Keep customIds, prefixes, permission checks, message text and timings identical unless the human approves a change at a checkpoint. Bugs found on the way go into `SEYFERT_MIGRATION.md` → "Found issues"; do not fix them in the migration diff.
4. **Big-bang on a branch.** Work on a new branch. The old discord.js commit is the rollback. Do not run discord.js and Seyfert gateway clients on the same token at the same time (duplicate event handling, split caches).
5. **Gate every phase.** A phase is done when the project typechecks and every file migrated in that phase loads (see "Verification"). Tick it in `SEYFERT_MIGRATION.md` before starting the next.
6. Follow the target repo's own conventions (CLAUDE.md, lint, path aliases, error-handling style) over the examples here.
7. **Comments explain why, not what it used to be.** Do not write mapping comments like `// discord.js guild.roles.cache` or `// was the global customId router`. After the merge discord.js is gone, so they become noise. The old ↔ new mapping lives in the PR diff and `SEYFERT_MIGRATION.md`. Do comment constraints a future reader could "simplify" away, stated without the old API: e.g. `// The API ignores position on create: create, then move`, `// cache only, no fetch`, `// explicit so the client default allowedMentions is not applied`.

## Workflow

### Step 1 - Inventory (read-only)

Collect facts; do not ask the human for anything you can grep. Fill the "Inventory" section of the plan template (`assets/migration-plan-template.md`).

- Runtime and build: Node vs Bun, `tsc` emit vs no-build, `tsconfig` (`moduleResolution`, `paths`, `verbatimModuleSyntax`), TypeScript version, lint/format tools.
- Client construction: intents, partials, `makeCache`, sweepers, presence, `allowedMentions`, sharding.
- Loaders: how commands, events, and components are discovered and registered.
- Surface counts - run these and record counts plus file lists:

```bash
rg -l "from ['\"]discord\.js['\"]" src | wc -l
rg -c "EmbedBuilder|ActionRowBuilder|ButtonBuilder|StringSelectMenuBuilder|ModalBuilder|AttachmentBuilder" src
rg -n "createMessageComponentCollector|awaitMessageComponent|awaitModalSubmit|createReactionCollector|awaitMessages" src
rg -n "\.cache\b" src | rg -o "(guilds|channels|members|roles|users|messages|emojis)\.cache" | sort | uniq -c
rg -n "members\.fetch|guilds\.fetch|channels\.fetch|messages\.fetch" src
rg -n "WebhookClient|\.createWebhook|REST\(|Routes\." src
rg -n "Events\.[A-Za-z]+|client\.on\(|client\.once\(" src
rg -n "\.partial\b|Partials\." src
rg -n "setTimeout|setInterval|CronJob|new Worker\(|new Queue\(" src
rg -n "\.put\(Routes\.application|commands\.set\(|application\.commands" src
```

- For every event, command, component handler, job, and startup function: one line on what it does and what it needs (gateway cache? REST only? DB?).
- Note anything that reads messages from other bots, `messageUpdate` old/new comparisons, reaction handlers on uncached messages, and long-lived timers.

### Step 2 - Checkpoints (ask the human)

Ask all checkpoints whose answers do not depend on each other in one round. Number them and give a recommended answer with the reason. Record every answer under "Decisions".

| # | Checkpoint | When to ask | Reference |
|---|---|---|---|
| C1 | **Startup placement**: for each function the old `ready` handler calls, does it go in `botReady` (`once: true`) or after `await client.start()`? | Always, per function | `references/lifecycle.md` |
| C2 | **Command registration**: Is this application id shared with another codebase or bot? If yes, **`client.uploadCommands()` is forbidden** (it bulk-PUTs and deletes commands it does not know about). | If any slash/context command exists | `references/commands.md` |
| C3 | **Prefix commands**: native Seyfert text commands with a custom `argsParser`, or keep a hand-written `messageCreate` handler? How are args parsed today (positional split, flags, quoted)? | If prefix commands exist | `references/commands.md` |
| C4 | **Cooldowns / permissions**: map the existing cooldown store to `@slipher/cooldown`? Map the permission checks to `defaultMemberPermissions` / `botPermissions` or keep custom checks? | If either exists | `references/commands.md` |
| C5 | **Component kind**, per component flow: collector (dies on restart) or `ComponentCommand` (survives restart)? Default to collector. Ask for each button or select that is posted once and expected to keep working (panels, role pickers, setup messages). | If components exist | `references/components.md` |
| C6 | **Cache policy**: which resources to cache, and limits (replaces `makeCache` + sweepers). Is `messageUpdate` "old message" needed? It only exists when the message is cached. | Always | `references/cache-and-fetch.md` |
| C7 | **REST / webhooks**: migrate `@discordjs/rest` / `WebhookClient` usage to Seyfert shorters and `client.proxy` now, or keep it as a later phase? | If found | `references/rest-webhooks.md` |
| C8 | **Out-of-scope features** found in inventory (sharding, HTTP mode, voice, i18n). | If found | Seyfert docs |
| C9 | **Anything ambiguous** that you found: a function whose purpose or dependencies you cannot tell from the code. Quote file:line, say what you think it does, ask "is this right?". | Whenever | - |

Do not proceed to Step 3 until every applicable checkpoint is answered.

### Step 3 - Write `SEYFERT_MIGRATION.md`

Copy `assets/migration-plan-template.md` to the target repo root as `SEYFERT_MIGRATION.md`. Fill in the inventory, decisions, phase checklist (one checkbox per file or group of files), a smoke-test checklist (one line per command, event and component that a human can trigger in a dev guild), and found issues. Show it to the human. This file lets the work resume across sessions and lets reviewers follow it in the PR.

### Step 4 - Migrate phase by phase

Create a branch (e.g. `migrate/seyfert`). Phases in order, each one gated:

1. **Toolchain** - install `seyfert` (and `@slipher/cooldown` if C4 says so), set up `tsconfig` decorators, add `seyfert.config.mjs`, and add `declare module 'seyfert'` typings. → `references/lifecycle.md`
2. **Client + lifecycle** - replace `new Client(...)` and the loaders; split the `ready` handler per C1; set up the cache per C6. → `references/lifecycle.md`, `references/cache-and-fetch.md`
3. **Events** - convert each event file to `createEvent`. → `references/events.md`
4. **Commands** - prefix and slash commands as `@Declare` classes; registration per C2. → `references/commands.md`
5. **Components** - collectors and `ComponentCommand`s per C5; global customId routers are removed. → `references/components.md`
6. **Builders and remaining discord.js calls** - embeds, rows, attachments, cache access. → `references/builders.md`, `references/cache-and-fetch.md`
7. **REST / webhooks / jobs** - per C7. → `references/rest-webhooks.md`
8. **Remove discord.js** - uninstall `discord.js` and `@discordjs/*`. Then `rg "discord\.js"` must return nothing in `src`. Keep `discord-api-types` only if code still imports it directly; Seyfert re-exports API types from `seyfert/lib/types`.

Per-phase rule: Large phases (events, commands, builders) may be split into batches, but the gate still applies per batch.

**Keeping every commit green in a big codebase.** Events, utils and commands call each other, so at some point one side is converted and the other is not.
- Add a temporary bridge, `legacy(x): any`, in one file. Wrap every call that crosses the Seyfert↔discord.js boundary with it. `rg "legacy\("` then counts the remaining work, and so do the `no-unsafe-argument` lint warnings. Delete the bridge in phase 8.
- For a shared helper module that both converted and unconverted code use (for example `utils/discord.ts` or pagination), copy the discord.js version to `legacy-<name>.ts` and sed the unconverted importers to it. Rewrite the original for Seyfert. Delete the copy once its last importer is converted.
- Convert in order: events → core helpers → feature groups (one commit each) → commands → components → REST. Move a helper up the order when a phase cannot typecheck without it. In practice the webhook module and pagination had to move early.
- A mechanical converter (import splitting, builder renames, `iconURL` → `iconUrl`) saves time. See gotcha #40 before trusting it.

### Step 5 - Verify and hand over

- Gates (every phase): the project's typecheck (`tsc --noEmit` or equivalent) passes, and the lint passes if the project has one.
- Load check: a throwaway script imports `client` and calls `await client.loadCommands()`, `loadEvents()` and `loadComponents()` without `start()`. Every file must load with no errors. Delete the script afterwards.
- Smoke test (human, dev bot token, dev guild): walk the smoke checklist in `SEYFERT_MIGRATION.md`. Never use the production token for this.
  Typecheck and load check cannot catch interaction routing (guild `guildId`, customId filters, collectors). Only a real click or command does, so every slash command and every persistent button needs one live try.
- Before uninstalling discord.js, list bare imports that are not in `package.json` (packages that only resolved through discord.js, e.g. `lodash`). See gotcha 47.
- Comment pass: `git diff <base> -- src | grep -E '^\+\s*(//|\*)' | grep -iE 'discord\.js|was |old '`. Remove mapping-only comments, reword the ones that carry a reason (rule 7). Verify it is comment-only: with comment lines stripped, every changed file must equal HEAD.
- Hand over as a **draft PR** while the human keeps testing. Expect the line count to stay about the same: helpers replace discord.js conveniences (`role.members`, `awaitReactions`, overwrite merge, mention formatters). The win is the dependency tree and removing hand-written routers and loaders, not fewer lines.
- Report: what changed, what passed, the items still unchecked, the found issues, and the rollback (redeploy the last discord.js commit).

Read `references/gotchas.md` before Step 4 and again before hand-over.

## References

- `references/lifecycle.md` - config, tsconfig, client, typings, where startup code goes
- `references/events.md` - event file shape, name mapping, argument shapes
- `references/commands.md` - `@Declare`, options, prefix commands, `argsParser`, cooldown, permissions, registration
- `references/components.md` - collectors vs `ComponentCommand`, modals
- `references/builders.md` - EmbedBuilder → Embed, and the rest
- `references/cache-and-fetch.md` - cache config, `.cache` replacements, fetching
- `references/rest-webhooks.md` - shorters, `client.proxy`, webhooks, threads
- `references/gotchas.md` - behavior differences that break silently
- `assets/migration-plan-template.md` - the `SEYFERT_MIGRATION.md` template
