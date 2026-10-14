---
icon: hand-pointer
---

# Nexus TextUI

> A point-based 3D world-anchored TextUI. Register a point once and the resource handles distance, the indicator, the prompt and the key press for you.

**Version:** 1.0.1 · **Type:** Standalone / client-only

***

## Description

Nexus TextUI is not a plain show and hide helper. It is a **point registry**. You register an interaction point once, with world coordinates, two distances, a prompt, a key and two optional callbacks, and from that moment the resource owns the entire lifecycle: it measures the distance to the player, decides whether to show nothing, a floating diamond indicator or the full prompt, projects the world coordinate onto the screen every frame, scales the element down as the player gets further away, listens for the key press and calls your handler. You never write a render loop, never call a show or hide function and never poll a distance yourself.

That design is the main difference from conventional TextUI resources. The usual pattern (`TextUI:Open(text)` inside your own distance loop and `TextUI:Close()` when the player walks away) puts the per-frame cost in every resource that uses it, and every one of them re-implements the same logic slightly differently. Here all of that cost is centralised in two threads inside `nexus_TextUI`, no matter how many resources and how many points are registered. A resource that uses it typically calls `Create` once at start and `Delete` once on stop, and does nothing in between.

Internally there are two cooperating threads with very different duty cycles:

* The **state thread** runs every 250 ms. It walks every registered point, computes the distance to the player, evaluates your `canInteract` predicate and decides which points are in view range and whether each one renders as an `indicator` (inside `viewDistance` but outside `interactionDistance`) or as `text` (inside `interactionDistance`). Points that dropped out of range since the previous pass are removed.
* The **render thread** runs at frame rate, but only while at least one point is visible. Otherwise it sleeps 200 ms per iteration. For each visible point it projects the world position to the screen, computes a scale between `0.6` and `1.0` across the gap between `viewDistance` and `interactionDistance`, polls the key when the player is inside `interactionDistance` and only sends a NUI message when a coordinate or the scale changed by more than 0.01. All changed points go in a single batched message, so a standing player generates no NUI traffic.

The interface keeps one DOM element per point and only repositions it. The visual is a two-box prompt: a square key badge and a separate rounded text box beside it, both in the Nexus purple with a looping shimmer sweep, which cross-fades into a pulsing diamond indicator when the player moves away past `interactionDistance`.

***

## Features

* **Point-based registration**: `Create(config)` registers a world point and the resource owns distance, visibility, rendering and input from then on.
* **Two-stage proximity rendering**: outside `interactionDistance` but inside `viewDistance`, the point renders as a pulsing purple diamond so players can see that something is there. Crossing `interactionDistance` cross-fades it into the full key and text prompt.
* **Distance-based scaling**: in the indicator band the element is scaled between `0.6` at `viewDistance` and `1.0` at `interactionDistance`, so distant points look distant.
* **Built-in key handling**: the key named in `config.key` is polled only while the player is inside `interactionDistance`, and your `onInteract` callback runs on press. You register no keybind and no control loop.
* **Conditional visibility (`canInteract`)**: an optional predicate re-evaluated every 250 ms. Return `false` and the point disappears entirely, indicator included, until it returns `true` again. Typical uses are job checks, not being in a vehicle, inventory checks and opening hours.
* **Separate key and text boxes**: a fixed 36 by 36 key badge and an auto-width text box, 8 px apart, each with its own entrance animation and shimmer.
* **Automatic key extraction**: the resource reads a `[KEY] Description` pattern from `config.text` and fills the key badge with the bracketed part. When the text has no brackets, the badge uses `config.key` and the text box shows the text as it is.
* **Change-gated NUI traffic**: positions are only pushed when something moved by more than 0.01 percent of the screen.
* **Adaptive render loop**: the frame-rate thread sleeps 200 ms whenever no point is visible.
* **Off-screen culling**: points behind the camera are parked outside the screen with scale `0` instead of being drawn in a wrong position.
* **Unlimited simultaneous points** from any number of resources, each with its own distances, key, text and predicate.
* **Fully standalone**: no framework, no database, no `ox_lib`, no target resource. It is a pure client-side resource.
* **Optional test commands**: with `Config.EnableTestCommands = true`, `/testui` creates a working demo point in front of you and `/deltestui` removes it.
* **Smooth transitions**: opacity fade, springy scale-in on each box, slide-in entrance and a short grace period on removal so elements fade out instead of popping.

***

## Dependencies

