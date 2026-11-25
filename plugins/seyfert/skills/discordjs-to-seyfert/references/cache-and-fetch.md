# Cache and fetching (checkpoint C6)

## Model

- The cache is **global**, not one cache per manager on each guild. Resources: `users`, `guilds`, `members`, `voiceStates`, `presences`, `bans`, `emojis`, `roles`, `channels`, `stickers`, `stageInstances`, `overwrites`, `messages` (plus `onPacket`, which turns off gateway writes entirely).
- Default adapter: `MemoryAdapter`. **Everything is cached, with no limit.** That includes `messages`, which a discord.js bot usually bounds with sweepers.
- Cache reads may be sync or async, depending on the adapter. Code that awaits `client.cache.x.get(...)` works with both.
- Intents still decide what can be cached. `members` needs `GuildMembers`; `presences` needs `GuildPresences`.

## Port `makeCache` + `sweepers`

```ts
import { LimitedMemoryAdapter } from 'seyfert'

client.setServices({
  cache: {
    adapter: new LimitedMemoryAdapter({
      message: { expire: 5 * 60_000, limit: 1_000 }, // ≈ sweepers.messages lifetime 300s
    }),
    disabledCache: { emojis: true, stickers: true, voiceStates: true, stageInstances: true }, // ≈ cacheWithLimits({ X: 0 })
  },
})
```

| discord.js `makeCache` / `sweepers` key | Seyfert |
|---|---|
| `GuildEmojiManager: 0`, `BaseGuildEmojiManager: 0` | `disabledCache.emojis` |
| `GuildStickerManager: 0` | `disabledCache.stickers` |
| `VoiceStateManager: 0` | `disabledCache.voiceStates` |
| `GuildInviteManager: 0` | no invite cache in Seyfert; nothing to do |
| `ThreadManager: 0` / `ThreadMemberManager: 0` | threads live in `channels`, so they cannot be disabled separately. Use `client.cache.channels!.filter = (ch) => ...` |
| `sweepers.messages { lifetime }` | `LimitedMemoryAdapter({ message: { expire } })` |
| `MessageManager: n` | `LimitedMemoryAdapter({ message: { limit: n } })` (`limit` and `expire` apply per storage bucket. Messages are bucketed per **guild** in 6.0.0 (`message.<guildId>`), unlike the per-channel discord.js limit, so size `limit` for the busiest guild. With `expire` set, each read refreshes the TTL.) |

Ask at C6:
- Does any handler compare an old message with the new one (`messageUpdate`)? Then keep `messages` and choose an `expire` at least as long as the longest edit window the code cares about.
- Does any feature need `presences`?

## `.cache` replacements

| discord.js | Seyfert (cache-first, falls back to REST unless noted) |
|---|---|
| `client.guilds.cache.get(id)` | `await client.guilds.fetch(id)`, or `client.cache.guilds?.get(id)` (cache only) |
| `guild.members.cache.get(uid)` | `await client.members.fetch(guildId, uid)`, or `client.cache.members?.get(uid, guildId)` |
| `guild.roles.cache.get(rid)` | `await client.roles.fetch(guildId, rid)`, or `client.cache.roles?.get(rid)` |
| `guild.roles.cache.filter(...)` | `(await client.roles.list(guildId)).filter(...)` |
| `client.channels.cache.get(id)` | `await client.channels.fetch(id)`, or `client.cache.channels?.get(id)` |
| `role.members` (members having a role) | `(await client.members.list(guildId, { limit: 1000 }, { force: true })).filter((m) => m.roles.keys.includes(rid))`, paginated with `after` for big guilds. Alternatively loop over the cache: `client.cache.members?.values(guildId)` |
| `member.roles.cache.has(rid)` | `member.roles.keys.includes(rid)` |
| `member.roles.add(rid)` / `remove(rid)` | `member.roles.add(rid)` / `remove(rid)`, or `client.members.addRole(gid, uid, rid)` / `removeRole(...)` (no reason argument) |
| `channel.messages.fetch(id)` | `await client.messages.fetch(messageId, channelId)` |
| `channel.messages.fetch({ limit })` | `await client.messages.list(channelId, { limit })` |
| `client.users.fetch(id)` | `await client.users.fetch(id)` |
| `guild.members.fetch({ user: ids })` (bulk by IDs over the gateway) | `await client.gateway.get(shardId)!.requestGuildMember({ guild_id, user_ids: ids, limit: 0 })` → `{ members: APIGuildMember[] }` (raw), or loop `client.members.fetch` (REST, 1 call per ID) |
| `guild.members.fetch()` (all members) | `client.members.list(guildId, { limit: 1000, after }, { force: true })` paginated, or `requestGuildMember({ guild_id, query: '', limit: 0 })` on the gateway |

- `fetch(..., force = true)` skips the cache. `list`/`search` take `{ force: true }` to skip the cache and `{ cacheQuery: false }` to skip cache-side pagination.
- **A cache-first `list` can be partial.** When the cache already has *some* members, `members.list` answers from the cache by default. Use `force: true` when the logic needs completeness (role sweeps, cleanup jobs).
- Ask at C9 when the old `fetch({ user: ids, time })` had a timeout: is the timeout behavior part of the logic, or only a safety net?
- Fetching a guild's shard ID for `requestGuildMember`: single-shard bots use `0`. Otherwise compute it with `client.gateway.calculateShardId(guildId)`. Check that method exists in the installed version.

Structure methods replace manager calls in many places (`message.reply`, `message.react`, `message.edit`, `message.delete`, `message.channel()`, `message.guild()`, `member.roles.add`). Prefer them when the structure is already in hand. Prefer the `client.<shorter>` call when you only have IDs.
