# Nexus Car Radio

A standalone in-car infotainment system that streams YouTube to everyone around the vehicle with logarithmic spatial audio, plus multiple database-backed personal playlists, queue playback and a persistent mini HUD with its own keybinds.

## 📝 Description

Nexus Car Radio replaces the stock vehicle radio with a purpose-built infotainment panel. The driver presses **M** and a wide, dark-purple console opens: a branded top bar with a live clock, two tabs (**PLAYER** and **PLAYLIST**), a 36-bar radial visualiser with a rotating scanner ring and a 20-bar linear waveform, the track's YouTube thumbnail with a crossfade-and-zoom transition on every song change, a clickable and draggable progress bar with real timestamps, transport controls (restart/previous, play/pause, next/stop, loop) and a volume rail. Below that sits a search row: paste a YouTube link, press **CARGAR**, get a preview card with the real title, channel and thumbnail, then press **PLAY**.

Audio is handled entirely client-side. The server never streams anything — it keeps an in-memory map of `activeRadios[vehicleNetId] = { url, title, playing, startedAt, currentTime, volume, ownerId, looped }` and broadcasts it. Each client runs its own hidden YouTube IFrame player per vehicle network ID, and a loop ticking every `Config.UpdateInterval` milliseconds decides what volume *that* client should hear each vehicle at. If you are sitting inside the vehicle you get the full configured volume with no attenuation; if you are outside, the volume follows a logarithmic falloff between `Config.FullVolumeDistance` and `Config.MaxDistance`. Because the position is re-read from the live vehicle entity every tick, **the music follows the car** — a vehicle driving past you gets louder and then quieter in real time, and a vehicle that is deleted or destroyed has its radio torn down automatically on every client.

Playlists are the feature that separates this from a simple URL player. Each player gets up to `Config.MaxPlaylists` named playlists, each holding up to `Config.MaxPlaylistSize` songs, stored in two MySQL tables that the resource **creates by itself on first start** — there is no SQL file to import. Ownership is enforced on every single write: saving, removing, renaming and deleting all verify `citizenid` against the row before touching anything, duplicates within a playlist are rejected, and the server refuses to delete your last remaining playlist so you are never left with none. Playing a song from a playlist also sets a **queue context**, so when that track ends the next one in the list starts automatically, and the next/previous buttons walk the playlist instead of just stopping.

The mini HUD is the quality-of-life layer. Once something is playing, a compact strip stays on screen with the track title, playback state, a progress bar and four buttons labelled with their keys — **J** volume down, **K** play/pause, **L** volume up, **M** open the full panel. Those are real `RegisterKeyMapping` bindings, so they work without NUI focus (you can change the volume while driving) and players can rebind them in the GTA settings menu. The HUD hides itself the moment you step out of the vehicle.

It is fully **standalone**: no ESX, no QBCore, no inventory, no target system. The only hard dependency is `oxmysql`, for the playlists.

## ✨ Features