| Dependency                           | Required | Notes                                                                                                                                             |
| ------------------------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Framework (ESX / QBCore)             | No       | Never referenced. Works on ESX, QBCore, QBox, vRP or a bare server.                                                                               |
| Database / MySQL                     | No       | No persistence of any kind and no `.sql` file.                                                                                                    |
| `ox_lib` / `qb-target` / `ox_target` | No       | This resource covers the prompt layer for point-based interactions. It does not build on a target system.                                         |
| Server-side script                   | None     | The resource is client-only. All registration happens on the client.                                                                              |
| Internet access on the client        | Optional | `html/index.html` loads the Inter font from Google Fonts. Without internet the prompt uses the generic sans-serif font and everything else works. |

The resource folder **must** be named exactly `nexus_TextUI` (capital `UI`). `client/_resource.lua` stops the resource with an error if the name is different, because the export namespace `exports['nexus_TextUI']` depends on it.

***

## Installation

1. Download the resource and place the folder in your `resources` directory.
2. Keep the exact folder name `nexus_TextUI`, with capital `U` and capital `I`. On a case-sensitive server, `nexus_textui` fails the name check and the resource does not start.
3. There is nothing to import. No SQL is needed.
4.  Add it to `server.cfg` **before** every resource that registers points. The export must exist when another resource calls `Create` at start:

    ```cfg
    ensure nexus_TextUI
    ensure nexus_multijob
    ensure my_shops
    ```
5. Restart the server.
6. Optional check: set `Config.EnableTestCommands = true` in `config.lua`, restart the resource and run `/testui` in game. A prompt appears about 2 m in front of you. Walk away and it turns into a pulsing diamond, then disappears past 5 m. Press `E` when close to get the confirmation message. Get into a vehicle and it disappears. Run `/deltestui` to remove it.
7. Register your own points with `exports['nexus_TextUI']:Create({ ... })` from any client script. Behaviour is configured per point, in the `Create` call.

***

## Configuration

### `config.lua`

The only server-wide option is in `config.lua`, which is not encrypted:

| Option                      | Default | Description                                                                            |
| --------------------------- | ------- | -------------------------------------------------------------------------------------- |
| `Config.EnableTestCommands` | `false` | Registers the `/testui` and `/deltestui` demo commands. Keep it `false` in production. |

### Point configuration

Every other tunable is a property of the individual interaction point, passed to `Create(config)`:

| Key                   | Type      | Default | Description                                                                                                                                                                                                                                           |
| --------------------- | --------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                  | string    | none    | **Required.** Unique identifier of the point. Used as the registry key, as the DOM id in the interface and as the argument of `Delete`. Without it, `Create` prints an error and registers nothing. Using an existing id replaces the previous point. |
| `coords`              | `vector3` | none    | **Required.** World position the prompt is anchored to. It must be a `vector3`, not a table of three numbers.                                                                                                                                         |
| `viewDistance`        | number    | none    | **Required.** Outer radius in meters. Beyond it nothing is drawn. Between `interactionDistance` and `viewDistance` the point renders as a diamond indicator.                                                                                          |
| `interactionDistance` | number    | none    | **Required.** Inner radius in meters. Inside it the point renders the full prompt and the key is polled. It should be smaller than `viewDistance`.                                                                                                    |
| `text`                | string    | none    | **Required.** The prompt. Write it as `'[E] Open the garage'`: the bracketed part fills the square key badge and the rest is the description. Without brackets, the badge uses `config.key` and the text box shows the whole string.                  |
| `key`                 | string    | none    | **Required for interaction.** Key that triggers `onInteract`, written as a name from the key table below. Names are uppercase and case-sensitive. An unknown name prints an error in the console when the point is created and the key does nothing.  |
| `onInteract`          | function  | `nil`   | Called without arguments when the player presses `key` inside `interactionDistance`. Leave it out for a purely informational prompt.                                                                                                                  |
| `canInteract`         | function  | `nil`   | Optional predicate evaluated every 250 ms. Return `false` to hide the point completely, return `true` (or leave the field out) to show it. It takes no arguments.                                                                                     |

The resource writes two helper fields into your table. Do not set them yourself: `formattedText` (the key badge and text box contents) and `lastScreenState` (cache used to skip unchanged frames).

### Key names

```
A B C D E F G H K L M N P Q R S T U V W X Y Z
1 2 3 4 5 6 7 8 9  -  =  ,  .  [  ]  ~
F1 F2 F3 F5 F6 F7 F8 F9 F10 F11
UpArr DownArr LeftArr RightArr   LEFT RIGHT TOP DOWN
SPACE ENTER TAB BACKSPACE DELETE CAPS ESC
LShift LEFTSHIFT LAlt LEFTALT LEFTCTRL RIGHTCTRL
HOME PAGEUP PAGEDOWN
NUM1 to NUM9   N4 N5 N6 N7 N8 N9 N+ N- NENTER
```

