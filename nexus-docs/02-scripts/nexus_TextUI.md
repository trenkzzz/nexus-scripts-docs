---
icon: hand-pointer
---

# Nexus TextUI

> A point-based 3D world-anchored TextUI — register a point once, and the resource handles distance, the indicator, the prompt and the keypress for you.

**Version:** 1.0.0 · **Type:** Standalone / client-only

***

## 📝 Description

Nexus TextUI is not a "show text / hide text" helper. It is a **point registry**. You register an interaction point once — world coordinates, two distances, a prompt, a key and two callbacks — and from that moment the resource owns the entire lifecycle: it measures the distance to the player, decides whether to show nothing, a floating diamond indicator or the full prompt, projects the world coordinate onto the screen every frame, scales the element down as the player gets further away, listens for the key press, and calls your handler. You never write a render loop, never call a show/hide function, and never poll a distance yourself.

That design is the main difference from conventional TextUI resources. The usual pattern (`TextUI:Open(text)` inside your own `while true do` distance loop, `TextUI:Close()` when the player walks away) puts the per-frame cost in _every_ resource that uses it, and every one of them re-implements the same logic slightly differently. Here all of that cost is centralised in two threads inside `nexus_TextUI`, no matter how many resources and how many points are registered. A resource that uses it typically calls `Create` once at start and `Delete` once on stop, and does nothing in between.

Internally there are two cooperating threads with very different duty cycles. The **state thread** runs every 250 ms: it walks every registered point, computes `#(point.coords - playerCoords)`, evaluates your `canInteract` predicate, and decides which points are in view range and whether each should render as an `indicator` (inside `viewDistance` but outside `interactionDistance`) or as `text` (inside `interactionDistance`). Points that dropped out of range since the last pass get a `remove` message. The **render thread** runs at frame rate, but only while at least one point is actually visible — otherwise it sleeps 200 ms per iteration. For each visible point it calls `GetScreenCoordFromWorldCoord`, computes a scale interpolated between `0.6` and `1.0` across the gap between `viewDistance` and `interactionDistance`, and — critically — **only sends a NUI message when a coordinate or the scale actually changed by more than 0.01**, batching every changed point into a single `updatePositions` message. A standing player generates no NUI traffic at all.

The NUI itself keeps a `Map` of live DOM elements keyed by interaction id, so an element is created once and then only repositioned through `style.left` / `style.top` / `style.transform`. The visual is a two-box prompt: a square key badge and a separate rounded text box beside it, both in the Nexus purple with a looping shimmer sweep, which cross-fades to a pulsing diamond indicator when the player backs away past `interactionDistance`.

## ✨ Features

* **Point-based registration, not show/hide** — `Create(config)` registers a world point; the resource owns distance, visibility, rendering and input from then on. No render loop in your resource.
* **Two-stage proximity rendering** — outside `interactionDistance` but inside `viewDistance`, the point renders as a pulsing purple diamond indicator so players can see _that_ something is there; crossing `interactionDistance` cross-fades it into the full key + text prompt.
* **Distance-based scaling** — while in the indicator band, the element is scaled between `0.6` (at `viewDistance`) and `1.0` (at `interactionDistance`) by linear interpolation, so distant points genuinely look distant.
* **Built-in key handling** — the key named in `config.key` is polled via `IsControlJustPressed` **only** while the player is inside `interactionDistance`, and your `onInteract` callback is invoked on press. You register no keybind and no control loop.
* **Conditional visibility (`canInteract`)** — an optional predicate re-evaluated every 250 ms. Return `false` and the point disappears entirely (indicator included) until it returns `true` again. Typical uses: job checks, "not in a vehicle", inventory checks, shop open/closed hours.
* **Two separate boxes for key and text** — a fixed 36×36 key badge and an auto-width text box, 8px apart, each with its own entrance animation (the text box is delayed 50 ms behind the key badge) and its own offset shimmer sweep.
* **Automatic key extraction from the prompt text** — the resource parses a `[KEY] Description` pattern out of `config.text` and uses the bracketed part to fill the key badge, falling back to `config.key` if the text does not match the pattern.
* **Change-gated NUI traffic** — positions are only pushed when something moved more than 0.01 screen percent, and all changed points go in one batched `updatePositions` message. Standing still costs zero messages.
* **Adaptive render loop** — the frame-rate thread short-circuits to a 200 ms sleep whenever no point is visible, so an empty or far-away server costs effectively nothing.
* **Off-screen culling** — points behind the camera are moved to `-1000, -1000` with scale `0` rather than being drawn at a wrong position.
* **Unlimited simultaneous points** — `activeInteractions` is a plain id-keyed table; dozens of points from many different resources coexist, each with its own distances, key, text and predicate.
* **Fully standalone** — no framework, no database, no `ox_lib`, no target resource, no config file. It is a pure client-side resource.
* **Built-in test commands** — `/testui` spawns a working interaction point 2 m in front of the player (view 5 m, interact 2 m, key `E`, blocked while in a vehicle) and `/deltestui` removes it, so you can verify the resource end-to-end before integrating.
* **Smooth enter/exit transitions** — 0.3 s opacity transition on the container, a springy `cubic-bezier(0.175, 0.885, 0.32, 1.275)` scale-in on each box, a `slide-in` entrance, and a 300 ms grace period on removal so elements fade out instead of popping.

