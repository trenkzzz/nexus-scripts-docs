---
icon: bell
---

# Nexus Notify

> A notification system where every player places, scales and styles their own notifications, and any resource can send them with a single export from the client or the server.

**Version:** 3.1.1 · **Type:** Standalone (no framework, no database)

***

## Description

Nexus Notify is a standalone NUI notification stack. Each player runs `/notifymove`, drags the notification anchor anywhere on the screen, scales the stack between 50% and 200%, optionally switches to a compact pill style, and saves. These preferences are stored per player on their own machine using FiveM KVP, so they persist across reconnects and server restarts without any database.

Three kinds of notification are available through the same `Alert` export:

* **Timed notification.** Shows for a duration, with a progress bar in the full style.
* **Persistent notification.** Identified by a `persistentId`, stays on screen until you remove it, and updates in place when you call `Alert` again with the same id.
* **Button notification.** Renders clickable buttons that fire a client event in your own resource. Buttons can also be bound to keyboard keys.

Notification types (colors, icons, glow) are defined in `config.lua`. Adding a new type requires no code changes.

***

## Features

* Per-player position, saved as a percentage of the screen so it is resolution independent.
* Per-player scale from 50% to 200% through a slider.
* Two styles: full card (icon, title, message, progress bar) and minimalist pill (icon and message only). Players can switch between them when `AllowUserMinimalistToggle` is enabled.
* Five included types: `info`, `success`, `warning`, `error` and `admin`. Add as many as you want.
* Optional server logo shown in the icon bubble instead of the per-type icon.
* Per-type sounds with a global switch and a volume setting.
* Text markup: GTA color codes (`~r~`, `~g~`, `~b~`, `~y~`, `~o~`, `~p~`, closed with `~s~`), `**bold**`, `*italic*` and `~strike~`.
* Persistent notifications that update in place.
* Button notifications with mouse click and optional keyboard keys.
* Notifications always grow away from the screen edge, whichever corner the player chooses.
* Client and server exports with identical behavior.
* A position test command available to every player.
* Optional debug commands.

***

## Dependencies

| Dependency                       | Required | Notes                                                        |
| -------------------------------- | -------- | ------------------------------------------------------------ |
| Framework (ESX / QBCore / other) | No       | The resource never reads a player object, job or identifier. |
| Database / MySQL                 | No       | Player preferences are stored in KVP. There is no SQL file.  |
| `ox_lib` or any UI library       | No       | The NUI is self-contained.                                   |

The resource folder must be named exactly `nexus_notify`. The export namespace and the NUI callback URLs depend on that name, and the resource stops loading with a console message if the folder is renamed.

***

## Installation

1. Download the resource and place the `nexus_notify` folder in your `resources` directory.
2. Do not rename the folder.
3. Add it to `server.cfg` before every resource that sends notifications through it:

```cfg
ensure nexus_notify
ensure nexus_bounty
ensure nexus_multijob
```

4. There is no SQL to import.
5. Open `config.lua` and adjust the defaults for new players: `DefaultPosition`, `DefaultScale`, `MinimalistMode` and `DefaultDuration`.
6. Optional: to use your own logo, place it in `html/`, set `Config.LogoFileName` to its file name and set `Config.UseLogoInsteadOfIcon = true`.
7. Optional: to use your own sounds, place `.wav` or `.ogg` files in `html/sounds/` and point the entries of `Config.Sounds` at them. The manifest already includes everything inside `html/sounds/` and `html/icons/`.
8. Restart the server.
9. In game, run `/showpos` to display one notification of every configured type, then `/notifymove` to position the stack and press Save.

***

## Configuration

Everything is in `config.lua`, which is not encrypted and can be edited freely.

### Appearance and defaults