There is no `F4`, no `F12`, no `I`, `J` or `O` and no `0` in the table.

### Styling

The visual identity is changed by editing `html/style.css`, which is never encrypted:

| What                   | Where                                                        | Current value           |
| ---------------------- | ------------------------------------------------------------ | ----------------------- |
| Prompt and badge color | `.key-box`, `.text-box` `background-color`                   | `#a958ff`               |
| Indicator color        | `.indicator-wrapper::before` `background-color` and `border` | `#a958ff` and `#8642c9` |
| Key badge size         | `.key-box` `width` and `height`                              | `36px` by `36px`        |
| Text box font size     | `.text-box` `font-size`                                      | `0.85em`                |
| Gap between boxes      | `.text-wrapper` `gap`                                        | `8px`                   |
| Indicator pulse speed  | `@keyframes pulse`                                           | `2s`                    |
| Shimmer sweep speed    | `@keyframes shimmer`                                         | `3s`                    |
| Fade in and out speed  | `.interaction-point` `transition`                            | `0.3s`                  |
| Font                   | `body` `font-family`                                         | `Inter`                 |

***

## Locales and editable strings

Nexus TextUI has no locale system. The text a player reads is the `text` you pass to `Create` from your own resource, so it is already in the language of that resource. Pass your own translated string, for example `text = ('[E] %s'):format(T('open_garage'))`.

| Where                                 | Strings                                               | Editable                                 |
| ------------------------------------- | ----------------------------------------------------- | ---------------------------------------- |
| `Create` call of the calling resource | Every prompt the player sees.                         | Yes, it belongs to the calling resource. |
| `html/style.css`, `html/index.html`   | No user-facing copy.                                  | Yes.                                     |
| `config.lua`                          | No strings.                                           | Yes.                                     |
| `client/main.lua`                     | Console error messages and the demo command messages. | No, the file is encrypted.               |

***

## Compatibility

| System                                                                  | How it integrates                                                                                                                                                                                                          |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Frameworks                                                              | None required. Job or grade checks are done by the consumer inside its own `canInteract` predicate, for example `canInteract = function() return PlayerData.job and PlayerData.job.name == 'police' end`.                  |
| Databases                                                               | None.                                                                                                                                                                                                                      |
| Notification systems                                                    | Not used. The demo commands use the plain GTA notification natives.                                                                                                                                                        |
| Target resources (`ox_target`, `qb-target`)                             | Can coexist. Use a target resource for interactions tied to entities and Nexus TextUI for interactions tied to fixed world coordinates. They draw independent interfaces.                                                  |
| Other TextUI resources (`ox_lib` TextUI, `esx_textui`, `cd_drawtextui`) | Can coexist. Both draw if you register the same interaction in both. Nexus TextUI exposes `Create` and `Delete` instead of `Open` and `Close`, so it is not a direct drop-in replacement. See the migration example below. |
| Nexus Scripts catalogue                                                 | Nexus resources with a TextUI option in their config, such as `nexus_boombox`, call `exports['nexus_TextUI']:Create` and `:Delete` from that branch.                                                                       |

Escrow: `config.lua` and the `html/` files are open and editable. `client/main.lua` and `client/_resource.lua` are encrypted.

***

## Developer API

### Exports

Both exports are client-side. The resource has no server script, so there are no server exports.

| Export                           | Parameters       | Returns | Description                                                                                                                                                                                                                   |
| -------------------------------- | ---------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exports['nexus_TextUI']:Create` | `config` (table) | nothing | Registers an interaction point. Requires `config.id`. It parses `config.text`, stores the point and, if the point is already visible, applies the new content. Nothing is drawn until the player walks inside `viewDistance`. |
| `exports['nexus_TextUI']:Delete` | `id` (string)    | nothing | Unregisters the point and removes its element from the screen. It does nothing if the id is not registered.                                                                                                                   |

```lua
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

exports['nexus_TextUI']:Delete('garage_pillbox')
```

**Return value**: neither export returns anything.

**Overwrite semantics**: calling `Create` again with an id that already exists replaces the stored point. This is the supported way to change the text, distances, key or predicate of a point. There is no separate update export.

**Creating a point from the server**: the resource has no server exports. Trigger your own client event and call the export there:

```lua
-- server
TriggerClientEvent('myresource:registerPoint', src, pointData)

