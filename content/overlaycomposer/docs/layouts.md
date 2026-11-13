---
title: "Layouts and editor"
description: "Arrange widgets on a canvas and give every OBS scene its own layout."
weight: 3
---

A **layout** is a set of layers (widgets) on a canvas of a fixed size. Each OBS scene can have its own layout; scenes without one show the **default** layout.

## Layouts

The left panel lists layouts. A row shows how many scenes the layout is bound to, and **live** marks the one that is on air now. **+ New**, **Duplicate** and **Delete** manage them. In the right panel you name the layout, mark it as default, choose the OBS scenes it belongs to (you can also add a scene by name) and set the canvas size.

## Layers

- Click a layer to select it, drag to move it, drag the corners to resize, use the handle on top to rotate.
- **Alt+click** selects the next layer under the cursor when layers lie on top of each other.
- Hold **Alt** and drag a handle to **crop** like in OBS; exact values are in the *Crop* section.
- The right panel edits position, size, rotation, opacity and scale. Double-click a name to rename a layer, the eye hides a layer.
- **Shift** while resizing keeps the aspect, while rotating snaps to 15°. Arrow keys nudge the layer (**Shift** = 10 px).

## Copy between layouts

Select a layer and press **Copy to layout…**, or **Ctrl+C**, switch layouts and **Ctrl+V**. A *linked* copy is the same widget in another place: it does not reload when the scene changes, so alerts and chat keep their state. Untick **Keep linked** for an independent copy. **Ctrl+V** in the same layout always makes an independent copy.

## History, layout mode, snapping

- **Ctrl+Z**, **Ctrl+Shift+Z** (or **Ctrl+Y**) and the ↶ ↷ buttons undo and redo. Quick consecutive edits merge into one step; the history survives saving.
- **Layout mode** (button or **L**) draws thin dashed outlines with names around all widgets and dims the content. It is handy for transparent widgets, and it is the mode in the screenshot on the first page.
- **Snap** aligns edges and centers to other widgets and the canvas, with pink guides. Hold **Ctrl** to bypass it for one gesture.

Save with **Save (Ctrl+S)**. Unsaved changes are marked in the header.
