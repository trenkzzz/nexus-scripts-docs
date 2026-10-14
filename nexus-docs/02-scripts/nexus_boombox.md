---
icon: boombox
---

# Nexus Boombox

> A portable, community-style boombox. Drop it anywhere, load an audio link or a file from your server, and everyone nearby hears it with realistic spatial audio.

**Version:** 1.0.0 · **Framework(s):** Standalone (optional ox\_inventory / qb-core / es\_extended) · **Database:** None

***

## Description

Nexus Boombox adds a placeable boombox to your server. A player places it on the ground, opens its retro interface, loads a track and everyone around hears it, louder when close and quieter as they walk away. The boombox is a shared object: it does not lock to its owner, so anyone in range can play, pause, change the volume, loop or stop it. This makes it a good fit for roleplay scenes such as parties, street corners and camps, where music is a shared favour and not a private possession.

The audio is always yours. The boombox plays direct audio links, online radio streams or audio files that you store inside the resource, so you decide what is available on your server and you can restrict the allowed domains with a single config option.

Playback is synchronised through a shared real-time clock. A player who walks into range, or who connects while the music is playing, starts hearing the track at the correct second.

***

## Features

* **Your own audio**: play direct links to audio files, online radio streams or files stored in `html/audio/`.
* **Allowed domain list**: `Config.AllowedHosts` limits links to the domains you trust. Local files are always allowed.
* **Spatial audio**: the volume falls off logarithmically with the distance. Full volume inside `Config.FullVolumeDistance` and silence beyond `Config.MaxDistance`.
* **Community design**: no ownership lock. Anyone in range can play, pause, change the volume, loop or stop the boombox.
* **Place and pick up in the world**: the prop is spawned for every player, with a place and pick-up animation, an interaction hint and a server-side distance check on pickup.
* **Synced for everyone**: late joiners and players walking into range start at the right second.
* **Volume ceiling**: `Config.MaxVolume` is enforced on the server and in the interface.
* **Inventory aware**: `none`, `ox_inventory`, `qb-core`, `es_extended` or `custom`.
* **Configurable notifications and hints**: native GTA feed and 3D text, `nexus_notify`, `nexus_TextUI` or your own system.
* **Item duplication protection**: the item is only returned after a valid, in-range pickup of a boombox that really existed.
* **Cleanup**: boomboxes are removed when their owner disconnects and when the resource stops.
* **7 languages**: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`, all editable.
* **Open configuration**: `config.lua`, `client/functions.lua`, `server/functions.lua`, locales and the whole interface are not encrypted.

***

## Dependencies

### Required

* OneSync. The server reads player positions to validate pickups.

### Optional

| Resource       | When it is needed                      |
| -------------- | -------------------------------------- |
| `ox_inventory` | `Config.Inventory = 'ox'`              |
| `qb-core`      | `Config.Inventory = 'qb'`              |
| `es_extended`  | `Config.Inventory = 'esx'`             |
| `nexus_notify` | `Config.NotifySystem = 'nexus_notify'` |
| `nexus_TextUI` | `Config.TextUISystem = 'nexus_TextUI'` |

### Server requirements

* No database and no SQL import. Boomboxes are not persisted across restarts.
* The resource folder must be named exactly `nexus_boombox`.

***

## Installation

1. Drop the `nexus_boombox` folder into your `resources` directory. Keep the folder name exactly `nexus_boombox`.
2.  Add it to your `server.cfg`, after your inventory and the optional Nexus resources you use:

    ```
    ensure nexus_notify
    ensure nexus_TextUI
    ensure nexus_boombox
    ```
3. Open `shared/config.lua` and set `Config.Inventory`, `Config.NotifySystem` and `Config.TextUISystem` to match your server.
4. If you use an inventory, create the item (the name comes from `Config.BoomboxItem`, default `boombox`) and make its use action run the command `useboombox`. See **Integration examples**.
5. Optional: copy your own audio files into `html/audio/` (see **Audio sources**).
6. Restart the server.

### Controls

| Action                     | Default                                                        |
| -------------------------- | -------------------------------------------------------------- |
| Place a boombox            | `/useboombox` command (bind it to the use action of your item) |
| Open the boombox interface | `E` near a boombox                                             |
| Pick up the boombox        | `G` near a boombox                                             |
| Close the interface        | `ESC`                                                          |

The keys `E` and `G` are fixed. `Config.InteractKeyLabel` and `Config.PickupKeyLabel` only change the label shown in the hint.

***

## Audio sources

The boombox plays audio that you provide. Three kinds of source are supported, and any of them can be typed or pasted in the input field of the interface.

| Source                       | Example                                   | Notes                                                                                                         |
| ---------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Direct link to an audio file | `https://cdn.myserver.com/music/song.mp3` | Supported formats: `mp3`, `ogg`, `wav`, `aac`, `flac`, `m4a` and `opus`. Use HTTPS links.                     |
| Online radio stream          | `https://stream.myradio.com/live`         | Icecast and Shoutcast style streams work. A live stream has no fixed length, so there is no position to seek. |
| Local file                   | `song.mp3`                                | A file stored in `html/audio/` of the resource. Type only the file name.                                      |

