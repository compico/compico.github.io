---
title: "Разработка плагинов"
description: "Самодостаточное руководство: HTML-виджеты и автономные плагины на Go (WebAssembly) для OverlayComposer."
weight: 7
---

Эта страница написана так, чтобы хватало её одной, в том числе ИИ-агенту, которому дали только эту ссылку (сырая версия Markdown заканчивается на `index.md`). Здесь два вида плагинов, выбери один:

| | Плагин-виджет (HTML) | Автономный плагин (WebAssembly) |
|---|---|---|
| Что это | папка с `plugin.json` и веб-страницей | один файл `.wasm`, работает внутри программы |
| Язык | HTML, JS, CSS (любой веб-стек) | Go (есть SDK); Rust и другие через ABI |
| Где работает | в изолированном iframe оверлея | на сервере, без окна |
| Подходит для | того, что рисуется на стриме: чат, счётчики, алерты | логики, которая должна работать всегда: награды Twitch, очереди, таймеры, HTTP-запросы, сообщения бота, события |
| Свой интерфейс | он и есть интерфейс | да: виджеты и страница настроек упаковываются в тот же файл |

Их можно сочетать: автономный плагин может нести свои виджеты, а виджет общается со своим плагином через `oc.plugin`.

---

## Часть 1. Плагин-виджет (HTML)

### Папка

```
plugins/
  my-widget/
    plugin.json
    index.html
    app.js
```

Папка плагинов открывается из **Add Source → Open plugins folder**. Виджет раздаётся по адресу `/plugins/<id>/`.

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

- `id`: `a-z A-Z 0-9 _ -`, до 64 символов, равен имени папки. `version`: semver, **поднимай при каждом изменении** (обновления предлагаются по версии).
- `entry`: страница, которая открывается по `/plugins/<id>/` (по умолчанию `index.html`).
- Типы настроек: `string`, `text`, `number`, `range`, `bool`, `color` (`#rrggbb`), `font`, `select` (нужны `options`). `min`, `max`, `step` для `number` и `range`; `group` — заголовок секции; `help` — подпись.
- Редактор сохраняет только изменённые пользователем значения, поэтому **значения по умолчанию должен знать и сам виджет**. Держи их равными значениям из манифеста.
- Настройки приходят виджету **query-параметрами** его адреса: `?size=32&color=%23ffffff`. Число — `32`, bool — `true` или `false`.

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

Фон страницы должен быть прозрачным: она рисуется поверх стрима. `oc.js` создаёт глобальный `oc`. **Полное описание типов — один файл `oc.d.ts`**, его отдаёт запущенная программа по `/plugin-sdk/oc.d.ts` (и он лежит в репозитории как `web/sdk/oc.d.ts`); отдай его редактору или агенту.

### API `oc`

| Вызов | Разрешение | Что делает |
|---|---|---|
| `oc.ready` | нет | промис со сведениями `{pluginId, sourceId, settings, permissions, preview}` |
| `oc.param(key, default)` | нет | настройка из query, приведённая к типу `default` |
| `oc.chat.subscribe({channel?})` | `chat.read` | начать читать чат; события через `oc.chat.on("message" \| "delete" \| "clear" \| "status", fn)` |
| `oc.chat.say(text, {channel?, as?, replyTo?})` | `chat.send` | написать от имени бота (`as: "bot"`, по умолчанию) или стримера |
| `oc.storage.get/set/delete/list(prefix)` | `storage` | JSON-значения плагина на этом компьютере |
| `oc.storage.update(key, fn)` | `storage` | безопасное «прочитал → изменил → записал» |
| `oc.storage.onChange(fn)` | `storage` | другая копия виджета изменила данные |
| `oc.net.localGet(target, path)` | `net.local` | GET к программе на этом компьютере (нужно правило `netLocal` в манифесте) |
| `oc.lifecycle.onStop(fn)` | нет | до 3 секунд на сохранение данных при выходе из программы |

