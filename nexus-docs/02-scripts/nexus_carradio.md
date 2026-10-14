---
icon: radio
---

# Nexus CarRadio

> An in-vehicle radio with an infotainment-style interface, spatial audio and persistent personal playlists. It plays the audio you provide: direct links, online radio streams and files stored on your server.

**Version:** 2.0.0 · **Framework(s):** Standalone (ESX, QBCore and any other framework through hooks) · **Database:** oxmysql

***

## Description

Nexus CarRadio brings an immersive radio to every vehicle of your server. The driver opens a sleek infotainment-style panel, loads a track and everybody in range hears the car with realistic spatial audio: full volume inside the vehicle, a smooth logarithmic fall-off outside it and silence beyond the configured distance. Every player keeps their own playlists, saved in the database, and can queue a whole playlist to play one song after another.

Playback is driver-owned and synchronised for everyone. The progress of the track is computed from a shared real-time clock, so passengers and bystanders always hear the song at the right position, including players who arrive when the music is already playing.

The resource is standalone. It works out of the box and connects to ESX, QBCore or any other framework through a small set of editable hook functions: notifications, item requirements and the identifier used to store playlists.

***

## Features

* **Your own audio**: direct links to audio files, online radio streams and files stored in `html/audio/`. The allowed domains can be restricted with `Config.AllowedHosts`.
* **Spatial audio**: full volume inside `Config.FullVolumeDistance` and for everyone inside the vehicle, logarithmic fall-off outside, silent beyond `Config.MaxDistance`.
* **Driver-owned radio**: only the player who started the radio can control it (play, pause, seek, volume, loop, stop). Other occupants and bystanders listen but cannot control it.
* **Persistent playlists**: each player has their own playlists in the database, with rename, delete and per-playlist song lists.
* **Playlist queue**: playing a song from a playlist continues with the next song of that playlist when the track ends.
* **Full controls**: play and pause, clickable progress bar, volume, loop.
* **Mini HUD and key bindings**: control playback and volume without opening the full panel.
* **Synchronised for everyone**: the progress is computed from a shared real-time clock, not from a local timer.
* **Server-side safety**: ownership checks on every action, volume clamped to 0.0 - 1.0, audio source validation, playlist input validated and every playlist query scoped to its owner.
* **7 languages**: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`.
* **Open configuration**: `config.lua`, `functions.lua`, locales and the whole interface are not encrypted.

***

## Dependencies

### Required

| Resource  | Notes                                            |
| --------- | ------------------------------------------------ |
| `oxmysql` | Stores the playlists.                            |
| OneSync   | The server validates the vehicle and its driver. |

### Optional

* An inventory resource if you want to require an item to use the radio (see **Integration examples**).

No ESX or QBCore is required.

### Server requirements

* The resource folder must be named exactly `nexus_carradio`.
* The tables are created automatically on the first start. No SQL import is needed.

***

## Installation

1. Drop the `nexus_carradio` folder into your `resources` directory. Keep the folder name exactly `nexus_carradio`.
2.  Make sure `oxmysql` starts before it. In `server.cfg`:

    ```
    ensure oxmysql
    ensure nexus_carradio
    ```
3. Start the server. The tables are created automatically (see **Database schema**).
4. Optional: edit `shared/config.lua` and `shared/functions.lua` to your liking.
5. Optional: copy your own audio files into `html/audio/` (see **Audio sources**).

### Controls

All keys can be rebound by each player in the FiveM key bindings menu.

| Action                                | Default | Command                |
| ------------------------------------- | ------- | ---------------------- |
| Open or close the radio (driver only) | `M`     | `nexusradio`           |
| Play or pause                         | `K`     | `nexusradio_playpause` |
| Volume down                           | `J`     | `nexusradio_voldown`   |
| Volume up                             | `L`     | `nexusradio_volup`     |

The radio panel closes by itself when the driver leaves the vehicle. The play, pause and volume keys act on the radio of the vehicle where the panel has been opened.

***

## Audio sources

The radio plays audio that you or your players provide. Three kinds of source are supported, and any of them can be typed or pasted in the input field of the panel.

| Source                       | Example                                   | Notes                                                                                                         |
| ---------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Direct link to an audio file | `https://cdn.myserver.com/music/song.mp3` | Supported formats: `mp3`, `ogg`, `wav`, `aac`, `flac`, `m4a` and `opus`. Use HTTPS links.                     |
| Online radio stream          | `https://stream.myradio.com/live`         | Icecast and Shoutcast style streams work. A live stream has no fixed length, so there is no position to seek. |
| Local file                   | `song.mp3`                                | A file stored in `html/audio/` of the resource. Type only the file name.                                      |

