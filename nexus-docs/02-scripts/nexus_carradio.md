# Nexus Car Radio

Immersive in-vehicle radio for FiveM. Drivers stream YouTube straight from a sleek infotainment-style interface, other players hear the car with realistic spatial audio, and every player keeps persistent personal playlists in the database.

---

## Features

- **YouTube streaming** – paste a link and play it from the vehicle.
- **Spatial audio** – nearby players hear the car radio with a logarithmic fall-off. Full volume inside `Config.FullVolumeDistance`, silent beyond `Config.MaxDistance`.
- **Driver-owned radio** – only the driver who started the radio can control it (play, pause, seek, volume, loop, stop).
- **Persistent playlists** – each player has their own playlists saved in the database (oxmysql).
- **Full controls** – play / pause, seek, clickable progress bar, volume, loop.
- **Mini HUD and key bindings** – control playback and volume without opening the full UI.
- **Synced for everyone** – progress is computed from a shared real-time clock.
- **Server-side safety** – ownership checks on every action, volume clamped to 0.0 – 1.0, playlist input validated.
- **7 languages** – en, es, de, fr, it, pt, zh.
- **Open configuration** – `config.lua`, `functions.lua`, locales and the whole NUI are not encrypted.

---

## Dependencies

| Resource | Required | Notes |
|----------|----------|-------|
| oxmysql | Yes | Playlists storage. |
| OneSync | Yes | Server-side vehicle and driver validation. |

Standalone: no ESX or QBCore required.

---

## Installation

1. Drop the `nexus_carradio` folder into your `resources` directory. **The folder must keep the name `nexus_carradio`.**
2. Make sure `oxmysql` starts before it. In `server.cfg`:

   ```
   ensure oxmysql
   ensure nexus_carradio
   ```

3. Start the server. The tables are created automatically on first start (see Database Schema).
4. Optional: edit `shared/config.lua` and `shared/functions.lua` to your liking.

### Controls

All keys can be rebound in the FiveM key bindings menu.

| Action | Default | Command |
|--------|---------|---------|
| Open / close the radio (driver only) | `M` | `nexusradio` |
| Play / pause | `K` | `nexusradio_playpause` |
| Volume down | `J` | `nexusradio_voldown` |
| Volume up | `L` | `nexusradio_volup` |

---

## Configuration

`shared/config.lua`:

| Option | Default | Description |
|--------|---------|-------------|
| `Config.Locale` | `'en'` | Language file from `locales/`. |
| `Config.MaxDistance` | `30.0` | Maximum distance at which others can hear the radio. |
| `Config.FullVolumeDistance` | `3.0` | Full volume inside this radius. |
| `Config.SyncInterval` | `500` | Nearby vehicle check interval in ms. |
| `Config.UpdateInterval` | `300` | Spatial volume refresh in ms. |
| `Config.MaxPlaylistSize` | `50` | Maximum songs per playlist. |
| `Config.MaxPlaylists` | `10` | Maximum playlists per player. |

`shared/functions.lua` contains the hooks you can adapt to your server:

- `ShowNotification(message)` – replace with your notification system.
- `CanUseRadio()` – return `false` to block the radio (for example when the player does not own a radio item).
- `GetCitizenId(src)` – identifier used to store playlists.

---

## Locales & Editable Strings

