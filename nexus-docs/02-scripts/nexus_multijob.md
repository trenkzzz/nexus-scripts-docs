# Nexus Notify

A fully client-configurable notification system for FiveM: every player drags, scales and styles their own notifications, and any resource can push them with a single export — from both client and server.

## 📝 Description

Nexus Notify replaces the stock GTA / framework notification feed with a standalone NUI notification stack that each player controls personally. A player runs `/notifymove`, drags the notification anchor anywhere on screen, scales it between 50% and 200% with a slider, optionally switches to a compact "minimalist" pill style, and saves. Those three settings are persisted **per player, per machine** using FiveM's KVP store (`SetResourceKvpInt` / `SetResourceKvpFloat`), so they survive reconnects and server restarts without any database.

Internally the resource is deliberately thin. `client.lua` owns a table of currently-displayed persistent notifications, resolves the requested notification type against `Config.Styles`, resolves the sound against `Config.Sounds`, injects the player's saved position/scale/style into the payload, and pushes a single `SendNUIMessage` with one of three actions: `show`, `update` or `hide`. All layout, animation, colour application, text markup parsing and the countdown timer live in `html/script.js`, which means adding a new notification type is a pure config change — a new entry in `Config.Styles` with an SVG filename, a hex colour, a card background and a glow string, and the NUI renders it with no code edits.

The notification stack is positional-aware: because the player can anchor notifications in any corner, the NUI recomputes `left/right/top/bottom`, `transform-origin`, `flex-direction` and `align-items` from the saved coordinates, and inserts new cards with `prepend` instead of `appendChild` when the anchor sits in the lower half of the screen. The result is that notifications always grow *away* from the screen edge regardless of where the player parked them.

Three notification shapes are supported through the same `Alert` export: a standard timed notification with a progress bar, a **persistent** notification addressed by a caller-supplied `persistentId` that stays on screen until explicitly removed (and that *updates in place* if `Alert` is called again with the same id), and a **button** notification that renders clickable buttons which fire client events back into your own resource. The server side is a thin relay: the same `Alert` / `Remove` export names exist server-side with a leading `target` argument, validate the target, and forward over `nexus_notify:client:Alert` / `nexus_notify:client:Remove`.

## ✨ Features

- **Per-player position** — `/notifymove` opens a draggable anchor panel; the player drags it anywhere, and the position is stored as a percentage of screen width/height so it is resolution-independent.
- **Per-player scale** — a slider in the move panel from `0.5` to `2.0` in `0.1` steps, applied to the whole stack through a CSS `transform: scale()`.
- **Two visual styles** — `style-full` (icon bubble, title, message, progress bar, 320px card) and `style-mini` (compact pill, no title, no progress bar, auto width), switchable by the player with a toggle switch in the move panel when `AllowUserMinimalistToggle` is enabled.
- **Persistence without a database** — position, scale and minimalist choice are written to FiveM KVP (`nexus_notify_pos_x`, `nexus_notify_pos_y`, `nexus_notify_scale`, `nexus_notify_minimalist`). No SQL file, no table, no framework dependency.
- **Reset-to-default button** — optional button in the move panel (`EnableResetButton`) that pulls `DefaultPosition` / `DefaultScale` back from config through the `resetPosition` NUI callback.
- **Fully config-driven notification types** — five shipped types (`info`, `success`, `warning`, `error`, `admin`), each with its own SVG icon file, accent colour, card background and glow. Add, edit or delete entries in `Config.Styles` and the NUI follows; an unknown type silently falls back to `info`.
- **Server-logo mode** — set `UseLogoInsteadOfIcon = true` and every notification renders your server logo (`LogoFileName`, served from `html/`) in the icon bubble instead of the per-type SVG.
- **Per-type sounds with a global kill switch** — `Config.Sounds` maps each type to a filename inside `html/sounds/`; `EnableSounds` and `DefaultVolume` control the feature globally, and the `playSound` argument overrides it per call (`true` = force, `false` = silence, `nil` = follow config).
- **Entrance / exit animations** — the full card expands from a circle to a 320px rounded card (`fullIn`), the icon pops with a rotate-and-overshoot keyframe (`iconPop`), the text fades in after the expansion finishes, and the exit collapses back into a circle and shrinks to zero height (`fullOut`). The mini style has its own `pillIn` / `pillOut` pair.
- **Progress bar** — timed notifications in full style draw a 2px bar in the notification's accent colour that fills across the configured duration. Persistent notifications and mini-style notifications draw no bar.
- **Text markup** — the message body supports GTA-style colour tags `~r~ ~g~ ~b~ ~y~ ~o~ ~p~` closed with `~s~`, plus `**bold**` and `*italic*`, parsed in the NUI and mapped to real CSS colours.
- **Persistent notifications with in-place update** — calling `Alert` again with the same `persistentId` sends `action = 'update'` and rewrites the title and message of the existing card instead of stacking a duplicate.
- **Button notifications** — pass `options.buttons`; each button renders in the card, and clicking it posts back through the `buttonClick` NUI callback, which `TriggerEvent`s the event name you supplied with the params you supplied. Buttons force full style even for players using minimalist mode.
- **Position-aware stacking** — cards are prepended instead of appended when the anchor is in the lower half of the screen, so the newest notification is always the one closest to the anchor.
- **Built-in position-check command** — `Config.TestNotifyCommand` (default `/showpos`) fires one notification of every configured type, staggered 600ms apart, so the player can confirm where they parked the stack.
- **Four optional debug commands** — gated behind `EnableDebugCommands`: one notification per type, show a persistent notification, hide that persistent notification, and show a two-button notification with working example handlers. All four register `chat:addSuggestion` entries.
- **Framework-free** — no ESX, no QBCore, no database, no `ox_lib`. It runs on a bare server.

