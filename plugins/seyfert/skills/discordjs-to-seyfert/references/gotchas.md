# Gotchas: differences that fail silently

Read this before migrating and again before hand-over. Each item was checked in Seyfert 6.0.0 source unless marked.

## Startup and loading
1. **`ready` ≠ `ClientReady`.** `ready` fires per shard before guilds arrive. Use `botReady` with `once: true`.
2. **Missing `experimentalDecorators` / `emitDecoratorMetadata`** → the build fails or commands lose metadata.
3. **Every file under `locations.*` is imported**, including helpers and `*.test.ts`. Anything that does not default-export a command, event or component is logged as a warning or ignored. Keep helpers and tests out of these directories.
4. **One handler per event name.** A second file for `messageCreate` silently replaces the first. Merge the listeners.
5. **`client.uploadCommands()` bulk-overwrites** the application's commands. Commands registered by other code under the same application id get deleted. See checkpoint C2.

## Commands
6. **Bots can trigger prefix commands.** Seyfert does not check `author.bot`. Filter in the `commands.prefix` callback.
7. **Default prefix args are `-flag value`**, not space-split. Port with a custom `argsParser`.
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

## Runtime
27. Verified working: Bun 1.4 + TypeScript 7.0 (native) typecheck with `moduleResolution: Bundler` and `locations.base: 'src'`. For Node + `tsc` emit, set `base: 'dist'` and make sure `seyfert.config.mjs` is resolvable from the cwd.
28. `HandleCommand` is not exported from the `seyfert` root in 6.0.0. Import it from `seyfert/lib/commands/handle`.