### Using local files

1. Copy the audio file into `nexus_boombox/html/audio/`.
2. Restart the resource so the new file is registered.
3. In the interface, type the file name, for example `song.mp3`. A file inside a subfolder is written with the folder, for example `party/song.mp3`.

Local file names can contain letters, numbers, spaces, `_`, `-` and `.`. The extension must be one of the supported formats. Paths with `..`, a leading `/`, a drive letter or a backslash are rejected.

### Restricting the allowed domains

Leave `Config.AllowedHosts` empty to allow links from any domain, or list the domains you control:

```lua
Config.AllowedHosts = { 'cdn.myserver.com', 'stream.myradio.com' }
```

Subdomains of a listed domain are accepted too (`music.cdn.myserver.com` is accepted when `cdn.myserver.com` is listed). A link from any other domain is rejected by the server and the player sees the message of the locale key `url_not_allowed`. Local files are always allowed.

{% hint style="info" %}
Make sure you have the rights to the audio you play on your server.
{% endhint %}

***

## Configuration

All options live in `shared/config.lua`.

| Option                      | Default                 | Description                                                                                   |
| --------------------------- | ----------------------- | --------------------------------------------------------------------------------------------- |
| `Config.Locale`             | `'en'`                  | Language file from `locales/`.                                                                |
| `Config.Debug`              | `false`                 | Skips the item checks and does not consume or return the item. Keep it `false` in production. |
| `Config.BoomboxItem`        | `'boombox'`             | Item required to place the boombox.                                                           |
| `Config.Inventory`          | `'none'`                | `'none'`, `'ox'`, `'qb'`, `'esx'` or `'custom'`. `'none'` does not require any item.          |
| `Config.NotifySystem`       | `'native'`              | `'native'`, `'nexus_notify'` or `'custom'`.                                                   |
| `Config.TextUISystem`       | `'native'`              | `'native'`, `'nexus_TextUI'` or `'custom'`.                                                   |
| `Config.BoomboxProp`        | `'prop_ghettoblast_02'` | World prop model.                                                                             |
| `Config.InteractDistance`   | `2.5`                   | Distance at which the hint appears and the keys work.                                         |
| `Config.FullVolumeDistance` | `3.0`                   | Listeners inside this radius hear full volume.                                                |
| `Config.MaxDistance`        | `25.0`                  | Beyond this distance the boombox is silent.                                                   |
| `Config.UpdateInterval`     | `300`                   | Spatial audio refresh interval in ms.                                                         |
| `Config.MaxVolume`          | `1.0`                   | Highest volume allowed (0.0 to 1.0), enforced on the server and in the interface.             |
| `Config.DefaultVolume`      | `0.7`                   | Volume used on the first play.                                                                |
| `Config.InteractKeyLabel`   | `'E'`                   | Label of the interact key in the hint.                                                        |
| `Config.PickupKeyLabel`     | `'G'`                   | Label of the pick-up key in the hint.                                                         |
| `Config.AllowedHosts`       | `{}`                    | Domains allowed for audio links. Empty allows any domain.                                     |

### Inventory modes

| `Config.Inventory` | Behaviour                                                                                                                          |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `'none'`           | No item is required. Anyone can run `/useboombox`.                                                                                 |
| `'ox'`             | Checks and consumes the item with `ox_inventory`.                                                                                  |
| `'qb'`             | Checks the item with `QBCore.Functions.HasItem` and adds or removes it on the server.                                              |
| `'esx'`            | Checks the ESX player inventory and adds or removes the item on the server.                                                        |
| `'custom'`         | Runs your own code. Edit `HasBoomboxItem` in `client/functions.lua` and the `give` and `take` functions in `server/functions.lua`. |

The item is consumed when the boombox is placed. It is returned when the player who placed the boombox picks it up. Server-side, every return needs a valid pickup registered first, so the item cannot be duplicated by triggering the event manually.

### Notification and hint systems