The title shown in the panel and saved in playlists is taken from the file name of the link, or from the domain of a stream.

### Using local files

1. Copy the audio file into `nexus_carradio/html/audio/`.
2. Restart the resource so the new file is registered.
3. In the panel, type the file name, for example `song.mp3`. A file inside a subfolder is written with the folder, for example `radio/song.mp3`.

Local file names can contain letters, numbers, spaces, `_`, `-` and `.`. The extension must be one of the supported formats. Paths with `..`, a leading `/`, a drive letter or a backslash are rejected.

### Restricting the allowed domains

Leave `Config.AllowedHosts` empty to allow links from any domain, or list the domains you control:

```lua
Config.AllowedHosts = { 'cdn.myserver.com', 'stream.myradio.com' }
```

Subdomains of a listed domain are accepted too. A link from any other domain is rejected by the server, in the moment of playing and when saving it to a playlist, and the player sees the message of the locale key `url_not_allowed`. Local files are always allowed.

{% hint style="info" %}
Make sure you have the rights to the audio you play on your server.
{% endhint %}

***

## Configuration

`shared/config.lua`:

| Option                      | Default | Description                                                                        |
| --------------------------- | ------- | ---------------------------------------------------------------------------------- |
| `Config.Locale`             | `'en'`  | Language file from `locales/`.                                                     |
| `Config.MaxDistance`        | `30.0`  | Maximum distance at which other players hear the radio.                            |
| `Config.FullVolumeDistance` | `3.0`   | Full volume inside this radius.                                                    |
| `Config.SyncInterval`       | `500`   | Interval in ms of the nearby vehicle check.                                        |
| `Config.UpdateInterval`     | `300`   | Interval in ms of the spatial volume refresh.                                      |
| `Config.MaxPlaylistSize`    | `50`    | Maximum songs per playlist.                                                        |
| `Config.MaxPlaylists`       | `10`    | Maximum playlists per player.                                                      |
| `Config.AllowedHosts`       | `{}`    | Domains allowed for audio links. Empty allows any domain. Subdomains are included. |

### Hooks in `shared/functions.lua`

| Function                    | Side   | Purpose                                                                                                             |
| --------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------- |
| `ShowNotification(message)` | Client | Replace it with your notification system. Commented examples for ox\_lib, ESX and QBCore are included.              |
| `CanUseRadio()`             | Client | Return `false` to block opening the radio, for example when the player has no radio item.                           |
| `GetCitizenId(src)`         | Server | Identifier used to store the playlists. Returns the Rockstar license by default. Return `nil` to ignore the action. |
| `T(key, ...)`               | Shared | Returns a translated string from the active locale.                                                                 |

***

## How it works

1. The driver presses `M`. The panel opens and the playlists of the player are loaded.
2. The driver types or pastes a source and presses play. The server checks that the player is the driver of the vehicle and that the source is allowed, then stores the radio with its start time and broadcasts it.
3. Every client plays the audio locally. The spatial loop adjusts the volume with the distance to each active vehicle every `Config.UpdateInterval` ms.
4. Only the owner of the radio can pause, seek, change the volume, loop or stop it. The server rejects every other attempt.
5. The radio keeps playing when the owner leaves the vehicle. It stops when the owner stops it, when the owner disconnects or when the vehicle no longer exists.

***

## Locales and editable strings