Language files are in `locales/` (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`). Change `Config.Locale` to switch language. To add a language, copy `locales/en.lua`, translate the values, set `Config.Locale` to its name and list the file under `files` in `fxmanifest.lua`.

| Key | Default (en) |
|-----|--------------|
| `must_be_driver` | You must be the driver |
| `default_playlist_name` | My Playlist |
| `new_playlist_name` | New Playlist |
| `max_playlists_reached` | Maximum playlists reached (%s) |
| `cannot_delete_last` | You cannot delete your only playlist |
| `playlist_deleted` | Playlist deleted |
| `playlist_not_found` | Playlist not found |
| `song_already_in_playlist` | Already in that playlist |
| `playlist_full` | Playlist full (%s) |
| `song_saved` | Saved: %s |

Interface texts live in `html/` and can be edited freely.

---

## Compatibility

- **Game:** FiveM (gta5), `cerulean` manifest.
- **Server:** OneSync and oxmysql required.
- **Frameworks:** standalone. Works with ESX, QBCore and any other framework through the hooks in `shared/functions.lua`.
- **Escrow:** config, `shared/functions.lua`, locales and the whole `html/` folder are open. Core client/server logic is encrypted.

---

## Developer API

### Exports

This resource does not expose exports.

### Events

**Client → Server**

| Event | Arguments | Description |
|-------|-----------|-------------|
| `nexus_carradio:playSong` | `vehicleNetId, url, title, volume, looped` | Starts a track. The sender must be the driver of the vehicle, or the owner of its active radio. |
| `nexus_carradio:setPaused` | `vehicleNetId, paused, currentTime` | Owner only. |
| `nexus_carradio:seek` | `vehicleNetId, currentTime` | Owner only. |
| `nexus_carradio:setVolume` | `vehicleNetId, volume` | Owner only. Clamped to 0.0 – 1.0. |
| `nexus_carradio:setLoop` | `vehicleNetId, looped` | Owner only. |
| `nexus_carradio:stop` | `vehicleNetId` | Owner only. |
| `nexus_carradio:requestAll` | – | Requests all active radios. |
| `nexus_carradio:requestPlaylists` | – | Requests the player's playlists. |
| `nexus_carradio:createPlaylist` | `name` | Creates a playlist (limited by `Config.MaxPlaylists`). |
| `nexus_carradio:renamePlaylist` | `playlistId, newName` | `newName` must be a non-empty string. |
| `nexus_carradio:deletePlaylist` | `playlistId` | The last playlist cannot be deleted. |
| `nexus_carradio:saveSong` | `playlistId, url, title` | Adds a song. |
| `nexus_carradio:removeSong` | `playlistId, songIndex` | Removes a song. |

**Server → Client**

| Event | Arguments | Description |
|-------|-----------|-------------|
| `nexus_carradio:sync` | `vehicleNetId, radioData` | Playback state changed. |
| `nexus_carradio:stopped` | `vehicleNetId` | Playback stopped. |
| `nexus_carradio:fullSync` | `radios` | Full state for a joining client. |
| `nexus_carradio:playlistsUpdate` | `playlists` | Playlists refreshed. |
| `nexus_carradio:notification` | `message` | Message shown in the UI. |

`radioData` fields: `url`, `title`, `playing`, `startedAt` (milliseconds, `os.time() * 1000`), `currentTime`, `volume`, `ownerId`, `looped`.

### Functions

| Function | Side | Description |
|----------|------|-------------|
| `T(key, ...)` | Shared | Returns a translated string. |
| `ShowNotification(message)` | Client | Notification hook. |
| `CanUseRadio()` | Client | Gate for opening the radio. |
| `GetCitizenId(src)` | Server | Returns the identifier used for playlists, or `nil` if none can be resolved. When it returns `nil` the action is ignored. |

### Database Schema

Created automatically on start.

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

### Integration Example

Use your framework's character id so playlists follow the character instead of the Rockstar license.

```lua
-- shared/functions.lua (server side branch)
function GetCitizenId(src)
    local QBCore = exports['qb-core']:GetCoreObject()
    local p = QBCore.Functions.GetPlayer(src)
    if p then return p.PlayerData.citizenid end
    return nil
end
```

Require an item to use the radio:

```lua
function CanUseRadio()
    return exports.ox_inventory:Search('count', 'car_radio') > 0
end
```

---

## FAQ

**Who can control the radio?**
The player who started it. Other players in the car can listen but not control it.

**Does it need ESX or QBCore?**
No. It is standalone. Use `GetCitizenId` to tie playlists to your framework's character id.

**Where are playlists stored?**
In the `nexus_radio_playlists` and `nexus_radio_songs` tables. By default they are tied to the player's Rockstar license.

**What if no identifier can be found for a player?**
Playlist actions are ignored instead of using a temporary key that would change on reconnect.

**Can I require an item to use the radio?**
Yes, edit `CanUseRadio()`.

**A video does not play.**
Some YouTube videos block embedding. Try another link.

---

## Changelog

### 2.1.0
- Fixed the progress clock: the server now sends `os.time() * 1000` so the UI and the server use the same clock.
- `playSong` now validates driver and radio ownership.
- `setVolume` is clamped to 0.0 – 1.0 on the server.
- `GetCitizenId` no longer returns an unstable `player_<id>` fallback. It returns `nil` and callers stop.
- `renamePlaylist` validates that the new name is a non-empty string.
- Removed the unguarded per-song server `print()`.
- `html/` is now outside the escrow.
- Replaced the outdated README.

### 2.0.0
- Persistent playlists with oxmysql.
- Mini HUD, key bindings and spatial audio improvements.