- **Driver-only infotainment panel** — opens with **M**, and only if you are in the driver's seat (`GetPedInVehicleSeat(veh, -1) == ped`); passengers get the `must_be_driver` notification. The panel auto-closes if you leave the vehicle or change vehicles.
- **Dual-mode interface** — a PLAYER tab (now-playing, visualiser, thumbnail, search, transport, volume) and a PLAYLIST tab (tab strip of your playlists with per-playlist song counts, and the song list).
- **36-bar radial visualiser + 20-bar waveform** — SVG lines generated at 10° intervals, driven by layered sine functions seeded per bar. It has three distinct states with real transitions: a gentle idle breathe when nothing is playing, a 700 ms eased **attack ramp** that interpolates from the idle snapshot into live values when playback starts, and a 900 ms eased **decay** back down when it stops. Not a looping GIF — the geometry is recomputed every frame.
- **Thumbnail with crossfade** — the YouTube `mqdefault` thumbnail, with a three-step fade-out → swap → fade-in-with-zoom transition whenever the track changes, and a clean teardown when the radio stops.
- **Clickable and draggable progress bar** — mousedown/mousemove/mouseup with `requestAnimationFrame` throttling; the displayed time follows the drag, and the seek is only committed to the server on release. Driver-only.
- **Real timestamps** — current position and duration read from the live YouTube player every 500 ms.
- **Logarithmic spatial audio** — full volume inside `Config.FullVolumeDistance`, silence past `Config.MaxDistance`, log falloff in between, recomputed every `Config.UpdateInterval` ms per vehicle per client.
- **Occupant override** — anyone physically inside the vehicle hears the configured volume at 100 %, bypassing the distance curve entirely, so the cabin always sounds right.
- **Multi-vehicle audio** — one hidden YouTube player per `vehicleNetId`; stand between two cars with different songs and you hear both, each at its own distance-based volume.
- **Automatic teardown** — if a vehicle no longer exists or is destroyed, the client drops its radio, destroys the player and clears ownership on the next tick. The server does the same for any radio owned by a disconnecting player.
- **Multiple named playlists** — create, rename and delete, up to `Config.MaxPlaylists`. A first playlist is created automatically on first use, named from the `default_playlist_name` locale key.
- **Queue playback** — playing a song from a playlist remembers the playlist and index; on natural song end the next track starts automatically. Next/previous walk the queue. The previous button restarts the current track if you are more than 3 seconds in, otherwise it goes back one — standard media-player behaviour.
- **Save to playlist with picker** — with a single playlist, **GUARDAR** saves immediately; with several, it switches to the PLAYLIST tab and arms a save-mode flag so the next playlist tab you click receives the song.
- **Server-side playlist guardrails** — ownership check on every write, duplicate-URL rejection, per-playlist song cap, per-player playlist cap, refusal to delete your only playlist, and name truncation to 64 characters / titles to 256.
- **Loop toggle** — synced through the server as part of radio state, so passengers see the same loop indicator. On song end with loop on, the client seeks to 0 and re-broadcasts the position.
- **Mini HUD with real keybinds** — `RegisterKeyMapping` for M / K / J / L, rebindable in GTA's own settings, working without NUI focus. The HUD shows title, state, progress and key hints, and hides itself when you are on foot.
- **Self-creating database schema** — `nexus_radio_playlists` and `nexus_radio_songs` are created with `CREATE TABLE IF NOT EXISTS` on resource start, with a cascading foreign key. No SQL import step.
- **Seven languages** — all Lua-side strings in `locales/*.lua`, loaded at runtime, used on both client and server.
- **Resource-name validation** — hard error on load if the folder is not named exactly `nexus_carradio`.
- **Late-join sync** — `requestAll` → `fullSync` one second after the resource starts on a client, so radios already playing are picked up.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **`oxmysql`** | **Required** | Declared in `fxmanifest.lua` both as `dependencies { 'oxmysql' }` and as `@oxmysql/lib/MySQL.lua`. Used for the playlist tables and the `MySQL.query.await` / `MySQL.insert.await` / `MySQL.scalar.await` API. Note this is the **oxmysql** API specifically, not the `MySQL.Async` wrapper. |
| Framework (ESX / QBCore) | **Not required** | Fully standalone. The resource identifies players by their `license:` identifier, not by a framework identifier. Optional hooks are commented into `shared/functions.lua` if you want framework identifiers instead. |
| SQL import | **Not needed** | The tables are created automatically at resource start. There is no `.sql` file in the package. |
| Internet access from the game client | **Required** | The NUI loads the YouTube IFrame API, Google Fonts (Bebas Neue, Outfit) and `noembed.com` for title/channel lookup. |
| Inventory / item | **Not required** | `CanUseRadio()` returns `true` unconditionally by default; commented examples for `ox_inventory` and QBCore are provided if you want to gate it behind an item. |
| Target / TextUI / keys / fuel / banking | **Not used** | — |
| OneSync | **Recommended** | Audio sync broadcasts use `TriggerClientEvent(-1, …)` and vehicles are resolved with `NetToVeh`, which needs network IDs to be valid across the session. |

**Load order:** `oxmysql` must start before `nexus_carradio`.

## ⚙️ Installation

1. **Unzip the resource.** Download from the cfx.re portal (Keymaster) and extract into `resources/`. The folder **must** be named exactly `nexus_carradio` — `shared/_resource.lua` raises a hard error otherwise.
2. **No SQL to import.** `nexus_radio_playlists` and `nexus_radio_songs` are created automatically the first time the resource starts. Just make sure the database user has `CREATE TABLE` permission on first boot.
3. **Set the `server.cfg` order.** `oxmysql` first:
   ```cfg
   ensure oxmysql
   ensure nexus_carradio
   ```
4. **Configure.** Open `shared/config.lua` and set `Config.Locale`, your audio distances (`Config.MaxDistance`, `Config.FullVolumeDistance`), and the playlist caps (`Config.MaxPlaylistSize`, `Config.MaxPlaylists`).
5. *(Optional)* **Gate it behind an item.** Edit `CanUseRadio()` in `shared/functions.lua` — commented `ox_inventory` and QBCore examples are already in the file.
6. *(Optional)* **Use framework identifiers.** Edit `GetCitizenId(src)` in `shared/functions.lua` — commented ESX and QBCore branches are already there. Do this **before** players build up playlists, or existing playlists will be orphaned under their old identifier.
7. **Restart.** A full server restart is recommended. Check the console for the two `CREATE TABLE` statements succeeding.
8. **Start using it.** Get into the driver's seat and press **M**. Paste a YouTube link in the search field, press **CARGAR**, then **PLAY**. While something is playing, **J** / **K** / **L** control volume and playback without opening the panel.

