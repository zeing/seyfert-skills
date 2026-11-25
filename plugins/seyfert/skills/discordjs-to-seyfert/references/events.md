# Events

## File shape

```ts
// discord.js
export default {
  name: Events.MessageCreate,
  once: false,
  async execute(message: Message) { /* ... */ },
}

// Seyfert - src/events/message-create.ts
import { createEvent } from 'seyfert'

export default createEvent({
  data: { name: 'messageCreate' },
  async run(message, client, shardId) { /* ... */ },
})
```

- Every event's run gets `(payload, client, shardId)`. The client is the **second** argument. Drop any closure over a global client if you can.
- **Only one handler per event name survives.** `EventHandler.load` stores `values[name] = instance`, so a later file silently replaces an earlier one (verified in 6.0.0, `lib/events/handler.js`). discord.js projects often have several listeners on one event, for example a command handler and a bot-message parser both on `MessageCreate`. Merge them into one event file that calls each handler in turn, keeping the original order. Move the handler bodies into plain modules outside `locations.events`.
- With native prefix commands, Seyfert runs your `messageCreate` event **and then** its own command handler for the same message, so the old command-dispatch listener goes away entirely.
- Payloads are Seyfert structures (`MessageStructure`, `GuildMemberStructure`, ...) or camelCased raw objects, never discord.js classes.

## Name mapping

The name is the gateway dispatch name in camelCase (`MESSAGE_REACTION_ADD` → `messageReactionAdd`). The type of `run`'s first argument is the return type of the matching hook in `node_modules/seyfert/lib/events/hooks/*.d.ts`, so look it up there.

| discord.js `Events.*` | Seyfert `name` | First `run` argument |
|---|---|---|
| `ClientReady` | `botReady` (all shards + guilds ready) | `ClientUserStructure` |
| *(none)* | `ready` (per shard, before guilds) | `ClientUserStructure` |
| `MessageCreate` | `messageCreate` | `MessageStructure` |
| `MessageUpdate` | `messageUpdate` | `[message, old \| undefined]` tuple |
| `MessageDelete` | `messageDelete` | cached message **or** `{ id, channelId, guildId }` |
| `MessageReactionAdd` / `Remove` | `messageReactionAdd` / `messageReactionRemove` | plain object `{ userId, channelId, messageId, guildId?, member?, emoji, burst, ... }` |
| `InteractionCreate` | `interactionCreate` | interaction (usually not needed - see `components.md`) |
| `GuildMemberAdd` | `guildMemberAdd` | `GuildMemberStructure` |
| `GuildMemberRemove` | `guildMemberRemove` | cached member **or** `{ user, guildId }`-like raw object |
| `GuildMemberUpdate` | `guildMemberUpdate` | `[member, old?]` tuple |
| `GuildUpdate` | `guildUpdate` | `[guild, old?]` tuple |
| `ThreadUpdate` | `threadUpdate` | `[thread, old?]` tuple |
| `ChannelUpdate` | `channelUpdate` | `[channel, old?]` tuple |

**Tuple events:** when a hook returns `[new, old]`, the first argument *is the tuple*:

```ts
createEvent({
  data: { name: 'messageUpdate' },
  async run([message, old], client) {
    if (!old) return // not in cache: old content unknown
  },
})
```

`old` is whatever was in cache when the update arrived. If the resource is disabled in the cache or has expired, `old` is `undefined`. Logic like "compare old vs new embeds" needs the `messages` cache on (checkpoint C6).

## Partials

Seyfert has no `Partials` and no `.partial` flag.
- Reaction events give IDs only. Fetch when you need the message: `await client.messages.fetch(messageId, channelId)`.
- Deletes give the cached structure or raw IDs. Narrow with `'content' in data` (or check for the structure class) before you use message fields.
- Drop every `if (x.partial) await x.fetch()` block and replace it with an explicit fetch by ID.

## Custom events

```ts
declare module 'seyfert' {
  interface CustomEvents { raidSpawned: (raidId: string) => void }
}
await client.events?.runCustom('raidSpawned', id)
```

Replaces `client.emit('custom', ...)`.