| Key                                | Type           | Default              | Description                                                                                                                                                                                                                       |
| ---------------------------------- | -------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.DefaultPosition`           | table `{x, y}` | `{ x = 99, y = 30 }` | Anchor used by players who never saved a position, as a percentage of screen width (`x`) and height (`y`). A `y` above 50 anchors the stack to the bottom half and new cards are inserted so they always grow away from the edge. |
| `Config.DefaultScale`              | number         | `1.0`                | Default scale. `1.0` is 100%. The in-game slider ranges from `0.5` to `2.0`.                                                                                                                                                      |
| `Config.UseLogoInsteadOfIcon`      | boolean        | `false`              | Shows `LogoFileName` in the icon bubble of every notification instead of the per-type icon.                                                                                                                                       |
| `Config.LogoFileName`              | string         | `'logo.png'`         | Logo file, relative to `html/`.                                                                                                                                                                                                   |
| `Config.EnableResetButton`         | boolean        | `true`               | Shows the Reset to Default button in the `/notifymove` panel.                                                                                                                                                                     |
| `Config.DefaultDuration`           | number (ms)    | `5000`               | Duration used when the caller does not pass one.                                                                                                                                                                                  |
| `Config.TestNotifyCommand`         | string         | `'showpos'`          | Command that shows one notification of every type, 600 ms apart. Written without the slash.                                                                                                                                       |
| `Config.MinimalistMode`            | boolean        | `false`              | Default style. `false` is the full card, `true` is the compact pill.                                                                                                                                                              |
| `Config.AllowUserMinimalistToggle` | boolean        | `true`               | Lets players choose their style in the `/notifymove` panel. When `false`, the toggle is hidden and every player uses `MinimalistMode`.                                                                                            |

### Sounds

| Key                    | Type    | Default   | Description                                                                                       |
| ---------------------- | ------- | --------- | ------------------------------------------------------------------------------------------------- |
| `Config.EnableSounds`  | boolean | `true`    | Global sound switch. Applies when the caller does not pass `playSound`.                           |
| `Config.DefaultVolume` | number  | `0.7`     | Playback volume from `0.0` to `1.0`.                                                              |
| `Config.Sounds`        | table   | see below | Maps a notification type to a file inside `html/sounds/`. A type without an entry plays no sound. |

```lua
Config.Sounds = {
    info    = 'opening-sound.wav',
    success = 'opening-sound.wav',
    warning = 'opening-sound.wav',
    error   = 'opening-sound.wav',
    admin   = 'opening-sound.wav',
}
```

### Button keybinds

| Key                           | Type    | Default     | Description                                                                                                                                         |
| ----------------------------- | ------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.EnableButtonKeybinds` | boolean | `true`      | Enables keyboard keys for notification buttons. When `false`, buttons can only be clicked with the mouse and the `key` field is ignored.            |
| `Config.ButtonKeyLabel`       | string  | `'%s (%s)'` | Format of the button text when a key is assigned. The first value is the label, the second is the key.                                              |
| `Config.DefaultButtonKeys`    | table   | `{}`        | Keys applied by button position when a button does not define its own `key`. Example: `{ 'F7', 'F8' }` gives the first button F7 and the second F8. |
| `Config.KeyMappings`          | table   | see file    | Maps key names to GTA control ids. Add entries to support more keys.                                                                                |

Available key names include letters, digits, `F1`, `F2`, `F3`, `F5` to `F11`, arrows (`UpArr`, `DownArr`, `LeftArr`, `RightArr`), numpad keys (`NUM1` to `NUM9`), `LShift`, `LAlt`, `ESC`, `TAB`, `ENTER`, `SPACE`, `BACKSPACE` and others. Key names are matched exactly as written in `Config.KeyMappings`, with an uppercase fallback.

### Styles

`Config.Styles` defines every notification type. The key is the value you pass as the `notificationType` argument. An unknown type falls back to `info`.

```lua
Config.Styles = {
    ['info'] = {
        iconFile   = 'icons/info.svg',
        color      = '#3b82f6',
        background = 'rgba(20, 20, 26, 0.97)',
        glow       = '0 0 15px rgba(59, 130, 246, 0.25)'
    },
}
```

| Field        | Description                                                                                                                                                                     |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `iconFile`   | Icon path relative to `html/`. Place custom icons in `html/icons/`. Ignored when `UseLogoInsteadOfIcon` is `true`.                                                              |
| `color`      | Accent color used for the border, icon bubble, title and progress bar. In the minimalist style the icon is drawn dark on the colored bubble, so choose a light or medium color. |
| `background` | Card background. `rgba` values are supported.                                                                                                                                   |
| `glow`       | Full CSS `box-shadow` value.                                                                                                                                                    |

