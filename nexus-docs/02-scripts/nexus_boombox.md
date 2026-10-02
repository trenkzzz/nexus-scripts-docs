# Nexus Boombox

A placeable, fully synchronised street boombox that streams any YouTube link with realistic logarithmic spatial audio — every player nearby hears the same song, at the same second, at a volume that depends on how far away they are standing.

## 📝 Description

Nexus Boombox turns a single inventory item into a shared, world-placed sound system. A player runs the open command (or uses the item), an animation plays while the ped crouches down, and a `prop_ghettoblast_02` prop is dropped on the ground in front of them. Walking up to that prop shows a floating interaction hint, and pressing the interact key opens a fully custom NUI: a photoreal 80s boombox with twin speakers, a DS-Digital LCD screen, a scrolling song-title marquee, a power button with a CRT-style scan-on animation, transport controls and a loop toggle.

Internally the resource is a **state-synchronisation layer, not an audio streamer**. The server never touches audio. It keeps an in-memory registry of every active boombox (`activeBoomboxes[boomboxId] = { x, y, z, heading, ownerServerId, radio }`) and broadcasts that state to every client. Each client then runs its own hidden YouTube IFrame player per boombox (`YTManager`, one 1×1 px iframe per `boomboxId` inside `#ytContainer`) and a loop that fires every `Config.UpdateInterval` milliseconds. That loop measures the distance from the local player to each boombox prop and sets the volume of that boombox's local YouTube player accordingly, so the same song is playing in every listener's browser at an independently-computed volume. The attenuation curve is logarithmic rather than linear — `1.0 - (log(1 + t*9) / log(10))` where `t` is the normalised distance between `Config.FullVolumeDistance` and `Config.MaxDistance` — which matches how human hearing perceives loudness far better than a straight falloff.

The prop itself is **client-side only and deliberately so**. The server stores the coordinates and heading, and every client spawns its own local, frozen, non-networked object with `CreateObjectNoOffset` + `PlaceObjectOnGroundProperly`. There is no entity ownership to fight over, no net ID to desync and nothing left behind on a restart. When a boombox is picked up, the server broadcasts the removal and every client deletes its own copy. A late-joining player triggers `nexus_boombox:requestAll` one second after the resource starts on their machine and receives the full world state in a single `fullSync` event, so boomboxes placed before they connected appear with the correct song already playing.

The problem it solves is the classic "party scene" gap: servers either have no portable music at all, or they have a radio that only the person who started it can hear. This gives a whole group a single shared audio source anchored to a physical object in the world, with no third-party audio resource, no streaming server and no framework requirement — it is fully standalone and only touches a framework if you ask it to, through the inventory branch.

## ✨ Features

