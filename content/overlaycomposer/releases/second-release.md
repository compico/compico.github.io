+++
title = "OverlayComposer: a mixer on the Deck, a store for standalone plugins, backups"
date = 2026-11-24T02:11:00+03:00
version = "v2026.11.23.6"
summary = "The Deck gets a full OBS mixer, mute and folder tiles and plugin buttons. Standalone plugins get a store with updates and rollback, chat commands and deprecation notices. Settings can be exported, snapshotted and kept in the cloud."
+++

This release is mostly about two things: the **Deck** becomes a real control panel for the stream, and **plugins** become easy to install, update and trust. Settings are no longer tied to one computer.

## Deck: from scene buttons to a stream panel

### A mixer, like in OBS

The Deck now has a ready-made **Mixer** group. It shows **every audio source that is active in OBS right now**, so when you change the scene and the set of sources changes, the mixer changes with it.

For each source you get:

- a **fader** that works like the one in OBS: the first half of the travel is fine adjustment, the rest is coarser down to silence, and it never goes above 0 dB;
- **mute** and **monitoring** buttons (a tap switches them on and off, the way you expect);
- a **level meter** with green, yellow and red zones, a peak mark, and a red dot when the sound clips, so a problem is visible from across the room.

It runs smoothly even on an old iPad with Safari 9. In the Deck editor press **+ Mixer**; the group has nothing to configure, only its width. On a narrow screen (a tablet held upright) it shrinks to fit.

### New kinds of tiles

- **Mute** tile: pick an audio source from the OBS mixer, and the tile mutes and unmutes it. It lights up while the source is muted.
- **Folder** tile: one tile opens a list of scenes (from 1 to 8). Pick one and the panel switches the scene and closes the list. The tile shows the name of the active scene.
- **Plugin** tiles: a tile can press a button of a standalone plugin. For the music plugin that is *play/pause*, *skip*, *skip and ban*. The tile lights up by the plugin's state (for example, while the music is playing), and a big tile can show **what is playing now**: the title and the artist.

In the editor, **What it does** lists everything: scene, folder, mute and every action that your installed plugins offer. A selected tile is removed with the **Delete** key.

### Faster

The panel used to ask OBS "anything new?" every two seconds. Now the server tells it at once, so highlights, texts and the active scene change immediately. If the connection is lost, the panel falls back to the old way by itself and reconnects.

## Plugins

### A store for standalone plugins

Standalone (WebAssembly) plugins used to be installed by copying a file. Now they live in the same **Plugin store** as widgets, in the **Store** tab of the **Plugins** window:

- **Install, update and remove with a button.** The plugin comes from the release of the repository you added, and the app checks that the file is exactly the one that was published.
- **Any version.** Choose the latest one, a range such as `^1.5`, or an exact version. Going back to an older version is a normal action, with a warning that the newer version may have changed the plugin's saved data.
- **See what changes before you update.** The window lists the changes of *every* version you skipped and the section **What to do after updating**, and shows the new permissions. *Cancel* changes nothing.
- **New permissions are never given silently.** After an update the plugin may ask for more; you decide in **Plugins**.
- **Notifications.** On start and by a button, the app checks installed plugins and puts a note about a new version into the bell, with its changes. It never installs anything by itself.

### The same plugins on another computer

**Export lock** saves the list of installed plugins with their exact versions; **Restore from lock…** installs the same set elsewhere. The app first shows a plan: what can be installed, what is already there, and what is a problem (the release was removed, the file changed, the source is not added) with the reason. Plugin files, permissions and settings are not in the list; the lock only says *which* plugin and *which* release.

### Retired plugins and their replacements

When a plugin is replaced by a better one, the repository can say so. The app then shows **Deprecated** in the store and in the Plugins window, with the reason and a button that leads to the replacement, and the bell tells those who have already installed it.

It is already used: **osu-skills** and **osu-np** are merged into one plugin, **osu-tools**.

### Chat commands

A plugin can now answer commands in the chat, such as `!np`. It receives **only the commands it declared**, not the whole chat, and it needs its own permission, `chat.command.read`. The permission window lists the exact commands, for example `!np`. This is deliberately separate from reading the chat, so you can see at a glance what a plugin can hear. The bot can answer **as a reply** to the viewer's message, and Twitch shows it as a reply.

### Updated plugins

- **music** 0.8 and 0.9: buttons for the Deck (*play/pause, skip, skip and ban*), tile highlight by the player's state and a **Now playing** tile.
- **osu-tools** 1.0: the `!np` command now works on the server, so no open overlay or widget is needed, plus the skills graph; osu!lazer is supported.
- Every plugin release can carry a list of changes and a note of what to do after updating, which is what you see in the update window.

If you write plugins, see the updated guide [Writing plugins](../../docs/plugin-development/): chat commands, Deck buttons, release layout and retiring a plugin.

## Settings you can take with you

**Settings → Backup** is new:

- **Export and import.** One file holds your sources, layouts, Deck, plugin list, store sources and, if you want, OBS address and plugin settings. When you import, nothing changes until you press *Import*: a window shows what is new, what replaces yours and what has a problem, and you tick what to apply. A layout that would replace yours can be imported as a copy.
- **Secrets stay home.** Twitch tokens, passwords, keys, paired devices and the store token are never written to the file. Plugin settings (they can hold tokens) are left out unless you tick them.
- **Local snapshots.** The app saves a snapshot when you close it (only if something changed), keeps a history (30 files by default) and restores any of them through the same import window. A snapshot is also made before every import, so a mistake can be undone.
- **Cloud storage.** Keep the file in any folder that Dropbox, OneDrive, Google Drive for desktop or Syncthing already syncs, or in **Google Drive** directly. For Google Drive the app uses a hidden application folder: the file does not appear among your documents, and the app cannot see anything else in your Drive. Nothing is uploaded or applied by itself; there are only buttons.

The editor also notices changes made in another window or on another device at once, without waiting.

[Download from GitHub Releases](https://github.com/compico/overlaycomposer/releases/tag/v2026.11.23.6) · [Documentation](../../docs/) · [Report a problem](https://github.com/compico/overlaycomposer/issues)