### Default keybinds

| Key | Command | Action |
|---|---|---|
| **M** | `/nexusradio` | Open / close the panel (driver only) |
| **K** | `/nexusradio_playpause` | Play / pause |
| **J** | `/nexusradio_voldown` | Volume −10 % |
| **L** | `/nexusradio_volup` | Volume +10 % |

All four are registered with `RegisterKeyMapping`, so each player can rebind them under **Settings → Key Bindings → FiveM**. There is no config key for them.

## 🔧 Configuration

Every key in `shared/config.lua`, with no exceptions:

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Locale` | string | `'en'` | Language file loaded from `locales/<locale>.lua` at resource start, on **both** client and server (`shared/functions.lua` is a shared script). Shipped: `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`. A missing file prints a warning and `T()` falls back to returning the raw key. |
| `Config.MaxDistance` | number | `30.0` | Maximum distance (game units) at which a vehicle's radio can be heard by someone outside it. Beyond this the computed volume is exactly `0.0`. |
| `Config.FullVolumeDistance` | number | `3.0` | Distance within which listeners outside the vehicle still hear it at 100 % of its configured volume. Must be smaller than `MaxDistance` or the falloff maths divides by zero. |
| `Config.SyncInterval` | number (ms) | `500` | **Currently not read anywhere in the code.** Documented in the config as the nearby-vehicle sync check interval, but the client only runs the `UpdateInterval` loop. See the incident report. |
| `Config.UpdateInterval` | number (ms) | `300` | How often the client recalculates each vehicle's distance and pushes a new volume to its local YouTube player. Lower values track fast-moving vehicles more smoothly, at the cost of more NUI messages per second. |
| `Config.MaxPlaylistSize` | number | `50` | Maximum songs per playlist. Enforced server-side on `saveSong`; the player gets the `playlist_full` notification with the limit substituted in. |
| `Config.MaxPlaylists` | number | `10` | Maximum playlists per player. Enforced server-side on `createPlaylist`; the player gets `max_playlists_reached` with the limit substituted in. |

### Spatial audio curve

```lua
-- client/client.lua
if insideThisVehicle then
    finalVolume = radio.volume            -- occupants always hear 100%
elseif distance <= Config.FullVolumeDistance then
    finalVolume = radio.volume
elseif distance >= Config.MaxDistance then
    finalVolume = 0.0
else
    local t      = (distance - Config.FullVolumeDistance)
                 / (Config.MaxDistance - Config.FullVolumeDistance)   -- 0..1
    local factor = 1.0 - (math.log(1.0 + t * 9.0) / math.log(10.0))
    finalVolume  = radio.volume * math.max(0.0, factor)