Language files are in `locales/` (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`). Change `Config.Locale` to switch language.

| Key                        | Default (en)                                    |
| -------------------------- | ----------------------------------------------- |
| `must_be_driver`           | You must be the driver                          |
| `default_playlist_name`    | My Playlist                                     |
| `new_playlist_name`        | New Playlist                                    |
| `max_playlists_reached`    | Maximum playlists reached (%s)                  |
| `cannot_delete_last`       | You cannot delete your only playlist            |
| `playlist_deleted`         | Playlist deleted                                |
| `playlist_not_found`       | Playlist not found                              |
| `song_already_in_playlist` | Already in that playlist                        |
| `playlist_full`            | Playlist full (%s)                              |
| `song_saved`               | Saved: %s                                       |
| `url_not_allowed`          | This audio source is not allowed on this server |

To add a language, copy `locales/en.lua`, translate the values, set `Config.Locale` to its name and list the file under `files` in `fxmanifest.lua`.

The interface texts live in `html/index.html` and `html/js/app.js` and can be edited freely.

***

## Compatibility

* **Game:** FiveM (gta5), `cerulean` manifest.
* **Server:** OneSync and oxmysql required.
* **Frameworks:** standalone. Works with ESX, QBCore and any other framework through the hooks in `shared/functions.lua`.
* **Escrow:** `shared/config.lua`, `shared/functions.lua`, locales and the whole `html/` folder are open. The core client and server logic is encrypted.

***

## Developer API

### Exports

This resource does not expose exports.

### Events - Client to server

| Event                             | Arguments                                  | Description                                                                                                                    |
| --------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `nexus_carradio:playSong`         | `vehicleNetId, url, title, volume, looped` | Starts a track. The sender must be the driver of the vehicle or the owner of its active radio, and the source must be allowed. |
| `nexus_carradio:setPaused`        | `vehicleNetId, paused, currentTime`        | Owner only.                                                                                                                    |
| `nexus_carradio:seek`             | `vehicleNetId, currentTime`                | Owner only.                                                                                                                    |
| `nexus_carradio:setVolume`        | `vehicleNetId, volume`                     | Owner only. Clamped to 0.0 - 1.0.                                                                                              |
| `nexus_carradio:setLoop`          | `vehicleNetId, looped`                     | Owner only.                                                                                                                    |
| `nexus_carradio:stop`             | `vehicleNetId`                             | Owner only.                                                                                                                    |
| `nexus_carradio:requestAll`       | none                                       | Requests all active radios. Sent automatically when the resource starts.                                                       |
| `nexus_carradio:requestPlaylists` | none                                       | Requests the playlists of the player.                                                                                          |
| `nexus_carradio:createPlaylist`   | `name`                                     | Creates a playlist, limited by `Config.MaxPlaylists`.                                                                          |
| `nexus_carradio:renamePlaylist`   | `playlistId, newName`                      | `newName` must be a non-empty string.                                                                                          |
| `nexus_carradio:deletePlaylist`   | `playlistId`                               | The last playlist of a player cannot be deleted.                                                                               |
| `nexus_carradio:saveSong`         | `playlistId, url, title`                   | Adds a song. The source is validated like in `playSong`.                                                                       |
| `nexus_carradio:removeSong`       | `playlistId, songIndex`                    | Removes a song.                                                                                                                |

### Events - Server to client

| Event                            | Arguments                 | Description                               |
| -------------------------------- | ------------------------- | ----------------------------------------- |
| `nexus_carradio:sync`            | `vehicleNetId, radioData` | The playback state changed.               |
| `nexus_carradio:stopped`         | `vehicleNetId`            | The playback stopped.                     |
| `nexus_carradio:fullSync`        | `radios`                  | Full state for a client that just joined. |
| `nexus_carradio:playlistsUpdate` | `playlists`               | The playlists were refreshed.             |
| `nexus_carradio:notification`    | `message`                 | Message shown in the panel.               |

`radioData` fields: `url`, `title`, `playing`, `startedAt` (milliseconds, `os.time() * 1000`), `currentTime`, `volume`, `ownerId` and `looped`.

`playlists` is a list of `{ id, name, songs }`, where `songs` is a list of `{ url, title }`.

### Editable functions

The hooks listed in **Configuration** are plain global functions in `shared/functions.lua`.

### Database schema

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

The column `citizenid` stores whatever `GetCitizenId` returns. If you change that function on a server that already has playlists, existing rows keep their old identifier.

### Integration examples

#### Playlists tied to the character (QBCore)

Use the character id so the playlists follow the character and not the Rockstar license.

```lua
function GetCitizenId(src)
    local QBCore = exports['qb-core']:GetCoreObject()
    local p = QBCore.Functions.GetPlayer(src)
    if p then return p.PlayerData.citizenid end
    return nil
end
```

#### Playlists tied to the character (ESX)

```lua
function GetCitizenId(src)
    local ESX = exports['es_extended']:getSharedObject()
    local xp = ESX.GetPlayerFromId(src)
    if xp then return xp.getIdentifier() end
    return nil
end
```

#### Require an item to use the radio

```lua
function CanUseRadio()
    return exports.ox_inventory:Search('count', 'car_radio') > 0
end
```

```lua
function CanUseRadio()
    local QBCore = exports['qb-core']:GetCoreObject()
    return QBCore.Functions.HasItem('car_radio')
end
```

#### Use your notification system

```lua
function ShowNotification(message)
    exports.nexus_notify:Alert('Car Radio', message, 5000, 'info')
end
```

```lua
function ShowNotification(message)
    lib.notify({ title = 'Car Radio', description = message, type = 'inform' })
end
```

#### Host your own music

Upload your tracks to a storage or CDN you control, then allow only that domain.

```lua
Config.AllowedHosts = { 'cdn.myserver.com' }
```

Players can then paste `https://cdn.myserver.com/music/song.mp3` in the panel, or you can ship the files inside `html/audio/` and tell players to type their file names.

#### Stop the radio from another script

The owner of a radio can stop it from client code, for example when the engine is turned off.

```lua
local veh = GetVehiclePedIsIn(PlayerPedId(), false)
if veh ~= 0 then
    TriggerServerEvent('nexus_carradio:stop', NetworkGetNetworkIdFromEntity(veh))
end
```

#### React to what is being played

Server-side listener, for example to log the sources players use.

```lua
AddEventHandler('nexus_carradio:playSong', function(vehicleNetId, url, title)
    local src = source
    print(('%s played %s'):format(GetPlayerName(src), url))
end)
```

***

## FAQ

**Who can control the radio?** The player who started it. Other players in the car can listen but not control it.

**Does it need ESX or QBCore?** No, it is standalone. Use `GetCitizenId` to tie playlists to the character id of your framework.

**Where are playlists stored?** In the `nexus_radio_playlists` and `nexus_radio_songs` tables. By default they are tied to the Rockstar license of the player.

**What happens if no identifier can be found for a player?** Playlist actions are ignored instead of using a temporary key that would change on reconnect.

**Can I require an item to use the radio?** Yes, edit `CanUseRadio()`.

**A link does not play.** Check that it points directly to an audio file or to a radio stream, that it uses HTTPS and that its domain is listed in `Config.AllowedHosts` when that list is not empty.

**A local file does not play.** Check that the file is inside `html/audio/`, that the extension is a supported format, that you typed the exact file name and that the resource was restarted after copying the file.

**The radio keeps playing after I leave the car.** The radio belongs to the player who started it and keeps running until that player stops it, disconnects or the vehicle is removed.

**Can two players share the same vehicle radio?** Only the owner controls the radio. Once the owner stops it, any driver can start a new one.

**Do passengers hear the radio at full volume?** Yes. Everybody inside the vehicle hears it at the volume set by the owner. Players outside hear it with the distance fall-off.

**Can I change the maximum number of playlists and songs?** Yes, with `Config.MaxPlaylists` and `Config.MaxPlaylistSize`.