Included types: `info` (`#3b82f6`), `success` (`#00FF00`), `warning` (`#f59e0b`), `error` (`#f43f5e`) and `admin` (`#9333ea`).

### Debug commands

| Key                             | Default               | Description                                                                                                    |
| ------------------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------- |
| `Config.EnableDebugCommands`    | `false`               | Registers the four commands below. They are available to every player, so keep this disabled on a live server. |
| `Config.DebugTestAllCommand`    | `'ndebug_all'`        | Shows one notification of every type.                                                                          |
| `Config.DebugPersistentCommand` | `'ndebug_persistent'` | Shows a persistent notification with id `debug_persistent`.                                                    |
| `Config.DebugPersistentHide`    | `'ndebug_hide'`       | Removes that persistent notification.                                                                          |
| `Config.DebugButtonsCommand`    | `'ndebug_buttons'`    | Shows a 15 second notification with Accept and Deny buttons bound to F7 and F8.                                |

***

## Player commands

| Command       | Description                                                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `/notifymove` | Opens the move panel. Drag the anchor, set the scale, choose the style if allowed, then press Save and Close. Running it again closes the panel. |
| `/showpos`    | Shows one notification of every type so the player can check the result. The name comes from `Config.TestNotifyCommand`.                         |

Player preferences are saved with the following KVP keys on the player's machine: `nexus_notify_pos_x`, `nexus_notify_pos_y`, `nexus_notify_pos_set`, `nexus_notify_scale` and `nexus_notify_minimalist` (`1` minimalist, `2` full, `0` or unset means the server default). Preferences are not stored on the server and do not follow the player to another computer.

***

## Locales

The resource has no locale system. Notification titles and messages are whatever your resources pass in. The move panel texts are in `html/index.html` and can be translated directly in that file.

***

## Compatibility

| System                       | Behavior                                                                                                                                                                                                          |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Frameworks                   | Works on ESX, QBCore, QBox or a bare server with no changes.                                                                                                                                                      |
| Other notification resources | Can coexist. Nexus Notify does not override `ESX.ShowNotification`, `QBCore.Functions.Notify` or native notifications. Resources keep using their current system until they call `exports['nexus_notify']:Alert`. |
| Nexus Scripts resources      | Select `nexus_notify` in the notification option of each resource's configuration to route its notifications here.                                                                                                |
| GTA color codes              | `~r~`, `~g~`, `~b~`, `~y~`, `~o~`, `~p~` and `~s~` are rendered by the NUI, so messages written for native notifications display correctly.                                                                       |

***

## Developer API

### Client exports

| Export                           | Parameters                                                              | Description                                                                           |
| -------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `exports['nexus_notify']:Alert`  | `title`, `text`, `duration`, `notificationType`, `playSound`, `options` | Shows a notification to the local player. Does nothing if `title` or `text` is empty. |
| `exports['nexus_notify']:Remove` | `persistentId`                                                          | Hides the persistent notification with that id. Does nothing if it is not visible.    |

| Argument           | Type        | Default                  | Notes                                                                                                                              |
| ------------------ | ----------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `title`            | string      | required                 | Shown in the accent color in the full style. Not shown in the minimalist style.                                                    |
| `text`             | string      | required                 | Message body. Supports the markup listed above. Shown on a single line and clipped with an ellipsis when too long.                 |
| `duration`         | number (ms) | `Config.DefaultDuration` | Use `0` together with `persistentId` for a notification that does not close by itself.                                             |
| `notificationType` | string      | `'info'`                 | Any key of `Config.Styles`.                                                                                                        |
| `playSound`        | boolean     | `Config.EnableSounds`    | `true` forces the sound, `false` mutes it, `nil` follows the configuration. The file comes from `Config.Sounds[notificationType]`. |
| `options`          | table       | `nil`                    | `{ persistentId = string, buttons = table }`                                                                                       |

**`options.persistentId`** makes the notification persistent. Calling `Alert` again with the same id updates the title and message of the visible card instead of creating another one. Persistent notifications have no progress bar.

**`options.buttons`** is an array of buttons. Your table is not modified.