end
```

With the defaults (`3.0` / `30.0`), a pedestrian 5 m from the car hears about 78 % of the base volume, 10 m about 55 %, 20 m about 21 %, and nothing past 30 m. The NUI additionally pauses any player whose computed volume drops below `0.01`, so distant vehicles do not keep decoding audio needlessly.

## 🌐 Locales & Editable Strings

**Shipped languages: 7** — `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`, all declared in `fxmanifest.lua` under `files{}` (necessary, because they are read at runtime with `LoadResourceFile` rather than loaded as scripts) and all covered by `escrow_ignore`.

All **10 keys** are present in all seven files, with identical key sets and no gaps:

| Key | Format args | Side | Used when |
|---|---|---|---|
| `must_be_driver` | — | client | A non-driver pressed M. |
| `default_playlist_name` | — | server | Name given to the playlist auto-created on a player's first use. |
| `new_playlist_name` | — | server | Fallback name if `createPlaylist` arrives with no name. |
| `max_playlists_reached` | `%s` = `Config.MaxPlaylists` | server | Playlist cap hit. |
| `cannot_delete_last` | — | server | Tried to delete the only remaining playlist. |
| `playlist_deleted` | — | server | Playlist deleted successfully. |
| `playlist_not_found` | — | server | Tried to save into a playlist you do not own (or that does not exist). |
| `song_already_in_playlist` | — | server | Duplicate URL in the same playlist. |
| `playlist_full` | `%s` = `Config.MaxPlaylistSize` | server | Song cap hit. |
| `song_saved` | `%s` = song title, truncated to 40 chars | server | Song saved. |

Editable files (`escrow_ignore`): `shared/config.lua`, `shared/functions.lua`, `locales/*.lua`.

### ⚠️ Strings that are NOT translatable

`client/client.lua`, `server/server.lua` and the whole `html/` folder are inside the escrow, and all contain hardcoded player-facing text:

| Location | Strings |
|---|---|
| `client/client.lua` | The four keybind descriptions shown in GTA's own Key Bindings menu: `'Nexus Car Radio — Abrir/Cerrar'`, `'Nexus Car Radio — Play/Pausa'`, `'Nexus Car Radio — Bajar volumen'`, `'Nexus Car Radio — Subir volumen'`; plus `'Mi Playlist'` in the legacy `playlistUpdate` compatibility handler |
| `html/index.html` | `Sin reproducción`, `STANDBY`, `NUEVA PLAYLIST`, `Nombre de la playlist...`, `CANCELAR`, `CONFIRMAR`, `ELIMINAR PLAYLIST`, `¿Seguro que quieres eliminar …? Esta acción no se puede deshacer.`, `HORA LOCAL`, `REPRODUCIENDO AHORA`, `SIN SEÑAL`, `GUARDAR`, `Pega un link de YouTube...`, `CARGAR`, `▶ PLAY`, and the `title=` tooltips (`Bajar volumen (J)`, `Subir volumen (L)`, `Abrir panel (M)`, `Repetir canción`, `Guardar en playlist`, `Nueva playlist`, `Renombrar`, `Eliminar playlist`) |
| `html/js/app.js` | `REPRODUCIENDO`, `PAUSADO`, `STANDBY`, `SIN SEÑAL`, `Sin reproducción`, `BUCLE ACTIVADO`, `BUCLE DESACTIVADO`, `ELIGE LA PLAYLIST DONDE GUARDAR`, `INTRODUCE UN LINK`, `LINK INVÁLIDO`, `SOLO EL CONDUCTOR`, `NUEVA PLAYLIST`, `RENOMBRAR PLAYLIST`, `PLAYLIST VACÍA`, `Todavía no has guardado ninguna canción en …`, `Usa el botón GUARDAR en el Player`, `CARGAR`, `VOL: nn%` |

Logged in the incident report.

## 🔗 Compatibility

| System | How it is selected | Notes |
|---|---|---|
| **Framework** | None needed | Fully standalone. Players are identified by their `license:` identifier via `GetCitizenId(src)`. ESX and QBCore branches are pre-written as comments in `shared/functions.lua` if you prefer framework identifiers. |
| **Database** | Fixed — `oxmysql` | Uses the oxmysql API (`MySQL.query.await`, `MySQL.insert.await`, `MySQL.scalar.await`) and declares `@oxmysql/lib/MySQL.lua`. `mysql-async` is **not** a drop-in substitute here — unlike `nexus_bounty`, this resource needs the oxmysql-style API. |
| **Notifications** | `ShowNotification()` in `shared/functions.lua` | Default is the native GTA feed. ox_lib, ESX and QBCore examples are pre-written as comments. ⚠️ The in-game notifications a player actually sees from this resource go through the **NUI toast** (`nexus_carradio:notification` → `showToast`), not through `ShowNotification()` — which, in the shipped code, is never called. Replace the toast only if you are rebuilding the UI. There is no `Config.NotifySystem` branch in this resource. |
| **Inventory** | `CanUseRadio()` in `shared/functions.lua` | Returns `true` by default. `ox_inventory` and QBCore item checks are pre-written as comments. The check runs client-side, in the open command, before the panel opens. |
| **Keys** | `RegisterKeyMapping` | M / K / J / L, rebindable per player in GTA settings. Not configurable from `config.lua`. |
| **TextUI / target / fuel / banking** | **Not used** | — |
| **OneSync** | Recommended | Vehicles are addressed by network ID and resolved with `NetToVeh` on each client. |
| **Other audio resources** | No conflict | All audio lives in hidden YouTube iframes inside the resource's own CEF instance. GTA audio banks, `xsound` and `interact-sound` are untouched. The stock GTA vehicle radio is also untouched — mute it yourself if you do not want both. |

## 💻 Developer API

### Client Exports

**None.** `nexus_carradio` declares no `exports` on the client.

### Server Exports

**None.** `nexus_carradio` declares no `exports` on the server. Integration goes through the events below and the editable functions in `shared/functions.lua`.

### Events — Emitted

Server → client:

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_carradio:sync` | server → all clients (`-1`) | `vehicleNetId` (number), `radioData` (table: `url`, `title`, `playing`, `startedAt`, `currentTime`, `volume`, `ownerId`, `looped`) | Any radio state change on any vehicle: new song, pause/resume, seek, volume, loop toggle. |
| `nexus_carradio:stopped` | server → all clients (`-1`) | `vehicleNetId` (number) | The radio was stopped by its owner, or its owner disconnected. |
| `nexus_carradio:fullSync` | server → **one** client | `radios` (table keyed by `vehicleNetId`) | Reply to `requestAll`; the full world radio state for a client that just started. |
| `nexus_carradio:playlistsUpdate` | server → **one** client | `playlists` = array of `{ id, name, songs = { { url, title }, … } }` | After every playlist read or write. This is the single source of truth the NUI renders from. |
| `nexus_carradio:notification` | server → **one** client | `msg` (string, already localised server-side via `T()`) | Playlist errors and confirmations. Rendered as an uppercased NUI toast. |

### Events — Listened

Client → server. Note the ownership model: `playSong` **registers** the caller as `ownerId`, and every subsequent control event is rejected unless `radio.ownerId == source`.

| Event | Payload | Purpose |
|---|---|---|
| `nexus_carradio:playSong` | `vehicleNetId`, `url`, `title`, `volume`, `looped` | Creates/replaces the radio entry for that vehicle, sets `playing = true`, `startedAt = GetGameTimer()`, `currentTime = 0`, `ownerId = source`, then broadcasts `sync`. ⚠️ **No ownership check** — see the incident report. |
| `nexus_carradio:setPaused` | `vehicleNetId`, `paused` (boolean), `currentTime` (seconds) | Owner only. Sets `playing = not paused`, stores `currentTime`, back-dates `startedAt`, broadcasts `sync`. |
| `nexus_carradio:seek` | `vehicleNetId`, `currentTime` (seconds) | Owner only. Stores the new position, back-dates `startedAt`, broadcasts `sync`. |
| `nexus_carradio:setVolume` | `vehicleNetId`, `volume` (0.0–1.0) | Owner only. Stores the base volume and broadcasts `sync`. Not clamped server-side. |
| `nexus_carradio:setLoop` | `vehicleNetId`, `looped` (boolean) | Owner only. Stores the flag and broadcasts `sync`. |
| `nexus_carradio:stop` | `vehicleNetId` | Owner only. Removes the radio entry and broadcasts `stopped`. |
| `nexus_carradio:requestAll` | — | Answered with `fullSync` to that player only. |
| `nexus_carradio:requestPlaylists` | — | Reads (and lazily creates) the caller's playlists, answers with `playlistsUpdate`. |
| `nexus_carradio:requestPlaylist` | — | Legacy alias; identical behaviour to `requestPlaylists`. |
| `nexus_carradio:createPlaylist` | `name` (string) | Checks `Config.MaxPlaylists`, truncates the name to 64 chars, inserts, re-sends playlists. |
| `nexus_carradio:renamePlaylist` | `playlistId`, `name` | `UPDATE … WHERE id = ? AND citizenid = ?` — ownership enforced in the query itself. Truncates to 64 chars. |
| `nexus_carradio:deletePlaylist` | `playlistId` | Refuses if it is the caller's only playlist (`cannot_delete_last`), otherwise deletes it (songs cascade) and re-sends playlists. |
| `nexus_carradio:saveSong` | `playlistId`, `url`, `title` | Verifies ownership → rejects duplicate URL in that playlist → checks `Config.MaxPlaylistSize` → inserts at `MAX(position) + 1` → re-sends playlists and notifies. |
| `nexus_carradio:removeSong` | `playlistId`, `index` (**1-based**) | Verifies ownership, reads the playlist's songs ordered by `position`, deletes the one at that index. The NUI sends `arrayIndex + 1`. |

Native events consumed: `playerDropped` (server, stops any radio owned by the leaving player), `onClientResourceStart` (client, late sync).

### NUI callbacks (client ← NUI)

| Callback | Body | Returns |
|---|---|---|
| `setFocus` | — | `'ok'`. Re-asserts `SetNuiFocus(true, true)` — used when the panel is re-opened from the mini HUD. |
| `close` | — | `'ok'` |
| `playSong` | `{ url, title, volume, looped }` | `'ok'` / `'no_vehicle'` |
| `setPaused` | `{ paused, currentTime }` | `'ok'` / `'no_vehicle'` |
| `seek` | `{ currentTime }` | `'ok'` / `'no_vehicle'` |
| `setVolume` | `{ volume }` | `'ok'` / `'no_vehicle'` |
| `setLoop` | `{ looped }` | `'ok'` / `'no_vehicle'` |
| `stop` | — | `'ok'` / `'no_vehicle'` |
| `saveSong` | `{ playlistId, url, title }` | `'ok'` |
| `removeSong` | `{ playlistId, index }` | `'ok'` |
| `createPlaylist` | `{ name }` | `'ok'` |
| `renamePlaylist` | `{ playlistId, name }` | `'ok'` |
| `deletePlaylist` | `{ playlistId }` | `'ok'` |
| `requestPlaylists` | — | `'ok'` |

All of them except `setFocus`, `close`, `saveSong`, `removeSong` and the playlist-management ones require `myVehicleNetId` to be set, i.e. the player must be the registered driver.

### NUI messages (client → NUI)

| `type` | Fields | Meaning |
|---|---|---|
| `open` | `isOwner`, `vehicleNetId` | Show the panel. `isOwner = false` disables the transport controls, the volume slider and the loop button. |
| `close` | — | Hide the panel. |
| `radioSync` | `radio` | Apply full radio state to the open panel: title, thumbnail, playing state, loop, volume, and load/seek the YouTube player. |
| `radioStopped` | — | Our vehicle's radio stopped: clear the title, reset progress, hide the mini HUD, destroy the player. |
| `stopRadio` | `vehicleNetId` | Destroy the local player for that vehicle (used for other people's vehicles too). |
| `fullSync` | `radios` (map) | Apply state for our own vehicle if present in the map. |
| `updateSpatial` | `vehicleNetId`, `volume`, `paused`, `url`, `title`, `currentTime`, `startedAt`, `looped` | The per-tick volume update, and what triggers the first load of a YouTube player for a vehicle you have just come within range of. |
| `playlistsUpdate` | `playlists` | Re-render the playlist tab strip and song list. |
| `miniVolChange` | `volume` | The J/L keybinds changed the volume without the panel open — update the slider and show a toast. |
| `notification` | `message` | Show an uppercased toast. |
| `hideMini` | — | Hide the mini HUD (sent when the player is on foot). |

### Editable Functions (`shared/functions.lua`)

The only unencrypted code file, loaded as a **shared** script so these exist on both client and server.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `T` | `T(key, ...) → string` | `client/client.lua` (the `must_be_driver` notification) and `server/server.lua` (all nine playlist strings) | Locale lookup. Returns `Lang[key]`, `string.format`-ed with the varargs if any, or the raw `key` if missing. Must always return a string — it is passed straight into `TriggerClientEvent` payloads and SQL parameters (`default_playlist_name` becomes a playlist name in the database). |
| `ShowNotification` | `ShowNotification(message)` | **Nowhere in the shipped code.** | Provided for your own extensions. Default is the native GTA feed; ox_lib, ESX and QBCore one-liners are pre-written as comments. Note the resource's own user-facing messages go through the NUI toast instead. |
| `CanUseRadio` | `CanUseRadio() → boolean` | `client/client.lua`, inside the `/nexusradio` command, **after** the driver check and **before** the panel opens | Gate the radio behind an item, a job, a vehicle class or anything else. Return `false` to silently block opening (no notification is shown on a `false` return — add your own inside the function if you want feedback). Default returns `true`. |
| `GetCitizenId` | `GetCitizenId(src) → string` | `server/server.lua`, at the top of every playlist operation (`sendPlaylists`, `createPlaylist`, `renamePlaylist`, `deletePlaylist`, `saveSong`, `removeSong`) | **The key integration point.** Returns the identifier playlists are keyed by in the database. Default walks the player's identifiers and returns the first one containing `license:`, falling back to `'player_' .. src` if none is found. Must return a **stable, persistent** string — the fallback is not stable across sessions, so a player with no license identifier would get a fresh empty playlist set each time they connect. ESX (`xPlayer.getIdentifier()`) and QBCore (`PlayerData.citizenid`) branches are pre-written as comments. Changing this after launch orphans existing playlists. |

Also defined: the `Lang` table, populated by the locale loader at the top of the file.

### Database Schema

Both tables are created automatically by `server/server.lua` in a `CreateThread` at resource start — there is **no SQL file to import**.

```sql
CREATE TABLE IF NOT EXISTS nexus_radio_playlists (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    citizenid  VARCHAR(64) NOT NULL,
    name       VARCHAR(64) NOT NULL DEFAULT 'Playlist',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_citizen (citizenid)
);

CREATE TABLE IF NOT EXISTS nexus_radio_songs (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    playlist_id INT NOT NULL,
    url         VARCHAR(512) NOT NULL,
    title       VARCHAR(256) NOT NULL,
    position    INT NOT NULL DEFAULT 0,
    FOREIGN KEY (playlist_id) REFERENCES nexus_radio_playlists(id) ON DELETE CASCADE,
    INDEX idx_playlist (playlist_id)
);
```

Notes for integrators:
- `citizenid` is whatever `GetCitizenId(src)` returns — by default a full `license:xxxxxxxx` string, not a framework citizen ID, despite the column name.
- Playlists are ordered by `created_at ASC`; songs by `position ASC`. `position` is assigned as `MAX(position) + 1` and is never renumbered, so gaps are normal after deletions. The NUI's `index` is the position in the *ordered result set*, not the `position` value.
- `ON DELETE CASCADE` means deleting a playlist row removes its songs automatically.
- No table stores radio playback state — that is in-memory only and is lost on restart, by design.

### Integration Example

Gating the radio behind an `ox_inventory` item and switching playlists to ESX identifiers, both in the unencrypted `shared/functions.lua`:

```lua
-- shared/functions.lua
function CanUseRadio()
    if exports.ox_inventory:Search('count', 'car_radio') > 0 then
        return true
    end
    ShowNotification(T('must_be_driver'))   -- or your own "no radio installed" string
    return false
end

function GetCitizenId(src)
    local ESX = exports['es_extended']:getSharedObject()
    local xPlayer = ESX.GetPlayerFromId(src)
    if xPlayer then return xPlayer.getIdentifier() end
    return 'player_' .. tostring(src)        -- keep a fallback
end
```

A separate resource that logs every song played and bans certain videos:

```lua
-- my_radio_addon/server.lua
local blocked = { 'dQw4w9WgXcQ' }

AddEventHandler('nexus_carradio:playSong', function(vehicleNetId, url, title)
    local src = source
    for _, id in ipairs(blocked) do
        if url:find(id, 1, true) then
            TriggerClientEvent('nexus_carradio:notification', src, 'THAT TRACK IS BLOCKED')
            TriggerEvent('nexus_carradio:stop', vehicleNetId)
            return
        end
    end
    print(('[radio-log] %s played "%s" on vehicle %s'):format(GetPlayerName(src), title, vehicleNetId))
end)
```

Reading a player's saved playlists from your own resource (the tables are plain SQL, so just query them):

```lua
-- my_radio_addon/server.lua
local function getPlaylists(src)
    local cid = GetCitizenId(src)   -- exported globally by nexus_carradio's shared script
                                    -- within that resource; replicate the logic here if separate
    local pls = MySQL.query.await(
        'SELECT p.id, p.name, COUNT(s.id) AS songs ' ..
        'FROM nexus_radio_playlists p LEFT JOIN nexus_radio_songs s ON s.playlist_id = p.id ' ..
        'WHERE p.citizenid = ? GROUP BY p.id ORDER BY p.created_at ASC',
        { cid }
    ) or {}
    return pls
end
```

Mirroring radio state on the client (e.g. to drive your own HUD or a police "loud music" callout):

```lua
local radios = {}

RegisterNetEvent('nexus_carradio:sync', function(netId, radio)
    radios[netId] = radio
end)

RegisterNetEvent('nexus_carradio:stopped', function(netId)
    radios[netId] = nil
end)

-- Is there a vehicle blasting music near me?
CreateThread(function()
    while true do
        Wait(2000)
        local myPos = GetEntityCoords(PlayerPedId())
        for netId, radio in pairs(radios) do
            if radio.playing and (radio.volume or 0) > 0.8 then
                local veh = NetToVeh(netId)
                if DoesEntityExist(veh) and #(myPos - GetEntityCoords(veh)) < 15.0 then
                    print(('[callout] loud vehicle nearby: %s'):format(radio.title or '?'))
                end
            end
        end
    end
end)
```

## ❓ FAQ

**I press M and nothing happens.**
You must be in the **driver's seat** — passengers get the `must_be_driver` notification, and on foot nothing happens at all. Also check that M is not bound to something else in **Settings → Key Bindings → FiveM**, and that `CanUseRadio()` has not been edited to return `false`.

**The panel opens but there is no sound.**
Audio comes from YouTube inside the game's CEF browser. Check that the game client has internet access and that YouTube is not blocked at network level, and try a different video — age-restricted, region-locked and embed-disabled videos fail inside an iframe even though they play fine on youtube.com. Then check that `Config.MaxDistance` is larger than `Config.FullVolumeDistance`.

**Players outside the car can't hear it.**
They need to be within `Config.MaxDistance` (default 30 units), and each of them needs their own working YouTube connection — your server does not relay the audio, every client fetches the stream independently. Also note the NUI pauses players whose computed volume drops below `0.01`, so someone right at the edge of the range may hear nothing.

**The song starts from the wrong position for people who join mid-track.**
Report this. There is a known unit mismatch between the server's `startedAt` (which is `GetGameTimer()`, milliseconds since the **server** started) and the NUI's elapsed-time calculation (which uses `Date.now()`, milliseconds since the Unix epoch). It is documented in the incident report.

**Only the driver can control playback.**
By design. The server records `ownerId` when `playSong` is received and rejects pause, seek, volume, loop and stop from anyone else. Passengers see the panel but with the controls disabled.

**Nobody's playlists are saving.**
Playlists need `oxmysql` running and started **before** `nexus_carradio`, and the database user needs `CREATE TABLE` permission on first boot. Check the server console on startup for errors from the two `CREATE TABLE IF NOT EXISTS` statements. Note this resource needs the **oxmysql** API specifically — `mysql-async` will not work here even though it does for some other Nexus resources.

**I changed `GetCitizenId` and everyone's playlists vanished.**
Playlists are keyed by whatever that function returns. Changing it changes the key, so old rows are orphaned under the previous identifier. Either migrate the `citizenid` column yourself with an `UPDATE`, or decide on the identifier before launch.

**"Already in that playlist" when saving a song I don't see.**
Duplicate detection is on the exact `url` string within that playlist. `youtu.be/xxxx` and `youtube.com/watch?v=xxxx` are different strings, so the same video can be saved twice under two URL forms — but re-saving the identical URL is rejected.

**I can't delete my last playlist.**
Deliberate. The server refuses (`cannot_delete_last`) so the save flow always has somewhere to put a song. Rename it instead.

**The next song doesn't play automatically.**
Auto-advance only happens when a queue context exists, which means the track must have been started **from the playlist tab** (the play button on a playlist row). A song started from the search box plays alone and then stops, because playing manually deliberately clears the queue.

**Can I translate the panel?**
Only the 10 Lua-side strings in `locales/*.lua`. Everything rendered inside the panel — `REPRODUCIENDO AHORA`, `SIN SEÑAL`, `GUARDAR`, `CARGAR`, the modals, the empty-playlist message — is hardcoded in `html/`, which is inside the escrow. The keybind descriptions in GTA's settings menu are hardcoded too.

**There's a `README.md` in the folder that contradicts this page.**
Ignore it — it is a stale development file from an earlier build (it refers to a `car_radio` folder, a `client/main.lua`, a `Config.OpenKey` key that no longer exists, and in-memory playlists that are now database-backed). This documentation reflects the shipped code. The incident report recommends deleting it from the package.

**Does it conflict with the stock GTA radio or with `xsound`?**
No conflicts. It never touches GTA audio banks or native audio. It also does not mute the vanilla vehicle radio — do that yourself if you want only one audio source.

### Before opening a ticket

- Make sure the resource folder name is exactly **`nexus_carradio`** — it refuses to start otherwise and prints why to the server console.
- Make sure you are on the latest version of the resource (**2.0.0** at the time of writing).
- Confirm `oxmysql` is started **before** `nexus_carradio` in `server.cfg`, and that the two `nexus_radio_*` tables exist.
- Re-read this FAQ page, and ignore the stale `README.md` in the resource folder.
- Have your `shared/config.lua`, your `server.cfg` boot order, and both the F8 client log and the server console output ready.

## 📋 Changelog

### 2.0.0 — Current release
As declared in `fxmanifest.lua` (`version '2.0.0'`). No per-version history file ships with the resource; the items below are the feature set present in this build, and the elements that have clearly moved on from the 1.x design still described by the bundled `README.md`:
- **Multiple named playlists** persisted in MySQL (`nexus_radio_playlists` / `nexus_radio_songs`, auto-created), replacing the single in-memory playlist of the earlier design.
- Per-player caps (`Config.MaxPlaylists`, `Config.MaxPlaylistSize`), duplicate rejection, ownership checks on every write, and refusal to delete the last playlist.
- **Queue playback** with auto-advance and playlist-aware next/previous.
- **Mini HUD** with its own `RegisterKeyMapping` bindings (M / K / J / L) that work without NUI focus.
- **Loop toggle** synced through server radio state.
- Rebuilt NUI: 36-bar radial visualiser with attack/decay ramps, thumbnail crossfade, draggable progress bar, playlist tab strip, create/rename/delete modals.
- Logarithmic spatial audio with an occupant override, per-vehicle YouTube players, and automatic teardown of destroyed vehicles.
- Seven locales (`es`, `en`, `de`, `fr`, `it`, `pt`, `zh`) loaded on both client and server.
- Resource-name validation on load.
- Legacy `nexus_carradio:playlistUpdate` event still handled for backward compatibility with 1.x integrations.

