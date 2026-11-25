# Builders

All of these come from `'seyfert'`. Enums (`ButtonStyle`, `MessageFlags`, `TextInputStyle`, `ChannelType`, ...) come from `'seyfert/lib/types'`.

| discord.js | Seyfert | Notes |
|---|---|---|
| `EmbedBuilder` | `Embed` | Same setter names: `setTitle`, `setDescription`, `setColor`, `setAuthor`, `setFooter`, `setFields`, `setImage`, `setThumbnail`, `setTimestamp`, `setURL`. Plus `addFields(...)`. No `spliceFields`: rebuild with `setFields`. |
| `ActionRowBuilder<ButtonBuilder>` | `ActionRow<Button>` | `.addComponents(...)` / `.setComponents([...])` |
| `ButtonBuilder` | `Button` | `.setCustomId().setLabel().setStyle(ButtonStyle.Primary).setEmoji().setDisabled()`; for link buttons use `.setURL()` with `ButtonStyle.Link` |
| `StringSelectMenuBuilder` + `StringSelectMenuOptionBuilder` | `StringSelectMenu` + `StringSelectOption` | `.setCustomId().setPlaceholder().setOptions([...])` / `.addOption()`, `.setValuesLength({ min, max })`; option: `.setLabel().setValue().setDescription().setEmoji().setDefault()` |
| `RoleSelectMenuBuilder` / `UserSelectMenuBuilder` / `ChannelSelectMenuBuilder` / `MentionableSelectMenuBuilder` | `RoleSelectMenu` / `UserSelectMenu` / `ChannelSelectMenu` / `MentionableSelectMenu` | |
| `ModalBuilder` + `TextInputBuilder` | `Modal` + `Label` + `TextInput` | see `components.md` |
| `AttachmentBuilder(buffer, { name })` | `new AttachmentBuilder().setName('a.png').setFile('buffer', buf)` | `setFile('url' \| 'path' \| 'buffer', value)`; pass via `files: [...]`; reference it as `attachment://a.png` |
| `ContainerBuilder`, `TextDisplayBuilder`, `SectionBuilder`, `SeparatorBuilder`, `MediaGalleryBuilder`, `ThumbnailBuilder`, `FileBuilder` | `Container`, `TextDisplay`, `Section`, `Separator`, `MediaGallery`, `Thumbnail`, `File` | Components v2: needs `flags: MessageFlags.IsComponentsV2`. Media items are `{ url }`. |
| `PollBuilder` | `Poll` | |
| `ButtonBuilder.from(component)` / `ActionRowBuilder.from(row)` | `fromComponent(apiComponent)` | Used to rebuild a received message's components (for example to disable them) |
| `PermissionFlagsBits.X` | the string `'X'` (`PermissionStrings`), or `PermissionFlagsBits` from `seyfert/lib/types` for bitfield math | |
| `Colors.Red` / `resolveColor` | `EmbedColors.Red`, or the number or hex string to `setColor` | |
| `time(date, 'R')`, `userMention`, `roleMention`, `channelMention` | `Formatter.timestamp(date, TimestampStyle.RelativeTime)`, or the template strings `<@id>` `<@&id>` `<#id>` | |
| `Collection` | `Collection` from `'seyfert'` (Map subclass), or a plain `Map` | |
| `embed.data.description` (read a built embed) | `embed.data.description` | `Embed.data` is the API object (snake_case) |
| A received embed (`message.embeds[0].description`) | `message.embeds[0].description` | Received embeds are structures with camelCase getters. Check field access per use. |

Rules:
- Builder setter **arguments** usually match discord.js. **Raw objects** passed in place of builders must be API snake_case (`custom_id`, `style`, `emoji: { name }`).
- `MessageFlags.Ephemeral` (64) replaces `ephemeral: true`.
- Do a mechanical rename pass on the import lines and constructors first, then typecheck. The compiler finds the methods with a different name.