## 📋 Dependencies

| Dependency | Required | Notes |
|---|---|---|
| Framework (ESX / QBCore) | ❌ No | The resource never touches a framework object. It is fully standalone. |
| Database / MySQL | ❌ No | Player settings use FiveM KVP, not SQL. There is no `.sql` file. |
| `ox_lib` or any UI library | ❌ No | The NUI is self-contained. |
| Internet access on the client | ⚠️ Soft | `html/style.css` imports the Roboto font from Google Fonts. Without internet the NUI falls back to the generic sans-serif; nothing breaks. |

The only hard requirement is that **the resource folder must be named exactly `nexus_notify`**. Both `_resource.lua` and `client.lua`/`server.lua` validate this on start and abort if it differs, because the export namespace (`exports['nexus_notify']`) and the NUI callback URLs (`https://nexus_notify/...`, hardcoded in `html/script.js`) depend on that exact name.

## ⚙️ Installation

1. **Download and extract the resource.** Download it from the cfx.re (Keymaster) portal and place the folder in your `resources` directory.
2. **Do not rename the folder.** It must stay `nexus_notify`. The NUI posts its callbacks to `https://nexus_notify/...`, so a renamed folder breaks the move menu, the reset button and all notification buttons.
3. **Add it to `server.cfg`.** It has no dependencies, so it can go anywhere — but put it **before** every resource that will send notifications through it, so the export exists by the time they start:
   ```cfg
   ensure nexus_notify
   -- then the resources that consume it
   ensure nexus_bounty
   ensure nexus_multijob
   ```
4. **Import no SQL.** There is nothing to import.
5. **Adapt `config.lua`.** Set `DefaultPosition`, `DefaultScale`, `MinimalistMode` and `DefaultDuration` to the defaults you want new players to get, and review `Config.Styles` if you want your own colours.
6. **Optional — use your own logo.** Drop your logo into `html/`, set `LogoFileName` to its filename and `UseLogoInsteadOfIcon = true`.
7. **Optional — add your own sounds.** Drop `.wav`/`.ogg` files into `html/sounds/` and point the entries of `Config.Sounds` at them. `html/sounds/*` is already covered by the `files` block in the manifest, so no manifest edit is needed.
8. **Restart.** A full server restart is recommended so other resources pick up the export.
9. **Verify.** In game, run `/showpos` — you should get one notification of every configured type. Then run `/notifymove`, drag the panel, set a scale, and press **Save & Close**.

## 🔧 Configuration

Everything lives in `config.lua`, which is listed in `escrow_ignore` and therefore ships unencrypted and fully editable.