***

## 📋 Dependencies

| Dependency                           | Required | Notes                                                                                                                                                                                                                          |
| ------------------------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Framework (ESX / QBCore)             | ❌ No     | Never referenced. Fully framework-agnostic — works on ESX, QBCore, QBox, vRP or a bare server.                                                                                                                                 |
| Database / MySQL                     | ❌ No     | No persistence of any kind, no `.sql` file.                                                                                                                                                                                    |
| `ox_lib` / `qb-target` / `ox_target` | ❌ No     | This resource _replaces_ the prompt layer of a target system for point-based interactions; it does not build on one.                                                                                                           |
| Server-side script                   | ❌ None   | The resource is **client-only** (`client_scripts` only — there is no `server_script` at all). All registration happens on the client.                                                                                          |
| Internet access on the client        | ⚠️ Soft  | `html/index.html` loads the Inter font from Google Fonts, and also pulls Font Awesome from a CDN (unused — see the incident log in the FAQ). Without internet the prompt falls back to the generic sans-serif; nothing breaks. |

The resource folder **must** be named exactly `nexus_TextUI` (note the capital `UI`). `client/_resource.lua` raises a hard `error()` and aborts the resource if it is not, because the export namespace `exports['nexus_TextUI']` depends on it.

***

## ⚙️ Installation

1. **Download and extract the resource.** Download it from the cfx.re (Keymaster) portal and place the folder in your `resources` directory.
2. **Keep the exact folder name `nexus_TextUI`** — capital `U`, capital `I`. On a case-sensitive Linux server, `nexus_textui` will fail the name check and the resource will not start.
3. **Import no SQL.** There is nothing to import.
4.  **Add it to `server.cfg`, before every resource that will register points.** The export must exist by the time another resource calls `Create` at start:

    ```cfg
    ensure nexus_TextUI
    -- then the resources that register interaction points
    ensure nexus_multijob
    ensure my_shops
    ```
