# Nexus Boombox

> A portable, community-style boombox — drop it, stream a YouTube link, and everyone nearby hears it with realistic spatial audio.

**Version:** 1.0.1 · **Framework(s):** Standalone (optional ox_inventory / qb-core / es_extended) · **Database:** None

---

## 📝 Description

Portable, community-style boombox for FiveM. Any player can drop a boombox in the world, paste a YouTube link, and everyone nearby hears it with realistic spatial audio. Whoever places it spends the item; anyone can use it; whoever picks it up gets the item back. Built for roleplay servers where music is a shared favor, not a private possession.

---

## ✨ Features

- **YouTube streaming** – paste any YouTube link (watch, youtu.be or embed) and play it straight from a retro boombox UI.
- **Spatial audio** – volume falls off logarithmically with distance. Full volume inside `Config.FullVolumeDistance`, silent beyond `Config.MaxDistance`.
- **Community boombox design** – no ownership locks. Anyone in range can play, pause, change volume, loop or stop. Only interaction distance is enforced.
- **Place / pick up in the world** – prop spawned for every player, with a pick-up animation, an interaction hint and a server-side distance check on pickup.
- **Synced for everyone** – the progress position is computed from a shared real-time clock, so late joiners and players walking into range start at the right second.
- **Configurable volume ceiling** – `Config.MaxVolume` is enforced on the server and in the UI.
- **Inventory aware** – `none`, `ox_inventory`, `qb-core`, `es_extended` or `custom`.
- **Configurable notifications and hints** – native GTA feed / 3D text, `nexus_notify`, `nexus_TextUI` or your own system.
- **Anti-duplication** – the item is only handed back after a valid, in-range pickup of a boombox that really existed.
- **7 languages** – en, es, de, fr, it, pt, zh. All strings are editable and outside the escrow.
- **Open configuration** – `config.lua`, `functions.lua`, locales and the whole NUI are not encrypted.

---

## 📋 Dependencies

| Resource | Required | Notes |
|----------|----------|-------|
| OneSync | Yes | The server reads player positions to validate pickups. |
| ox_inventory / qb-core / es_extended | Optional | Only if `Config.Inventory` is not `none`. |
| nexus_notify | Optional | Only if `Config.NotifySystem = 'nexus_notify'`. |
| nexus_TextUI | Optional | Only if `Config.TextUISystem = 'nexus_TextUI'`. |

No database is used. Boomboxes are not persisted across restarts.

---

## ⚙️ Installation

1. Drop the `nexus_boombox` folder into your `resources` directory. **The folder must keep the name `nexus_boombox`.**
2. Add it to your `server.cfg`:

   ```
   ensure nexus_boombox
   ```

3. Open `shared/config.lua` and set `Config.Inventory` to match your server.
4. If you use an inventory, create the item (name from `Config.BoomboxItem`, default `boombox`) and make its "use" action run the command `useboombox` (see the Integration Example below).
5. Restart the server.

### Controls

| Action | Default |
|--------|---------|
| Place a boombox | `/useboombox` command (bind it to your item use) |
| Open the boombox UI | `E` near a boombox |
| Pick up the boombox | `G` near a boombox |
| Close the UI | `ESC` |

---

## 🔧 Configuration

All options live in `shared/config.lua`.

| Option | Default | Description |
|--------|---------|-------------|
| `Config.Locale` | `'en'` | Language file from `locales/`. |
| `Config.Debug` | `false` | Skips item checks. Keep `false` in production. |
| `Config.BoomboxItem` | `'boombox'` | Item required to place the boombox. |
| `Config.Inventory` | `'none'` | `none`, `ox`, `qb`, `esx` or `custom`. `none` does not require any item. |
| `Config.NotifySystem` | `'native'` | `native`, `nexus_notify` or `custom`. |
| `Config.TextUISystem` | `'native'` | `native`, `nexus_TextUI` or `custom`. |
| `Config.BoomboxProp` | `'prop_ghettoblast_02'` | World prop model. |
| `Config.InteractDistance` | `2.5` | Distance at which the hint appears and keys work. |
| `Config.FullVolumeDistance` | `3.0` | Listeners inside this radius hear full volume. |
| `Config.MaxDistance` | `25.0` | Beyond this distance the boombox is silent. |
| `Config.UpdateInterval` | `300` | Spatial audio refresh in ms. |
| `Config.MaxVolume` | `1.0` | Highest volume allowed (0.0 – 1.0), enforced on server and UI. |
| `Config.DefaultVolume` | `0.7` | Volume used on first play. |
| `Config.InteractKeyLabel` | `'E'` | Label shown in the hint (display only). |
| `Config.PickupKeyLabel` | `'G'` | Label shown in the hint (display only). |

---

## 🌐 Locales & Editable Strings