Заметки: ключ хранилища — до 128 символов `A–Z a–z 0–9 _ . : / -`, значение — любой JSON до 256 КБ, квота — 5000 ключей и 8 МБ. В предпросмотре редактора `say` отключён (ответы дублировались бы). Отправка ограничена примерно 18 сообщениями за 30 с на бота.

```js
oc.chat.on("message", async (m) => {
  if (m.text.startsWith("!task ")) {
    await oc.storage.update("tasks", (list = []) => [...list, m.text.slice(6)]);
  }
});
oc.storage.onChange((c) => { if (c.key === "tasks") render(c.value || []); });
```

### Песочница

Виджет работает в iframe с `sandbox="allow-scripts allow-forms allow-popups allow-modals"` и **без** `allow-same-origin`. У него «непрозрачный» origin: нет `localStorage`, cookie и доступа к API программы. `fetch` и `WebSocket` к внешним адресам работают (другая сторона должна разрешать CORS). Собственные файлы плагина (`/plugins/<id>/…`) отдаются с CORS `*`.

### Публикация

Положи плагин в репозиторий GitHub как `plugins/<id>/` (с `plugin.json` и файлом входа) и поделись адресом репозитория: пользователи добавляют его в **Add Source → Plugin store…**. Примеры — [compico/oc-plugins](https://github.com/compico/oc-plugins) (`osu-np`, `fps-meter`). Файл `changelog.json` рядом с `plugin.json` показывается пользователям после обновления: `[{ "version": "1.2.0", "date": "2026-10-27", "changes": ["что нового"], "migration": "что сделать после обновления" }]`.

---

## Часть 2. Автономный плагин на Go

Автономный плагин **пассивен**: он объявляет маршруты и обработчики, а вызывает их хост. Сам он ничего не слушает и в фоне не работает. Долгоживущими ресурсами (таймерами, сетевыми запросами, соединением с Twitch) владеет хост.

### Подготовка

Нужен Go 1.24 или новее (`//go:wasmexport`). У SDK нет зависимостей.

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

Сборка:

```
GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o hello.wasm .
```

Установка: скопируй `hello.wasm` в папку плагинов (**🧩 Plugins → Open folder**), нажми **Rescan**, затем выдай запрошенные разрешения (**🧩 Plugins**, карточка плагина). Проверка: `http://127.0.0.1:8000/plugins/hello/api/greet?name=oc` вернёт `{"count":1,"hello":"oc"}`.

Правила, без которых не заработает:

- `New` вызывается **один раз, из `init()`**; в одном `.wasm` один плагин. Пути маршрутов начинаются с `/`.
- id плагина не должен совпадать с id HTML-плагина.
- Плагин на Go весит около 3 МБ (рантайм Go); это нормально.

### Что нужно учитывать в устройстве плагина

- **Вызовы идут по одному**, на вызов не больше 10 с. Паника, trap или таймаут пересоздают экземпляр; 5 сбоев за минуту переводят плагин в `failed` до ручного перезапуска.
- **Состояние храни в хранилище хоста** (`Store`/`Load`), а не в переменных: экземпляр пересоздаётся после сбоя, после смены разрешений и после смены настроек.
- **`OnInit` выполняется много раз.** Пока пользователь не выдал разрешение, операции хоста возвращают `permission denied`. В `OnInit` это обычное состояние: не возвращай ошибку (ошибка из `OnInit` переводит плагин в `failed`).
- **Обработчик маршрута отвечает сразу.** Долгая работа идёт через операции хоста, результат которых приходит позже событием (`Fetch` → `OnHTTPResponse`, `SetTimer` → `OnTimer`).
- В `wasip1` нет потоков: горутины работают только внутри вызова хоста.
- `time.Now()` и `crypto/rand` работают как обычно (время в UTC).
- Лог: `ocplugin.Info/Warn/Error/Debug(msg, "ключ", значение, …)`, а также stdout/stderr, попадают в лог программы как `plugin=<id>`.

### Маршруты и запросы

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

`p.GET/POST/PUT/PATCH/DELETE(path, handler)` становятся `/plugins/<id>/api<path>`. У `Ctx` есть `Query(key)`, `Bind(&v)`, `JSON(status, v)`, `NoContent()`, `Error(status, msg)` и поля `Method`, `Path`. Возвращённая `error` превращается в голый 500 (подробности только в лог). Паника в обработчике превращается в 500, плагин продолжает работать. Ответ всегда `application/json`.

Безопасность маршрутов: запрос с чужого origin (в том числе из iframe виджета) программа отклоняет, другим устройствам нужен токен сопряжения, поэтому **GET-обработчики не должны иметь побочных эффектов** (их может вызвать любая страница). Любой ввод считай недоверенным.

### Операции хоста (разрешения)

Разрешения перечисляются в `Meta.Permissions`, выдаёт их пользователь.

| Разрешение | SDK | Что делает |
|---|---|---|
| `storage` (`PermStorage`) | `Load[T](key)`, `Store(key, v)`, `Remove(key)` | JSON-значения в базе программы (личные для плагина) |
| `timer` (`PermTimer`) | `SetTimer(id, d, repeat)`, `ClearTimer(id)`, `p.OnTimer(id, f)` | от 50 мс до 24 ч, до 16 таймеров, тот же id заменяет |
| `net.http` (`PermNetHTTP`) | `Fetch(id, FetchRequest{Method, URL, Body})`, `p.OnHTTPResponse(id, f)` | только `http(s)`; тело запроса до 256 КБ, ответ до 1 МБ, 15 с, 8 одновременно, до 3 редиректов; **адреса loopback, частных сетей, link-local и CGNAT запрещены** |
| `twitch.redemption` (`PermRedeem`) | `p.OnRedemption(f)`, `FulfillRedemption(rewardID, id)`, `CancelRedemption(rewardID, id)` | награды за баллы канала; отмена возвращает баллы |
| `events.publish` (`PermPublish`) | `Publish(topic, v)`, `PublishRetained(topic, v)` | события для виджетов, аддонов и Deck |
| `chat.send` (`PermChat`) | `Say(text)` | сообщение в канале стримера **от имени бота**: до 300 символов, всплеск 3, затем одно в 3 с; канал и аккаунт выбирает хост |

Неудавшаяся операция возвращает `error` (например, `permission denied`).

### Таймер

```go
p.OnInit(func() error {
	_ = ocplugin.SetTimer("tick", 30*time.Second, true) // denied before the grant: that is fine
	return nil
})
p.OnTimer("tick", func() { ocplugin.Info("tick") })
```

### HTTP-запрос

```go
_ = ocplugin.Fetch("rates", ocplugin.FetchRequest{URL: "https://example.com/api/rates"})
p.OnHTTPResponse("rates", func(r ocplugin.HTTPResponse) {
	if r.Error != "" || r.Status != 200 {
		return
	}
	_ = ocplugin.Store("rates", r.Body) // r.Body is a string
})
```

### Награда Twitch

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

У `Redemption` есть поля `ID, RewardID, RewardTitle, UserID, UserLogin, UserName, Input, Status, RedeemedAt`. Награда должна быть **создана самой программой** (в настройках плагина: **New reward…**), иначе Twitch не даст менять её статус. Программа сама подписывается на EventSub Twitch, пока у включённого плагина есть разрешение `twitch.redemption`; повторные доставки одного погашения отбрасываются по id.

### Настройки (configSchema)

`Meta.ConfigSchema` описывает форму, которую показывает программа: поля типа `string`, `integer`/`number`, `boolean` с `title` и `description`. `"format": "twitch.reward"` даёт выбор награды. Значения читай через `p.Config(&struct{...})` (доступно с `OnInit`); сохранение настроек перезапускает плагин.

### Публикация событий

Объяви топики в `Meta.Events` и добавь разрешение `events.publish`:

```go
Events: []ocplugin.EventDecl{{ID: "now", Name: "Now playing"}, {ID: "commands"}},
```

`ocplugin.Publish("commands", v)` публикует событие; `PublishRetained("now", v)` ещё и хранит последнее значение, чтобы новый подписчик получил его сразу (для состояния, не для команд). Топик: `a-z0-9._-`, до 64 символов, до 32 на плагин; данные — JSON до 64 КБ; в среднем около 20 событий/с на плагин.

Подписчики:

- виджет или страница настроек того же плагина: `oc.plugin.subscribe("now", (data, meta) => …)`, где `meta` — `{topic, seq, retained}`;
- внешняя программа: WebSocket `ws://127.0.0.1:8000/plugins/<id>/ws`. Кадры — текстовый JSON: отправь `{"t":"sub","topic":"now"}`, получай `{"t":"event","topic":"now","data":…,"seq":7,"retained":true}`; время от времени шли `{"t":"ping"}` и переподключайся при потере связи (после `sub` retained-значение приходит снова). Допустимые origin: пустой, тот же хост, `music-application://desktop`.

### Интерфейс внутри .wasm: виджеты и страница настроек

Положи HTML, JS, CSS, картинки и шрифты в папку (например, `ui/`), объяви их в `Meta` и упакуй папку **в тот же `.wasm`** утилитой `ocpack`:

```go
Widgets:  []ocplugin.Widget{{ID: "queue", Name: "Queue", Entry: "queue.html", Width: 360, Height: 240}},
Settings: "settings.html", // a page shown under 🧩 Plugins → Settings
```

```
# in a clone of github.com/compico/overlaycomposer
go run ./cmd/ocpack pack hello.wasm ./ui          # appends the oc.assets section; running again replaces it
go run ./cmd/ocpack list hello.wasm
```

В GitHub Actions используй `uses: compico/overlaycomposer/actions/ocpack@<тег-или-коммит>` с входами `wasm:` и `ui:`. Правила: относительные пути с `/` и символами `A–Z a–z 0–9 . _ ~ @ -`, без `..`, без скрытых файлов; расширения только `html htm js mjs css json txt png jpg jpeg gif webp svg ico woff woff2 mp3 ogg wav`; до 256 файлов, до 2 МБ на файл, 8 МБ всего. Если файла из `Entry` или `Settings` нет, плагин не загрузится.

Страницы раздаются по `/plugins/<id>/ui/<путь>` (относительные ссылки вроде `app.js` работают). Виджеты появляются в **Add Source → Widgets** с пометкой *standalone*. В странице подключи SDK и вызывай свой плагин:

```html
<script src="/plugin-sdk/oc.js"></script>
<script>
  oc.plugin.get("/queue").then(({ body }) => render(body));   // GET /plugins/<id>/api/queue
  oc.plugin.post("/started", { id: "abc" });
  oc.plugin.subscribe("now", (data) => show(data));
</script>
```

`oc.plugin.call/get/post` идут через страницу оверлея, а не через `fetch` из песочницы; они достают только маршруты, объявленные **этим** плагином (до 8 одновременных вызовов, 15 с). Предпросмотр редактора тоже вызывает плагин (`oc.ready` даёт `info.preview === true`): не отправляй из предпросмотра действия, меняющие состояние. На странице настроек доступны также `oc.storage.*` (с разрешением `storage`) и те же `oc.plugin.*`.

Каждая страница плагина отдаётся с `Content-Security-Policy: sandbox allow-scripts`; к API программы она обратиться не может.

### Тесты

Слой SDK собирается только под `wasip1`, но обработчики — обычные функции. Подмени хост в обычном `go test`:

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

Логику (очереди, разбор) держи в функциях, не трогающих SDK, и тестируй их напрямую; `wasm/music` в [oc-plugins](https://github.com/compico/oc-plugins/tree/main/wasm/music) — полный пример.

### Публикация автономного плагина

В [compico/oc-plugins](https://github.com/compico/oc-plugins) (или в своём репозитории, устроенном так же): плагин лежит в `wasm/<id>/` со своим `go.mod` (имя папки равно `id`), необязательной `ui/` и `README.md`.

- `node scripts/wasm.mjs build <id>` выполняет `go vet`, `go test`, собирает `dist/<id>.wasm` (упаковывая `ui/` через `ocpack`) и печатает метаданные, прочитанные из файла. `npm test` проверяет все манифесты.
- Поднимай `Version` при каждом изменении кода. После merge в `main` CI создаёт релиз `<id>-v<версия>` с `<id>.wasm` и sha256; PR, который меняет код без повышения версии, не проходит проверку.
- Пользователи ставят файл из релиза (см. [Виджеты и плагины](../plugins/)).

---

## Часть 3. Другие языки (ABI)

Файл `.wasm` для цели `wasip1` (reactor) с этими экспортами и одним импортом работает на любом языке. Всё, кроме сигнатур, — JSON, поэтому новые возможности не ломают готовые плагины. `i64` — это `(ptr << 32) | len` буфера в памяти плагина, `0` — пусто.

| Экспорт | Назначение |
|---|---|
| `alloc(n: i32) -> i32` | выделить не менее `n` байт |
| `dealloc(ptr: i32)` | освободить буфер |
| `oc_meta() -> i64` | **чистая** функция: JSON метаданных (вызывается на одноразовом экземпляре, `host_call` там недоступен) |
| `oc_init(ptr, len) -> i64` | `{"apiVersion":1,"config":{…}}`; результат игнорируется |
| `oc_call(name_ptr, name_len, arg_ptr, arg_len) -> i64` | обработчик маршрута (по имени из `oc_meta`) или `on_event` |

Импорт (модуль `oc`): `host_call(op_ptr, op_len, arg_ptr, arg_len) -> i64` выполняет операцию хоста: на входе JSON-аргумент, на выходе JSON-результат (`{"error":"…"}` при ошибке). Если экспортирован `_initialize`, хост его вызывает. Go: `-buildmode=c-shared` с `//go:wasmexport` и `//go:wasmimport`; Rust: `cdylib` для `wasm32-wasip1`.

Результат `oc_meta`:

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

Обработчик маршрута получает `{"method","path","query":{…},"body":<json|null>}` и возвращает `{"status":200,"body":<json>}`. События приходят в обработчик с именем `on_event` как `{"type","data"}`: `twitch.redemption`, `timer` (`{"id"}`), `http.response` (`{"id","status","body","error"}`).

Операции хоста (`host_call`): `log`, `storage.get/set/delete`, `timer.set/clear`, `http.fetch`, `twitch.redemption.update`, `events.publish`, `chat.send`. Аргументы — JSON-объекты, которые строят функции Go SDK выше (например, `storage.set` принимает `{"key","value"}`, `timer.set` — `{"id","ms","repeat"}`, `chat.send` — `{"text"}`).

SDK для Rust (как `ocplugin` для Go) запланирован; см. [issue #136](https://github.com/compico/overlaycomposer/issues/136). Плагины на Rust на том же ABI весят около 100 КБ.

---

## Чек-лист для агента

1. Выбери вид плагина (часть 1 или 2). Если нужны награды Twitch, таймеры, сообщения бота или работа без открытого виджета, это автономный плагин.
2. Напиши `Meta`/`plugin.json` с **минимумом** разрешений.
3. Состояние держи в хранилище; не полагайся на переменные и фоновую работу; на маршруты отвечай сразу.
4. Любой текст зрителя (`Redemption.Input`, сообщения чата) считай недоверенным; не собирай из него HTML через `innerHTML`.
5. Поднимай версию при каждом изменении.
6. Собери (`GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o <id>.wasm .`), установи, выдай разрешения в **🧩 Plugins**, проверь лог (`plugin=<id>`).
