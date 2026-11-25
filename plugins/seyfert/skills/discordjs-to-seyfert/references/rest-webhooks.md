# REST, webhooks, threads, jobs (checkpoint C7)

## Shorters first, `client.proxy` for the rest

Seyfert exposes typed "shorters" on the client: `client.messages`, `channels`, `members`, `roles`, `guilds`, `users`, `threads`, `webhooks`, `reactions`, `emojis`, `bans`, `interactions`, `applications`, ... For any route without a shorter, `client.proxy` is a typed route builder over the REST client:

```ts
// @discordjs/rest:  rest.post(Routes.channelMessages(id), { body })
await client.proxy.channels(channelId).messages.post({ body: { content: 'hi' } })
// rest.patch(Routes.guildMember(g, u), { body, reason })
await client.proxy.guilds(guildId).members(userId).patch({ body: { nick: 'x' }, reason: 'why' })
```

Bodies and queries are API snake_case. You can pass `reason` on proxy calls where the shorter has no reason argument (for example `members.addRole`).

| discord.js / @discordjs/rest | Seyfert |
|---|---|
| `channel.send(x)` / `rest.post(Routes.channelMessages)` | `client.messages.write(channelId, x)` |
| `message.edit(x)` | `client.messages.edit(messageId, channelId, x)` or `message.edit(x)` |
| `message.delete()` | `client.messages.delete(messageId, channelId, reason?)` |
| `message.crosspost()` | `client.messages.crosspost(messageId, channelId)` |
| `channel.bulkDelete(ids)` | `client.messages.purge(ids, channelId)` |
| `user.send(x)` / `createDM` + post | `client.users.write(userId, x)` (or `client.users.createDM(userId)`) |
| `channel.threads.create(...)` / `message.startThread(...)` | `client.threads.create(channelId, body)` / `client.channels.thread(channelId, body)`. For a thread on a message use `client.proxy.channels(channelId).messages(messageId).threads.post({ body })` |
| `thread.setLocked(true)` / `setArchived` | `client.threads.lock(threadId, true)` / `client.threads.edit(threadId, { archived: true })` |
| `guild.channels.fetchActiveThreads()` | `client.threads.listGuildActive(guildId)` |
| `role.edit(x)` / `guild.roles.create(x)` | `client.roles.edit(guildId, roleId, x)` / `client.roles.create(guildId, x)` |
| `channel.permissionOverwrites.edit(...)` | `client.channels.editOverwrite(channelId, overwriteId, body)` |
| `rest.put(Routes.applicationGuildCommands(...))` | see `commands.md` → Registration (**C2**) |

A hand-written REST wrapper service (`discord-rest.ts` style) usually shrinks to almost nothing. Ask (C7) whether to delete it or to keep it as a thin facade over the shorters, so call sites stay unchanged.

## Webhooks

```ts
// discord.js
const hook = new WebhookClient({ id, token })
await hook.send({ content, threadId, username, avatarURL })

// Seyfert - no client object per webhook; call by id + token
await client.webhooks.writeMessage(id, token, {
  body: { content, username, avatar_url: avatarURL },
  query: { thread_id: threadId, wait: true }, // wait:true → returns the message
})
await client.webhooks.editMessage(id, token, { messageId, body: { content } })
```

- A `WebhookClient({ url })` becomes `id` and `token` parsed from the URL (`/webhooks/<id>/<token>`). Keep the parsing in one helper.
- `threadId` → `query.thread_id`; `avatarURL` → `avatar_url`; files → `body.files`.
- Webhook calls need only REST. Modules that send webhooks at boot can run after `client.start()` (see `lifecycle.md`).

## Jobs and queues (cron, BullMQ, setInterval)

- Jobs that use **only** REST: start them after `await client.start()`, and pass in or import the `client`.
- Jobs that read gateway cache or need guild state: start them in `botReady` with `once: true`.
- Separate worker processes with no gateway connection: `await client.start({}, false)` loads handlers and creates the shard manager without connecting (`start(options?, execute = true)`, 6.0.0). It still needs the token and calls `GET /gateway/bot`. Shorters and `client.proxy` work after that.
- `setTimeout` timers that remove roles or end events are in-memory, just like before. The restore-on-boot logic stays, and it goes into `botReady` (see `lifecycle.md`).