### Visual

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.DefaultPosition` | table `{x, y}` | `{ x = 99, y = 30 }` | Default anchor for players who have never moved their notifications, as a **percentage** of screen width (`x`) and height (`y`). A value above `50` is interpreted by the NUI as "anchored to the right / bottom edge", which flips the stacking direction and the transform origin. |
| `Config.DefaultScale` | number | `1.0` | Default scale multiplier for players who have never scaled their notifications. `1.0` = 100%, `0.5` = 50%, `1.5` = 150%. The in-game slider clamps player choices to `0.5`–`2.0`. |
| `Config.UseLogoInsteadOfIcon` | boolean | `false` | `true` renders `LogoFileName` in the icon bubble of every notification instead of the per-type SVG. `false` uses each type's `iconFile`. |
| `Config.LogoFileName` | string | `'logo.png'` | Filename of the logo, resolved relative to `html/`. Must also be present in the manifest's `files` block (`html/logo.png` already is). |
| `Config.EnableResetButton` | boolean | `true` | Shows the **Reset to Default** button inside the `/notifymove` panel. |
| `Config.DefaultDuration` | number (ms) | `5000` | Duration applied when the caller does not pass a `duration`. `5000` = 5s. |
| `Config.TestNotifyCommand` | string | `'showpos'` | Name of the always-available command that fires one notification per configured type so players can see where their stack sits. Registered without the `/`. |
| `Config.MinimalistMode` | boolean | `false` | Default card style. `false` = full card (icon, title, message, progress bar). `true` = compact pill (icon + message only). |
| `Config.AllowUserMinimalistToggle` | boolean | `true` | `true` shows the Minimalist Mode toggle switch inside the `/notifymove` panel so players can choose. `false` hides the toggle and also makes the `saveMinimalist` callback a no-op, locking everyone to `MinimalistMode`. |

### Sounds

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.EnableSounds` | boolean | `true` | Global sound switch. When `false`, no notification plays audio unless the caller explicitly passes `playSound = true`. |
| `Config.DefaultVolume` | number | `0.7` | Volume passed to the NUI. *Note: the NUI currently constructs the `Audio` object without applying this value — see the FAQ.* |
| `Config.Sounds` | table | see below | Maps a notification type name to a filename inside `html/sounds/`. A type with no entry here plays nothing. |