- **World-placed prop with animation** — `prop_ghettoblast_02` (configurable) spawned 0.9 units in front of the player, frozen, collision enabled and snapped to the ground. The `pickup_object` / `pickup_low` native animation plays for 1400 ms on both place and pickup, so the action reads correctly to other players.
- **Custom NUI boombox (not a generic menu)** — a single screen containing: two speaker assemblies with cone/ring/cap/mesh layers and a glow overlay that activates only while audio is playing; an LCD panel with an ON state and an OFF state; a `VOL nn%` badge; an idle placeholder line; a dual-copy marquee track for long song titles; a YouTube URL input; a power button with halo; and a four-button transport row (volume down, play/pause, loop, volume up).
- **Power state with scan-on animation** — the LCD starts off. Pressing power triggers a one-shot CRT scan overlay (`screenScanOn.classList.add('scanning')`) and fades the screen content in after 180 ms. Reopening a boombox that is already playing restores the ON state directly, without replaying the animation.
- **Scrolling title marquee with computed timing** — the title is rendered twice in one track; the offset is calculated from the real measured span width plus an 80 px gap, and the animation duration is derived from the text length (`clamp(spanWidth / 30, 8s, 22s)`), so short titles do not crawl and long titles do not blur past. The marquee only runs while the song is actually playing.
- **Logarithmic spatial audio** — full volume inside `Config.FullVolumeDistance`, silence beyond `Config.MaxDistance`, log-curve falloff in between, recomputed every `Config.UpdateInterval` ms per boombox per client.
- **Multi-boombox support** — `YTManager` keys every player, ready flag, pending-load entry and end-of-song callback by `boomboxId`, so several boomboxes can play different songs simultaneously and a player standing between two of them hears both, each at its own distance-based volume.
- **Real YouTube title resolution** — the NUI immediately shows the pasted URL as a provisional title for instant feedback, then fetches the real title from `noembed.com` in parallel and replaces it when it arrives, falling back to `UNKNOWN TRACK` if the lookup fails. Playback is never blocked on the fetch.
- **Loop toggle** — client-side loop flag. On the YouTube `ENDED` state (`e.data === 0`) the end callback either re-sends `playSong` with the same URL to restart the track server-side for everyone, or stops the boombox and clears the UI.
- **Pause / resume with time preservation** — pausing sends the current playback position to the server, which stores `currentTime` and back-dates `startedAt`, so a listener who arrives during a pause and a listener who was there all along resume from the same position.
- **Shared volume** — volume is server state, not per-listener preference. Any player at the boombox can change it in 10 % steps and the new base volume is broadcast to everyone, then scaled locally by distance.
- **Late-join full sync** — `requestAll` → `fullSync` rebuilds every prop and its radio state for a player who just connected or just restarted the resource.
- **Owner-disconnect cleanup** — `playerDropped` removes every boombox whose `ownerServerId` matches the leaving player and broadcasts the pickup, so nothing is orphaned in the world.
- **Resource-stop cleanup** — `onClientResourceStop` deletes every local prop, so a `/restart nexus_boombox` leaves no floating objects behind.
- **Pluggable inventory** — `Config.Inventory` selects `none` / `ox` / `qb` / `esx` / `custom`. The check, the removal and the return of the item each live in an unencrypted branch you can edit.
- **Seven languages** — all Lua-side player-facing strings in `locales/*.lua`, loaded at runtime with `LoadResourceFile`.
- **Resource-name validation** — `shared/_resource.lua` hard-errors on load if the folder is not named exactly `nexus_boombox`.
- **Debug bypass** — `Config.Debug = true` makes `HasBoomboxItem()` always return true and skips item add/remove, for testing without touching an inventory.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| Framework (ESX / QBCore) | **Optional** | Only used by the inventory branch. With `Config.Inventory = 'none'` the resource is fully standalone. |
| `ox_inventory` | Optional | Only when `Config.Inventory = 'ox'`. |
| `qb-core` | Optional | Only when `Config.Inventory = 'qb'`. |
| `es_extended` | Optional | Only when `Config.Inventory = 'esx'`. |
| Database | **Not used** | No SQL file, no tables, no queries. All state is in server memory and is intentionally lost on restart. |
| Internet access from the game client | **Required** | The NUI loads the YouTube IFrame API, Google Fonts, `fonts.cdnfonts.com` (DS-Digital) and `noembed.com`. |
| An inventory item named per `Config.BoomboxItem` | Conditional | Required for every `Config.Inventory` value except `none` and `custom`. |

There is no `dependencies {}` block in `fxmanifest.lua`, so the resource will start regardless of load order. The inventory exports are resolved lazily, at the moment a player tries to place a boombox, not at resource start.

## ⚙️ Installation

1. **Unzip the resource.** Download it from the cfx.re portal (Keymaster) and extract the folder into your `resources/` directory. The folder **must** be named exactly `nexus_boombox` — `shared/_resource.lua` raises a hard error and refuses to start otherwise.
2. **No SQL import.** This resource has no database component. Skip this step.
3. **Add it to `server.cfg`.** There is no load-order requirement, but if you use an inventory integration, start that inventory first:
   ```cfg
   ensure ox_inventory      # only if Config.Inventory = 'ox'
   ensure nexus_boombox
   ```
4. **Create the item** (only if `Config.Inventory` is not `none`/`custom`). Register an item whose name matches `Config.BoomboxItem` (default `boombox`) in your inventory's item list, and give it an image.
5. **Configure.** Open `shared/config.lua` and set `Config.Locale`, `Config.Inventory`, `Config.BoomboxItem`, and your audio distances.
6. **Restart.** A full server restart is recommended over `refresh; ensure`.
7. **Start using it.** Run `/useboombox` to place it. Walk within `Config.InteractDistance` (default 2.5) of the prop: a hint appears above it. Press **E** to open the boombox UI, or **G** to pick it up. Inside the UI, press the power button, paste a YouTube link, and press play.

## 🔧 Configuration

