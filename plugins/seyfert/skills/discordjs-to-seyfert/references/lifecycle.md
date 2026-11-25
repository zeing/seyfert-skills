# Lifecycle: toolchain, client, startup

## tsconfig

Seyfert's `@Declare`, `@Options`, `@Middlewares` are legacy decorators. Add:

```jsonc
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

If you leave these out, the build fails or the commands lose their metadata. Verified with TypeScript 7.0 (native) typecheck + Bun 1.4 runtime. If the project uses `verbatimModuleSyntax` with CJS output, remove it (Seyfert docs). It is fine with ESM + `noEmit`.

## seyfert.config.mjs (project root)

```js
// @ts-check
import { config } from 'seyfert'

export default config.bot({
  token: process.env.BOT_TOKEN ?? '',
  intents: ['Guilds', 'GuildMessages', 'MessageContent'], // same set as the old client
  locations: {
    base: 'src', // 'src' when run directly (Bun / tsx); 'dist' when compiled with tsc
    commands: 'commands',
    events: 'events',
    components: 'components',
  },
})
```

- Intents are string names of `GatewayIntentBits` (`'GuildMembers'`, `'GuildPresences'`, `'GuildMessageReactions'`, ...).
- Each `locations` entry is a directory that Seyfert loads recursively. Every file must **default-export** a command class, a `createEvent(...)` result, or a component class. Move helpers out of these directories, or keep a `_`-prefix skip convention by wrapping the handler `filter` (see `gotchas.md` #3).
- `client.getRC()` reads the config at runtime.

## Client + typings

```ts
// src/client.ts
import { Client, type ParseClient } from 'seyfert'

export const client = new Client({
  commands: { prefix: (m) => (m.author.bot ? [] : ['!']), reply: () => true }, // only if prefix commands exist
  allowedMentions: { parse: ['roles'], replied_user: true }, // snake_case API shape
  presence: () => ({ status: 'online', since: null, afk: false, activities: [{ name: 'x', type: 3 }] }),
})

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>
    // plugins: typeof plugins     // when using definePlugins(...)
    // middlewares: typeof middlewares
  }
  interface InternalOptions {
    withPrefix: true // only if prefix commands exist; makes ctx.message typed
  }
}
```

- Option objects that go to the Discord API (`allowedMentions`, `presence`, message bodies) use **snake_case API shapes**, not discord.js camelCase.
- `client.setServices({ cache, middlewares, handleCommand, langs })` is called **before** `start()`.
- Replace augmented properties on the discord.js client (`client.commands = new Collection()`, `client.cooldowns`, and so on) with Seyfert's own registries or with module-level state. Do not hang state on `client`.

## Startup: where code goes (checkpoint C1)

discord.js usually crams everything into `client.once('ready')`. In Seyfert, split it by what each piece needs:

| The function needs... | Put it | Why |
|---|---|---|
| Guild/member/role/channel cache, or fetches guild data (restoring role timers, syncing members, a cron job that edits roles) | `src/events/bot-ready.ts` → `createEvent({ data: { name: 'botReady', once: true }, run })` | `botReady` fires after `GUILDS_READY` on every shard, so the cache is populated |
| Only REST, the DB, a queue, or Redis (queue workers, a "bot started" webhook, HTTP servers) | `src/index.ts` after `await client.start()` | Does not need the gateway; failure is visible at boot |
| Nothing from Discord (config validation, DB connection) | `src/index.ts` before `client.start()` | Fail fast before connecting |

```ts
// src/index.ts
import { client } from '~/client'

await connectDb()
await client.start()
startNotificationWorker() // REST-only
```

```ts
// src/events/bot-ready.ts
import { createEvent } from 'seyfert'

export default createEvent({
  data: { name: 'botReady', once: true },
  async run(user, client) {
    await restoreRoleTimers(client) // needs guild + members
    startRoleSyncCron(client)
  },
})
```

- There is also a raw `ready` event that fires **per shard**, before the guilds have arrived. Do not port `Events.ClientReady` to `ready`.
- Always use `once: true` for one-shot startup work.
- A function that sets `setTimeout` timers from persisted state (for example "remove a role at time T") is restore-on-boot logic. It belongs in `botReady`. Ask (C9) whether the live trigger that originally sets those timers still exists.
- Functions that used to take `client: Client` from discord.js now take `UsingClient` (`import type { UsingClient } from 'seyfert'`) or import the singleton `client`.

## Logger (checkpoint C10)

Seyfert's `Logger` (`client.logger`, `seyfert` export `Logger`, `LogLevels`) supports:
- console output
- `saveOnFile`
- one global hook: `Logger.customize((self, level, args) => args | undefined)`

It has **no transports**, and no `verbose`/`http`/`silly` levels (Debug, Info, Warn, Error, Fatal only).

**Keep the existing logger (default).** It is independent of discord.js: only switch its imports (e.g. a webhook transport that used `WebhookClient` → a Seyfert webhook send, gotcha 29). Seyfert's own warnings (e.g. `<cmd> command does not have 'run' callback`) then still go only to the console.

**Switch to Seyfert's `Logger`.** Wrap the default formatter and send webhooks as a side effect:

```ts
import { Logger, LogLevels } from 'seyfert'

const base = Logger.getCustomizer()
Logger.customize((self, level, args) => {
  if (level >= LogLevels.Warn && !isDev) void sendLogWebhook(level, args).catch(() => null) // never throw here
  return base(self, level, args) // keep Seyfert's console format
})
```

Before switching, check:
- **Call shape.** Winston-style calls (`logger.error('msg', { error, ...meta })`) must still render the same in the log channel. Parse the meta object out of `args`.
- **Volume.** The hook is global (every `Logger` instance, including Seyfert's), so filter levels and environments.
- **Call sites.** Keep a module exporting the same `logger.error/warn/info/debug` names so they do not change.
- **Scope.** Do it as its own change after the migration is merged. It touches every log line, not the discord.js removal.

## Loaders

Delete hand-written `readdir` + `import()` loaders for commands and events. `client.start()` loads `locations.*`. Hot reload: `client.commands.reloadAll()` (v6 throws on a bad file).