Language files are in `locales/` (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`). Change `Config.Locale` to switch language. To add a language, copy `locales/en.lua`, translate the values and set `Config.Locale` to the new file name (also list the file under `files` in `fxmanifest.lua`).

| Key | Default (en) |
|-----|--------------|
| `no_item` | You do not have a boombox |
| `already_placed` | You already have a boombox placed. Pick it up first. |
| `picked_up` | Boombox picked up |
| `interact_hint` | [%s] Use Boombox   [%s] Pick up |

The UI texts live in `html/index.html` and `html/js/app.js`. They are not encrypted and can be edited freely.

---

## 🔗 Compatibility

- **Game:** FiveM (gta5), `cerulean` manifest.
- **Server:** OneSync required.
- **Frameworks:** standalone. Inventory support for ox_inventory, qb-core and es_extended, plus a `custom` hook.
- **Escrow:** `config`, `client/functions.lua`, `server/functions.lua`, locales and the whole `html/` folder are open. Core client/server logic is encrypted.

---

## 💻 Developer API

### Exports

This resource does not expose exports.

### Events — Emitted

Client → Server:

| Event | Arguments | Description |
|-------|-----------|-------------|
| `nexus_boombox:place` | `x, y, z, heading` | Registers a new boombox. |
| `nexus_boombox:pickup` | `boomboxId` | Removes a boombox. Validated by distance. |
| `nexus_boombox:playSong` | `boomboxId, url, title, volume` | Starts a track. Volume is clamped to `Config.MaxVolume`. |
| `nexus_boombox:setPaused` | `boomboxId, paused, currentTime` | Pause or resume. |
| `nexus_boombox:setVolume` | `boomboxId, volume` | Changes volume. Clamped to `Config.MaxVolume`. |
| `nexus_boombox:stop` | `boomboxId` | Stops playback. |
| `nexus_boombox:requestAll` | – | Requests the full list of boomboxes. |
| `nexus_boombox:server:item` | `action ('add'|'remove'), item` | Inventory hook. `add` only works after a valid pickup. |

Server → Client:

| Event | Arguments | Description |
|-------|-----------|-------------|
| `nexus_boombox:placed` | `id, propNetId, x, y, z, heading, ownerServerId` | A boombox was placed. |
| `nexus_boombox:sync` | `boomboxId, radioData` | Playback state changed. |
| `nexus_boombox:stopped` | `boomboxId` | Playback stopped. |
| `nexus_boombox:pickedUp` | `boomboxId` | A boombox was removed. |
| `nexus_boombox:fullSync` | `boomboxes` | Full state for a joining client. |

`radioData` fields: `url`, `title`, `playing`, `startedAt` (milliseconds, `os.time() * 1000`), `currentTime`, `volume`, `ownerId`, `looped`.

### Events — Listened

Both the client and server listen for their own counterpart of the events above (e.g. the server listens for `nexus_boombox:place`, the client listens for `nexus_boombox:placed`) — there are no additional listened-only events outside this pair.

### Editable Functions

Editable in `client/functions.lua` and `server/functions.lua`.

| Function | Side | Description |
|----------|------|-------------|
| `T(key, ...)` | Client | Returns a translated string. |
| `HasBoomboxItem()` | Client | Item check by `Config.Inventory`. Edit the `custom` branch for your inventory. |
| `RemoveBoomboxItem()` / `AddBoomboxItem()` | Client | Ask the server to take or return the item. |
| `ShowNotification(message)` | Client | Routes through `Config.NotifySystem`. |
| `DrawInteractText(coords, text)` / `HideInteractText()` | Client | Routes through `Config.TextUISystem`. |
| `give(src, item)` / `take(src, item)` | Server | Inventory add and remove. Edit the `custom` branch. |
| `GrantItemReturn(src)` | Server | Registers one valid item return for a player. Called after a validated pickup. |

### Database Schema

None. No SQL is required.

### Integration Example

Make an inventory item place the boombox by running the command from its use callback.

```lua
-- ox_inventory: data/items.lua
['boombox'] = {
    label = 'Boombox',
    weight = 2500,
    stack = false,
    close = true,
    client = { event = 'my_resource:useBoombox' }
},
```

```lua
-- your own client script
RegisterNetEvent('my_resource:useBoombox', function()
    ExecuteCommand('useboombox')
end)
```

Custom notify or TextUI: set `Config.NotifySystem = 'custom'` and write your call in the `custom` branch of `ShowNotification` in `client/functions.lua`.

---

## ❓ FAQ

**Can only the person who placed the boombox use it?**
No. It is a community boombox by design. Anyone within range can control it.

**Who gets the item back?**
The player who picks it up (the one who presses `G` and passes the distance check).

**Does it need a database?**
No.

**Are boomboxes saved after a restart?**
No. They are removed on restart and when their owner disconnects.

**Why does the progress bar match for everyone?**
Playback start time is stored as real time (`os.time() * 1000`) and every client calculates the position against the same clock.

**A video does not play.**
Some YouTube videos block embedding. Try another link.

**The item is not returned.**
Make sure `Config.Inventory` matches your inventory and the item name matches `Config.BoomboxItem`. The item is only returned after a valid pickup in range.

---

## 📋 Changelog

### 1.0.1
- Fixed the progress clock: the server now sends `os.time() * 1000` so the UI and the server use the same clock.
- Pickup is now validated by player distance.
- Fixed an item duplication exploit: `nexus_boombox:server:item('add')` now requires a valid pickup.
- Added `Config.NotifySystem` and `Config.TextUISystem`.
- `Config.MaxVolume` is now enforced on server and UI.
- Removed unguarded server `print()` calls.
- UI strings translated to English and `html/` moved out of the escrow.

### 1.0.0
- Initial release.
