# Commands

## Class shape (slash and prefix share it)

```ts
import { Command, type CommandContext, createStringOption, createUserOption, Declare, IgnoreCommand, Options, type PermissionStrings } from 'seyfert'

const options = {
  user: createUserOption({ description: 'target', required: true }),
  reason: createStringOption({ description: 'why' }),
}

@Declare({
  name: 'warn',
  description: 'Warn a member',
  aliases: ['w'], // prefix only
  defaultMemberPermissions: ['ManageMessages'], // PermissionStrings, type-checked
  botPermissions: ['SendMessages'],
  ignore: IgnoreCommand.Slash, // prefix-only. IgnoreCommand.Message = slash-only. Omit = both
  // guildId: ['123'],          // guild-scoped slash registration
})
@Options(options)
export default class WarnCommand extends Command {
  override async run(ctx: CommandContext<typeof options>) {
    ctx.options.user // typed from `options`
    await ctx.write({ content: 'done' }).catch((): null => null)
  }

  override async onPermissionsFail(ctx: CommandContext, missing: PermissionStrings) {
    await ctx.write({ content: `Missing: ${missing.join(', ')}` }).catch((): null => null)
  }
}
```

| discord.js | Seyfert `CommandContext` |
|---|---|
| `interaction.reply(x)` / `message.reply(x)` | `ctx.write(x)` (prefix + `commands.reply: () => true` → real reply) |
| `reply(...)` then use the returned message | `const msg = await ctx.write(x, true)` (withResponse) |
| `deferReply()` | `ctx.deferReply(ephemeral?)` |
| `editReply(x)` | `ctx.editOrReply(x)` / `ctx.editResponse(x)` |
| `followUp(x)` | `ctx.followup(x)` |
| `interaction.user` / `message.author` | `ctx.author` |
| `interaction.member` | `ctx.member` |
| `interaction.guildId` / `channelId` | `ctx.guildId` / `ctx.channelId` |
| `interaction.guild` | `await ctx.guild()` (REST/flow) or `ctx.guild('cache')` |
| `message` (prefix) | `ctx.message` (needs `InternalOptions.withPrefix: true`; `undefined` for slash) |
| `{ ephemeral: true }` | `{ flags: MessageFlags.Ephemeral }` (`MessageFlags` from `seyfert/lib/types`) |

Message bodies use API snake_case (`allowed_mentions`, `message_reference`), and builders go in as-is (`embeds: [new Embed()]`).

## Prefix commands (checkpoint C3)

Enable on the client:

```ts
new Client({
  commands: {
    prefix: (message) => (message.author.bot ? [] : [process.env.PREFIX!]),
    reply: () => true, // reply to the triggering message (default: plain channel send)
  },
})
```

- **Seyfert does not ignore bot authors** in prefix handling (`HandleCommand.message` has no `author.bot` check in 6.0.0). Return `[]` from `prefix` for bots, as above, or for whatever the old handler filtered (DMs, webhooks, specific channels).
- Prefix matching is case-sensitive, and the command name comes from the first 1-3 words (to support subcommands).
- The default `argsParser` expects flags: `!cmd -user @x -reason spam`. discord.js bots usually split on spaces into `args: string[]`. To keep that UX, subclass `HandleCommand` (import from `seyfert/lib/commands/handle`; it is not re-exported from `seyfert` in 6.0.0):

```ts
import type { Command, MessageStructure, SubCommand } from 'seyfert'
import { HandleCommand } from 'seyfert/lib/commands/handle'

/** Space-split args → declared options in declaration order; the last option takes the remainder. */
export class PositionalHandleCommand extends HandleCommand {
  override argsParser(content: string, command: SubCommand | Command, _message: MessageStructure) {
    const words = content.split(' ').filter(Boolean)
    const options = command.options ?? []
    const args: Record<string, string> = {}
    options.forEach((option, i) => {
      if (i >= words.length) return
      args[option.name] = i === options.length - 1 ? words.slice(i).join(' ') : words[i]
    })
    return args
  }
}

client.setServices({ handleCommand: PositionalHandleCommand })
```

  Verified on 6.0.0: `'alpha beta gamma'` with options `target, rest` → `{ target: 'alpha', rest: 'beta gamma' }`.
  - Options are resolved by type after parsing: user, channel and role options accept mentions or IDs, and boolean accepts `yes|y|true`.
  - A required option that is missing calls `onOptionsError`.
  - The string returned for an option is still raw, so it is the place for any quoting rules the old bot had.