Every key in `shared/config.lua`, with no exceptions:

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Locale` | string | `'en'` | Language file loaded from `locales/<locale>.lua` at resource start. Shipped values: `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`. If the file is missing, `T()` falls back to returning the raw key and a warning is printed. |
| `Config.Debug` | boolean | `false` | Development bypass. When `true`, `HasBoomboxItem()` always returns `true` and `RemoveBoomboxItem()` / `AddBoomboxItem()` become no-ops, so you can test placement without an inventory or an item. Leave `false` in production. |
| `Config.BoomboxItem` | string | `'boombox'` | Inventory item name checked before placing, removed on place and returned on pickup. Ignored when `Config.Inventory = 'none'`. The server-side item handler also validates the incoming item name against this value before touching the inventory. |
| `Config.Inventory` | string | `'none'` | Which inventory to use: `'none'` (no item required at all), `'ox'` (`ox_inventory`), `'qb'` (`qb-core`), `'esx'` (`es_extended`), `'custom'` (write your own check in `client/functions.lua` and your own add/remove in `server/functions.lua`). Note that `'custom'` also skips the add/remove server round-trip entirely, so you implement both halves yourself. |
| `Config.BoomboxProp` | string | `'prop_ghettoblast_02'` | Model name of the world object. Any static prop model works; the model is requested with a 5-second timeout and placement is aborted if it never loads. |
| `Config.InteractDistance` | number | `2.5` | Radius (game units) in which the floating interaction hint is drawn and the E / G keys become active. Also used for the auto-close check: the UI closes when you walk further than `InteractDistance + 1.5` from the boombox you have open. |
| `Config.FullVolumeDistance` | number | `3.0` | Inside this radius listeners hear the boombox at 100 % of its configured base volume, with no attenuation. |
| `Config.MaxDistance` | number | `25.0` | Beyond this radius the computed volume is exactly `0.0`. Must be greater than `FullVolumeDistance` or the falloff maths divides by zero. |
| `Config.UpdateInterval` | number (ms) | `300` | How often the client spatial-audio loop recalculates distance and pushes a new volume to each local YouTube player. Lower values track fast-moving players more smoothly at the cost of more NUI messages. |
| `Config.MaxVolume` | number (0.0–1.0) | `1.0` | **Currently not read anywhere in the code.** The NUI clamps volume to `1.0` and the server clamps to `math.max(0.0, math.min(1.0, volume))`, neither of which consults this key. See the incident report. |
| `Config.DefaultVolume` | number (0.0–1.0) | `0.7` | Base volume used by the server when a `playSong` event arrives without a volume, and by the client falloff calculation when a radio entry has no volume set. The NUI also independently defaults its own slider state to `0.7`. |
| `Config.InteractKeyLabel` | string | `'E'` | **Display only.** Substituted into the `interact_hint` locale string as the first `%s`. The actual key is hardcoded to control `38` (E) in the protected client loop; changing this label does not rebind anything. |
| `Config.PickupKeyLabel` | string | `'G'` | **Display only.** Substituted into `interact_hint` as the second `%s`. The actual key is hardcoded to control `47` (G). |

### Spatial audio curve

The volume a listener hears is computed per-boombox, per-client, every `UpdateInterval` ms:

```lua
-- client/client.lua
if distance <= Config.FullVolumeDistance then
    return baseVolume                      -- inside the full-volume bubble
elseif distance >= Config.MaxDistance then
    return 0.0                             -- out of range
else
    local t      = (distance - Config.FullVolumeDistance)
                 / (Config.MaxDistance - Config.FullVolumeDistance)   -- 0..1
    local factor = 1.0 - (math.log(1.0 + t * 9.0) / math.log(10.0))
    return baseVolume * math.max(0.0, factor)