5. **Restart.** A full server restart is recommended.
6. **Verify.** In game, run `/testui`. A prompt should appear \~2 m in front of you; back away and it should become a pulsing diamond, then disappear past 5 m. Press `E` while close to get the confirmation notification. Get into a vehicle and it should vanish (that is the demo's `canInteract` predicate). Run `/deltestui` to clean up.
7. **Integrate.** Register your own points with `exports['nexus_TextUI']:Create{ ... }` from any client script. There is no config file to edit — behaviour is per-point, passed in the `Create` call.

***

## 🔧 Configuration

**Nexus TextUI ships no `config.lua`.** This is deliberate: there are no server-wide settings, because every tunable is a property of the individual interaction point and is passed into `Create`. The complete surface is therefore the `config` table of `Create(config)`:

| Config key            | Type      | Default | Description                                                                                                                                                                                                                                                                                                         |
| --------------------- | --------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                  | string    | —       | **Required.** Unique identifier for the point. Used as the table key in `activeInteractions`, as the DOM element id in the NUI, and as the argument to `Delete`. If omitted, `Create` prints an error and returns without registering anything. Re-using an existing id silently **overwrites** the previous point. |
| `coords`              | `vector3` | —       | **Required.** The world position the prompt is anchored to. Must be a vector3 (the code uses `#(config.coords - playerCoords)` and `config.coords.x/.y/.z`), not a table of three numbers.                                                                                                                          |
| `viewDistance`        | number    | —       | **Required.** Outer radius in metres. Beyond this, the point renders nothing at all. Inside it but outside `interactionDistance`, the point renders as a pulsing diamond indicator.                                                                                                                                 |
| `interactionDistance` | number    | —       | **Required.** Inner radius in metres. Inside it, the point renders the full key + text prompt and the key is polled. Must be smaller than `viewDistance` for the indicator stage to exist and for the scaling to behave as intended.                                                                                |
| `text`                | string    | —       | **Required.** The prompt. Write it as `'[E] Open the garage'`: the bracketed part fills the square key badge, and the rest is the description. If the string does not match the `[…]…` pattern, the badge falls back to `config.key` and the text box shows the raw string.                                         |
| `key`                 | string    | —       | **Required for interaction.** The key that triggers `onInteract`, given as a name from the internal `Keys` table (e.g. `'E'`, `'F'`, `'G'`, `'SPACE'`, `'F7'`). **Case-sensitive and uppercase** — `'e'` is not a valid entry and will error when the player walks into range.                                      |
| `onInteract`          | function  | `nil`   | Called with no arguments when the player presses `key` inside `interactionDistance`. Omit it for a purely informational prompt.                                                                                                                                                                                     |
| `canInteract`         | function  | `nil`   | Optional predicate, re-evaluated every 250 ms. Return `false` to hide the point completely (indicator included); return `true` (or omit the field) to show it. Takes no arguments.                                                                                                                                  |

Fields the resource writes into your table — do not set them yourself:

| Field             | Set by   | Purpose                                                                      |
| ----------------- | -------- | ---------------------------------------------------------------------------- |
| `formattedText`   | `Create` | `{ key = <badge text>, text = <text-box text> }`, derived from `text`/`key`. |
| `lastScreenState` | `Create` | `{ x, y, scale }` cache used by the render thread to skip unchanged frames.  |

Valid `key` names (the internal `Keys` table, names are exact and case-sensitive):

```
A B C D E F G H K L M N P Q R S T U V W X Y Z
0-9 (as "1".."9")  -  =
F1 F2 F3 F5 F6 F7 F8 F9 F10 F11
UpArr DownArr LeftArr RightArr   LEFT RIGHT TOP DOWN
SPACE ENTER TAB BACKSPACE DELETE CAPS ESC
LShift LEFTSHIFT LAlt LEFTALT LEFTCTRL RIGHTCTRL
HOME PAGEUP PAGEDOWN  ,  .  [  ]  ~
NUM1..NUM9   N4 N5 N6 N7 N8 N9 N+ N- NENTER
```

Note there is no `F4` and no `F12` entry. An unrecognised key name is not caught at registration time — it resolves to `nil` and throws inside `IsControlJustPressed` the moment the player walks into `interactionDistance`, so check spelling and case carefully (see the FAQ).

### Styling

With no config file, the visual identity is changed by editing `html/style.css` directly, which ships as a plain NUI asset and is never encrypted. The values worth knowing:

| What                  | Where                                                      | Current value          |
| --------------------- | ---------------------------------------------------------- | ---------------------- |
| Prompt / badge colour | `.key-box`, `.text-box` `background-color`                 | `#a958ff`              |
| Indicator colour      | `.indicator-wrapper::before` `background-color` / `border` | `#a958ff` / `#8642c9`  |
| Key badge size        | `.key-box` `width`/`height`                                | `36px` × `36px`        |
| Text box font size    | `.text-box` `font-size`                                    | `0.85em`               |
| Gap between boxes     | `.text-wrapper` `gap`                                      | `8px`                  |
| Indicator pulse speed | `@keyframes pulse`                                         | `2s`                   |
| Shimmer sweep speed   | `@keyframes shimmer`                                       | `3s`                   |
| Fade in/out speed     | `.interaction-point` `transition`                          | `0.3s`                 |
| Font                  | `body` `font-family`                                       | `Inter` (Google Fonts) |

***

## 🌐 Locales & Editable Strings

**Nexus TextUI has no locale system, and the prompts do not need one** — the text a player reads is the `text` you pass into `Create` from your own resource, so it is already in whatever language your resource is localised to. A consumer resource should pass its own translated string, e.g. `text = ('[E] %s'):format(T('open_garage'))`.

| Where                               | Strings                                                                                            | Editable without escrow?                                                                                          |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Caller's `Create` call              | Every prompt the player actually sees.                                                             | ✅ Yes — owned by the calling resource.                                                                            |
| `html/style.css`, `html/index.html` | No user-facing copy at all (the NUI builds its text purely from the NUI messages).                 | ✅ Yes — plain assets.                                                                                             |
| `client/main.lua`                   | The `Create` validation error, and the four strings of the `/testui` / `/deltestui` demo commands. | ❌ No — `client/main.lua` is **not** listed in `escrow_ignore` (the manifest has no `escrow_ignore` block at all). |
| `client/_resource.lua`              | The resource-name validation message.                                                              | ❌ No, and it is in Spanish.                                                                                       |

**Reported to the incident log:**

* The resource ships **no `escrow_ignore` block whatsoever**, so if it is uploaded to the escrow nothing is left editable. For a resource with no `config.lua`, `functions.lua` or `locales.lua`, that is consistent — but it also means the demo-command strings cannot be changed by a customer.
* The validation error inside `Create` was tagged `[mi_textui]` (a leftover resource name) and written in Spanish. **Corrected** to `[nexus_TextUI]` in English.
* `client/_resource.lua` prints in Spanish, and `fxmanifest.lua`'s `description` is in Spanish (`'Sistema de Text UI 3D Standalone y Optimizado'`) while the rest of the catalogue is in English. Reported, not changed.
* The `/testui` demo strings are hardcoded Spanish inside the escrow (`'PARA LA PRUEBA'`, `'¡La interacción de prueba ha funcionado!'`, `'Punto de prueba creado.'`, `'Punto de prueba eliminado.'`). Reported, not changed — they are developer-facing test commands, but they do ship.

***

## 🔗 Compatibility

| System                                                                      | How it integrates                                                                                                                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frameworks**                                                              | None required, none selected. There is no `Config.Framework`. Job/grade gating is done by the consumer inside its own `canInteract` predicate, which keeps the resource framework-neutral: `canInteract = function() return PlayerData.job and PlayerData.job.name == 'police' end`. Works identically on ESX, QBCore, QBox, vRP or a bare server. |
| **Databases**                                                               | None.                                                                                                                                                                                                                                                                                                                                              |
| **Notification systems**                                                    | Not used by this resource. The only notifications it produces are in the `/testui` demo commands, which use the plain GTA natives (`SetNotificationTextEntry` / `DrawNotification`) specifically so the resource stays dependency-free.                                                                                                            |
| **Target resources (`ox_target`, `qb-target`, `bt-target`)**                | Can coexist, and the two solve different problems. Use a target resource for interactions tied to _entities_ (peds, vehicles, props with bone offsets); use Nexus TextUI for interactions tied to _fixed world coordinates_, which is cheaper and gives you the distance indicator for free. Nothing conflicts — they draw independent NUIs.       |
| **Other TextUI resources (`ox_lib` TextUI, `esx_textui`, `cd_drawtextui`)** | Can coexist, but both will draw if you register the same interaction in both. This resource deliberately exposes `Create`/`Delete` rather than `Open`/`Close`, so it is **not** a drop-in replacement for an `Open`/`Close` API — see the FAQ for the adapter pattern.                                                                             |
| **Keys / fuel / banking / inventory**                                       | Not used. Any such check belongs in your `canInteract` predicate or your `onInteract` handler.                                                                                                                                                                                                                                                     |
| **Nexus Scripts catalogue**                                                 | Other Nexus resources that expose a TextUI branch in their config call `exports['nexus_TextUI']:Create` / `:Delete` from that branch.                                                                                                                                                                                                              |

***

## 💻 Developer API

> This is the section third-party developers integrate against. Everything below was read directly out of `client/main.lua`, `html/script.js` and `fxmanifest.lua`.

### Exports

Both exports are **client-side only**. There is no server script in this resource, so there are no server exports.

| Export                           | Parameters       | Returns | Description                                                                                                                                                                                                                                                                                                                                                           |
| -------------------------------- | ---------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exports['nexus_TextUI']:Create` | `config` (table) | nothing | Registers an interaction point. Requires `config.id`; without it, prints `[nexus_TextUI] Error: Create() called without a unique id.` and returns. Parses `config.text` into `config.formattedText`, initialises `config.lastScreenState`, and stores the table under `activeInteractions[config.id]`. Nothing is drawn until the player walks inside `viewDistance`. |
| `exports['nexus_TextUI']:Delete` | `id` (string)    | nothing | Unregisters the point and sends `{ action = 'remove', id = id }` to the NUI. Returns immediately (no-op) if `id` is not registered.                                                                                                                                                                                                                                   |

Both functions are also declared as globals (`Create` / `Delete`) inside the resource's own Lua state, which is why `/testui` can call `Create(...)` directly — but from **your** resource, always go through the export.

```lua
-- Register a point
exports['nexus_TextUI']:Create({
    id                  = 'garage_pillbox',
    coords              = vector3(215.76, -810.12, 30.73),
    viewDistance        = 12.0,
    interactionDistance = 2.0,
    text                = '[E] Open the garage',
    key                 = 'E',
    onInteract          = function()
        TriggerEvent('mygarage:open', 'pillbox')
    end,
    canInteract         = function()
        return not IsPedInAnyVehicle(PlayerPedId(), false)
    end,
})

-- Remove it
exports['nexus_TextUI']:Delete('garage_pillbox')
```

**Return value:** neither export returns anything, so there is no success/failure value to check. `Create` fails silently (apart from the console line) only when `id` is missing.

**Overwrite semantics:** calling `Create` again with an id that already exists replaces the stored table outright. That is the supported way to change a point's text, distances, key or predicate — there is no `Update` export. The NUI element is reused, and the new text is picked up on the next 250 ms state pass.

**Server exports — none.** `fxmanifest.lua` declares no `server_script` / `server_scripts` entry. If you need to create a point from the server, trigger your own client event and call the export there:

```lua
-- your server.lua
TriggerClientEvent('myresource:registerPoint', src, pointData)

-- your client.lua
RegisterNetEvent('myresource:registerPoint', function(data)
    exports['nexus_TextUI']:Create({
        id = data.id, coords = vector3(data.x, data.y, data.z),
        viewDistance = 10.0, interactionDistance = 2.0,
        text = '[E] ' .. data.label, key = 'E',
        onInteract = function() TriggerServerEvent('myresource:use', data.id) end,
    })
end)
```

### Events — Emitted

| Event    | Side | Payload | When                                                                                                                                                                                                                                               |
| -------- | ---- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| _(none)_ | —    | —       | **The resource emits no Lua events at all** — not on create, not on delete, not on interact. All communication back to your code goes through the `onInteract` / `canInteract` callbacks you supplied, which are called directly as Lua functions. |

If you need an event-based integration (for logging, analytics or anti-cheat), raise it yourself inside your own `onInteract`:

```lua
onInteract = function()
    TriggerEvent('myresource:textuiUsed', 'garage_pillbox')
    TriggerServerEvent('myresource:logInteraction', 'garage_pillbox')
end
```

### Events — Listened

| Event    | Side | Payload | Purpose                                                                                                                                                                                                                                                           |
| -------- | ---- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| _(none)_ | —    | —       | The resource registers **no** `RegisterNetEvent` and **no** `AddEventHandler` for gameplay logic. It cannot be driven by events, only by its exports. This also means a malicious client cannot make another player's screen show a prompt through this resource. |

### Commands

| Command      | Restricted | Purpose                                                                                                                                                                                             |
| ------------ | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/testui`    | No         | Registers a demo point (`id = 'test_interaction'`) 2 m in front of the player, `viewDistance = 5.0`, `interactionDistance = 2.0`, key `E`, with `canInteract` returning `false` while in a vehicle. |
| `/deltestui` | No         | Deletes `test_interaction`.                                                                                                                                                                         |

Both are registered unconditionally — there is no `Config.Debug` gate — so **any player on your server can run them**. They only create a point for the caller's own client and cannot affect anyone else, but it does mean the demo commands (and their hardcoded Spanish strings) ship live on every server; see the incident log above.

### NUI Messages (Lua → JavaScript)

Documented for anyone forking `html/`.

| Action            | Payload                                    | Sent by                                                       | Effect                                                                                                                                                                                                                                                                                                                                                            |
| ----------------- | ------------------------------------------ | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `updateContent`   | `{ item = { id, type, content? } }`        | the 250 ms state thread, once per in-range point per pass     | Creates the DOM element on first sight (a `.indicator-wrapper` plus a `.text-wrapper` containing `.key-box` and `.text-box`), writes `content.key` / `content.text` when `type == 'text'`, and swaps the `show-indicator` / `show-text` class. `type` is `'text'` inside `interactionDistance`, `'indicator'` outside it. `content` is only present for `'text'`. |
| `updatePositions` | `{ positions = { {id, x, y, scale}, … } }` | the frame-rate render thread, **only when something changed** | Sets `style.left = x + '%'`, `style.top = y + '%'` and `style.transform = translate(-50%,-50%) scale(scale)` on each listed element. Batched: all changed points in one message.                                                                                                                                                                                  |
| `remove`          | `{ id = id }`                              | `Delete`, and the state thread for points that left range     | Drops the `visible` class and removes the element from the DOM and from the JS `Map` after 300 ms.                                                                                                                                                                                                                                                                |

### Editable Functions

**There is no `functions.lua`.** Unlike the gameplay resources in the catalogue, Nexus TextUI has no external system to branch on — no notification backend, no framework, no target — so there is nothing to abstract behind editable functions. The two customisation surfaces are:

1. **Per point, at call time** — `text`, `key`, the two distances, `onInteract` and `canInteract` are all supplied by the caller, so behaviour is customised per interaction rather than per server.
2. **`html/style.css`** — the whole visual identity (colours, sizes, animation timings, font). NUI files ship as plain assets and are never encrypted, so a customer can restyle the prompt freely without touching Lua.

### Database Schema

**None.** The resource creates no tables, ships no `.sql` file, and persists nothing — not even KVP. Every registered point lives only in the client's memory and is gone on resource restart or player reconnect, which is why consumers should register their points from a start-up thread or on `playerLoaded` rather than once in a one-shot event.

### Integration Example

A complete job-gated shop point that registers on player load, re-registers after a resource restart, respects the player's job through `canInteract`, and cleans up after itself.

```lua
-- ───────────────────────── client.lua of your resource
local POINTS = {
    { id = 'ammunation_1', coords = vector3(22.09, -1107.28, 29.79),  label = 'Ammunation' },
    { id = 'ammunation_2', coords = vector3(810.25, -2157.60, 29.61), label = 'Ammunation' },
}

local registered = false

local function hasTextUI()
    return GetResourceState('nexus_TextUI') == 'started'
end

local function registerPoints()
    if not hasTextUI() or registered then return end
    for _, p in ipairs(POINTS) do
        exports['nexus_TextUI']:Create({
            id                  = p.id,
            coords              = p.coords,
            viewDistance        = 15.0,   -- diamond indicator from 15 m
            interactionDistance = 1.8,    -- prompt + key from 1.8 m
            text                = ('[E] %s'):format(p.label),
            key                 = 'E',
            canInteract         = function()
                -- hidden in a vehicle, while dead, and outside opening hours
                if IsPedInAnyVehicle(PlayerPedId(), false) then return false end
                if IsEntityDead(PlayerPedId()) then return false end
                local h = GetClockHours()
                return h >= 8 and h < 22
            end,
            onInteract          = function()
                TriggerServerEvent('myshop:open', p.id)
            end,
        })
    end
    registered = true
end

local function unregisterPoints()
    if not hasTextUI() then return end
    for _, p in ipairs(POINTS) do
        exports['nexus_TextUI']:Delete(p.id)
    end
    registered = false
end

-- Register on spawn and on framework load; points live in client memory only,
-- so they must be re-created after any restart or reconnect.
AddEventHandler('playerSpawned', registerPoints)
RegisterNetEvent('esx:playerLoaded',     registerPoints)
RegisterNetEvent('QBCore:Client:OnPlayerLoaded', registerPoints)

CreateThread(function()
    while not hasTextUI() do Wait(500) end
    registerPoints()
end)

-- Always clean up, or stale prompts stay on screen until the player reconnects
AddEventHandler('onClientResourceStop', function(res)
    if res == GetCurrentResourceName() then unregisterPoints() end
end)

-- If nexus_TextUI itself restarts, our points are gone from ITS memory — re-add them
AddEventHandler('onClientResourceStart', function(res)
    if res == 'nexus_TextUI' then
        registered = false
        Wait(500)
        registerPoints()
    end
end)
```

**Dynamic text** — because there is no `Update` export, re-`Create` with the same id:

```lua
local function setShopPrompt(id, coords, label)
    exports['nexus_TextUI']:Create({
        id = id, coords = coords,
        viewDistance = 15.0, interactionDistance = 1.8,
        text = ('[E] %s'):format(label),      -- new text
        key = 'E',
        onInteract = function() TriggerServerEvent('myshop:open', id) end,
    })
end

setShopPrompt('ammunation_1', POINTS[1].coords, 'Ammunation — CLOSED')
```

**Adapter for code written against an `Open`/`Close` TextUI** — the two models are not equivalent (this resource owns the distance check, so there is nothing to "open"), but a screen-anchored stand-in can be built by registering a point on the player's own position. The clean migration is to delete your distance loop and let the point own it:

```lua
-- Before (Open/Close style, your own loop)
CreateThread(function()
    while true do Wait(5)
        if #(GetEntityCoords(PlayerPedId()) - shopCoords) < 2.0 then
            TextUI:Open('[E] Shop')
            if IsControlJustPressed(0, 38) then openShop() end
        else
            TextUI:Close()
        end
    end
end)

-- After (Nexus TextUI, no loop)
exports['nexus_TextUI']:Create({
    id = 'shop', coords = shopCoords,
    viewDistance = 12.0, interactionDistance = 2.0,
    text = '[E] Shop', key = 'E',
    onInteract = openShop,
})
```

***

## ❓ FAQ

**I called `Open()` / `Close()` and got a nil error.** Those functions do not exist. This resource exposes **`Create(config)`** and **`Delete(id)`** — a point registry, not a show/hide helper. See the adapter pattern in the Integration Example: you register the point once and delete your distance loop entirely.

**The key letter appears twice — the badge shows `E` and the text box shows `[E] Open the garage`.** Known display issue, logged as an incident. `Create` fills the key badge from the bracketed part of `text`, but then rebuilds the text-box string as `"[KEY] rest"` including the bracket, so the key renders in both boxes. Until it is fixed, the clean workaround is to **not** use the bracket pattern: pass `text = 'Open the garage'` and `key = 'E'`. The fallback branch then sets the badge from `config.key` and the text box to your string verbatim, giving the intended `[E] | Open the garage` layout.

**Nothing appears at all.** Check, in order: (1) the folder is named exactly `nexus_TextUI` with capital `UI` — on Linux the check is case-sensitive and the resource aborts; (2) `coords` is a real `vector3`, not `{x, y, z}` — the code uses vector arithmetic and will error on a plain table; (3) you are inside `viewDistance`; (4) your `canInteract` is not returning `false` (remember a predicate that returns _nothing_ returns `nil`, which is falsy — always `return true`); (5) `nexus_TextUI` is started _before_ the resource that calls `Create`.

**The prompt shows but pressing the key does nothing.** `config.key` must be an **uppercase name from the internal `Keys` table** — `'E'`, not `'e'`, and not the numeric control id. An unrecognised name resolves to `nil` and will throw inside `IsControlJustPressed` the moment you walk into `interactionDistance`. Also confirm you actually passed `onInteract`: without it the prompt is purely informational and the press is ignored. And note the key is only polled **inside `interactionDistance`**, not inside `viewDistance`.

**The prompt flickers / dims rhythmically while I stand next to it.** Known issue, logged as an incident. The 250 ms state thread re-sends `updateContent` for every in-range point on every pass, and the NUI handler removes and re-adds the `visible` class each time, restarting the 0.3 s opacity transition roughly four times a second. It is cosmetic — the interaction still works.

**My points vanished after I restarted a resource.** Expected: registrations live only in the calling client's memory. If **your** resource restarts, re-register on start. If **`nexus_TextUI`** restarts, its `activeInteractions` table is emptied and every consumer must re-register — hook `onClientResourceStart` for `'nexus_TextUI'` as shown in the Integration Example.

**I deleted a point and `onInteract` still fired once afterwards.** Known race, logged as an incident. `Delete` clears the registry and tells the NUI to remove the element immediately, but the frame-rate render thread works from a second table that is only resynchronised on the next 250 ms pass — so for up to 250 ms after `Delete`, a key press can still reach your `onInteract`. Guard inside your handler (e.g. a `local deleted = true` flag) if a late call would be harmful.

**`canInteract` is not reacting fast enough.** It is evaluated on the 250 ms state thread, so a state change can take up to a quarter of a second to hide or show the point. That is the intended trade-off for the resource's low cost; do not use `canInteract` for anything that must be instantaneous. Keep the predicate cheap, too — it runs for every registered point, four times a second.

**Can two resources register the same `id`?** They can, and the second one silently overwrites the first, including its `onInteract`. Namespace your ids with your resource name (`myshop:ammunation_1`) to avoid collisions.

**How many points can I register?** There is no hard limit. The 250 ms thread is O(number of registered points) and the frame-rate thread is O(number of _visible_ points), so cost scales with how many points are near the player, not how many exist. A few hundred registered points with a handful visible is comfortable.

**Can I change the purple colour or the box sizes?** Yes — edit `html/style.css` (`.key-box`, `.text-box`, `.indicator-wrapper::before`). NUI files are plain assets and are never encrypted, so this survives the escrow. There is no config key for it.

**Can I register a point from the server?** Not directly — the resource is client-only and has no server exports. Trigger your own client event and call `Create` there (example in the Developer API section).

**Does it draw through walls?** Yes. The resource uses `GetScreenCoordFromWorldCoord` with no occlusion or raycast check, so a point behind a wall is still visible if the player is within `viewDistance` and facing it. Add your own visibility test in `canInteract` if that matters.

**Can I use it without a framework?** Yes. The resource never references ESX, QBCore or any other framework; it is pure client-side Lua plus NUI.

### Before opening a ticket

* Make sure the resource's name is exactly **`nexus_TextUI`** (capital `U`, capital `I`).
* Make sure you are using the latest version of the resource (**1.0.0**).
* Run `/testui` in game. If the demo point works, the resource is fine and the problem is in the calling resource's `Create` call — check `coords` is a `vector3`, `key` is an uppercase name from the `Keys` table, and `canInteract` explicitly returns `true`.
* Confirm `nexus_TextUI` starts **before** the resources that register points, and that `GetResourceState('nexus_TextUI') == 'started'` when you call `Create`.
* Check F8 for a Lua error from your own `onInteract` / `canInteract` callback — an error thrown inside your callback surfaces as the prompt "not working".
* Review this FAQ page.

***

## 📋 Changelog

**Current version: 1.0.0** (`fxmanifest.lua`)

* `1.0.0` — initial release. Point-registry API (`Create` / `Delete`), two-stage proximity rendering (diamond indicator → key + text prompt), distance-based scaling, built-in key handling, `canInteract` predicate, change-gated batched NUI position updates, adaptive render loop, and the `/testui` / `/deltestui` demo commands.

> The known issues listed in the FAQ above — the duplicated key badge/text display, the 250 ms flicker, and the delete-race condition on `onInteract` — are open incidents against this version and are not yet fixed in a released update.
