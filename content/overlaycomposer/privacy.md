---
title: "Privacy Policy"
description: "What data OverlayComposer handles, where it stays, and how the optional Google Drive backup works."
---

*Last updated: 21 November 2026.*

OverlayComposer is a program that runs on your own computer. It has no accounts, no developer-operated servers and no analytics. **The developer does not receive, collect or store any of your data.** This page explains what the program keeps on your computer and what it sends to third parties on your behalf.

## What stays on your computer

Layouts, widget sources, settings, plugin data and the connections you configure (OBS address and password, Twitch accounts) are stored in the program's data folder on your computer. Passwords and tokens are encrypted with a key that is stored next to them. Nothing is uploaded unless you use one of the features below.

## What the program sends, and where

- **Twitch.** If you connect a Twitch account, the program talks to Twitch directly to read chat and channel events and to send messages you ask it to send. The access tokens stay on your computer.
- **GitHub.** The program checks GitHub for new versions (this can be switched off in `config.json`) and downloads plugins from the GitHub repositories you add to the plugin store.
- **Websites you add as widget sources.** Those sites receive the requests your browser source makes, like any web page.
- **Google Drive (optional).** Only if you press “Connect” in Settings → Backup → Cloud storage. See below.

## Third-party plugins and widgets

Plugins and widget sources written by other people are separate software, not part of OverlayComposer. What they do with data (for example, which websites they contact) is decided by their authors and the permissions you grant them in the program, and is described in their own documentation or privacy policy. OverlayComposer does not give plugins access to your Google Drive connection.

## Google Drive backup

The backup feature saves a settings file to a folder named “Overlay Composer” in **your** Google Drive and reads it back when you ask. It uses the `drive.file` permission: the program can see and change only the files it created itself, and cannot see anything else in your Drive.

- Files travel directly between your computer and Google. They do not pass through any server of the developer.
- The only Google data the program stores is an OAuth refresh token, kept encrypted on your computer. “Disconnect” in the same screen deletes it. You can also revoke access at any time in your [Google Account permissions](https://myaccount.google.com/permissions).
- The settings file contains layouts, source addresses, deck panel, the list of plugins and similar settings. It never contains Twitch accounts, OBS passwords, or store tokens. The optional “Plugin settings” part, which is off by default, can contain tokens or addresses that a plugin keeps in its settings: tick it only if you want them copied to your Drive.
- Google user data is used only to provide this backup feature. It is not shared with anyone, not used for advertising, and not read by people.

OverlayComposer's use and transfer of information received from Google APIs adheres to the [Google API Services User Data Policy](https://developers.google.com/terms/api-services-user-data-policy), including the Limited Use requirements.

## Children

The program is not directed at children and does not knowingly process their data.

## Changes and contact

If this policy changes, the date above changes. Questions: open an issue at [github.com/compico/overlaycomposer](https://github.com/compico/overlaycomposer/issues).