end
```

With the defaults (`3.0` / `25.0`), a listener at 5 m hears roughly 74 % of the base volume, at 10 m roughly 48 %, and at 20 m roughly 14 % — a fast initial drop followed by a long quiet tail, which is what real outdoor sound does.

## 🌐 Locales & Editable Strings

**Shipped languages: 7** — `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`, all listed in `fxmanifest.lua` under `files{}` (required, because they are read at runtime with `LoadResourceFile`, not loaded as scripts) and all covered by `escrow_ignore`.

All four keys exist in all seven files, with no gaps:

| Key | Format args | Used by |
|---|---|---|
| `no_item` | — | Shown when `HasBoomboxItem()` returns false. |
| `already_placed` | — | Shown when the player already has a boombox placed (`myBoombox` is set). |
| `picked_up` | — | Shown after a successful pickup. |
| `interact_hint` | `%s` = `Config.InteractKeyLabel`, `%s` = `Config.PickupKeyLabel` | The floating 3D text drawn above a nearby boombox. |

The loader lives in `client/functions.lua` (outside escrow) and is a plain `load()` of the locale file, so you can add a language simply by copying `en.lua`, translating the four values, and adding the filename to `files{}` in `fxmanifest.lua`.

### ⚠️ Strings that are NOT translatable

The NUI is **inside the escrow**. `fxmanifest.lua` declares `escrow_ignore` as `shared/config.lua`, `client/functions.lua`, `server/functions.lua` and `locales/*.lua` — `html/index.html`, `html/style.css`, `html/js/app.js` and `html/js/youtube.js` are **not** in that list, and every string drawn inside the boombox UI is hardcoded Spanish in those protected files:

| String | File | Where it appears |
|---|---|---|
| `ESPERANDO CANCION...` | `html/index.html` | LCD idle line |
| `PRESIONA POWER` | `html/index.html` | LCD off-state hint |
| `PEGA URL DE YOUTUBE...` | `html/index.html` | URL input placeholder |
| `VOL 70%` | `html/index.html` | Initial volume badge (overwritten by JS, same format) |
| `— — —` | `html/index.html` | LCD off-state dashes |
| `PEGA UNA URL DE YOUTUBE` | `html/js/app.js` | Toast on empty input |
| `URL INVALIDA` | `html/js/app.js` | Toast on failed URL validation |
| `UNKNOWN TRACK` | `html/js/app.js` | Title fallback when the noembed lookup fails |
| `VOL ` prefix | `html/js/app.js` | `updateVolUI()` |

This is logged in the incident report. There is no supported way for a customer to translate the boombox screen.

## 🔗 Compatibility

| System | How it is selected | Notes |
|---|---|---|
| **Framework** | Implicit, via `Config.Inventory` | Fully standalone with `Config.Inventory = 'none'`. ESX and QBCore are only reached through the inventory branch. |
| **Inventory** | `Config.Inventory` | `none` / `ox` / `qb` / `esx` / `custom`. `ox` uses `exports.ox_inventory:Search('count', item)` client-side and `AddItem` / `RemoveItem` server-side. `qb` uses `QBCore.Functions.HasItem` and `Player.Functions.AddItem` / `RemoveItem`. `esx` iterates `ESX.GetPlayerData().inventory` client-side and uses `addInventoryItem` / `removeInventoryItem` server-side. |
| **Notifications** | Not configurable | `ShowNotification()` in `client/functions.lua` uses the native GTA feed (`SetNotificationTextEntry` / `DrawNotification`). The file is outside escrow so you can replace the body with `nexus_notify`, ox_lib, okokNotify or anything else, but there is no `Config.NotifySystem` branch as in other Nexus scripts — see the incident report. |
| **TextUI / target** | Not used | The interaction hint is drawn directly with `SetDrawOrigin` + `DrawText` in `DrawInteractText()` (outside escrow, so replaceable with `nexus_TextUI`'s `Create(config)` if you want). No target resource is involved. |
| **Keys / fuel / banking** | Not used | No vehicle, no money, no key system involvement. |
| **OneSync** | Compatible | All broadcasts use `TriggerClientEvent(..., -1, ...)`. Props are local objects, so there is no entity-ownership or culling problem. |
| **Audio resources** | No conflict | Audio is played inside the resource's own CEF browser instance via hidden YouTube iframes. It does not touch GTA audio banks, `xsound`, `interact-sound` or any other audio resource. |

## 💻 Developer API

### Client Exports

**None.** `nexus_boombox` declares no `exports` on the client side. Integration is done through the events below.

### Server Exports

**None.** `nexus_boombox` declares no `exports` on the server side.

### Events — Emitted

Server → client, all broadcast to `-1` (every player) unless stated otherwise:

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_boombox:placed` | server → all clients | `boomboxId` (string), `propNetId` (always `nil`), `px`, `py`, `pz` (numbers), `heading` (number), `ownerServerId` (number) | A player placed a boombox. Every client spawns its own local prop and registers the entry; the client whose server ID matches `ownerServerId` also marks it as its own. |
| `nexus_boombox:sync` | server → all clients | `boomboxId` (string), `radioData` (table: `url`, `title`, `playing`, `startedAt`, `currentTime`, `volume`, `ownerId`, `looped`) | Any radio state change: new song, pause, resume, volume change. |
| `nexus_boombox:stopped` | server → all clients | `boomboxId` (string) | The song was stopped (power off, natural end without loop, or explicit stop). The boombox itself stays in the world. |
| `nexus_boombox:pickedUp` | server → all clients | `boomboxId` (string) | The boombox was removed from the world — either picked up by a player, or auto-removed because its owner disconnected. Every client deletes its local prop. |
| `nexus_boombox:fullSync` | server → **one** client | `allBoomboxes` (table keyed by `boomboxId`, each `{ x, y, z, heading, ownerServerId, propNetId, radio }`) | Reply to `requestAll`. Sent only to the requesting player. |

### Events — Listened

Client → server:

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `nexus_boombox:place` | client → server | `px`, `py`, `pz`, `heading` | Registers a new boombox, generates a sequential ID (`bb_1`, `bb_2`, …) and broadcasts `placed`. |
| `nexus_boombox:pickup` | client → server | `boomboxId` | Removes the boombox from the server registry and broadcasts `pickedUp`. |
| `nexus_boombox:playSong` | client → server | `boomboxId`, `url`, `title`, `volume` | Builds the `radio` table (`playing = true`, `startedAt = GetGameTimer()`, `currentTime = 0`) and broadcasts `sync`. |
| `nexus_boombox:setPaused` | client → server | `boomboxId`, `paused` (boolean), `currentTime` (seconds) | Sets `playing = not paused`, stores `currentTime`, back-dates `startedAt` to `GetGameTimer() - currentTime * 1000`, broadcasts `sync`. |
| `nexus_boombox:setVolume` | client → server | `boomboxId`, `volume` (0.0–1.0) | Clamps to 0.0–1.0 and broadcasts `sync`. |
| `nexus_boombox:stop` | client → server | `boomboxId` | Clears the `radio` table and broadcasts `stopped`. |
| `nexus_boombox:requestAll` | client → server | — | Asks for the full world state; answered with `fullSync` to that player only. |
| `nexus_boombox:server:item` | client → server | `action` (`'add'` \| `'remove'`), `item` (string) | Inventory bridge. Returns immediately if `Config.Inventory == 'none'` or if `item ~= Config.BoomboxItem`. Defined in `server/functions.lua`, outside escrow. |

Native events consumed: `playerDropped` (server, owner cleanup), `onClientResourceStart` / `onClientResourceStop` (client, late sync and prop cleanup).

### NUI callbacks (client ← NUI)

Registered in `client/client.lua`. Useful if you rebuild the UI:

| Callback | Body | Returns |
|---|---|---|
| `close` | — | `'ok'` |
| `playSong` | `{ url, title, volume }` | `'ok'`, or `'no_boombox'` if no boombox is in range |
| `setPaused` | `{ paused, currentTime }` | `'ok'` / `'no_boombox'` |
| `setVolume` | `{ volume }` | `'ok'` / `'no_boombox'` |
| `stop` | — | `'ok'` / `'no_boombox'` |

> ⚠️ `html/js/app.js` also posts a `setFocus` callback when the URL input gains focus, but **no `setFocus` callback is registered in `client/client.lua`**. The `fetch` fails silently (it is `.catch(() => {})`-ed), so there is no visible break, but the call does nothing. Logged in the incident report.

### NUI messages (client → NUI)

| `type` | Fields | Meaning |
|---|---|---|
| `open` | `boomboxId`, `isOwner`, `radio` | Open the UI, restoring the ON state if `radio.url` is set. |
| `close` | — | Close the UI. |
| `radioSync` | `radio` | Update title / playing / volume for the open boombox. |
| `radioStopped` | — | The open boombox's song was stopped externally. |
| `stopBoombox` | `boomboxId` | Destroy the local YouTube player for that boombox. |
| `updateSpatial` | `boomboxId`, `volume`, `paused`, `url`, `title`, `currentTime`, `startedAt` | The per-tick spatial update; also what triggers the initial load of a YouTube player for a boombox the listener has not heard yet. |
| `notification` | `message` | Show a toast (uppercased by the NUI). Not currently sent by the Lua side. |

### Editable Functions (`client/functions.lua`, `server/functions.lua`)

Both files are in `escrow_ignore` and are the supported integration surface.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `T` | `T(key, ...) → string` | Everywhere player-facing text is built (`client/client.lua`) | Locale lookup. Returns `Lang[key]`, `string.format`-ed with the varargs if any were passed, or the raw `key` if the entry is missing. Must always return a string. |
| `HasBoomboxItem` | `HasBoomboxItem() → boolean` | `placeBoombox()` in `client/client.lua`, before anything else happens | Must return `true` if the player may place a boombox. Returns `true` unconditionally when `Config.Debug` is on or `Config.Inventory` is `'none'`/`'custom'`. Write your own check in the `custom` branch. |
| `RemoveBoomboxItem` | `RemoveBoomboxItem()` | `placeBoombox()`, immediately after the place event is sent | Should consume one item. Fires `nexus_boombox:server:item` with `'remove'`. No return value used. |
| `AddBoomboxItem` | `AddBoomboxItem()` | `pickupBoombox()`, only when the picker is the owner | Should return one item. Fires `nexus_boombox:server:item` with `'add'`. No return value used. |
| `ShowNotification` | `ShowNotification(message)` | `placeBoombox()` (no item / already placed) and `pickupBoombox()` (picked up) | Display a short notification. Default implementation is the native GTA feed. Replace the body to use your own notify resource. |
| `DrawInteractText` | `DrawInteractText(coords, text)` | The interaction thread, every frame while a boombox is within `Config.InteractDistance` | Draw the hint. Called at 0 ms intervals, so keep it cheap — no `Wait`, no HTTP, no entity creation. `coords` is a `vector3`; the default draws at `z + 0.9`. |
| `PlayPlaceAnimation` | `PlayPlaceAnimation(ped)` | `placeBoombox()`, **before** the server event is sent | Plays `pickup_object` / `pickup_low` for 1400 ms and **blocks** (`Wait(ANIM_DURATION)`) before returning. Any replacement should also block for the duration you want the player held in place. |
| `PlayPickupAnimation` | `PlayPickupAnimation(ped)` | `pickupBoombox()`, before the server event | Currently delegates to `PlayPlaceAnimation`. Override independently if you want a different clip for pickup. |
| `give` *(local)* | `give(src, item)` | `nexus_boombox:server:item` with `'add'` | Server-side item grant, branched by `Config.Inventory`. `local` to `server/functions.lua`. |
| `take` *(local)* | `take(src, item)` | `nexus_boombox:server:item` with `'remove'` | Server-side item removal, branched by `Config.Inventory`. |

Globals also exposed by `client/functions.lua`: `Lang` (the locale table), and the module-local animation constants `ANIM_DICT` (`'pickup_object'`), `ANIM_CLIP` (`'pickup_low'`), `ANIM_DURATION` (`1400`).

### Database Schema

**None.** `nexus_boombox` creates no tables and runs no queries. Placed boomboxes live in the server's `activeBoomboxes` table and are gone after a server restart, by design.

### Integration Example

A logging + permissions resource that records every boombox placement and blocks songs from a banned-URL list. Nothing here touches protected code.

```lua
-- my_boombox_addon/server.lua

local banned = { 'dQw4w9WgXcQ' }

-- Log every placement. 'placed' is broadcast to -1, so the server can also
-- just listen to the inbound 'place' event instead; this hooks the outbound one
-- by registering for the same client event name is NOT possible server-side,
-- so we listen to the inbound request:
AddEventHandler('nexus_boombox:place', function(x, y, z, heading)
    local src = source
    print(('[boombox-log] %s placed a boombox at %.2f %.2f %.2f'):format(
        GetPlayerName(src), x, y, z))
end)

AddEventHandler('nexus_boombox:playSong', function(boomboxId, url, title)
    local src = source
    for _, id in ipairs(banned) do
        if url:find(id, 1, true) then
            TriggerClientEvent('chat:addMessage', src, { args = { 'Boombox', 'That track is blocked.' } })
            -- Stop it again right away
            TriggerEvent('nexus_boombox:stop', boomboxId)
            return
        end
    end
    print(('[boombox-log] %s -> %s (%s)'):format(GetPlayerName(src), title, boomboxId))
end)
```

```lua
-- Reading boombox state from the client side of another resource
-- (mirror the sync events into your own table)
local known = {}

RegisterNetEvent('nexus_boombox:sync', function(boomboxId, radio)
    known[boomboxId] = radio
    if radio and radio.playing then
        print(('[addon] %s is now playing %s'):format(boomboxId, radio.title or '?'))
    end
end)

RegisterNetEvent('nexus_boombox:pickedUp', function(boomboxId)
    known[boomboxId] = nil
end)
```

To swap in `nexus_notify` without touching protected code, edit `client/functions.lua`:

```lua
function ShowNotification(message)
    exports['nexus_notify']:Alert('Boombox', message, 5000, 'info', true)
end
```

## ❓ FAQ

**The boombox UI opens but no sound comes out.**
The NUI streams from YouTube inside the game's CEF browser. Check, in order: (1) the game client has internet access and YouTube is not blocked at network level; (2) the video is not age-restricted, region-blocked or embed-disabled — those fail inside an iframe even though they play on youtube.com; (3) `Config.MaxDistance` is larger than `Config.FullVolumeDistance`; (4) you actually pressed the power button — the play button returns early while the boombox is powered off.

**Other players can't hear my boombox.**
Volume is distance-based. Confirm they are within `Config.MaxDistance` (default 25 units) and that `Config.UpdateInterval` is not set to something enormous. Also note each listener needs their own working YouTube connection: the audio is not relayed by your server, every client fetches the stream independently.

**The interaction hint doesn't appear / E and G do nothing.**
The hint is only drawn within `Config.InteractDistance` (default 2.5 units), and only while no boombox UI is open. The keys are the hardcoded controls `38` (E) and `47` (G) — `Config.InteractKeyLabel` and `Config.PickupKeyLabel` only change the text in the hint, they do not rebind anything. If you need different keys you need a code change.

**I set `Config.Inventory = 'ox'` and placing throws an error.**
The inventory exports are only resolved at the moment you place, so a missing or not-yet-started inventory resource shows up as a runtime error rather than a startup warning. Make sure `ox_inventory` is started **before** `nexus_boombox` in `server.cfg` and that an item named `Config.BoomboxItem` exists in its item list.

**All my boomboxes disappeared after a restart.**
Expected. There is no database: the server keeps placed boomboxes in memory only. Restarting the server or the resource clears them (and `onClientResourceStop` deletes the props so nothing is left floating).

**A boombox vanished while someone was using it.**
If the player who placed it disconnects, `playerDropped` removes it for everyone. That is deliberate — it prevents orphaned props.

**Someone else picked up my boombox.**
The pickup check is positional, not ownership-based: any player standing within `Config.InteractDistance` can press G. The item is only returned to the owner; if a non-owner picks it up, the boombox is destroyed and nobody gets an item back. If you need this locked down, see the incident report — it needs a code change, it is not configurable.

**Can two boomboxes play different songs at once?**
Yes. Every boombox gets its own hidden YouTube player keyed by `boomboxId`, and each one's volume is computed independently from your distance to that specific prop.

**Can I translate the text on the boombox screen?**
Only the four Lua-side strings (`locales/*.lua`). The text inside the UI itself (`ESPERANDO CANCION...`, `PRESIONA POWER`, `PEGA URL DE YOUTUBE...`, the error toasts) is hardcoded in `html/`, which is inside the escrow and cannot be edited.

**Does it conflict with `xsound` / `interact-sound` / other audio resources?**
No. It never touches GTA audio banks or native audio; everything happens in its own browser instance.

### Before opening a ticket

- Make sure the resource folder name is exactly **`nexus_boombox`** — the resource refuses to start otherwise, and prints the reason to the server console.
- Make sure you are on the latest version of the resource (**1.0.0** at the time of writing).
- Re-read this FAQ page.
- Set `Config.Debug = true` and reproduce — it removes the inventory from the equation entirely.
- Have your `shared/config.lua`, your `server.cfg` boot order, and the full F8 **and** server-console output ready.

## 📋 Changelog

### 1.0.0 — Initial release
As declared in `fxmanifest.lua` (`version '1.0.0'`). No prior version history is shipped with the resource.
- Placeable boombox prop with place/pickup animation.
- Custom NUI boombox with LCD, speaker glow, marquee title, power state and transport controls.
- Logarithmic distance-based spatial audio, recomputed per client per `Config.UpdateInterval`.
- Multi-boombox support with independent YouTube players.
- Loop toggle, shared volume, pause with position preservation.
- Late-join full sync and owner-disconnect cleanup.
- Inventory abstraction for `none` / `ox` / `qb` / `esx` / `custom`.
- Seven locales (`es`, `en`, `de`, `fr`, `it`, `pt`, `zh`).
