---
title: "Widgets and plugins"
description: "Add widgets, install plugins from the store, and manage standalone plugins."
weight: 6
---

There are three ways to get a widget onto a layout. All start with **+ Add Source**.

## 1. A web page (iframe)

Enter the URL of any widget (the same address you would give to an OBS Browser Source). If the site refuses to be embedded (alerts of some donation services), tick **Load via server proxy**: the app opens the page through itself and strips the headers that forbid embedding. HTTP, WebSocket and long-poll work through it.

## 2. A local HTML file

**Add Source → HTML file**: choose a `.html` file (CSS, JS and images next to it are picked up) or a whole folder. The app creates a plugin without permissions in `plugins/local-<name>/` and puts it on the scene. Edits to those files are picked up when the widget page reloads. Up to 64 files and 6 MB; with several HTML files name the main one `index.html`.

## 3. Plugins

A plugin is a ready-made widget with settings and, if needed, access to Twitch chat, storage or a server part.

**Add Source → Plugin store…** lists plugin repositories. The default is [compico/oc-plugins](https://github.com/compico/oc-plugins); you can add any repository as `owner/repo`, `owner/repo@branch-or-tag` or a github.com link. The catalog shows each plugin's version, requested permissions and the commit it is built from, and installs, updates and removes plugins with a button.

If a plugin needs a newer plugin API than your Overlay Composer has (the version is shown in the settings as **Plugin API**), the store shows **Update the app** instead of **Install**: update Overlay Composer first. An update is offered when the version in the repository is higher (or the files changed). New permissions are confirmed separately. To pin the code use a tag or a commit: `owner/repo@v1.2.0`. For private repositories add a fine-grained GitHub token (`Contents: Read-only`) in the store dialog; it is stored encrypted and sent only to `api.github.com`.

Installed widgets appear in **Add Source → Widgets**. After adding one, select its layer and edit its settings in the right panel (only the values you changed are saved; ↺ resets to the default).

### Permissions

A widget asks for what it needs; the list is shown when you add it and can be changed in the layer properties. Without a permission the call is rejected.

| Permission | What it allows |
|---|---|
| `chat.read` | read the chat of a Twitch channel |
| `chat.send` | write to the chat as the bot or the streamer |
| `storage` | keep JSON data of the plugin on this computer |
| `net.local` | GET requests to programs on this computer (for example tosu) |

A widget is third-party code. It runs in a sandboxed iframe with no access to the app API, but everything you permit it does on your behalf: install plugins only from authors you trust.

## Standalone plugins (WebAssembly)

Besides widgets there are **standalone** plugins: one `.wasm` file that runs inside the app, with no window of its own. It answers HTTP requests, receives events (for example a channel point reward), stores data, writes to the chat as the bot, publishes events to other programs, and can bring its own widgets and a settings page.

Install: open **🧩 Plugins** in the header and go to the **Store** tab. It lists the standalone plugins of the repositories you added; **Install** downloads the release and checks that the file is exactly the published one. You can also download `<id>.wasm` from the plugin's release (for [compico/oc-plugins](https://github.com/compico/oc-plugins/releases) the tags are `<id>-v<version>`) and put it into the plugins folder; **Rescan** picks up new files. In the **Installed** tab the plugin card lets you grant permissions, edit settings and restart it.

A new plugin gets **no permissions** until you tick them.

### Updates and rollback

- On start and by a button the app checks installed plugins; a new version appears in the notification center (the bell). Nothing is installed by itself.
- Before updating the window shows the changes of **every** version you skipped and, where the author wrote it, **What to do after updating**. If the new version asks for more permissions, they are listed and stay **not granted** until you grant them.
- You can pick any version: the latest, a range such as `^1.5`, or an exact one. Going back to an older version is possible; the newer one may have changed the plugin's saved data, so the window warns you.
- **Export lock** saves the list of installed plugins with exact versions, **Restore from lock…** installs the same set on another computer after showing a plan. Files, permissions and settings are not in it. (The same list is part of the settings backup.)
- A plugin marked **Deprecated** has been replaced by its author; the card shows the reason and a button to the replacement.

![Settings page of the music requests plugin](/overlaycomposer/img/plugin-settings.png)
*The settings page of the `music` plugin (track requests for channel points): the reward, the player, the queue and the ban list.*

| Permission | What it allows |
|---|---|
| `storage` | keep data |
| `net.http` | outgoing HTTP requests (never to loopback or private networks) |
| `twitch.redemption` | receive reward redemptions and fulfil or cancel them |
| `timer` | host timers |
| `events.publish` | publish events to subscribers |
| `chat.send` | write to the chat of the streamer's channel as the bot, also as a reply to a message |
| `chat.command.read` | receive the chat commands (`!np`…) that the plugin lists; the permission window shows exactly which. The rest of the chat stays hidden |
| `chat.read` | read **the whole chat** of the streamer's channel: every viewer's message, not only commands (for chat widgets); the window says so explicitly |
| `net.local` | talk to a program on this computer at the addresses the plugin lists (for example `127.0.0.1:24050`); the window shows them |

Giving or changing permissions restarts the plugin. A plugin that crashes 5 times in a minute is stopped until you restart it. Everything the plugin logs goes to the app log tagged `plugin=<id>`.

## Write your own

See [Writing plugins](../plugin-development/): HTML widgets and standalone plugins in Go.