* `native`: GTA feed notification and 3D text above the boombox.
* `nexus_notify`: notifications are sent with the `Alert` export of nexus\_notify.
* `nexus_TextUI`: the hint is split in one floating prompt per key, anchored to the boombox, using the `Create` and `Delete` exports of nexus\_TextUI.
* `custom`: write your own call in `ShowNotification`, `DrawInteractText` and `HideInteractText` in `client/functions.lua`.

***

## How it works

1. The player runs `/useboombox`. The inventory check runs, the placing animation plays and the boombox is registered on the server.
2. The server tells every client to spawn the prop at the same coordinates. A player can have one boombox placed at a time.
3. Anyone within `Config.InteractDistance` sees the hint. `E` opens the interface and `G` picks the boombox up.
4. When a track is loaded, the server stores it with its start time and broadcasts it. Every client plays the audio locally and adjusts the volume with the distance to the boombox.
5. When the owner picks it up or disconnects, the boombox is removed for everyone.

The interface closes automatically when the player walks away from the boombox.

***

## Locales and editable strings

Language files are in `locales/` (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`). Change `Config.Locale` to switch language.

| Key               | Default (en)                                         |
| ----------------- | ---------------------------------------------------- |
| `no_item`         | You do not have a boombox                            |
| `already_placed`  | You already have a boombox placed. Pick it up first. |
| `picked_up`       | Boombox picked up                                    |
| `interact_hint`   | \[%s] Use Boombox \[%s] Pick up                      |
| `url_not_allowed` | This audio source is not allowed on this server      |

In `interact_hint` the first `%s` is `Config.InteractKeyLabel` and the second one is `Config.PickupKeyLabel`.

To add a language, copy `locales/en.lua`, translate the values, set `Config.Locale` to the new file name and list the file under `files` in `fxmanifest.lua`.

The interface texts live in `html/index.html` and `html/js/app.js` and can be edited freely.

***

## Compatibility

* **Game:** FiveM (gta5), `cerulean` manifest.
* **Server:** OneSync required.
* **Frameworks:** standalone. Inventory support for ox\_inventory, qb-core and es\_extended, plus a `custom` hook.
* **Escrow:** `shared/config.lua`, `client/functions.lua`, `server/functions.lua`, locales and the whole `html/` folder are open. The core client and server logic is encrypted.

***

## Developer API

### Exports

This resource does not expose exports.

### Events - Client to server

| Event                       | Arguments                          | Description                                                                                                                 |
| --------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `nexus_boombox:place`       | `x, y, z, heading`                 | Registers a new boombox at the given position.                                                                              |
| `nexus_boombox:pickup`      | `boomboxId`                        | Removes a boombox. The server checks that the player is close to it.                                                        |
| `nexus_boombox:playSong`    | `boomboxId, url, title, volume`    | Starts a track. The source is validated (format and `Config.AllowedHosts`) and the volume is clamped to `Config.MaxVolume`. |
| `nexus_boombox:setPaused`   | `boomboxId, paused, currentTime`   | Pauses or resumes.                                                                                                          |
| `nexus_boombox:setVolume`   | `boomboxId, volume`                | Changes the volume, clamped to `Config.MaxVolume`.                                                                          |
| `nexus_boombox:stop`        | `boomboxId`                        | Stops the playback.                                                                                                         |
| `nexus_boombox:requestAll`  | none                               | Requests the full list of boomboxes. Sent automatically when the resource starts.                                           |
| `nexus_boombox:server:item` | `action ('add' or 'remove'), item` | Gives or takes the configured item. `add` only works after a valid pickup.                                                  |

### Events - Server to client

| Event                       | Arguments                                        | Description                                       |
| --------------------------- | ------------------------------------------------ | ------------------------------------------------- |
| `nexus_boombox:placed`      | `id, propNetId, x, y, z, heading, ownerServerId` | A boombox was placed.                             |
| `nexus_boombox:sync`        | `boomboxId, radioData`                           | The playback state changed.                       |
| `nexus_boombox:stopped`     | `boomboxId`                                      | The playback stopped.                             |
| `nexus_boombox:pickedUp`    | `boomboxId`                                      | A boombox was removed.                            |
| `nexus_boombox:fullSync`    | `boomboxes`                                      | Full state for a client that just joined.         |
| `nexus_boombox:urlRejected` | none                                             | The audio source sent by the player was rejected. |

`radioData` fields: `url`, `title`, `playing`, `startedAt` (milliseconds, `os.time() * 1000`), `currentTime`, `volume`, `ownerId` and `looped`.

You can listen to the server events from your own scripts, for example to log what is being played:

```lua
AddEventHandler('nexus_boombox:playSong', function(boomboxId, url, title, volume)
    local src = source
    print(('%s played %s'):format(GetPlayerName(src), url))
end)
```

### Editable functions

Editable in `client/functions.lua` and `server/functions.lua`.

| Function                                                | Side   | Description                                                                        |
| ------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------- |
| `T(key, ...)`                                           | Client | Returns a translated string.                                                       |
| `HasBoomboxItem()`                                      | Client | Item check by `Config.Inventory`. Edit the `custom` branch for your own inventory. |
| `RemoveBoomboxItem()` / `AddBoomboxItem()`              | Client | Ask the server to take or return the item.                                         |
| `ShowNotification(message)`                             | Client | Routes the message through `Config.NotifySystem`.                                  |
| `DrawInteractText(coords, text)` / `HideInteractText()` | Client | Draw and hide the hint through `Config.TextUISystem`.                              |
| `PlayPlaceAnimation(ped)` / `PlayPickupAnimation(ped)`  | Client | Animations played when placing and picking up.                                     |
| `give(src, item)` / `take(src, item)`                   | Server | Inventory add and remove. Edit the `custom` branch.                                |
| `GrantItemReturn(src)`                                  | Server | Registers one valid item return for a player. Called after a validated pickup.     |

### Database schema

None.

### Integration examples

#### ox\_inventory

Set `Config.Inventory = 'ox'` and create the item. The use action runs the command that places the boombox.

```lua
['boombox'] = {
    label = 'Boombox',
    weight = 2500,
    stack = false,
    close = true,
    client = { event = 'my_resource:useBoombox' }
},
```

```lua
RegisterNetEvent('my_resource:useBoombox', function()
    ExecuteCommand('useboombox')
end)
```

#### QBCore

Set `Config.Inventory = 'qb'`, add the item to `qb-core/shared/items.lua` and register it as usable.

```lua
-- server side
QBCore.Functions.CreateUseableItem('boombox', function(source)
    TriggerClientEvent('my_resource:useBoombox', source)
end)
```

#### ESX

Set `Config.Inventory = 'esx'`, add the item to the `items` table of your database and register it as usable.

```lua
-- server side
ESX.RegisterUsableItem('boombox', function(source)
    TriggerClientEvent('my_resource:useBoombox', source)
end)
```

#### Custom inventory

Set `Config.Inventory = 'custom'` and write your checks in the `custom` branch of the three functions.

```lua
-- client/functions.lua
elseif inv == 'custom' then
    return exports.my_inventory:HasItem(Config.BoomboxItem)
```

```lua
-- server/functions.lua
elseif inv == 'custom' then
    exports.my_inventory:AddItem(src, item, 1)
```

#### Custom notifications

Set `Config.NotifySystem = 'custom'` and write your call in the matching branch of `ShowNotification`.

```lua
elseif Config.NotifySystem == 'custom' then
    exports.my_notify:Send(message)
```

#### Restrict the boombox to a job

With `Config.Inventory = 'custom'` you can check anything you like before the boombox is placed.

```lua
elseif inv == 'custom' then
    local data = exports.my_jobs:GetPlayerJob()
    return data and data.name == 'dj'
```

***

## FAQ

**Can only the person who placed the boombox use it?** No. It is a community boombox by design. Anyone within range can control it.

**Who gets the item back?** The player who placed the boombox gets the item when they pick it up. A boombox picked up by another player is removed from the world.

**How many boomboxes can a player place?** One at a time. The player has to pick the first one up before placing another.

**Does it need a database?** No.

**Are boomboxes saved after a restart?** No. They are removed on restart and when their owner disconnects.

**Why does the progress bar match for everyone?** The start time of the track is stored as real time (`os.time() * 1000`) and every client calculates the position against the same clock.

**A link does not play.** Check that it points directly to an audio file or to a radio stream, that it uses HTTPS and that its domain is listed in `Config.AllowedHosts` when that list is not empty.

**A local file does not play.** Check that the file is inside `html/audio/`, that the extension is a supported format, that you typed the exact file name and that the resource was restarted after copying the file.

**The item is not returned.** Make sure `Config.Inventory` matches your inventory and that the item name matches `Config.BoomboxItem`. The item is returned when the player who placed the boombox picks it up in range.

**Can I use my own boombox model?** Yes. Set `Config.BoomboxProp` to the model name of any prop available on your server.

**Can I change the interaction keys?** The keys are `E` for the interface and `G` for pick-up. Only the labels shown in the hint are configurable, with `Config.InteractKeyLabel` and `Config.PickupKeyLabel`.
