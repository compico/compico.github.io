---
title: "Deck: control panel for a tablet"
description: "Big Metro-style tiles that switch OBS scenes from a tablet or an old iPad, laid out in the editor."
weight: 5
---

**Deck** is a page at `http://<computer IP>:<port>/deck`: a start screen with large coloured tiles. A tap on a tile switches the OBS scene. It works in old browsers too (an iPad with Safari 9 is supported), so a retired tablet becomes a stream remote.

![The Deck page: groups of tiles with horizontal scrolling](/overlaycomposer/img/deck.png)
*Groups of tiles scroll horizontally. The white frame marks the current scene; the small square in the corner shows the connection state.*

## Open it on the tablet

1. Allow other devices: [Settings → Network](../settings/), *All network interfaces*, restart.
2. On the tablet open `http://<computer IP>:<port>/pair?next=/deck`, enter the code from Settings → Remote devices. After pairing the Deck opens.
3. Connect obs-websocket ([Connecting to OBS](../obs-setup/)), otherwise there is nothing to switch.

The layout is a grid of **4 rows** of square cells; the screen height is divided by four, so it looks the same in any orientation. Groups have titles and are laid out left to right.

## Tile sizes

| Size (w×h cells) | Use |
|---|---|
| 1×1 | a small button |
| 2×1 | a wide scene button (the default) |
| 3×1 | a long title |
| 2×3 | a big tile |

## The editor

Press **▦ Deck** in the editor header. The editor turns into the Deck editor (**← Overlay editor** returns).

![The Deck editor: groups and tiles on the left, canvas in the middle, tile properties on the right](/overlaycomposer/img/deck-editor.png)

- **Left:** the groups, **+ Group**, buttons **+ 1×1 / + 2×1 / + 3×1 / + 2×3** that add a tile to the current group (the first scene not used yet), and the list of the group's tiles.
- **Center:** the canvas. Drag tiles to move them, also between groups. A tile that does not fit is not moved.
- **Right:** the selected tile: title, the scene to switch to, **Ask for a second tap**, size (sizes that do not fit are disabled), colour, icon. With no tile selected it shows the group: title, order, delete.
- **Save deck** in the header applies the layout; **Reset to default** makes one tile per OBS scene.

If the layout was changed elsewhere in the meantime, saving offers to reload it. Closing with unsaved changes asks for confirmation. If OBS gives no scene list the scene field is a text box: type the scene name exactly as in OBS.

### Risky scenes

Tick **Ask for a second tap** on a tile (the screenshot shows it as `confirm`): the first tap only arms the tile and the second one switches the scene, so a stray touch cannot cut your stream.

## Security

The Deck needs a paired device like the editor. A tap sends only the tile id: the action always comes from the saved layout, so a forged request cannot ask for an arbitrary scene, and the scene must exist in OBS. Taps are rate limited.

The layout itself is stored in OverlayComposer; the Deck page and the editor read the same data.
