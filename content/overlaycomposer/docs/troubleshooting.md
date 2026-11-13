---
title: "Troubleshooting"
description: "Checks for common overlay, OBS, Deck and plugin problems."
weight: 8
---

## The overlay is not visible in OBS

1. Open `http://127.0.0.1:8000/overlay?debug=1` in a browser, or append `?debug=1` to the Browser Source URL. A corner label shows the WebSocket state, the layout, the number of objects and the last error.
2. Open **⚙ Settings → Application → Open log folder** and read the latest lines. `client …` is what the overlay reported, `http …` is what OBS requested, `proxy: …` is what a proxied site answered. If `/overlay` never appears in the log, OBS did not reach the server: check the URL and port.
3. Frequent page reloads can make a widget's site answer `Too Many Requests`: wait a bit.

To debug a Browser Source in DevTools start OBS with `--remote-debugging-port=9222 --remote-allow-origins=http://localhost:9222`, then open `http://localhost:9222/` in Chrome.

## A widget refuses to be embedded

Enable **Load via server proxy** for that source. If it still misbehaves, switch on **Settings → Network → Log proxy traffic** and send the log (tokens are masked).

## The scene list is empty or the layout does not switch

Connect obs-websocket: [Connecting to OBS](../obs-setup/). Check **Settings → OBS**: the status line shows the connection and how many scenes are known. In OBS enable *Tools → WebSocket Server Settings → Enable WebSocket server*.

## Animated widgets stutter while a game is running

The likely cause is how Windows shares the GPU between the game and the OBS browser source. What helped in practice: OBS → Settings → Advanced → turn off **Enable Browser Source Hardware Acceleration** and restart OBS. Also try limiting the game FPS, raising OBS priority (Settings → Advanced → Process priority) and setting OBS to *High performance* in Windows Graphics settings. The **fps-meter** plugin from oc-plugins shows FPS and frame time on the same layout, so you can see the numbers.

## A tablet cannot open the editor or the Deck

Check that **Listen on: All network interfaces** is saved and the app restarted, the device is paired ([Settings](../settings/)) and the port is open in the firewall. The proxy port (`port + 1`) must be free too. `/deck/probe` is a compatibility page: it shows what an old browser supports and whether the test WebSocket connects.

## The Deck does nothing when I tap

The page shows a connection state in the corner. A tap fails if OBS is not connected or the scene no longer exists in OBS (rename it in the tile properties). A tile with **second tap** needs two taps.

## Twitch: a plugin cannot read or write the chat

Open **Settings → Twitch**: the account must be connected, with the scopes listed there. After an update that adds scopes the notification center asks to reconnect. Messages written by `chat.send` need the **Bot** account.

## More help

Open an issue in the [OverlayComposer repository](https://github.com/compico/overlaycomposer/issues). Remove tokens, private URLs and other secrets from logs before sharing them.
