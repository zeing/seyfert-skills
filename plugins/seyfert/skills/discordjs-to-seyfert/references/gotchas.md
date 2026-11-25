# Gotchas: differences that fail silently

Read this before migrating and again before hand-over. Each item was checked in Seyfert 6.0.0 source unless marked.

## Startup and loading
1. **`ready` ≠ `ClientReady`.** `ready` fires per shard before guilds arrive. Use `botReady` with `once: true`.
2. **Missing `experimentalDecorators` / `emitDecoratorMetadata`** → the build fails or commands lose metadata.
3. **Every `.ts`/`.js` file under `locations.*` is imported** (only `.d.ts` is skipped), including helpers, `*.test.ts` and `_disabled.ts`. To keep a "skip files starting with `_`" convention, wrap the handler filter before `start()`: `const f = client.commands.filter; client.commands.filter = (p) => f(p) && !basename(p).startsWith('_')`. The same works for `client.events` and `client.components`. Verified in 6.0.0. Skipped files are still typechecked.
4. **One handler per event name.** A second file for `messageCreate` silently replaces the first. Merge the listeners.
5. **`client.uploadCommands()` bulk-overwrites** the application's commands. Commands registered by other code under the same application id get deleted. See checkpoint C2.

## Commands
6. **Bots can trigger prefix commands.** Seyfert does not check `author.bot`. Filter in the `commands.prefix` callback.
7. **Default prefix args are `-flag value`**, not space-split. Port with a custom `argsParser`.
7a. **Prefix and command-name matching is case-sensitive.** Prefix matching is `content.startsWith(prefix)` and name matching is `name === raw || aliases.includes(raw)` (6.0.0). There is no option to change this. If the old bot lowercased input, have the `prefix` callback return the prefix exactly as the user typed it when it matches case-insensitively, and override `resolveCommandFromContent` to lowercase the first word (see `commands.md`).
7b. Seyfert calls `trimStart()` after the prefix, so `! ping` runs `ping`. A discord.js handler that split on `' '` would not have run it.
8. **Prefix permission checks are guild-level**, the owner bypasses them, and a failure is silent unless `onPermissionsFail` is defined. discord.js code often checked channel permissions.
9. `ctx.message` exists only with `InternalOptions.withPrefix: true`, and only for prefix runs.
10. Without `commands.reply: () => true`, `ctx.write` in a prefix command sends a plain channel message instead of a reply.
11. Errors thrown in `run` go to `onRunError` or `commands.defaults.onRunError`. With neither set, they are only logged. Set a default that matches the old error UX.

## Events and cache
12. **`messageUpdate` old message exists only if cached.** With the `messages` cache disabled or expired, `old` is `undefined`.
13. **The default cache is unbounded**, messages included. Configure `LimitedMemoryAdapter` to replace the sweepers.
14. Message cache limits are bucketed **per guild**, not per channel.
15. **No partials.** Reaction events give IDs. Deletes give the cached structure or raw IDs. Fetch explicitly.
16. **A cache-first `members.list` can be partial.** Pass `{ force: true }` when completeness matters.
17. There is no bulk "fetch these member IDs" over REST. Use `shard.requestGuildMember({ user_ids })` (gateway) or loop.
18. Tuple events (`messageUpdate`, `guildMemberUpdate`, `guildUpdate`, `threadUpdate`, `channelUpdate`) pass `[new, old]` as the **first argument**.

## Components
19. **Collectors die on restart** (same as discord.js). Messages that must stay clickable need `ComponentCommand`.
20. **Every matching `ComponentCommand` runs.** Overlapping customId prefixes double-handle.
21. One collector per message. Attaching a second one replaces the first.
22. Keep customIds unchanged. Old messages still carry them.

## Payload shapes
23. API bodies are **snake_case** (`allowed_mentions`, `thread_id`, `avatar_url`, `custom_id`). discord.js camelCase options are silently ignored, so typecheck strictly.
24. `ephemeral: true` → `flags: MessageFlags.Ephemeral`.
25. `user.tag` has no `#0` suffix in v6. Prefer `user.username`.
26. `member.moderatable()` checks `ModerateMembers` in v6.