| Field    | Type   | Description                                                                                     |
| -------- | ------ | ----------------------------------------------------------------------------------------------- |
| `label`  | string | Button text.                                                                                    |
| `key`    | string | Optional keyboard key, for example `'F7'`. The label is formatted with `Config.ButtonKeyLabel`. |
| `event`  | string | Client event triggered with `TriggerEvent` when the button is clicked or its key is pressed.    |
| `params` | any    | Single value passed as the argument of that event.                                              |

Behavior of button notifications:

* They always use the full style, even for players in minimalist mode.
* Clicking a button triggers its event and closes the notification.
* When a button has a key, pressing the key triggers the event and closes the notification. If two buttons share the same key, only the first one fires.
* If the notification has keys but no `persistentId`, an internal id is generated and the notification is removed when its duration ends.
* If you pass a `persistentId` together with keys, your id is kept and `Remove(id)` works normally.

```lua
exports['nexus_notify']:Alert('Garage', 'Vehicle stored.', 4000, 'success', true)

exports['nexus_notify']:Alert('Bank', 'You received ~g~$1,250~s~ from **payroll**.', 6000, 'info')

exports['nexus_notify']:Alert('Delivery', 'Packages left: 4', 0, 'info', false, {
    persistentId = 'delivery_progress'
})
exports['nexus_notify']:Alert('Delivery', 'Packages left: 3', 0, 'info', false, {
    persistentId = 'delivery_progress'
})
exports['nexus_notify']:Remove('delivery_progress')

exports['nexus_notify']:Alert('Invite', 'Join the crew?', 15000, 'info', true, {
    buttons = {
        { label = 'Accept', key = 'F7', event = 'mycrew:accept', params = { id = 12 } },
        { label = 'Deny',   key = 'F8', event = 'mycrew:deny',   params = { id = 12 } },
    }
})
```

### Server exports

The same exports exist on the server with `target` as the first argument.

| Export                           | Parameters                                                                        | Description                                               |
| -------------------------------- | --------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `exports['nexus_notify']:Alert`  | `target`, `title`, `text`, `duration`, `notificationType`, `playSound`, `options` | Sends the notification to the target player.              |
| `exports['nexus_notify']:Remove` | `target`, `persistentId`                                                          | Removes a persistent notification from the target player. |

| `target`                      | Effect                                                              |
| ----------------------------- | ------------------------------------------------------------------- |
| A player's server id          | Sends to that player.                                               |
| `-1`                          | Sends to all connected players.                                     |
| An invalid or disconnected id | Nothing is sent and an error line is printed in the server console. |

```lua
exports['nexus_notify']:Alert(source, 'Shop', 'Purchase complete.', 4000, 'success', true)

exports['nexus_notify']:Alert(-1, 'Server', 'Restart in ~r~5 minutes~s~.', 10000, 'warning', true)

exports['nexus_notify']:Alert(source, 'Jail', 'Time left: 10 min', 0, 'error', false, {
    persistentId = 'jail_timer'
})
exports['nexus_notify']:Remove(source, 'jail_timer')

exports['nexus_notify']:Alert(source, 'Admin', 'Report assigned to you.', 20000, 'admin', true, {
    buttons = {
        { label = 'Teleport', key = 'F7', event = 'myadmin:tpToReport', params = reportId },
    }
})
```

Button events always run on the client of the player who received the notification.

### Events

| Event                        | Side               | Payload                                           | Description                                                                                                         |
| ---------------------------- | ------------------ | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `nexus_notify:client:Alert`  | Client (net event) | `title, text, duration, type, playSound, options` | Same effect as the client `Alert` export. Used by the server export and can be triggered with `TriggerClientEvent`. |
| `nexus_notify:client:Remove` | Client (net event) | `persistentId`                                    | Same effect as the client `Remove` export.                                                                          |

The resource registers no server events, so a client cannot use it to show notifications to other players.

### Adding a notification type

```lua
-- config.lua
['heist'] = {
    iconFile   = 'icons/heist.svg',
    color      = '#facc15',
    background = 'rgba(26, 20, 6, 0.97)',
    glow       = '0 0 18px rgba(250, 204, 21, 0.35)'
},

-- optional sound
Config.Sounds.heist = 'alarm.wav'
```

```lua
exports['nexus_notify']:Alert('Heist', 'Vault drilling started.', 8000, 'heist', true)
```