-- client
RegisterNetEvent('myresource:registerPoint', function(data)
    exports['nexus_TextUI']:Create({
        id = data.id,
        coords = vector3(data.x, data.y, data.z),
        viewDistance = 10.0,
        interactionDistance = 2.0,
        text = '[E] ' .. data.label,
        key = 'E',
        onInteract = function() TriggerServerEvent('myresource:use', data.id) end,
    })
end)
```

### Events - Emitted

The resource emits no Lua events. All communication back to your code goes through the `onInteract` and `canInteract` callbacks, which are called directly as Lua functions. For an event-based integration, raise the event inside your own callback:

```lua
onInteract = function()
    TriggerEvent('myresource:textuiUsed', 'garage_pillbox')
    TriggerServerEvent('myresource:logInteraction', 'garage_pillbox')
end
```

### Events - Listened

The resource registers no network events and no gameplay event handlers. It is driven only by its exports, so another client or script cannot make a prompt appear on your screen through this resource.

### Commands

Registered only when `Config.EnableTestCommands = true`.

| Command      | Purpose                                                                                                                                                                                                                            |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/testui`    | Registers the demo point `test_interaction` 2 m in front of the player: `viewDistance = 5.0`, `interactionDistance = 2.0`, key `E`, text `TEST POINT`, hidden while the player is in a vehicle. Pressing `E` shows a confirmation. |
| `/deltestui` | Deletes `test_interaction`.                                                                                                                                                                                                        |

### NUI messages (Lua to JavaScript)

For anyone adapting `html/`.

| Action            | Payload                                        | Sent by                                                      | Effect                                                                                                                                                                                                                                                                                                 |
| ----------------- | ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `updateContent`   | `{ item = { id, type, content? } }`            | The 250 ms state thread, once per in-range point per pass    | Creates the element on first sight, writes the key and text when `type` is `'text'` and switches between the indicator and the prompt. `type` is `'text'` inside `interactionDistance` and `'indicator'` outside it. Repeated messages for an element that is already visible do not restart its fade. |
| `updatePositions` | `{ positions = { { id, x, y, scale }, ... } }` | The render thread, only when something changed               | Sets left, top and scale of each listed element.                                                                                                                                                                                                                                                       |
| `remove`          | `{ id = id }`                                  | `Delete` and the state thread for points that left the range | Fades the element out and removes it after 300 ms. If the point comes back in range before that, the removal is cancelled and the element stays.                                                                                                                                                       |

### Editable functions

There is no `functions.lua`. There is no external system to branch on, so the customisation surfaces are:

1. **Per point**, at call time: `text`, `key`, the two distances, `onInteract` and `canInteract`.
2. **`html/style.css`**: colors, sizes, animation timings and font.

### Database schema

None. The resource creates no tables and persists nothing. Every registered point lives only in the memory of the client and is gone on resource restart or reconnect, so register points from a start-up thread or on player load.

### Integration examples

#### Job-gated shop points

Registers on player load, re-registers after a restart of nexus\_TextUI, respects opening hours and cleans up when the resource stops.

```lua
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
            viewDistance        = 15.0,
            interactionDistance = 1.8,
            text                = ('[E] %s'):format(p.label),
            key                 = 'E',
            canInteract         = function()
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

AddEventHandler('playerSpawned', registerPoints)
RegisterNetEvent('esx:playerLoaded', registerPoints)
RegisterNetEvent('QBCore:Client:OnPlayerLoaded', registerPoints)

CreateThread(function()
    while not hasTextUI() do Wait(500) end
    registerPoints()
end)

AddEventHandler('onClientResourceStop', function(res)
    if res == GetCurrentResourceName() then unregisterPoints() end
end)

AddEventHandler('onClientResourceStart', function(res)
    if res == 'nexus_TextUI' then
        registered = false
        Wait(500)
        registerPoints()
    end
end)
```

#### Dynamic text

There is no update export. Call `Create` again with the same id:

```lua
local function setShopPrompt(id, coords, label)
    exports['nexus_TextUI']:Create({
        id = id,
        coords = coords,
        viewDistance = 15.0,
        interactionDistance = 1.8,
        text = ('[E] %s'):format(label),
        key = 'E',
        onInteract = function() TriggerServerEvent('myshop:open', id) end,
    })
end

setShopPrompt('ammunation_1', POINTS[1].coords, 'Ammunation - Closed')
```

#### Migrating from an Open and Close TextUI