```lua
Config.Sounds = {
    info    = 'opening-sound.wav',
    success = 'opening-sound.wav',
    warning = 'opening-sound.wav',
    error   = 'opening-sound.wav',
    admin   = 'opening-sound.wav',
}
```
The key must match the notification type name exactly. The value is resolved by the NUI as `sounds/<value>`, so the file must sit in `html/sounds/` and be covered by the manifest's `files { 'html/sounds/*' }` entry (it already is).

### Styles

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Styles` | table of tables | 5 entries | The complete list of notification types. The key is the type name you pass as the `notificationType` argument. An unrecognised type falls back to `Config.Styles['info']`. |

Each entry has exactly four fields:

```lua
Config.Styles = {
    ['info'] = {
        iconFile   = 'icons/info.svg',                 -- SVG path relative to html/
        color      = '#3b82f6',                        -- accent: border, title, icon bubble, progress bar
        background = 'rgba(20, 20, 26, 0.97)',         -- card background (rgba supported for transparency)
        glow       = '0 0 15px rgba(59, 130, 246, 0.25)' -- full CSS box-shadow value
    },
    -- add your own:
    ['police'] = {
        iconFile   = 'icons/police.svg',
        color      = '#1d4ed8',
        background = 'rgba(10, 14, 30, 0.97)',
        glow       = '0 0 18px rgba(29, 78, 216, 0.35)'
    },
}
```

| Field | Type | Applied to | Notes |
|---|---|---|---|
| `iconFile` | string | `<img src>` in the icon bubble | Relative to `html/`. Must be inside `html/icons/` to be covered by the manifest's `files { 'html/icons/*' }` entry. Ignored when `UseLogoInsteadOfIcon = true`. |
| `color` | string (hex) | card `border-color`, icon bubble `background-color`, title colour (full style), progress bar colour | In mini style the SVG is rendered with `filter: brightness(0)`, i.e. black on the coloured bubble, so pick a light-to-mid accent. |
| `background` | string (CSS colour) | card `background-color` | Falls back to `rgba(20, 20, 26, 0.97)` in the NUI if omitted. |
| `glow` | string (CSS box-shadow) | card `box-shadow` | Full shadow value, not just a colour. |

Shipped types: `info` (`#3b82f6`), `success` (`#00FF00`), `warning` (`#f59e0b`), `error` (`#f43f5e`), `admin` (`#9333ea`).

### Debug

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.EnableDebugCommands` | boolean | `false` | Master switch for the four debug commands below. **Leave this `false` in production** — the commands are registered unrestricted, so any player could run them. |
| `Config.DebugTestAllCommand` | string | `'ndebug_all'` | Fires one notification per configured type, staggered 600ms. |
| `Config.DebugPersistentCommand` | string | `'ndebug_persistent'` | Shows a persistent `warning` notification with id `debug_persistent`. |
| `Config.DebugPersistentHide` | string | `'ndebug_hide'` | Removes the `debug_persistent` notification. |
| `Config.DebugButtonsCommand` | string | `'ndebug_buttons'` | Shows a 15s `info` notification with **Accept** / **Deny** buttons wired to two example handlers that reply with a success/error notification. |

## 🌐 Locales & Editable Strings

**Nexus Notify has no locale system, and it does not need one for its own output** — every string a player sees in a notification is the `title` and `text` *you* pass in from your own resource, in whatever language you want.

| Where | Strings | Editable without escrow? |
|---|---|---|
| `config.lua` | All config values and comments. | ✅ Yes — in `escrow_ignore`. |
| `html/index.html` | The `/notifymove` panel UI: `Notification Position & Scale`, `Drag to set your preferred position.`, `Notification Size (…%)`, `Minimalist Mode`, `Save & Close`, `Reset to Default`. | ✅ Yes — NUI files ship as plain assets and are never encrypted. Edit them directly to translate the move panel. |
| `client.lua` | `'You have moved the notifys here'` (the `/showpos` message) and the four debug notification bodies. | ❌ No — `client.lua` is inside the escrow. |
| `_resource.lua`, `client.lua`, `server.lua` | The resource-name validation messages. | ❌ No (`client.lua`/`server.lua`), ✅ `_resource.lua` is not escrow-ignored either. |

**Reported to the incident log:** the `/showpos` notification body, the four debug notification bodies and the resource-name error messages are hardcoded **inside the escrow** and cannot be translated by a customer. `_resource.lua` additionally prints its message in Spanish while the rest of the resource is in English. Neither affects normal gameplay — `/showpos` is a diagnostic command and the debug commands are off by default — but a `Config.Text = { ... }` block in `config.lua` would remove the limitation.

## 🔗 Compatibility

| System | How it integrates |
|---|---|
| **Frameworks** | None required. The resource is framework-agnostic and never reads a player object, job or identifier. It works on ESX, QBCore, vRP, QBox or a bare server with no changes. |
| **Databases** | None. Player preferences use FiveM's KVP store, scoped to the resource and to the individual player's machine. |
| **Other notification resources** | Can coexist. Nexus Notify does **not** override `ESX.ShowNotification`, `QBCore.Functions.Notify`, `chat:addMessage` or the GTA natives, and it does not register itself as a drop-in replacement for `okokNotify` / `mythic_notify`. Resources keep using whatever they already use until you point them at `exports['nexus_notify']:Alert`. |
| **Nexus Scripts catalogue** | Other Nexus resources select their notification backend through their own `Config.NotifySystem` / `Config.NotifyStyle` key and their own `functions.lua`; choosing the `nexus` / `nexus_notify` branch there makes them call this resource's `Alert` export. Nexus Notify itself needs no configuration for that. |
| **GTA colour codes** | The NUI parses `~r~ ~g~ ~b~ ~y~ ~o~ ~p~` + `~s~` itself, so messages written for ESX/QBCore native notifications display correctly without rewriting them. |
| **TextUI / target / keys / fuel / banking** | Not used. This resource has no world interaction at all. |

## 💻 Developer API

> This is the section third-party developers integrate against. Everything below was read directly out of `client.lua`, `server.lua` and `html/script.js`.

### Client Exports

Call these from any **client** script.

| Export | Parameters | Returns | Description |
|---|---|---|---|
| `exports['nexus_notify']:Alert` | `title` (string), `text` (string), `duration` (number\|nil), `notificationType` (string\|nil), `playSound` (boolean\|nil), `options` (table\|nil) | nothing | Shows a notification to the local player. Returns early and does nothing if `title` *or* `text` is falsy. |
| `exports['nexus_notify']:Remove` | `persistentId` (string) | nothing | Hides and forgets the persistent notification with that id. No-op if no notification with that id is currently tracked. |

**Argument detail for `Alert`:**

| Argument | Type | Default when `nil` | Notes |
|---|---|---|---|
| `title` | string | — | **Required.** Rendered in the accent colour in full style. **Not rendered at all in minimalist style** (`.notification-title { display: none }`). |
| `text` | string | — | **Required.** The message body. Supports `~r~ ~g~ ~b~ ~y~ ~o~ ~p~` … `~s~`, `**bold**` and `*italic*`. Single-line: it is clipped with an ellipsis, not wrapped. |
| `duration` | number (ms) | `Config.DefaultDuration` | Pass `0` together with `options.persistentId` for a notification that never auto-hides. |
| `notificationType` | string | `'info'` | Any key of `Config.Styles`. An unknown value silently falls back to `info`. |
| `playSound` | boolean | `Config.EnableSounds` | `true` forces sound even if `EnableSounds = false`; `false` forces silence; `nil` follows config. The actual file comes from `Config.Sounds[notificationType]` — a type missing from that table plays nothing even when `playSound = true`. |
| `options` | table | `nil` | `{ persistentId = string, buttons = table }`. See below. |

**`options.persistentId`** (string) — makes the notification persistent. Calling `Alert` again with the same id sends `action = 'update'` and rewrites the existing card's title and message in place instead of stacking a second card. Persistent notifications render **no progress bar**. Remove them with `:Remove(id)`.

**`options.buttons`** (array of tables) — each entry:

| Field | Type | Description |
|---|---|---|
| `label` | string | Button text. If `key` is set, the label is rewritten in place to `"<label> (<key>)"`. |
| `key` | string | Optional key hint, e.g. `'F7'`. **See the FAQ: this only appends text to the label — the keybind itself is not currently wired up.** |
| `event` | string | Name of the **client** event triggered via `TriggerEvent` when the button is clicked. |
| `params` | any | Single value passed as the sole argument to that event. |

Clicking any button hides the notification immediately. Button notifications always render in full style, even for a player who chose minimalist mode.

```lua
-- Simple
exports['nexus_notify']:Alert('Garage', 'Vehicle stored.', 4000, 'success', true)

-- With markup, following the player's sound preference
exports['nexus_notify']:Alert('Bank', 'You received ~g~$1,250~s~ from **payroll**.', 6000, 'info')

-- Persistent status card, updated in place
exports['nexus_notify']:Alert('Delivery', 'Packages left: 4', 0, 'info', false, {
    persistentId = 'delivery_progress'
})
exports['nexus_notify']:Alert('Delivery', 'Packages left: 3', 0, 'info', false, {
    persistentId = 'delivery_progress'   -- updates the same card
})
exports['nexus_notify']:Remove('delivery_progress')

-- Buttons
exports['nexus_notify']:Alert('Invite', 'Join the crew?', 15000, 'info', true, {
    buttons = {
        { label = 'Accept', key = 'F7', event = 'mycrew:accept', params = { id = 12 } },
        { label = 'Deny',   key = 'F8', event = 'mycrew:deny',   params = { id = 12 } },
    }
})
```

### Server Exports

Call these from any **server** script. Identical names, with `target` inserted as the **first** argument.

| Export | Parameters | Returns | Description |
|---|---|---|---|
| `exports['nexus_notify']:Alert` | `target` (number), `title`, `text`, `duration`, `notificationType`, `playSound`, `options` | nothing | Relays to `nexus_notify:client:Alert` on the target. Validates the target first: if `target ~= -1` and `GetPlayerName(target)` is falsy, it prints `[nexus_notify] ERROR: Alert called with invalid target: <target>` and returns without sending. |
| `exports['nexus_notify']:Remove` | `target` (number), `persistentId` (string) | nothing | Relays to `nexus_notify:client:Remove`. Same target validation and same error message pattern (`Remove called with invalid target`). |

| `target` value | Effect |
|---|---|
| a player's server id | Sends to that player only. |
| `-1` | Broadcasts to **all** connected players. |
| anything else / a disconnected id | Nothing is sent; an error line is printed to the server console. |

```lua
-- One player
exports['nexus_notify']:Alert(source, 'Shop', 'Purchase complete.', 4000, 'success', true)

-- Everyone
exports['nexus_notify']:Alert(-1, 'Server', 'Restart in ~r~5 minutes~s~.', 10000, 'warning', true)

-- Persistent, server-driven, then cleared
exports['nexus_notify']:Alert(source, 'Jail', 'Time left: 10 min', 0, 'error', false, {
    persistentId = 'jail_timer'
})
exports['nexus_notify']:Remove(source, 'jail_timer')

-- Buttons: the events fire on the receiving player's CLIENT
exports['nexus_notify']:Alert(source, 'Admin', 'Report assigned to you.', 20000, 'admin', true, {
    buttons = {
        { label = 'Teleport', key = 'F7', event = 'myadmin:tpToReport', params = reportId },
    }
})
```

### Events — Emitted

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_notify:client:Alert` | server → client | `title, text, duration, type, playSound, options` | Emitted by the server-side `Alert` export after target validation. Received by `client.lua`, which forwards it straight into the client `Alert` export. |
| `nexus_notify:client:Remove` | server → client | `persistentId` | Emitted by the server-side `Remove` export. |
| *(caller-defined)* | client | the button's `params` value | `TriggerEvent(data.event, data.params)` fired from the `buttonClick` NUI callback when a player clicks a notification button. The event name and payload are entirely yours. |
| `chat:addSuggestion` | client | `'/<command>', '<description>'` | Emitted once at start for each of the four debug commands, only when `Config.EnableDebugCommands = true`. |
| `nexus_notify:debug:accept` / `nexus_notify:debug:deny` | client | `{}` | Example button events used only by the `ndebug_buttons` debug command. Handled internally; do not rely on them. |

### Events — Listened

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `nexus_notify:client:Alert` | client | `title, text, duration, type, playSound, options` | Entry point for server-originated notifications. **You can trigger this directly** with `TriggerClientEvent` if you prefer events to exports — it has the exact same effect as the client export. |
| `nexus_notify:client:Remove` | client | `persistentId` | Entry point for server-originated removals. |

> There is **no** `nexus_notify:server:*` event. Server-side usage goes through the exports; the server registers no net events, so a malicious client cannot make another player show a notification through this resource.

### NUI Callbacks

Internal, but documented because they are part of the client surface if you fork the NUI.

| Callback | Posted from | Payload | Effect in `client.lua` |
|---|---|---|---|
| `savePosition` | Save & Close button | `{ x, y, scale }` | Writes KVP `nexus_notify_pos_x`, `nexus_notify_pos_y`, `nexus_notify_scale`; closes move mode; releases NUI focus. |
| `saveMinimalist` | Minimalist toggle | `{ value = boolean }` | Writes KVP `nexus_notify_minimalist` as `1`/`0` — **only if** `Config.AllowUserMinimalistToggle` is `true`. |
| `resetPosition` | Reset to Default button | `{}` | Responds with `{ pos = Config.DefaultPosition, scale = Config.DefaultScale }`. Note this only repositions the *panel* in the NUI; nothing is persisted until the player presses Save. |
| `buttonClick` | A notification button | `{ event, params }` | `TriggerEvent(data.event, data.params)`. |

### NUI Messages (Lua → JavaScript)

| Action | Payload | Handled by |
|---|---|---|
| `show` | `{ data = notificationData }` | `showNotification()` — builds and inserts the card. |
| `update` | `{ data = notificationData }` | `updateNotification()` — rewrites `.notification-title` (`textContent`) and `.notification-message` (`innerHTML`, through `formatText`) of the card matching `data.persistentId`. |
| `hide` | `{ id = persistentId }` | `hideNotification()` — adds `.hiding`, then removes the node after the exit animation (400ms full / 380ms mini). |
| `moveMode` | `{ enabled, currentPos, currentScale, showReset, allowToggle, minimalist }` | `toggleMoveMode()` — shows/hides the drag panel. |
| `setMinimalist` | `{ value }` | Handled in `html/script.js` but **never sent by `client.lua`** — a dead branch. Noted in the incident log. |

The `notificationData` table pushed on `show` / `update`:

```lua
{
    type         = notificationType,            -- resolved type name
    title        = title,
    text         = text,
    duration     = duration or Config.DefaultDuration,
    playSound    = soundFile or false,          -- filename inside html/sounds/, or false
    volume       = Config.DefaultVolume,
    useLogo      = Config.UseLogoInsteadOfIcon,
    logoFile     = Config.LogoFileName,
    position     = userPosition,                -- the player's saved {x, y}
    scale        = userScale,                   -- the player's saved scale
    persistentId = options and options.persistentId,
    buttons      = options and options.buttons,
    hasKeybinds  = hasKeybinds,
    style        = Config.Styles[notificationType],  -- the whole style table
    minimalist   = userMinimalist,
}
```

### Editable Functions

**There is no `functions.lua` in this resource**, and it needs none: Nexus Notify is the notification backend, not a consumer of one, so it has no external system to branch on. The customisation surface is `config.lua` (types, colours, icons, sounds, defaults) plus the NUI files in `html/`, all of which ship unencrypted.

`exports.lua` exists in the folder and is listed in `escrow_ignore`, but it is **not** loaded by any `shared_scripts` / `client_scripts` / `server_scripts` entry in `fxmanifest.lua`. It is a plain-text API cheat-sheet shipped for the customer to read — it is intentionally not executable code (running it would error, since it calls the exports with undefined variables). Noted in the incident log so it is not mistaken for a dead script.

### Database Schema

**None.** This resource creates no tables and ships no `.sql` file. All persistence is FiveM KVP:

| KVP key | Type | Written by | Holds |
|---|---|---|---|
| `nexus_notify_pos_x` | int | `savePosition` | Anchor X as a percentage of screen width. |
| `nexus_notify_pos_y` | int | `savePosition` | Anchor Y as a percentage of screen height. |
| `nexus_notify_scale` | float | `savePosition` | Scale multiplier. |
| `nexus_notify_minimalist` | int (`0`/`1`) | `saveMinimalist` | Whether the player chose the compact pill style. |

KVP is stored client-side per machine, so a player's layout does not follow them to another PC, and it is not readable from the server.

### Integration Example

A delivery job that uses a persistent notification as a live progress HUD, a button notification to offer a bonus run, and a broadcast when the route is finished.

```lua
-- ───────────────────────── client.lua of your resource
local remaining = 5
local HUD_ID    = 'mydelivery_hud'

local function hasNotify()
    return GetResourceState('nexus_notify') == 'started'
end

local function notify(title, text, duration, nType, sound, options)
    if hasNotify() then
        exports['nexus_notify']:Alert(title, text, duration, nType, sound, options)
    else
        SetNotificationTextEntry('STRING')
        AddTextComponentString(text)
        DrawNotification(false, false)
    end
end

-- Persistent HUD: same id every time, so the card updates instead of stacking
local function refreshHud()
    notify('Delivery Route', ('Packages left: ~y~%d~s~'):format(remaining), 0, 'info', false, {
        persistentId = HUD_ID
    })
end

RegisterNetEvent('mydelivery:packageDelivered', function()
    remaining = remaining - 1
    if remaining > 0 then
        refreshHud()
    else
        if hasNotify() then exports['nexus_notify']:Remove(HUD_ID) end
        notify('Delivery Route', 'Route complete. Head back to the depot.', 6000, 'success', true)
    end
end)

-- Offer a bonus run with buttons
RegisterNetEvent('mydelivery:offerBonus', function(bonusId)
    notify('Bonus Run', 'A rush order is available. ~g~+$500~s~', 20000, 'warning', true, {
        buttons = {
            { label = 'Take it', key = 'F7', event = 'mydelivery:acceptBonus', params = bonusId },
            { label = 'Pass',    key = 'F8', event = 'mydelivery:declineBonus', params = bonusId },
        }
    })
end)

AddEventHandler('mydelivery:acceptBonus', function(bonusId)
    TriggerServerEvent('mydelivery:claimBonus', bonusId)
end)

AddEventHandler('mydelivery:declineBonus', function()
    notify('Bonus Run', 'Offer declined.', 3000, 'info')
end)

-- Make sure the HUD never survives a resource stop
AddEventHandler('onResourceStop', function(res)
    if res == GetCurrentResourceName() and hasNotify() then
        exports['nexus_notify']:Remove(HUD_ID)
    end
end)

-- ───────────────────────── server.lua of your resource
RegisterNetEvent('mydelivery:claimBonus', function(bonusId)
    local src = source
    -- … validate and pay …
    exports['nexus_notify']:Alert(src, 'Bonus Run', 'Bonus paid: ~g~$500~s~', 5000, 'success', true)
    exports['nexus_notify']:Alert(-1, 'Depot', 'A rush order was just claimed.', 5000, 'info', false)
end)
```

Adding your own notification type takes no code at all — append to `Config.Styles`, drop the SVG in `html/icons/`, and call it:

```lua
-- config.lua
['heist'] = {
    iconFile   = 'icons/heist.svg',
    color      = '#facc15',
    background = 'rgba(26, 20, 6, 0.97)',
    glow       = '0 0 18px rgba(250, 204, 21, 0.35)'
},
-- optionally: Config.Sounds.heist = 'alarm.wav'

-- anywhere
exports['nexus_notify']:Alert('Heist', 'Vault drilling started.', 8000, 'heist', true)
```

## ❓ FAQ

**The `key` on my notification buttons does nothing — the button works when clicked, but pressing the key does not.**
Correct, and this is a current limitation rather than a misconfiguration. Setting `key` appends `" (F7)"` to the button label and flags the notification as persistent internally, but the client keybind-listening thread in `client.lua` has no body yet — its inner loop over the tracked notifications is empty. Use the buttons with the mouse, or register your own `RegisterKeyMapping` / `IsControlJustPressed` handler in your resource and call `exports['nexus_notify']:Remove(id)` yourself.

**I passed a `persistentId` *and* buttons with keys, and now `Remove(myId)` does not remove it.**
When any button has a `key`, the resource overwrites your `persistentId` with an internally generated one (`_keybind_notify_<random>`). Your own id is never registered, so `Remove` cannot find it. If you need to remove a notification by id, do not set `key` on its buttons.

**New players do not get `Config.DefaultPosition` — notifications appear in the top-left corner.**
This is a known bug, not a config mistake. The resource reads the saved position with `GetResourceKvpInt`, which returns `0` (not `nil`) when the key has never been written, so the `or Config.DefaultPosition.x` fallback never runs and a brand-new player ends up at `0, 0`. Workaround: tell players to run `/notifymove` once and press **Save & Close**, which writes real KVP values. It is listed in the incident log for a code fix.

**A player turned minimalist mode off but it keeps coming back on.**
Same class of bug. The saved value `0` (meaning "off") is treated as "nothing saved", so the code falls back to `Config.MinimalistMode`. If you have `MinimalistMode = true` in config, players cannot persistently opt out. Set `MinimalistMode = false` as the server default and let players opt *in*, which does persist correctly.

**My notification titles are invisible.**
You (or the player) are in minimalist mode. The pill style deliberately hides the title (`.notification-title { display: none }`) and shows only the icon and the message. Put anything essential in `text`, not `title`, if your server runs `MinimalistMode = true`.

**Long messages get cut off with "…".**
By design. Both styles set `white-space: nowrap` with `text-overflow: ellipsis`, so notifications are strictly single-line. Split long content across two notifications, or use a persistent notification that you update.

**`Config.DefaultVolume` has no effect.**
The volume is sent to the NUI in the payload but `html/script.js` creates the sound with `new Audio('sounds/' + file).play()` without assigning `.volume`. Until that is wired up, adjust loudness in the audio file itself. Logged as an incident.

**No sound plays for one of my notification types.**
The type must have an entry in `Config.Sounds` *and* the file must exist in `html/sounds/`. A type present in `Config.Styles` but absent from `Config.Sounds` is silent even with `playSound = true`, because the resource resolves the filename from `Config.Sounds[notificationType]`.

**I renamed the folder and the move menu, the reset button and all notification buttons stopped working.**
The folder must be exactly `nexus_notify`. `_resource.lua` aborts the resource outright on a mismatch, and even if it did not, `html/script.js` posts its callbacks to the hardcoded URL `https://nexus_notify/...`. Rename it back.

**Do I have to replace every `ESX.ShowNotification` call in my other scripts?**
No. Nexus Notify does not hijack framework notifications — it adds a parallel API. Existing calls keep going to your framework. Migrate resource by resource by swapping the call for `exports['nexus_notify']:Alert`, or, for Nexus Scripts resources, just select the `nexus` branch in that resource's own notification config key.

**Can the server remove a persistent notification from every player at once?**
Yes: `exports['nexus_notify']:Remove(-1, 'my_id')`.

**Notifications stack downward but my anchor is at the bottom of the screen.**
They should not — the NUI flips `flex-direction` and prepends new cards when `y > 50`. If stacking looks wrong, the player likely saved a position with `y` just under `50`. Have them run `/notifymove` and drag further toward the edge.

**Is anything stored server-side?**
No. Player layout preferences live in client-side KVP only. They do not follow the player to a different computer and cannot be read, reset or migrated from the server.

### Before opening a ticket

- Make sure the resource's name is exactly **`nexus_notify`**.
- Make sure you are using the latest version of the resource (**3.1.1**).
- Confirm `nexus_notify` starts **before** the resource that is calling its exports, and that `GetResourceState('nexus_notify') == 'started'` at the moment you call them.
- Run `/showpos` in game: if you see one notification per configured type, the resource itself is fine and the problem is in the calling resource.
- If a type renders wrong, check that its `Config.Styles` entry has all four fields and that the `iconFile` exists in `html/icons/`.
- Review this FAQ page.

## 📋 Changelog

**Current version: 3.1.1** (`fxmanifest.lua`)

- `3.1.1` — current release. Full/minimalist dual style with a per-player toggle, config-driven notification types (`Config.Styles`), per-type sounds, persistent and button notifications, client and server export pairs, player-saved position and scale via KVP, and four optional debug commands.

No earlier version history is recorded in the resource.
