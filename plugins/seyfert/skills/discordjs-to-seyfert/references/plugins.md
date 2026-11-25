# Seyfert plugins (checkpoint C11)

Seyfert has official plugins under `@slipher/*`, wired with `definePlugins(...)` and `new Client({ plugins })`. For each library or hand-written subsystem the bot already has, ask whether to keep it or move to the plugin. **Default: keep what works and migrate only discord.js in this pass.** Exception: `@slipher/cooldown` when C4 replaces a hand-written cooldown. Plugin adoption is a separate change after the migration is merged.

Checked on npm 2026-11 against Seyfert 6.0.0. Most `@slipher/*` peer on `seyfert >=5.0.0` but were built and tested on 5.2. Treat v6 support as unverified until the gates and a smoke test pass. Re-check versions and peers with `npm view <pkg> version peerDependencies` before recommending one.

## Inventory

```bash
jq -r '.dependencies + .devDependencies | keys[]' package.json \
  | rg '^(winston|pino|bunyan|bullmq|bull|bee-queue|agenda|cron|node-cron|croner|node-schedule|ioredis|redis|keyv|chokidar|nodemon)$'
rg -ln "setInterval\(|CronJob|new Queue\(|new Worker\(|createLogger\(" src
```

## Mapping

| Existing | Plugin | Recommend | Notes |
|---|---|---|---|
| hand-written cooldown map | `@slipher/cooldown` | **yes (C4)** | `@Cooldown.user(ms)` + `@Middlewares(['cooldown'])`. Verified on v6. Gotcha 44 covers the custom cooldown text. |
| winston / pino / custom webhook transport | `@slipher/logger` | ask (C10), later | `LoggerAdapter { write(entry) }` lets you write a webhook transport in ~30 lines. Has pino/evlog adapters and per-command context (command, guildId, userId). It does not capture Seyfert's internal `client.logger` warnings: add a `Logger.customize` hook for those. Old `logger.error(msg, meta)` calls need mapping. |
| `cron` / `node-cron` / `setInterval` jobs | `@slipher/scheduler` | ask, low value for a few jobs | `@Cron(expr)` / `@Interval('5m')`, `memory()` or BullMQ `persistent()` driver. Persistent tasks need stable ids, and removing one from code does not remove it from Redis. |
| BullMQ queues/workers | `@slipher/queues` | **usually no** | Typed `@Processor`/`@Process`. Existing queues often have custom job ids, dedup, retry and late-job rules, and jobs already in Redis under the old queue name. Moving them is a data migration, not a refactor. |
| no command tests | `@slipher/testing` | optional | Tests commands and components without a gateway. Useful to replace part of the manual smoke test. |
| in-memory cache | `@slipher/redis-adapter` | only for multi-process or workers | A single-process bot is faster with `LimitedMemoryAdapter`. |
| `chokidar`/`nodemon` reload | `@slipher/watcher` | check the peer | It peered on `seyfert ^5` at check time. |
| text-command args parsers | `yunaforseyfert`, `@konkonbot/parser` (community) | no, by default | They change argument syntax. A custom `argsParser` keeps the old behavior. `yunaforseyfert` peered on `^5.1`. |
| general utils | `@fallencodes/seyfert-utils` (community) | check the peer | It peered on `<6.0.0` at check time. |

## Writing your own plugin

Do not extract the migration's helpers into a plugin while only one bot uses them: the API would be guesswork. Typical helpers are POST-only command registration, a case-insensitive prefix handler, a reaction waiter and formatter copies. When a second bot migrates and needs the same pieces, extract them into a shared package. Publish only the generally useful ones (e.g. the reaction waiter, POST-only registration).