### Integration example

A delivery resource that uses a persistent notification as a progress display, offers a bonus with buttons, and falls back to the native notification if Nexus Notify is not running.

```lua
-- client
local remaining = 5
local HUD_ID = 'mydelivery_hud'

local function notify(title, text, duration, nType, sound, options)
    if GetResourceState('nexus_notify') == 'started' then
        exports['nexus_notify']:Alert(title, text, duration, nType, sound, options)
    else
        BeginTextCommandThefeedPost('STRING')
        AddTextComponentSubstringPlayerName(text)
        EndTextCommandThefeedPostTicker(false, false)
    end
end

RegisterNetEvent('mydelivery:packageDelivered', function()
    remaining = remaining - 1
    if remaining > 0 then
        notify('Delivery Route', ('Packages left: ~y~%d~s~'):format(remaining), 0, 'info', false, {
            persistentId = HUD_ID
        })
    else
        exports['nexus_notify']:Remove(HUD_ID)
        notify('Delivery Route', 'Route complete.', 6000, 'success', true)
    end
end)

RegisterNetEvent('mydelivery:offerBonus', function(bonusId)
    notify('Bonus Run', 'A rush order is available. ~g~+$500~s~', 20000, 'warning', true, {
        buttons = {
            { label = 'Take it', key = 'F7', event = 'mydelivery:acceptBonus',  params = bonusId },
            { label = 'Pass',    key = 'F8', event = 'mydelivery:declineBonus', params = bonusId },
        }
    })
end)

AddEventHandler('mydelivery:acceptBonus', function(bonusId)
    TriggerServerEvent('mydelivery:claimBonus', bonusId)
end)
```

```lua
-- server
RegisterNetEvent('mydelivery:claimBonus', function(bonusId)
    local src = source
    exports['nexus_notify']:Alert(src, 'Bonus Run', 'Bonus paid: ~g~$500~s~', 5000, 'success', true)
end)
```

***

## FAQ

**Notifications appear in a corner I did not choose.** New players use `Config.DefaultPosition`. Players can run `/notifymove`, drag the anchor and press Save to set their own.

**My notification titles are not visible.** The minimalist style hides the title and shows only the icon and message. Put essential information in `text` if your server uses `MinimalistMode = true`.

**Long messages are cut off.** Messages are shown on a single line and clipped with an ellipsis. Split long content across several notifications or use a persistent notification that you update.

**A player turned minimalist mode off but it returns after reconnecting.** When `AllowUserMinimalistToggle` is `false`, the server value of `MinimalistMode` always wins. Set it to `true` to let players choose.

**No sound plays for one of my types.** The type needs an entry in `Config.Sounds` and the file must exist in `html/sounds/`. `playSound = true` does not play anything for a type without a sound entry.

**The volume is too loud or too quiet.** Change `Config.DefaultVolume` between `0.0` and `1.0`.

**The button keys do nothing.** Check that `Config.EnableButtonKeybinds` is `true`, that the key name exists in `Config.KeyMappings`, and that the notification is still visible. Keys only work while the notification is on screen.

**I want every button to have a key without setting it in each call.** Fill `Config.DefaultButtonKeys`, for example `{ 'F7', 'F8' }`. A key defined directly on a button takes priority.

**The move panel, reset button or buttons stopped working after renaming the folder.** The folder must be named `nexus_notify`. Rename it back.

**Do I have to replace every `ESX.ShowNotification` call?** No. Nexus Notify adds a parallel API and does not intercept framework notifications. Replace calls resource by resource with `exports['nexus_notify']:Alert`.

**Can I remove a persistent notification from every player at once?** Yes: `exports['nexus_notify']:Remove(-1, 'my_id')`.

**Is anything stored on the server?** No. Player layout preferences live in KVP on the player's own computer.

### Before opening a ticket

* Confirm the folder is named exactly `nexus_notify`.
* Confirm you are on the latest version.
* Confirm `nexus_notify` starts before the resource that calls it and that `GetResourceState('nexus_notify')` returns `started`.
* Run `/showpos`. If you see one notification per type, the resource works and the issue is in the calling resource.
* If a type renders incorrectly, check that its `Config.Styles` entry has all four fields and that the icon exists in `html/icons/`.