The two models are different: this resource owns the distance check, so there is nothing to open. The migration is to delete your distance loop and let the point own it.

```lua
-- before
CreateThread(function()
    while true do
        Wait(5)
        if #(GetEntityCoords(PlayerPedId()) - shopCoords) < 2.0 then
            TextUI:Open('[E] Shop')
            if IsControlJustPressed(0, 38) then openShop() end
        else
            TextUI:Close()
        end
    end
end)

-- after
exports['nexus_TextUI']:Create({
    id = 'shop',
    coords = shopCoords,
    viewDistance = 12.0,
    interactionDistance = 2.0,
    text = '[E] Shop',
    key = 'E',
    onInteract = openShop,
})
```

#### Visibility rules with canInteract

```lua
canInteract = function()
    local job = ESX.GetPlayerData().job
    return job ~= nil and job.name == 'mechanic'
end
```

#### Hide a point behind walls

The prompt is drawn without an occlusion check. Add your own line-of-sight test when needed:

```lua
canInteract = function()
    local ped = PlayerPedId()
    local p = GetEntityCoords(ped)
    local ray = StartShapeTestLosProbe(p.x, p.y, p.z + 0.6, pointCoords.x, pointCoords.y, pointCoords.z, 1, ped, 4)
    local _, hit = GetShapeTestResult(ray)
    return hit == 0
end
```

***

## FAQ

**I called `Open()` or `Close()` and got a nil error.** Those functions do not exist. This resource exposes `Create(config)` and `Delete(id)`. See the migration example: register the point once and delete your distance loop.

**What does the key badge show?** The bracketed part of `text`. With `text = '[E] Open the garage'` the badge shows `E` and the text box shows `Open the garage`. Without brackets, the badge shows `config.key` and the text box shows the whole text.

**Nothing appears at all.** Check in order: the folder is named exactly `nexus_TextUI`; `coords` is a real `vector3` and not `{x, y, z}`; you are inside `viewDistance`; your `canInteract` is not returning `false` (a predicate that returns nothing returns `nil`, which counts as false, so always `return true`); and `nexus_TextUI` starts before the resource that calls `Create`.

**The prompt shows but pressing the key does nothing.** `config.key` must be an uppercase name from the key table, for example `'E'` and not `'e'` or a numeric control id. An unknown name prints an error in the console when the point is created. Also confirm that you passed `onInteract`. Without it the prompt is informational. The key is only read inside `interactionDistance`.

**My points vanished after I restarted a resource.** Registrations live only in the memory of the client. If your resource restarts, register again on start. If `nexus_TextUI` restarts, every consumer must register again. Hook `onClientResourceStart` for `'nexus_TextUI'` as in the integration example.

**Can `onInteract` run once after I delete a point?** `Delete` removes the point from the registry immediately, so its callback is not called afterwards.

**`canInteract` reacts with a small delay.** It is evaluated on the 250 ms state thread, so a change can take up to a quarter of a second to hide or show the point. Keep the predicate cheap, because it runs for every registered point in range four times a second.

**Can two resources register the same `id`?** They can, and the second one replaces the first, including its `onInteract`. Prefix your ids with your resource name, for example `myshop:ammunation_1`.

**How many points can I register?** There is no hard limit. The state thread depends on the number of registered points and the render thread on the number of visible points, so a few hundred registered points with a handful visible is comfortable.

**Can I change the purple color or the box sizes?** Yes. Edit `html/style.css`. There is no config key for it.

**Does it draw through walls?** Yes. There is no occlusion check, so a point behind a wall is visible when the player is inside `viewDistance` and facing it. Add your own test in `canInteract` if that matters.

**Can I use it without a framework?** Yes. The resource never references ESX, QBCore or any other framework.

**The font looks different.** The Inter font is loaded from Google Fonts. Without internet access on the client, the generic sans-serif font is used. To ship the font with your server, add the font file to `html/` and point `font-family` in `html/style.css` to it.

### Before opening a ticket

* Make sure the resource name is exactly `nexus_TextUI`.
* Make sure you are using the latest version of the resource (**1.0.1**).
* Set `Config.EnableTestCommands = true`, restart the resource and run `/testui`. If the demo point works, the resource is fine and the problem is in the `Create` call of the calling resource: `coords` must be a `vector3`, `key` an uppercase name from the key table and `canInteract` must return `true` explicitly.
* Confirm that `nexus_TextUI` starts before the resources that register points and that `GetResourceState('nexus_TextUI') == 'started'` when you call `Create`.
* Check the F8 console for a Lua error in your own `onInteract` or `canInteract` callback.
