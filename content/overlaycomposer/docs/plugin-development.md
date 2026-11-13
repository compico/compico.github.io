---
title: "Writing plugins"
description: "A self-contained guide: HTML widgets and standalone Go (WebAssembly) plugins for OverlayComposer."
weight: 7
---

This page is written to be enough on its own, also for an AI agent that gets only this link (the raw Markdown version ends in `index.md`). It covers two kinds of plugins; pick one:

| | HTML widget plugin | Standalone plugin (WebAssembly) |
|---|---|---|
| What it is | a folder with `plugin.json` and a web page | one `.wasm` file, runs inside the app |
| Language | HTML, JS, CSS (any web stack) | Go (SDK provided); Rust and others through the ABI |
| Runs | in a sandboxed iframe of the overlay | on the server, no window |
| Good for | what is drawn on the stream: chat, counters, alerts | logic that must keep running: Twitch rewards, queues, timers, HTTP calls, bot messages, events |
| Can have a UI | it is the UI | yes: widgets and a settings page packed into the same file |

They combine: a standalone plugin can ship its own widgets, and a widget talks to its plugin through `oc.plugin`.

---

## Part 1. HTML widget plugin

### Folder

```
plugins/
  my-widget/
    plugin.json
    index.html
    app.js
```

The plugins folder is opened from **Add Source → Open plugins folder**. The widget is served at `/plugins/<id>/`.

### plugin.json

```json
{
  "id": "my-widget",
  "name": "My widget",
  "version": "1.0.0",
  "description": "What the widget does",
  "entry": "index.html",
  "defaultSize": { "width": 480, "height": 200 },
  "permissions": ["chat.read", "storage"],
  "settings": [
    { "key": "channel", "label": "Channel", "type": "string", "default": "" },
    { "key": "size", "label": "Font size", "type": "number", "default": 32, "min": 8, "max": 200 },
    { "key": "color", "label": "Color", "type": "color", "default": "#ffffff" },
    { "key": "mode", "label": "Mode", "type": "select", "default": "a",
      "options": [ { "value": "a", "label": "Mode A" }, { "value": "b", "label": "Mode B" } ] }
  ]
}
```

- `id`: `a-z A-Z 0-9 _ -`, up to 64 characters, equal to the folder name. `version`: semver, **raise it on every change** (updates are offered by version).
- `entry`: the page opened at `/plugins/<id>/` (default `index.html`).
- Setting types: `string`, `text`, `number`, `range`, `bool`, `color` (`#rrggbb`), `font`, `select` (needs `options`). `min`, `max`, `step` for `number` and `range`; `group` titles a section; `help` is a caption.
- The editor stores only values the user changed, so **the widget must know the defaults too**. Keep them equal to the manifest.
- Settings reach the widget as **query parameters** of its address: `?size=32&color=%23ffffff`. Numbers are `32`, booleans `true` or `false`.

### index.html

```html
<!doctype html>
<meta charset="utf-8">
<style>body { margin: 0; background: transparent; color: #fff; font: 32px sans-serif; }</style>
<div id="out"></div>
<script src="/plugin-sdk/oc.js"></script>
<script>
  const size = oc.param("size", 32);          // reads the query, cast to the type of the default
  document.body.style.fontSize = size + "px";
  oc.chat.on("message", (m) => {
    document.getElementById("out").textContent = m.displayName + ": " + m.text;
  });
  oc.chat.subscribe();                         // chat of the streamer's channel
</script>
```

The page background must be transparent: it is drawn over the stream. `oc.js` gives the global `oc`. The **complete typed API is one file**, `oc.d.ts`, served by a running app at `/plugin-sdk/oc.d.ts` (and in the repository at `web/sdk/oc.d.ts`); give it to your editor or agent.

### The `oc` API

| Call | Permission | Does |
|---|---|---|
| `oc.ready` | none | promise with `{pluginId, sourceId, settings, permissions, preview}` |
| `oc.param(key, default)` | none | a setting from the query, cast to the type of `default` |
| `oc.chat.subscribe({channel?})` | `chat.read` | start reading the chat; events through `oc.chat.on("message" \| "delete" \| "clear" \| "status", fn)` |
| `oc.chat.say(text, {channel?, as?, replyTo?})` | `chat.send` | write as the bot (`as: "bot"`, default) or the streamer |
| `oc.storage.get/set/delete/list(prefix)` | `storage` | JSON values of the plugin on this computer |
| `oc.storage.update(key, fn)` | `storage` | safe read-modify-write |
| `oc.storage.onChange(fn)` | `storage` | another copy of the widget changed the data |
| `oc.net.localGet(target, path)` | `net.local` | GET to a program on this computer (needs a `netLocal` rule in the manifest) |
| `oc.lifecycle.onStop(fn)` | none | up to 3 s to save data when the app quits |

