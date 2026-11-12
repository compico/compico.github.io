---
title: "Connecting to OBS"
description: "Configure one Browser Source for your OverlayComposer canvas."
weight: 2
---

# Connecting OverlayComposer to OBS

1. Start OverlayComposer.
2. In OBS, add a **Browser Source**.
3. Set its URL to http://127.0.0.1:8000/overlay.
4. Set its width and height to match your OBS canvas, for example 1920 × 1080.
5. Add that same Browser Source to the other scenes where you want the overlay.

OverlayComposer switches the visible layout when the active OBS scene changes. If a scene does not have its own layout, the default layout is shown.

The default server listens on 127.0.0.1:8000. You can change the listen address and port in Settings. Only expose the server to your local network if you understand the security implications; the app does not currently provide user authentication.
