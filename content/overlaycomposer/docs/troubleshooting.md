---
title: "Troubleshooting"
description: "Checks for common overlay and OBS issues."
weight: 5
---

# Troubleshooting

## The overlay is not visible in OBS

1. Open http://127.0.0.1:8000/overlay?debug=1 in a browser or append ?debug=1 to the OBS Browser Source URL.
2. Check the status shown in the overlay.
3. Open Settings → Open log folder in OverlayComposer and inspect recent entries.
4. Confirm that the Browser Source URL and port match the app's settings.

## Widgets stutter while a game is running

On some Windows setups, OBS Browser Source hardware acceleration can compete with a game for GPU resources. As a test, open OBS → Settings → Advanced, disable Browser Source Hardware Acceleration, and restart OBS.

## More help

Open an issue in the [OverlayComposer repository](https://github.com/compico/overlaycomposer/issues). Remove tokens, private URLs, and other sensitive information from logs before sharing them.