## Found while migrating a real bot (Seyfert 6.0.0)
29. **The client's `allowedMentions` is applied to every body Seyfert builds**, including `client.webhooks.writeMessage` and `messages.write` (`transformMessageBody`). A discord.js `WebhookClient` had no client defaults. Set `allowed_mentions` explicitly on webhook sends to keep the old pings.
30. **There is no reaction collector** (`awaitReactions`/`createReactionCollector`). Keep a small waiter registry that your `messageReactionAdd` event feeds *before* any event-level filtering. discord.js collectors saw every reaction.
31. `messageUpdate`'s first tuple item is typed `OmitInsert<MessageStructure, 'tts', …>`. It is a `Message` at runtime, so cast it once in the event (`as unknown as MessageStructure`).
32. Discord still sends the deprecated message `interaction` field (`name`, `user`), and Seyfert copies it onto the structure, but its types omit it. Read it through one typed accessor.
33. Threads: discord.js `thread.locked` / `thread.archived` become `thread.threadMetadata?.locked` / `.archived`.
34. `StringSelectMenu.setValuesLength({ max })` also sets `min` to `undefined`. Port `setMaxValues(n)` as `setValuesLength({ min: menu.data.min_values, max: n })`.
35. REST errors are `SeyfertError`. The Discord JSON code is at `error.metadata.response.code`, where discord.js had `DiscordAPIError.code`.
36. discord.js `role.members` and `channel.messages.cache` are **cache-only** views. Port them to cache reads (`client.cache.members.values(guildId)`, `client.cache.messages.values(channelId)`), not REST, to keep the semantics.
37. `members.fetch({ query, limit })` (a gateway query) maps to `client.members.search(guildId, { query, limit }, true)`. Without `true`, Seyfert may answer from cache.
38. discord.js `member.roles.remove([...ids])` PATCHes the full role list from cache, so successive calls can re-add roles. Prefer one `DELETE` per owned role through `client.proxy...roles(id).delete({ reason })`, which also keeps the audit reason that the shorters drop.
39. Rebuild received components and embeds with `component.toBuilder()` / `embed.toBuilder()`. Container builder methods are `setColor` and `addComponents`, not `setAccentColor` or `addTextDisplayComponents`. The separator enum is `Spacing`.
40. **Automated renames corrupt string literals.** A regex `Message` → `MessageStructure` also rewrote log and embed text ("Game Message Error"). Rename types only in type positions, then grep the diff for structure names inside quotes.

41. Interaction methods take flags/booleans, not discord.js option objects: `interaction.deferReply({ flags: Ephemeral })` → `interaction.deferReply(MessageFlags.Ephemeral)`; `CommandContext` / `ComponentContext` take `deferReply(true)`. `ComponentContext` has no `editResponse`: use `editOrReply`.
42. Raw select options (`addOption({ label, value, emoji: '<:name:id>' })` objects) are sent as-is, so emoji strings are not parsed like discord.js did. Parse to `{ id, name }` first, and skip `setEmoji('')` (it throws).
43. With `strictNullChecks: false` (common in old discord.js repos), `ctx.options` collapses to `never`. Read options through one cast (`ctx.options as { args?: string }`), not per access.
44. `@slipher/cooldown`: object options of `@Middlewares` lose typing. To print the old cooldown text, call `client.cooldown.check({ name: ctx.fullCommandName, target, guildId })` in `onMiddlewaresError`. Use the **full** name: for subcommands, `ctx.command.name` does not resolve the parent's bucket.
45. Roles: discord.js `roles.create({ position })` is a create **plus** `setPosition`, since the API ignores `position` on create. Port it as `client.roles.create` then `client.roles.editPositions`. `role.hexColor` / `role.iconURL()` have no Seyfert getters, and an icon URL must be sent as a data URI (`data:image/png;base64,…`).
46. Editing a permission overwrite with `PUT .../permissions/:id` replaces it. discord.js `permissionOverwrites.edit` merged the new value into the existing allow/deny bits, so read and merge before the PUT.
47. **Removing `discord.js` removes its transitive packages.** Code importing `lodash`, `undici` or `ws` that only resolved through discord.js breaks at runtime, not in `tsc` when types come from elsewhere. Before uninstalling, list bare imports that are not in `package.json`.
48. `@discordjs/formatters` is not a drop-in for Seyfert `Formatter`: Seyfert has no mention helpers and orders `codeBlock` / `messageLink` arguments differently. Copy the few functions you use into a local module, and compare outputs against the package before removing it.
49. A shell `&&` chain that greps for a just-written line stops silently when the formatter has re-wrapped it, so later files are never written and `tsc` still passes. Confirm new files exist in the load check (count commands and components).

50. **Guild commands registered by hand must carry `guildId`.** An interaction from a guild-registered command has `guild_id`, and `HandleCommand.getCommand` then only matches commands whose `guildId` includes that guild. If you POST commands per guild yourself (instead of `@Declare({ guildId })` + `uploadCommands`), set `command.guildId = [guildId]` too, or every subcommand logs `undefined <sub> command does not have 'run' callback` and Discord shows "The application did not respond". Set it after `toJSON()` if the POST body must not change. Typecheck and load check cannot catch this: only a real interaction does.

## Runtime
27. Verified working: Bun 1.4 + TypeScript 7.0 (native) typecheck with `moduleResolution: Bundler` and `locations.base: 'src'`. For Node + `tsc` emit, set `base: 'dist'` and make sure `seyfert.config.mjs` is resolvable from the cwd.
28. `HandleCommand` is not exported from the `seyfert` root in 6.0.0. Import it from `seyfert/lib/commands/handle`.