- When a command took a free-form `args: string[]` and did its own parsing, the least invasive port is a single last string option (`args`), then `ctx.options.args?.split(' ') ?? []` inside `run`. Ask at C3 whether to do this or to model real options.
- Slash commands and prefix commands share the class. A prefix-only command must have `ignore: IgnoreCommand.Slash`, or registration will try to upload it.

## Permissions (checkpoint C4)

For prefix commands, Seyfert checks `defaultMemberPermissions` against the member's **guild-level** permissions (`client.members.permissions`). The guild owner always passes, and a failure only calls `onPermissionsFail`. With no handler, the command does nothing and sends no reply.

discord.js bots often checked **channel-level** permissions (`message.member.permissionsIn(channel)` / `channel.permissionsFor`). If behavior must stay identical, do one of these:
- override `checkMemberPermissions(command, context, permissions)` in the custom `HandleCommand` to compute channel permissions (`await client.channels.memberPermissions(channelId, member)`), or
- keep the check in a middleware.

Record which one you chose. `PermissionFlagsBits.X` → the string `'X'` in `PermissionStrings`.

## Cooldowns (checkpoint C4)

The official plugin is `@slipher/cooldown` (2.x for Seyfert 6). It does not need a module-level `Collection`.

```ts
import { cooldown } from '@slipher/cooldown'
import { Client, definePlugins } from 'seyfert'

const plugins = definePlugins(cooldown({ middleware: true }))
export const client = new Client({ plugins })
declare module 'seyfert' { interface SeyfertRegistry { plugins: typeof plugins } }
```

```ts
import { Cooldown } from '@slipher/cooldown'
import { Middlewares } from 'seyfert'

@Cooldown.user(5_000) // also .guild / .channel / .global / .custom(resolver, ms)
@Middlewares(['cooldown'])
export default class X extends Command { /* ... */ }
```

A command with no cooldown before must stay without one. Record any cooldown changes as decisions.

## Registration (checkpoint C2) - irreversible risk

`client.uploadCommands()` does a **bulk PUT** of global commands, plus a PUT per guild for guild-scoped ones. Discord **deletes every command not in the body**. This deletes commands registered by other code under the same application id.

- **Application id used only by this bot:** `await client.uploadCommands({ cachePath: './commands-cache.json' })` once `start()` resolves. `cachePath` skips the upload when nothing changed.
- **Shared application id** (another codebase or service registers commands under the same id): never call `uploadCommands`. Upsert per command. A POST with an existing name overwrites that command and leaves the others alone:

```ts
for (const command of client.commands.values) {
  if ('ignore' in command && command.ignore === IgnoreCommand.Slash) continue // ContextMenuCommand has no `ignore`
  await client.proxy.applications(client.applicationId).guilds(GUILD_ID).commands.post({ body: command.toJSON() })
}
```

Deleting a command that is gone from the code is a separate step that the human approves (DELETE by command id).

## Context menus, autocomplete, subcommands

- Context menu: `extends ContextMenuCommand` with `@Declare({ type: ApplicationCommandType.User | Message })`.
- Autocomplete: `createStringOption({ autocomplete: async (interaction) => interaction.respond([...]) })`. It lives on the option, so the `interactionCreate` switch goes away.
- Subcommands: `extends SubCommand`, attached via `@Options([Sub1, Sub2])` on the parent, or with `@AutoLoad()` and a directory. Read https://www.seyfert.dev/docs/learn/commands/subcommands before porting.