Notes: the storage key is up to 128 characters of `A–Z a–z 0–9 _ . : / -`, a value is any JSON up to 256 KB, the quota is 5000 keys and 8 MB. In the editor preview `say` is disabled (replies would be duplicated). Sending is limited to about 18 messages per 30 s per bot.

```js
oc.chat.on("message", async (m) => {
  if (m.text.startsWith("!task ")) {
    await oc.storage.update("tasks", (list = []) => [...list, m.text.slice(6)]);
  }
});
oc.storage.onChange((c) => { if (c.key === "tasks") render(c.value || []); });
```

### Sandbox

A widget runs in an iframe with `sandbox="allow-scripts allow-forms allow-popups allow-modals"` and **without** `allow-same-origin`. It has an opaque origin: no `localStorage`, no cookies, no access to the app API. `fetch` and `WebSocket` to external addresses work (the other side must allow CORS). The plugin's own files (`/plugins/<id>/…`) are served with CORS `*`.

### Publish

Put the plugin into a GitHub repository as `plugins/<id>/` (with `plugin.json` and the entry file) and share the repository address: users add it in **Add Source → Plugin store…**. See [compico/oc-plugins](https://github.com/compico/oc-plugins) for examples (`osu-np`, `fps-meter`). A `changelog.json` next to `plugin.json` is shown to users after an update: `[{ "version": "1.2.0", "date": "2026-10-27", "changes": ["what is new"], "migration": "what to do after updating" }]`.

---

## Part 2. Standalone plugin in Go

A standalone plugin is **passive**: it declares routes and handlers, and the host calls them. It does not listen to anything by itself and does not run in the background. The host owns long-lived resources (timers, network requests, the Twitch connection).

### Set up

Go 1.24 or newer is needed (`//go:wasmexport`). The SDK has no dependencies.

```
mkdir hello && cd hello
go mod init example.com/hello
go get github.com/compico/overlaycomposer/pkg/ocplugin
```

`main.go`:

```go
package main

import "github.com/compico/overlaycomposer/pkg/ocplugin"

func init() {
	p := ocplugin.New(ocplugin.Meta{
		ID:          "hello", // a-z 0-9 -, up to 64 characters; also the URL segment
		Name:        "Hello",
		Version:     "0.1.0", // semver
		Permissions: []string{ocplugin.PermStorage},
	})
	p.GET("/greet", func(c *ocplugin.Ctx) error {
		n, _, _ := ocplugin.Load[int]("count")
		_ = ocplugin.Store("count", n+1)
		return c.JSON(200, map[string]any{"hello": c.Query("name"), "count": n + 1})
	})
}

func main() {} // required to build a reactor; stays empty
```

Build:

```
GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o hello.wasm .
```

Install: copy `hello.wasm` into the plugins folder (**🧩 Plugins → Open folder**), press **Rescan**, then grant the permissions the plugin asks for (**🧩 Plugins**, the plugin card). Test: `http://127.0.0.1:8000/plugins/hello/api/greet?name=oc` returns `{"count":1,"hello":"oc"}`.

Rules that make it work:

- `New` is called **once, from `init()`**; one `.wasm` holds one plugin. Route paths start with `/`.
- The plugin id must not equal the id of an HTML plugin.
- A Go plugin weighs about 3 MB (the Go runtime); that is normal.

### Behaviour you must design for

- **Calls come one at a time**, a call is limited to 10 s. A panic, trap or timeout recreates the instance; 5 failures within a minute put the plugin into `failed` until a manual restart.
- **Keep state in the host storage** (`Store`/`Load`), not in variables: the instance is recreated after a failure, after permissions change and after settings change.
- **`OnInit` runs many times.** Before the user grants a permission, host operations return `permission denied`. In `OnInit` that is normal: do not return an error (an error from `OnInit` puts the plugin into `failed`).
- **A route handler answers immediately.** Long work goes through host operations that return later as events (`Fetch` → `OnHTTPResponse`, `SetTimer` → `OnTimer`).
- There are no threads in `wasip1`: goroutines run only inside a host call.
- `time.Now()` and `crypto/rand` work normally (time is UTC).
- Logging: `ocplugin.Info/Warn/Error/Debug(msg, "key", value, …)`, plus stdout/stderr, goes to the app log as `plugin=<id>`.

### Routes and requests

```go
p.POST("/add", func(c *ocplugin.Ctx) error {
	var in struct{ Text string `json:"text"` }
	if err := c.Bind(&in); err != nil {
		return c.Error(400, "bad json")
	}
	if in.Text == "" {
		return c.Error(422, "text is empty")
	}
	return c.JSON(201, map[string]any{"ok": true})
})
```

`p.GET/POST/PUT/PATCH/DELETE(path, handler)` become `/plugins/<id>/api<path>`. `Ctx` has `Query(key)`, `Bind(&v)`, `JSON(status, v)`, `NoContent()`, `Error(status, msg)`, plus `Method` and `Path`. A returned `error` becomes a bare 500 (details only go to the log). A panic in a handler becomes a 500 and the plugin keeps running. The response is always `application/json`.

Safety of routes: a request from another origin (including a widget iframe) is rejected by the app, other devices need a pairing token, so **GET handlers must be free of side effects** (any page can trigger them). Treat all input as untrusted.

### Host operations (permissions)

Declare permissions in `Meta.Permissions`; the user grants them.

| Permission | SDK | Does |
|---|---|---|
| `storage` (`PermStorage`) | `Load[T](key)`, `Store(key, v)`, `Remove(key)` | JSON values in the app database (private to the plugin) |
| `timer` (`PermTimer`) | `SetTimer(id, d, repeat)`, `ClearTimer(id)`, `p.OnTimer(id, f)` | 50 ms to 24 h, up to 16 timers, the same id replaces |
| `net.http` (`PermNetHTTP`) | `Fetch(id, FetchRequest{Method, URL, Body})`, `p.OnHTTPResponse(id, f)` | `http(s)` only; request body up to 256 KB, response up to 1 MB, 15 s, 8 at a time, up to 3 redirects; **loopback, private, link-local and CGNAT addresses are blocked** |
| `twitch.redemption` (`PermRedeem`) | `p.OnRedemption(f)`, `FulfillRedemption(rewardID, id)`, `CancelRedemption(rewardID, id)` | channel point rewards; cancelling refunds the points |
| `events.publish` (`PermPublish`) | `Publish(topic, v)`, `PublishRetained(topic, v)` | events for widgets, add-ons and the Deck |
| `chat.send` (`PermChat`) | `Say(text)` | a message in the streamer's channel **as the bot**: up to 300 characters, burst 3 then 1 per 3 s; the host chooses channel and account |

A failed operation returns an `error` (for example `permission denied`).

### Timer

```go
p.OnInit(func() error {
	_ = ocplugin.SetTimer("tick", 30*time.Second, true) // denied before the grant: that is fine
	return nil
})
p.OnTimer("tick", func() { ocplugin.Info("tick") })
```

### HTTP request

```go
_ = ocplugin.Fetch("rates", ocplugin.FetchRequest{URL: "https://example.com/api/rates"})
p.OnHTTPResponse("rates", func(r ocplugin.HTTPResponse) {
	if r.Error != "" || r.Status != 200 {
		return
	}
	_ = ocplugin.Store("rates", r.Body) // r.Body is a string
})
```

### Twitch reward

```go
p := ocplugin.New(ocplugin.Meta{
	ID: "orders", Name: "Orders", Version: "0.1.0",
	Permissions: []string{ocplugin.PermStorage, ocplugin.PermRedeem, ocplugin.PermChat},
	ConfigSchema: map[string]any{
		"rewardId": map[string]any{"type": "string", "format": "twitch.reward", "title": "Reward"},
	},
})
p.OnRedemption(func(r ocplugin.Redemption) {
	// r.Input is text typed by the viewer: untrusted
	var cfg struct{ RewardID string `json:"rewardId"` }
	_ = p.Config(&cfg)
	if r.RewardID != cfg.RewardID {
		return
	}
	// ... validate r.Input, save the order with Store ...
	_ = ocplugin.FulfillRedemption(r.RewardID, r.ID) // or CancelRedemption: the points are returned
	_ = ocplugin.Say("@" + r.UserName + " accepted")
})
```

`Redemption` has `ID, RewardID, RewardTitle, UserID, UserLogin, UserName, Input, Status, RedeemedAt`. The reward must be **created by the app** (in the plugin settings: **New reward…**), otherwise Twitch refuses to change its status. The app subscribes to Twitch EventSub automatically while an enabled plugin has the `twitch.redemption` permission; repeated deliveries of the same redemption are dropped by id.

### Settings (configSchema)

`Meta.ConfigSchema` describes the form shown by the app: fields of type `string`, `integer`/`number`, `boolean`, with `title` and `description`. `"format": "twitch.reward"` makes a reward picker. Read the values with `p.Config(&struct{...})` (available from `OnInit`); saving the settings restarts the plugin.

### Publishing events

Declare topics in `Meta.Events` and add the `events.publish` permission:

```go
Events: []ocplugin.EventDecl{{ID: "now", Name: "Now playing"}, {ID: "commands"}},
```

`ocplugin.Publish("commands", v)` publishes an event; `PublishRetained("now", v)` also keeps the last value so a new subscriber gets it at once (use for state, not for commands). A topic is `a-z0-9._-`, up to 64 characters, up to 32 per plugin; data is JSON up to 64 KB; about 20 events/s on average per plugin.

Subscribers:

- a widget or settings page of the same plugin: `oc.plugin.subscribe("now", (data, meta) => …)`, where `meta` is `{topic, seq, retained}`;
- an external program: WebSocket `ws://127.0.0.1:8000/plugins/<id>/ws`. Frames are JSON text: send `{"t":"sub","topic":"now"}`, receive `{"t":"event","topic":"now","data":…,"seq":7,"retained":true}`; send `{"t":"ping"}` now and then and reconnect on loss (the retained value comes again after `sub`). Allowed origins: none, the same host, `music-application://desktop`.

### UI inside the .wasm: widgets and a settings page

Put HTML, JS, CSS, images and fonts into a folder (for example `ui/`), declare them in `Meta`, and pack the folder **into the same `.wasm`** with `ocpack`:

```go
Widgets:  []ocplugin.Widget{{ID: "queue", Name: "Queue", Entry: "queue.html", Width: 360, Height: 240}},
Settings: "settings.html", // a page shown under 🧩 Plugins → Settings
```

```
# in a clone of github.com/compico/overlaycomposer
go run ./cmd/ocpack pack hello.wasm ./ui          # appends the oc.assets section; running again replaces it
go run ./cmd/ocpack list hello.wasm
```

In GitHub Actions use `uses: compico/overlaycomposer/actions/ocpack@<tag-or-commit>` with `wasm:` and `ui:` inputs. Rules: relative paths with `/` and the characters `A–Z a–z 0–9 . _ ~ @ -`, no `..`, no hidden files; extensions only `html htm js mjs css json txt png jpg jpeg gif webp svg ico woff woff2 mp3 ogg wav`; up to 256 files, 2 MB per file, 8 MB in total. If the file named by `Entry` or `Settings` is missing the plugin does not load.

Pages are served at `/plugins/<id>/ui/<path>` (relative links like `app.js` work). Widgets appear in **Add Source → Widgets** marked *standalone*. In the page include the SDK and call your own plugin:

```html
<script src="/plugin-sdk/oc.js"></script>
<script>
  oc.plugin.get("/queue").then(({ body }) => render(body));   // GET /plugins/<id>/api/queue
  oc.plugin.post("/started", { id: "abc" });
  oc.plugin.subscribe("now", (data) => show(data));
</script>
```

`oc.plugin.call/get/post` go through the overlay page, not through `fetch` from the sandbox; they reach only routes declared by **this** plugin (up to 8 concurrent calls, 15 s). The editor preview calls the plugin too (`oc.ready` gives `info.preview === true`): do not send state-changing actions from a preview. The settings page also has `oc.storage.*` (with the `storage` permission) and the same `oc.plugin.*`.

Every plugin page is served with `Content-Security-Policy: sandbox allow-scripts`; it cannot reach the app API.

### Tests

The SDK layer builds only for `wasip1`, but handlers are plain functions. Replace the host in an ordinary `go test`:

```go
func TestGreet(t *testing.T) {
	store := map[string]json.RawMessage{}
	ocplugin.SetHost(func(op string, arg []byte) []byte {
		switch op {
		case "storage.get":
			var a struct{ Key string }
			_ = json.Unmarshal(arg, &a)
			if v, ok := store[a.Key]; ok {
				return []byte(`{"found":true,"value":` + string(v) + `}`)
			}
			return []byte(`{"found":false}`)
		case "storage.set":
			var a struct {
				Key   string
				Value json.RawMessage
			}
			_ = json.Unmarshal(arg, &a)
			store[a.Key] = a.Value
		}
		return []byte(`{}`)
	})
	// call your logic (kept in separate functions) and assert
}
```

Keep the logic (queues, parsing) in functions that do not touch the SDK and test those directly; `wasm/music` in [oc-plugins](https://github.com/compico/oc-plugins/tree/main/wasm/music) is a full example.

### Publish a standalone plugin

In [compico/oc-plugins](https://github.com/compico/oc-plugins) (or your own repository built the same way): the plugin lives in `wasm/<id>/` with its own `go.mod` (the folder name equals the `id`), optional `ui/` and a `README.md`.

- `node scripts/wasm.mjs build <id>` runs `go vet`, `go test`, builds `dist/<id>.wasm` (packing `ui/` with `ocpack`) and prints the metadata read from the file. `npm test` validates every manifest.
- Raise `Version` for every code change. After merge to `main` CI creates a release `<id>-v<version>` with `<id>.wasm` and a sha256; a PR that changes code without raising the version fails the check.
- Users install the file from the release (see [Widgets and plugins](../plugins/)).

---

## Part 3. Other languages (ABI)

A `.wasm` file for the `wasip1` target (a reactor) with these exports and one import works in any language. Everything except signatures is JSON, so new features do not break existing plugins. `i64` is `(ptr << 32) | len` of a buffer in the plugin memory, `0` means empty.

| Export | Purpose |
|---|---|
| `alloc(n: i32) -> i32` | allocate at least `n` bytes |
| `dealloc(ptr: i32)` | free a buffer |
| `oc_meta() -> i64` | **pure** function: metadata JSON (called on a throwaway instance, `host_call` is not available) |
| `oc_init(ptr, len) -> i64` | `{"apiVersion":1,"config":{…}}`; the result is ignored |
| `oc_call(name_ptr, name_len, arg_ptr, arg_len) -> i64` | a route handler (by the name from `oc_meta`) or `on_event` |

Import (module `oc`): `host_call(op_ptr, op_len, arg_ptr, arg_len) -> i64` runs a host operation: JSON argument in, JSON result out (`{"error":"…"}` on failure). If `_initialize` is exported the host calls it. Go: `-buildmode=c-shared` with `//go:wasmexport` and `//go:wasmimport`; Rust: `cdylib` for `wasm32-wasip1`.

`oc_meta` result:

```json
{
  "id": "music", "name": "Music requests", "version": "0.1.0", "apiVersion": 1,
  "permissions": ["storage", "timer", "net.http", "twitch.redemption", "events.publish", "chat.send"],
  "subscriptions": ["twitch.redemption"],
  "routes": [{"method": "GET", "path": "/next", "handler": "next"}],
  "events": [{"id": "now", "name": "Now playing"}],
  "widgets": [{"id": "queue", "name": "Queue", "entry": "queue.html", "width": 360, "height": 240}],
  "settings": "settings.html",
  "configSchema": {"rewardId": {"type": "string", "format": "twitch.reward", "title": "Reward"}}
}
```

A route handler gets `{"method","path","query":{…},"body":<json|null>}` and returns `{"status":200,"body":<json>}`. Events go to the handler named `on_event` with `{"type","data"}`: `twitch.redemption`, `timer` (`{"id"}`), `http.response` (`{"id","status","body","error"}`).

Host operations (`host_call`): `log`, `storage.get/set/delete`, `timer.set/clear`, `http.fetch`, `twitch.redemption.update`, `events.publish`, `chat.send`. Arguments are the JSON objects shown by the Go SDK functions above (for example `storage.set` takes `{"key","value"}`, `timer.set` takes `{"id","ms","repeat"}`, `chat.send` takes `{"text"}`).

A Rust SDK (like `ocplugin` for Go) is planned; see the [issue #136](https://github.com/compico/overlaycomposer/issues/136). Rust plugins on the same ABI weigh about 100 KB.

---

## Checklist for an agent

1. Choose the kind (Part 1 or Part 2). If the plugin needs Twitch rewards, timers, bot messages or must survive without an open widget, it is standalone.
2. Write `Meta`/`plugin.json` with the **minimum** permissions.
3. Keep state in storage; never rely on variables or background work; answer routes immediately.
4. Treat every viewer text (`Redemption.Input`, chat messages) as untrusted; never build HTML from it with `innerHTML`.
5. Raise the version on every change.
6. Build (`GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o <id>.wasm .`), install, grant permissions in **🧩 Plugins**, check the log (`plugin=<id>`).
