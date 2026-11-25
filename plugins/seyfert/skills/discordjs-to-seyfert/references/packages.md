# discord.js sub-packages and transitive dependencies

Removing `discord.js` is not one `bun remove`. Inventory every package from the discord.js family that the bot imports **directly**, plus the ones it only gets **transitively**, and give each a replacement before phase 8.

## Inventory

```bash
# direct deps from the family
jq -r '.dependencies + .devDependencies | keys[]' package.json | rg '^(discord\.js|@discordjs/|discord-api-types)'
# where each is imported
rg -n "from ['\"](@discordjs/[a-z-]+|discord-api-types[^'\"]*)['\"]" src
# bare imports not declared in package.json (only resolve through discord.js or another dep)
#   run a small script: collect `from '<pkg>'` specifiers in src, drop node:/relative/alias, diff with package.json
```

Record each package in `SEYFERT_MIGRATION.md` → phase 8 with its replacement.

## Replacements (Seyfert 6.0.0)

| Package | Seyfert / replacement | Notes |
|---|---|---|
| `discord.js` | `seyfert` | the migration itself |
| `@discordjs/formatters` | local `utils/format.ts` (copy the functions you use) | Seyfert `Formatter` has no `userMention`/`roleMention`/`channelMention`/`chatInputApplicationCommandMention`, and `codeBlock` / `messageLink` take arguments in a different order. Copy the used functions, compare outputs against the package once, then remove it (gotcha 48). |
| `@discordjs/rest` | `client.proxy` / shorters (`client.messages`, `client.roles`, …) | checkpoint C7, `rest-webhooks.md`. `Routes.x(...)` → proxy path chain. A standalone script without a client can use Seyfert's `ApiHandler`. |
| `@discordjs/builders` | `Embed`, `ActionRow`, `Button`, `StringSelectMenu`, `Modal`, `AttachmentBuilder` from `seyfert` | `builders.md`. Slash builders → `@Declare` + `@Options` classes. |
| `@discordjs/collection` | `Collection` / `LimitedCollection` from `seyfert`, or a plain `Map` | Not a drop-in: no `first()`, among others. Prefer a `Map` or array where the code only needs get/set/iterate. |
| `@discordjs/ws` | built into the Seyfert client (`client.gateway`) | standalone gateway use is out of scope |
| `@discordjs/voice` | out of scope (checkpoint C8) | Seyfert has no built-in voice |
| `@discordjs/util`, `@discordjs/rest` helpers (`makeURLSearchParams`) | `URLSearchParams` / proxy `query` option | |
| `discord-api-types` (`/v10`) | `seyfert/lib/types` | same names (`APIEmbed`, `RESTPost…`, `ButtonStyle`, `PermissionFlagsBits`, …). Remove the direct dependency. |

## Transitive packages

discord.js pulls in `undici`, `ws`, `lodash.snakecase`, `@sapphire/*`, `tslib` and more. Code that imports one of these without declaring it keeps working until `bun remove discord.js`, then breaks **at runtime**. `tsc` may still pass if types come from elsewhere (gotcha 47; e.g. `import _ from 'lodash'` in a logger transport). For each such import, either add the package to `package.json` or switch to a declared one (`lodash` → `lodash-es`).

Packages that come through *other* dependencies (e.g. `winston-transport` via `winston`) are unaffected. Note them anyway.

## Phase 8 gate

```bash
rg -n "discord\.js|@discordjs/|discord-api-types" src package.json   # only comments / local file names
bun install && <typecheck> && <tests> && <load check>                  # after the removal, not before
```
