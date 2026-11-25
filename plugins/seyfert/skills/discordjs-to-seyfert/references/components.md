# Components: collectors vs ComponentCommand (checkpoint C5)

## Choose per flow

| Flow | Use | Why |
|---|---|---|
| Interactive reply that only matters for a few minutes (pagination, confirm/cancel, wizard, "pick one" after a command) | **Collector** on the sent message | Same lifetime as the discord.js collector; the code stays next to the command |
| Message posted once and expected to keep working forever (role panels, setup panels, ticket buttons, buttons on long-lived messages) | **`ComponentCommand`** in `locations.components` | Collectors are in-memory and die on restart; `ComponentCommand` matches any message by customId |

The default is the collector. Ask the human about each candidate for `ComponentCommand` and list them in `SEYFERT_MIGRATION.md`. Hints that a flow is persistent:
- a `*-setup` / panel command posts the message
- the customId encodes data (`role-<id>`)
- today's handler lives in a global `interactionCreate` router rather than a collector

**Keep customIds byte-identical.** Messages that were already posted carry the old ids.

## Collector port

```ts
// discord.js
const msg = await interaction.reply({ components: [row], fetchReply: true })
const collector = msg.createMessageComponentCollector({ time: 60_000, filter: (i) => i.user.id === user.id })
collector.on('collect', async (i) => { if (i.customId === 'next') await i.update(...) })
collector.on('end', () => msg.edit({ components: [] }))

// Seyfert
const msg = await ctx.write({ components: [row] }, true) // or client.messages.write(...)
const collector = msg.createComponentCollector({
  timeout: 60_000, // discord.js `time`
  idle: 30_000, // discord.js `idle`
  filter: (i) => i.user.id === ctx.author.id,
  onStop: async (reason) => { await msg.edit({ components: [] }).catch((): null => null) },
})
collector.run('next', async (i) => { await i.update({ ... }) }) // customId: string | string[] | RegExp
collector.run(/^page-\d+$/, async (i) => { /* ... */ })
```

- `awaitMessageComponent({ filter, time })` → `const i = await collector.waitFor('confirm', 30_000)` (resolves `null` on timeout).
- `collector.stop(reason)` and `collector.resetTimeouts()` exist. There is no `collected` collection. Keep your own counter if the code used `collected.size`.
- Only **one** collector per message is tracked (keyed by message id). Merge discord.js code that attached two collectors to the same message.
- Interaction methods inside `run`: `i.update(body)`, `i.deferUpdate()`, `i.write(body)`, `i.deferReply()`, `i.editOrReply(body)`, `i.followup(body)`, `i.modal(modal)`. Check select values with `i.isStringSelectMenu() && i.values`.

## ComponentCommand port (replaces global customId routers)

```ts
// src/components/role-panel.ts
import { ComponentCommand, type ComponentContext } from 'seyfert'

export default class RolePanelButton extends ComponentCommand {
  componentType = 'Button' as const // 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'ChannelSelect' | 'MentionableSelect'

  override filter(ctx: ComponentContext<typeof this.componentType>) {
    return ctx.customId.startsWith('role-panel-')
  }

  async run(ctx: ComponentContext<typeof this.componentType>) {
    const roleId = ctx.customId.slice('role-panel-'.length)
    await ctx.client.members.addRole(ctx.guildId!, ctx.author.id, roleId).catch((): null => null)
    await ctx.write({ content: 'Done', flags: 64 }).catch((): null => null)
  }
}
```

- Write one class per customId prefix (or per component type). Delete the `interactionCreate` switch.
- Collectors are checked first. A `ComponentCommand` only receives interactions whose message has no live collector matching that customId.
- **Every** `ComponentCommand` whose `componentType` and `filter` match will run. There is no first-match-wins (`executeComponent` loops over all of them, 6.0.0). Make the prefixes in the filters mutually exclusive: `'role-'` also matches `'role-panel-'`.
- `customId` may be a property (`customId = 'exact-id'` or a RegExp) instead of `filter`.
- Middlewares work the same way as on commands (`@Middlewares([...])`), and so do cooldowns.

## Modals

```ts
import { Label, Modal, TextInput } from 'seyfert'
import { TextInputStyle } from 'seyfert/lib/types'

const modal = new Modal()
  .setCustomId('feedback')
  .setTitle('Feedback')
  .setComponents([new Label().setLabel('Text').setComponent(new TextInput().setCustomId('text').setStyle(TextInputStyle.Paragraph))])
  .run(async (m) => { const text = m.getInputValue('text', true); await m.write({ content: text }) })

await ctx.modal(modal) // or i.modal(modal)
```

`awaitModalSubmit` → `.run(...)` on the modal, or a `ModalCommand` class (`extends ModalCommand`, `filter`/`customId`) for persistent modals. In Seyfert 6 a `Label` wraps each `TextInput`; it replaces the discord.js pattern of one action row per text input.
