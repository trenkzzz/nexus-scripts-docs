# Nexus Deathscreen

A two-stage death and unconscious screen with a free-orbit death camera, an animated ECG vitals panel, and a medically-worded cause-of-death line generated from the actual weapon and body part that killed you.

## 📝 Description

Nexus Deathscreen replaces the default `~r~WASTED` overlay with two distinct, purpose-built screens. The **unconscious** screen is what a downed-but-savable player sees: a pulsing heart whose beat rate slows as the countdown runs out, a scrolling three-segment ECG waveform with a live BPM readout that degrades toward flatline, a MIN:SEC countdown with a progress bar, the street name where the player went down, and a paramedic-call block with a cooldown bar and a dot counter for the remaining calls. The **death** screen is the point of no return: a skull, a `WASTED` title, the cause-of-death line, and a two-phase respawn timer.

The death phase model is the core of the script. Phase 1 is dead time the player cannot escape — a countdown labelled "waiting for medical response" with a vertical drain bar and a spinner, during which no respawn is possible, only another medic call. When phase 1 expires the script switches to phase 2: the respawn panel flies in from the left edge, a second countdown starts, and the respawn key becomes live. If the player does nothing, phase 2's expiry forces the respawn automatically. Both durations, and the unconscious duration before them, are independent config values so you can line them up with whatever your ambulance job does.

The cause-of-death line is not a random string. On death the client reads `GetPedCauseOfDeath` and `GetPedLastDamageBone`, maps the weapon hash to a damage category (pistol, SMG, rifle, sniper, shotgun, explosive, fire, heavy, blade, blunt, fist, explosion, vehicle, fall, water, electric) and the bone to a body zone (head, neck, chest, abdomen, shoulder, arm, hand, hip, leg, foot), then looks up the matching clinical phrase in the active locale. A head shot from a pistol reads *"Gunshot to the head — instant death"*; the same pistol in the femoral artery reads *"Bullet to the leg — femoral artery severed"*. Every category carries a `_default` for unmapped zones, and an unmapped weapon falls back to a random line from a short generic list, so there is no path to an empty string.

Around all this sits a camera and a presentation layer. The death camera has three modes: an automatic slow orbit, a static offset shot, and a mouse-driven free orbit where the player rotates and pitches the camera around their own corpse and zooms with the scroll wheel, with a shape-test raycast that pulls the camera in front of any wall it would otherwise clip through. Three visual effect presets (`minimal`, `vignette`, `blackout`) change the intensity of the vignette, the blood overlay, the glitch layer, the red flash and the scanline. Framework coupling is deliberately thin: everything that touches your ambulance job lives in `shared/functions.lua`, outside the escrow, as six short override functions.

## ✨ Features

- **Two separate screens** — an unconscious/laststand screen and a death screen, each with its own layout, animation set and interaction model, driven by two public client events so your ambulance job decides which one applies.
- **Smooth unconscious → death transition** — when the unconscious timer expires the script sends a dedicated `switchToDead` message so the NUI cross-fades (400 ms out, then in) instead of flashing black between a hide and a show.
- **Animated ECG vitals panel** — three chained SVG paths scroll continuously; the waveform morphs from a full QRS complex to a nearly flat trace once the player enters the critical window, and the scroll speed is tied to the computed BPM.
- **Decaying heart rate** — BPM is derived from the remaining time (`25 + 50 × timeLeft/totalTime`) with a small sine jitter, and the same value drives the heart icon's pulse animation duration, the pulse ring and the ECG scroll speed. Below 30 seconds everything turns red and a `CRITICAL` label appears; below 15 seconds the BPM readout starts blinking.
- **Street-name context** — the unconscious screen reads `GetStreetNameAtCoord` and displays "waiting for paramedics in <street>", falling back to "Los Santos".
- **Medically-worded cause of death** — 16 weapon categories × up to 9 body zones of distinct clinical phrases per language, resolved from the real weapon hash and impact bone, with a `_default` per category and a 5-entry random fallback list.
- **Two-phase respawn timer** — phase 1 is a forced wait with a vertical drain bar and a dot spinner; phase 2 flies in from off-screen left, enables the respawn key and auto-respawns on expiry. Setting phase 1 to `0` starts phase 2 immediately.
- **Independent paramedic-call systems** — the unconscious screen and the death screen each have their own enable flag, maximum number of calls, cooldown and displayed key. Each shows a cooldown progress bar that drains in real time, a live "next call in Ns" countdown, and a row of dots that grey out as calls are spent, then a terminal "no calls left" state.
- **Free-orbit death camera** — `orbit_free` mode: mouse X and Y rotate and pitch the orbit around the corpse (clamped by `orbitMinPitch`/`orbitMaxPitch`), mouse wheel zooms between `orbitMinRadius` and `orbitMaxRadius`, and the camera always points at the ped. A `StartShapeTestRay` from the corpse to the camera position pulls the camera 0.12 m in front of any geometry it hits, so it does not end up inside a wall.
- **Two more camera modes** — `orbit` (automatic rotation at `orbitSpeed` degrees per frame) and `static` (a fixed offset shot). Blend-in and blend-out times are configurable, as is the FOV.
- **Three visual effect presets** — `minimal`, `vignette` and `blackout`, applied as a CSS class on the screen root. `blackout` adds pseudo-element noise and chromatic-shift layers and a flicker animation; `vignette` adds a pulsing vignette and a stronger blood overlay; `minimal` keeps the overlays subtle.
- **Configurable control blocking** — `none`, `keyboard`, `mouse` or `keyboard+mouse`, set independently for the unconscious and the dead state. A thread disables the relevant control list every frame while the screen is up.
- **HUD suppression** — while either screen is visible, `SetFrontendActive(false)` and `HideHudAndRadarThisFrame()` run every frame, so the minimap and HUD cannot bleed through.
- **Cause-agnostic entry points** — `nexus_deathscreen:client:unconscious` drives the full unconscious → death pipeline (the script owns the timer), while `nexus_deathscreen:client:death` jumps straight to the death screen for a script that wants to own the timing itself. The direct event is idempotent: if the death screen is already up it is ignored, so a duplicate event from your ambulance job cannot reset a screen that just opened.
- **Transition handshake** — at the moment it switches from unconscious to dead the script fires `nexus_deathscreen:client:onTransitionToDead` locally and `nexus_deathscreen:server:onDeath` on the server, so your ambulance job can sync its own internal "is dead" state to the exact instant Nexus decided.
- **Revive integration** — `nexus_deathscreen:client:revive` clears the timers, stops the camera and hides everything; `nexus_deathscreen:client:doSpawn(coords)` additionally resurrects the ped at given coordinates, restores health to 200 and clears blood damage.
- **Seven languages** — `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`, all with identical key sets including the full death-phrase tree, all outside the escrow.
- **Per-key locale fallback** — `_T` falls back to Spanish per missing key rather than per missing file, and wraps `string.format` in a `pcall` so a bad placeholder in a translation prints the raw string instead of erroring.
- **Debug commands** — with `Config.Debug = true`, `/deathtest [dead|unconscious]` opens either screen on demand and `/deathhide` closes it.
- **Resource-name guard** — `shared/_resource.lua` aborts the resource with a clear console error if the folder is not named `nexus_deathscreen`.
- **Thin framework coupling** — six override functions in `shared/functions.lua`, outside the escrow, are the only place the script talks to your ambulance job.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **es_extended (ESX)** | Conditional | Required when `Config.Framework = 'esx'`. The client resolves it with `exports['es_extended']:getSharedObject()` if `es_extended` is started, otherwise falls back to the legacy `esx:getSharedObject` event. |
| **qb-core (QBCore)** | Conditional | Required when `Config.Framework = 'qb'`. Resolved with `exports['qb-core']:GetCoreObject()`. |
| **An ambulance / EMS job** | **Required in practice** | The script draws the screens; it does not decide when a player is down, and it does not revive anyone. `shared/functions.lua` targets `esx_ambulancejob` (`esx_ambulancejob:onPlayerDistress`, `esx_ambulancejob:onRemoveItemsAfterRPDeath`) and `qb-ambulancejob` (`hospital:server:ambulanceAlert`, `hospital:client:Revive`, `hospital:client:RespawnAtHospital`) out of the box. |
| **Something that fires the entry events** | **Required** | Your ambulance job (or your own code) must trigger `nexus_deathscreen:client:unconscious` and/or `nexus_deathscreen:client:death`. Nothing in this resource hooks `baseevents` or polls player health. |
| **Database** | Not used | No SQL, no persistence, no oxmysql. |
| **Notification resource** | Not used | The script has no notification layer. |

**Nothing else.** No target system, no TextUI, no fuel, keys or banking. Internet access is used only to load the Bebas Neue font from Google Fonts in the NUI.

## ⚙️ Installation

1. **Extract the resource.** Download it from the cfx.re (Keymaster) portal and place the folder in `resources`. The folder **must** be named exactly `nexus_deathscreen` — `shared/_resource.lua` stops the resource otherwise.
2. **No database import.** This resource creates no tables and stores nothing.
3. **Order your `server.cfg`.** Your framework and your ambulance job should start before this resource:
   ```cfg
   ensure es_extended            # or qb-core
   ensure esx_ambulancejob       # or qb-ambulancejob
   ensure nexus_deathscreen
   ```
4. **Set the framework and language** in `shared/config.lua`:
   ```lua
   Config.Framework = 'esx'   -- 'esx' or 'qb'
   Config.Locale    = 'en'    -- es | en | de | fr | it | pt | zh
   ```
5. **Line the timers up with your ambulance job.** This is the step people skip, and it is the one that matters. `shared/functions.lua` documents the mapping at the top of the file:
   - `Config.UnconsciousTime` should equal `Config.ReviveInterval` in `qb-ambulancejob/config.lua`;
   - `Config.RespawnWaitTime` should equal `Config.DeathTime` in `qb-ambulancejob/config.lua`;
   - `Config.RespawnForceTime` is yours to choose (30 s by default) and runs *after* the wait time.
6. **Disable your ambulance job's own death UI.** `qb-ambulancejob` and `esx_ambulancejob` both draw their own text/overlay; leave them on and you will see two death screens at once.
7. **Trigger the screens.** Call the public events from your ambulance job:
   ```lua
   -- player went down but is savable
   TriggerEvent('nexus_deathscreen:client:unconscious')

   -- player is dead outright, no laststand
   TriggerEvent('nexus_deathscreen:client:death')

   -- a medic revived them
   TriggerEvent('nexus_deathscreen:client:revive')
   ```
   In `qb-ambulancejob`, `nexus_deathscreen:client:unconscious` belongs where `laststand.lua` puts the player into last stand, and `nexus_deathscreen:client:death` where the resource marks the player dead.
8. **Adjust the camera and effects.** Pick `Config.DeathCamera.mode` (`orbit_free` by default) and `Config.VisualEffect` (`minimal` by default).
9. **Test before going live.** Set `Config.Debug = true`, restart, then use `/deathtest unconscious` and `/deathtest dead` to verify timings, the camera and the key hints. Set it back to `false` for production.

## 🔧 Configuration

Everything lives in `shared/config.lua`, which ships outside the escrow.

### General

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Framework` | string | `'esx'` | `'qb'` for QBCore or `'esx'` for ESX. Selects which core object the client resolves, and which branch every function in `shared/functions.lua` takes. |
| `Config.Locale` | string | `'es'` | Active language: `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`. Falls back to `es` per missing key, so a partial translation still renders. |
| `Config.HelpKey` | string | `'g'` | **Display only.** Uppercased and used to render the `PRESS [%s] TO RESPAWN` hint and as the key the NUI's own keydown listener matches. The actual respawn binding is control 47 (`G`) in the client input loop and is not derived from this value — change this only if you have also changed the control, or the hint will lie. |

### Timers (all in seconds)

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.UnconsciousTime` | number | `15` | How long the unconscious screen lasts before the script transitions to the death screen. Match this to `Config.ReviveInterval` in `qb-ambulancejob`. |
| `Config.RespawnWaitTime` | number | `15` | Phase 1: dead time during which respawn is impossible. Match this to `Config.DeathTime` in `qb-ambulancejob`. Set to `0` to skip straight to phase 2. |
| `Config.RespawnForceTime` | number | `30` | Phase 2: how long the player may stay dead with the respawn key available before the respawn is forced. Set to `0` to disable the forced respawn (the player then stays dead until they press the key or a medic revives them). |

### Visual effect

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.VisualEffect` | string | `'minimal'` | `'minimal'`, `'vignette'` or `'blackout'`. Applied as the CSS class `fx-<value>` on the screen root. `minimal` keeps the overlays subtle; `vignette` adds a pulsing vignette, a stronger blood layer and a visible scanline; `blackout` adds noise and chromatic-shift pseudo-element layers plus a flicker animation and is the most aggressive. Also affects the unconscious screen's corner frame. |

### Control blocking

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Controls.unconscious` | string | `'none'` | Controls disabled while the unconscious screen is up: `'none'`, `'keyboard'`, `'mouse'` or `'keyboard+mouse'`. |
| `Config.Controls.dead` | string | `'none'` | Same, for the death screen. Note that with `orbit_free` the camera reads mouse input through `GetDisabledControlNormal`, so blocking the mouse does **not** break the free camera. |

Keyboard mode disables controls `30, 31, 22, 24, 25, 36, 44, 37, 23, 75` (movement, jump, attack, aim, duck, cover, weapon select, enter/exit vehicle). Mouse mode disables `1, 2, 24, 25` (look X/Y, attack, aim).

```lua
Config.Controls = {
    unconscious = 'none',
    dead        = 'none',
}
```

### Death camera

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.DeathCamera.enabled` | boolean | `true` | Master switch. When `false` no scripted camera is created and the default game camera stays. |
| `Config.DeathCamera.mode` | string | `'orbit_free'` | `'orbit'` → automatic rotation, no input. `'orbit_free'` → the player rotates/pitches with the mouse and zooms with the wheel; the camera always looks at the corpse. `'static'` → a fixed camera at `offset` looking at the corpse. |
| `Config.DeathCamera.blendInTime` | number (ms) | `1000` | Transition time when the scripted camera takes over. |
| `Config.DeathCamera.blendOutTime` | number (ms) | `400` | Transition time when the camera is released on respawn or revive. |
| `Config.DeathCamera.fov` | number (deg) | `80.0` | Field of view. |
| `Config.DeathCamera.orbitRadius` | number (m) | `2.8` | Initial orbit radius. Used by `orbit` and as the starting radius for `orbit_free`. |
| `Config.DeathCamera.orbitHeight` | number (m) | `1.8` | Initial height above the ped. Used by `orbit`; `orbit_free` derives height from the pitch angle after the first frame. |
| `Config.DeathCamera.orbitSpeed` | number (deg/frame) | `0.2` | **`orbit` only.** Rotation per frame. |
| `Config.DeathCamera.orbitMouseSens` | number | `7.0` | **`orbit_free` only.** Mouse sensitivity; higher is faster. |
| `Config.DeathCamera.orbitMinPitch` | number (deg) | `-30.0` | **`orbit_free` only.** Lowest vertical angle, below the horizon. |
| `Config.DeathCamera.orbitMaxPitch` | number (deg) | `75.0` | **`orbit_free` only.** Highest vertical angle, above the ped. |
| `Config.DeathCamera.orbitMinRadius` | number (m) | `1.2` | **`orbit_free` only.** Closest the camera may zoom to the corpse. |
| `Config.DeathCamera.orbitMaxRadius` | number (m) | `6.0` | **`orbit_free` only.** Furthest the camera may zoom out. |
| `Config.DeathCamera.orbitScrollSpeed` | number (m/tick) | `0.3` | **`orbit_free` only.** Zoom step per wheel notch. |
| `Config.DeathCamera.offset` | table `{x, y, z}` | `{ x = 0.0, y = -2.5, z = 1.8 }` | **`static` only.** Camera position relative to the corpse. |

```lua
Config.DeathCamera = {
    enabled = true,
    mode    = 'orbit_free',   -- 'orbit' | 'orbit_free' | 'static'

    blendInTime  = 1000,
    blendOutTime = 400,
    fov          = 80.0,

    -- shared by orbit / orbit_free
    orbitRadius = 2.8,
    orbitHeight = 1.8,

    -- 'orbit' only
    orbitSpeed = 0.2,

    -- 'orbit_free' only
    orbitMouseSens   = 7.0,
    orbitMinPitch    = -30.0,
    orbitMaxPitch    =  75.0,
    orbitMinRadius   =  1.2,
    orbitMaxRadius   =  6.0,
    orbitScrollSpeed =  0.3,

    -- 'static' only
    offset = { x = 0.0, y = -2.5, z = 1.8 },
}
```

`orbit_free` controls: move the mouse to rotate and pitch, wheel up to move closer, wheel down to move away.

### Paramedic call — unconscious screen

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.UnconsciousHelpCall.enabled` | boolean | `true` | When `false` the whole call block is hidden on the unconscious screen. |
| `Config.UnconsciousHelpCall.maxCalls` | number | `3` | Maximum calls per down. `0` means unlimited (the dot counter is hidden and the "no calls left" state is never reached). |
| `Config.UnconsciousHelpCall.cooldown` | number (s) | `20` | Time between calls. The NUI draws a draining bar and a live countdown for exactly this long. |
| `Config.UnconsciousHelpCall.key` | string | `'g'` | **Display only.** Uppercased for the on-screen hint. The real binding is control 47 (`G`) in the client input loop. |

### Paramedic call — death screen

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.DeadHelpCall.enabled` | boolean | `true` | When `false` the call block is hidden on the death screen. |
| `Config.DeadHelpCall.maxCalls` | number | `3` | Maximum calls while dead. `0` means unlimited. Counted independently from the unconscious calls and reset when the screen closes. |
| `Config.DeadHelpCall.cooldown` | number (s) | `30` | Time between calls. |
| `Config.DeadHelpCall.key` | string | `'h'` | **Display only.** Uppercased for the on-screen hint. The real binding is control 74 (`H`) in the client input loop. |

### Cause of death

`Config.DeathCauses` holds only the *mappings*; the text itself is in `locales/<lang>.lua` under the `death` table. Two lookup tables:

**`Config.DeathCauses.Weapons`** — weapon hash → category. 16 categories are mapped out of the box: `pistol`, `smg`, `rifle`, `sniper`, `shotgun`, `explosive`, `fire`, `fire_env`, `heavy`, `blade`, `blunt`, `fist`, `explosion`, `vehicle`, `fall`, `water`, `electric`.

**`Config.DeathCauses.Zones`** — impact bone id → body zone. Mapped zones: `head`, `neck`, `chest`, `abdomen`, `shoulder`, `arm`, `hand`, `hip`, `leg`, `foot`.

```lua
Config.DeathCauses = {}

-- weapon hash → category
Config.DeathCauses.Weapons = {
    [0x1B06D571] = 'pistol',   -- WEAPON_PISTOL
    [0x22D8FE39] = 'smg',      -- WEAPON_MICROSMG
    [0xBFEFFF6D] = 'rifle',    -- WEAPON_ASSAULTRIFLE
    [0x0D5A796B] = 'fall',     -- WEAPON_FALL
    -- …
}

-- impacted bone → body zone
Config.DeathCauses.Zones = {
    [31086] = 'head',
    [24818] = 'chest',
    [58271] = 'leg',
    -- …
}
```

To add a weapon, add its hash and a category. To add a *new category*, add the hash mapping here **and** add the category to the `death` table in **every** `locales/<lang>.lua` with at least a `_default` key — otherwise that language falls through to the generic `death_fallback` list for kills with that weapon.

### Debug

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Debug` | boolean | `false` | Registers `/deathtest [dead\|unconscious]` and `/deathhide`, and prints a hint line on resource start. Leave `false` in production: the commands are not permission-checked, so any player could open their own death screen. |
| `Config.DebugAdminOnly` | boolean | `false` | **Declared but not read by any file in this version.** It does not gate the debug commands. |

## 🌐 Locales & Editable Strings

Seven languages ship, all outside the escrow: `locales/es.lua`, `en.lua`, `de.lua`, `fr.lua`, `it.lua`, `pt.lua`, `zh.lua`. Key sets are **identical across all seven** (verified): the same 17 flat string keys, the same 17 `death` categories, and a `death_fallback` list in each.

**File layout.** `locales/_init.lua` seeds the global `Locales` table; each `locales/<lang>.lua` assigns `Locales['<lang>']`; `locales.lua` (at the resource root, also outside the escrow) then defines the three resolver functions. All four are loaded as `shared_scripts` via `locales/*.lua`, so adding an eighth language is a drop-in — no manifest change needed.

**Resolvers** (in `locales.lua`):

- `_T(key, ...)` — returns `L[key]`, else the Spanish value, else the key itself. Applies `string.format` inside a `pcall`, so a malformed placeholder renders the raw string rather than throwing. `nil` arguments become empty strings.
- `_TDeath(category, zone)` — returns `L.death[category][zone]`, falling back to `[category]._default`, falling back to a random entry from `death_fallback`.
- `_TAll()` — flattens every *string* value (Spanish first, then the active language on top) into one table that is pushed to the NUI as `data.texts`. Nested tables such as `death` are deliberately excluded, since the death line is resolved on the Lua side and sent as `deathReason`.

**Locale keys**

| Key | Used by | Notes |
|---|---|---|
| `unconscious_title` | NUI | Unconscious screen title. |
| `unconscious_wait` | NUI | Prefix for "…in <street>". |
| `u_call_hint` | NUI | `PRESS [%s] TO CALL FOR HELP` — `%s` is `Config.UnconsciousHelpCall.key`. |
| `u_call_sent` | NUI | Confirmation after a call. |
| `u_call_cooldown` | NUI | `NEXT CALL IN %ds` — `%s` is the live countdown. |
| `u_call_no_calls` | NUI | Terminal state when calls are exhausted. |
| `death_reason_default` | NUI | Fallback cause of death if none was supplied. |
| `phase1_msg` | NUI | Phase 1 label. |
| `phase2_msg` | NUI | Phase 2 label. |
| `phase2_hint` | NUI | `PRESS [%s] TO RESPAWN` — `%s` is `Config.HelpKey`. |
| `dead_call_hint` | NUI | `PRESS [%s] TO CALL A MEDIC` — `%s` is `Config.DeadHelpCall.key`. |
| `dead_call_sent` | NUI | Confirmation. |
| `dead_call_cooldown` | NUI | `NEXT CALL IN %ds`. |
| `dead_call_no_calls` | NUI | Terminal state. |
| `death` (table) | Lua | Cause-of-death phrases, keyed `[category][zone]` with a `_default` per category. |
| `death_fallback` (list) | Lua | Generic phrases used when the weapon category is unknown. |
| `death_title` | — | **Present in all 7 locales but never applied.** The death screen title is the literal `WASTED` in `html/index.html`. |
| `help_hint`, `help_sent` | — | **Present in all 7 locales but never applied.** Superseded by `u_call_hint` / `u_call_sent`. |
| `critical` | — | **Present in all 7 locales but never applied.** The `CRITICAL` label is hardcoded as `CRÍTICO` in `html/index.html`. |
| `seconds_label` | — | **Present in all 7 locales but never applied.** The `SECONDS` label is hardcoded as `SEGUNDOS` in `html/index.html`. |

**Escrow status of strings.** No player-facing string lives inside the escrow. `shared/config.lua`, `shared/functions.lua`, `locales.lua` and `locales/*.lua` are all in `escrow_ignore`, and `html/index.html`, `html/style.css` and `html/app.js` ship as `files` and are therefore never encrypted either.

**But four labels are not translated.** Because `critical`, `seconds_label` and `death_title` are never read, the following remain hardcoded in `html/index.html` regardless of `Config.Locale`:

| Element | Hardcoded text |
|---|---|
| `#critical-lbl` | `CRÍTICO` |
| `.timer-lbl` (unconscious, ×2) | `MIN` and `SEG` |
| `.d-timer-lbl` (death, ×2) | `SEGUNDOS` |
| `.wasted-title` | `WASTED` (intentional — the GTA term) |

They are plain HTML and easy to edit, but they will not follow `Config.Locale`. The placeholder values for every other element in `index.html` (`INCONSCIENTE`, `Esperando paramédicos en Downtown Los Santos`, `PRESIONA [G] PARA PEDIR AYUDA`, `Heridas mortales`, …) are overwritten by `app.js` on the first message, so those are harmless.

Also worth noting: `shared/functions.lua` passes two Spanish strings to `qb-ambulancejob`'s alert — `'Jugador inconsciente solicita ayuda'` and `'Jugador muerto solicita ayuda médica'`. These reach the medics' notification, not the dying player, and are directly editable in that file.

## 🔗 Compatibility

| System | How it is selected | Notes |
|---|---|---|
| **Framework** | `Config.Framework = 'esx' \| 'qb'` | ESX via `exports['es_extended']:getSharedObject()` with a legacy `esx:getSharedObject` event fallback; QBCore via `exports['qb-core']:GetCoreObject()`. Every branch in `shared/functions.lua` keys off the same value. |
| **Ambulance / EMS job** | `shared/functions.lua` | **ESX:** distress → `esx_ambulancejob:onPlayerDistress`; respawn → `esx_ambulancejob:onRemoveItemsAfterRPDeath`; revive is left to `esx_ambulancejob`. **QBCore:** distress → `hospital:server:ambulanceAlert`; respawn → `nexus_deathscreen:server:doRespawn` → `hospital:client:RespawnAtHospital` (which picks the nearest hospital, assigns a bed and bills the player); revive → `hospital:client:Revive`. Any other EMS resource is supported by rewriting these six functions. |
| **Notifications** | not used | The script draws its own NUI and never calls a notification resource. |
| **TextUI** | not used | Key hints are rendered inside the NUI. |
| **Target system** | not used | No interaction points. |
| **Keys / fuel / banking** | not used | — |
| **Database** | not used | Nothing is persisted. |
| **HUD resources** | automatic | `SetFrontendActive(false)` + `HideHudAndRadarThisFrame()` every frame while a screen is up. A custom HUD rendered through its own NUI layer is outside this script's control and may still show through — hide it from your own resource on `nexus_deathscreen:client:onTransitionToDead`. |
| **Other death screens** | **conflict** | Disable the death UI in `esx_ambulancejob` / `qb-ambulancejob` (and any standalone wasted-screen resource), or two overlays will render simultaneously. |
| **Fonts** | external | `html/index.html` loads Bebas Neue from Google Fonts. Without outbound internet from the game client the NUI falls back to the generic sans-serif stack and will look noticeably different. |

## 💻 Developer API

### Client Exports

**None.** `nexus_deathscreen` registers no client export. The integration surface is the event set below plus the six functions in `shared/functions.lua`.

### Server Exports

**None.** `nexus_deathscreen` registers no server export.

### Events — Emitted

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_deathscreen:client:onTransitionToDead` | client → client (local `TriggerEvent`) | *none* | The unconscious timer reached 0 and the death screen has just been shown. This is the hook for syncing your ambulance job's internal "player is dead" state to Nexus's timing. |
| `nexus_deathscreen:server:onDeath` | client → server | *none* | Fired at the same instant as the above, and also on the direct `nexus_deathscreen:client:death` path. The server handler calls `OnPlayerDeath(source)` in `shared/functions.lua`. |
| `nexus_deathscreen:server:doRespawn` | client → server | *none* | Fired from `OnRespawnTriggered` on the QBCore branch only. The server handler calls `DoServerRespawn(source)`. |
| `esx_ambulancejob:onPlayerDistress` | client → server | *none* | ESX branch of `OnHelpKeyPressed` and `OnDeadHelpCallTriggered`. |
| `hospital:server:ambulanceAlert` | client → server | `message: string` | QBCore branch of the same two functions. |
| `esx_ambulancejob:onRemoveItemsAfterRPDeath` | client → client | *none* | ESX branch of `OnRespawnTriggered`. |
| `hospital:client:Revive` | server → client | *none* | QBCore branch of `OnPlayerRevived`. |
| `hospital:client:RespawnAtHospital` | server → client | *none* | QBCore branch of `DoServerRespawn`. |

All of the above except the first three are emitted from `shared/functions.lua` and are therefore yours to change.

### Events — Listened

These are the **public entry points**. Call them from your ambulance job or your own code.

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `nexus_deathscreen:client:unconscious` | client, local `AddEventHandler` | *none* | Show the unconscious screen. The script reads the current street name itself, owns the countdown, and after `Config.UnconsciousTime` seconds transitions to the death screen, starts the camera and the two respawn phases, and fires the transition events. **This is the normal entry point.** |
| `nexus_deathscreen:client:death` | client, local `AddEventHandler` | *none* | Show the death screen directly, for a death with no laststand or for a script that owns its own down-timer. Resolves the cause of death, starts the camera and the two phases, and fires `nexus_deathscreen:server:onDeath`. **Idempotent:** returns immediately if the death screen is already showing, so a duplicate event from `qb-ambulancejob` right after the internal transition cannot reset the screen. |
| `nexus_deathscreen:client:revive` | server → client (`RegisterNetEvent`) and local | *none* | A medic revived the player: clear timers, release the camera, hide the NUI. Does **not** touch the ped — your revive script handles health and position. |
| `nexus_deathscreen:client:doSpawn` | server → client (`RegisterNetEvent`) | `coords: { x, y, z, heading? }` | Hard respawn: `NetworkResurrectLocalPlayer` at those coordinates, health to 200, `ClearPedBloodDamage`, then hide the NUI. Use it when you want Nexus to perform the resurrection instead of your framework. |
| `nexus_deathscreen:server:onDeath` | client → server | *none* | Server-side handler; calls `OnPlayerDeath(source)`. |
| `nexus_deathscreen:server:doRespawn` | client → server | *none* | Server-side handler; calls `DoServerRespawn(source)`. |

### NUI Callbacks

| Callback | Payload | Effect |
|---|---|---|
| `respawn` | `{ forced: boolean }` | Triggers a respawn, but only while the death screen is up **and** phase 2 is active. Note that the script runs with `SetNuiFocus(false, false)` throughout, so the NUI's own keydown listener never fires in practice and this callback is effectively unused — the live path is control 47 in the client input loop. It is kept for a custom interface that takes focus. |

### Editable Functions (functions.lua)

`shared/functions.lua` is a `shared_script` listed in `escrow_ignore`, so all six functions exist on both client and server and all six are editable. The comment headers mark which side each one actually runs on.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `OnHelpKeyPressed` | `OnHelpKeyPressed()` | **Client** — `DoUnconsciousHelpCall`, after the cooldown and max-calls checks pass, while the unconscious screen is up. | Alert the medics that an unconscious player needs help. Ships with `esx_ambulancejob:onPlayerDistress` (ESX) and `hospital:server:ambulanceAlert` with a Spanish message (QB). Return value ignored. |
| `OnDeadHelpCallTriggered` | `OnDeadHelpCallTriggered(callNumber)` | **Client** — `DoDeadHelpCall`, same checks, while the death screen is up. | Alert the medics that a dead player is calling. `callNumber` is the 1-based index of this call (1, 2, 3…), so you can escalate — e.g. ping a different channel on the third call. Return value ignored. |
| `OnRespawnTriggered` | `OnRespawnTriggered(isForced)` | **Client** — `DoRespawn`, immediately before the NUI is hidden. | Perform the respawn. `isForced` is `true` when `Config.RespawnForceTime` ran out, `false` when the player pressed the respawn key. Ships with `esx_ambulancejob:onRemoveItemsAfterRPDeath` (ESX) and `TriggerServerEvent('nexus_deathscreen:server:doRespawn')` (QB). Return value ignored. **This is where the player actually gets moved** — leave it empty and the screen will close with the player still a corpse. |
| `OnRespawnDone` | `OnRespawnDone()` | **Client** — the last line of `HideNui()`, so it also runs on a revive and on `/deathhide`. | Post-respawn cleanup. Ships empty for ESX; for QB it resets `isDead`, `deathTime` and clears `SetEntityInvincible`. Return value ignored. Because it runs on *every* hide, keep it idempotent. |
| `OnPlayerDeath` | `OnPlayerDeath(source)` | **Server** — the `nexus_deathscreen:server:onDeath` handler. | React to a confirmed death: logging, a Discord webhook, dropping items, a death counter. `source` is the player server id. Ships as a no-op for both frameworks (both ambulance jobs manage their own state). Return value ignored. |
| `OnPlayerRevived` | `OnPlayerRevived(source, revivedBy)` | **Server** — called from `DoServerRespawn`. Not called anywhere else in the shipped code, so `revivedBy` is always `nil` as shipped. | Apply the actual revive server-side. Ships empty for ESX and `TriggerClientEvent('hospital:client:Revive', source)` for QB. Return value ignored. |
| `DoServerRespawn` | `DoServerRespawn(source)` | **Server** — the `nexus_deathscreen:server:doRespawn` handler. | Physically respawn the player at a hospital. Calls `OnPlayerRevived(source, nil)` and then, on the QB branch, `hospital:client:RespawnAtHospital` (which handles nearest hospital, bed and bill). Return value ignored. |

Swapping in a different EMS resource is a single-file change:

```lua
-- shared/functions.lua  —  example: route distress to your own dispatch

function OnHelpKeyPressed()
    TriggerServerEvent('my_dispatch:emergency', 'EMS', 'Unconscious civilian')
end

function OnDeadHelpCallTriggered(callNumber)
    local priority = callNumber >= 3 and 'CODE_RED' or 'EMS'
    TriggerServerEvent('my_dispatch:emergency', priority, 'Dead civilian requesting a medic')
end

function OnRespawnTriggered(isForced)
    TriggerServerEvent('my_hospital:respawnMe', isForced)
end
```

### Database Schema

**None.** `nexus_deathscreen` creates no tables, ships no `.sql` file and persists nothing. Call counters, timers and screen state are per-session and live in client memory only; they reset when the screen closes.

### Integration Example

A complete wiring of `nexus_deathscreen` into a custom EMS flow: a third-party resource owns the down/dead state and the dispatch, Nexus owns the presentation. Nothing escrowed is touched.

```lua
-- ────────────────────────────────────────────────────────────
-- my_ems/client.lua  —  deciding WHEN each screen shows
-- ────────────────────────────────────────────────────────────

local downed = false

-- Player dropped below the death threshold but is savable
RegisterNetEvent('my_ems:client:down', function()
    if downed then return end
    downed = true
    TriggerEvent('nexus_deathscreen:client:unconscious')
end)

-- Instant kill (explosion, headshot, /kill) — skip the laststand
RegisterNetEvent('my_ems:client:killed', function()
    downed = true
    TriggerEvent('nexus_deathscreen:client:death')
end)

-- A medic got to them in time
RegisterNetEvent('my_ems:client:revived', function()
    downed = false
    TriggerEvent('nexus_deathscreen:client:revive')
end)

-- Nexus's own unconscious timer expired: the player is now officially dead.
-- Sync our state and hide our custom HUD, which Nexus cannot reach.
AddEventHandler('nexus_deathscreen:client:onTransitionToDead', function()
    downed = true
    TriggerEvent('my_hud:client:setVisible', false)
    TriggerServerEvent('my_ems:server:markDead')
end)


-- ────────────────────────────────────────────────────────────
-- shared/functions.lua  —  deciding WHAT each action does
-- ────────────────────────────────────────────────────────────

function OnHelpKeyPressed()
    TriggerServerEvent('my_ems:server:distress', 'unconscious')
end

function OnDeadHelpCallTriggered(callNumber)
    -- escalate: calls 1-2 go to EMS, call 3+ also pings police
    TriggerServerEvent('my_ems:server:distress', 'dead', callNumber)
end

function OnRespawnTriggered(isForced)
    -- our own hospital handles position, health and the bill
    TriggerServerEvent('my_ems:server:respawn', isForced)
end

function OnRespawnDone()
    TriggerEvent('my_hud:client:setVisible', true)
end

function OnPlayerDeath(source)
    -- server side: audit the death
    local name = GetPlayerName(source) or 'unknown'
    print(('[my_ems] %s (%s) died'):format(name, source))
    MySQL.insert('INSERT INTO my_death_log (player, at) VALUES (?, ?)',
        { GetPlayerIdentifier(source, 0), os.time() })
end


-- ────────────────────────────────────────────────────────────
-- my_ems/server.lua  —  respawning the player ourselves
-- ────────────────────────────────────────────────────────────

RegisterNetEvent('my_ems:server:respawn', function(isForced)
    local src = source
    -- Let Nexus do the resurrection at our own hospital bed
    TriggerClientEvent('nexus_deathscreen:client:doSpawn', src, {
        x = 298.5, y = -584.6, z = 43.26, heading = 160.0
    })
end)

-- A medic used a defibrillator: close the screen without respawning
RegisterNetEvent('my_ems:server:medicRevive', function(targetId)
    TriggerClientEvent('nexus_deathscreen:client:revive', targetId)
    TriggerClientEvent('my_ems:client:revived', targetId)
end)
```

Adding a new weapon category end-to-end:

```lua
-- shared/config.lua
Config.DeathCauses.Weapons[0x47757124] = 'bow'   -- WEAPON_RAYCARBINE, as an example

-- locales/en.lua  (and de, fr, it, pt, zh, es — all of them)
death = {
    -- …
    bow = {
        head    = 'Arrow through the skull — instant death',
        chest   = 'Arrow to the chest — pierced lung',
        _default = 'Penetrating arrow wound',
    },
}
```

Omit the category from one language and that language silently falls back to its generic `death_fallback` list for kills with that weapon.

## ❓ FAQ

**Nothing happens when I die.**
This resource does not detect death. It only draws screens when something triggers `nexus_deathscreen:client:unconscious` or `nexus_deathscreen:client:death`. Add those `TriggerEvent` calls to your ambulance job (for `qb-ambulancejob`, in `laststand.lua` where last stand begins and where the player is marked dead), or verify with `Config.Debug = true` and `/deathtest dead` that the screens themselves work.

**I see two death screens stacked on top of each other.**
Your ambulance job's own death UI is still enabled. Disable the wasted/laststand overlay in `esx_ambulancejob` or `qb-ambulancejob` (and any standalone death-screen resource).

**I changed `Config.HelpKey` to `'f'` and the hint says F, but F does nothing.**
`Config.HelpKey`, `Config.UnconsciousHelpCall.key` and `Config.DeadHelpCall.key` are **display-only** in this version. The real bindings are hardcoded in the client input loop: control 47 (`G`) for "call for help" while unconscious and for respawning in phase 2, and control 74 (`H`) for "call a medic" while dead. Changing the config changes the label, not the key.

**The respawn key does nothing on the death screen.**
Respawn is only available in **phase 2**. During phase 1 (`Config.RespawnWaitTime` seconds, 15 by default) the key is deliberately inert and the screen shows "waiting for medical response". Set `Config.RespawnWaitTime = 0` if you want phase 2 immediately.

**The timers do not match my ambulance job — the player respawns while still revivable, or stays down after the medic timer ended.**
This is the single most common misconfiguration. `Config.UnconsciousTime` must equal `Config.ReviveInterval` and `Config.RespawnWaitTime` must equal `Config.DeathTime` in `qb-ambulancejob/config.lua` (the equivalents in whatever you use). The mapping is written out at the top of `shared/functions.lua`.

**The cause of death is always generic ("Multiple trauma", "Severe internal bleeding"…).**
That is the `death_fallback` list, used when the killing weapon's hash is not in `Config.DeathCauses.Weapons`. Custom or addon weapons are not mapped out of the box. Add the hash and a category to `Config.DeathCauses.Weapons`; if you invent a new category, add it to the `death` table in all seven locale files too.

**The cause of death ignores where I was hit.**
The bone was not in `Config.DeathCauses.Zones`, so the category's `_default` phrase was used. Add the bone id to the zone map. Note that explosions, fire, drowning and electrocution deliberately only define `_default` — a body zone is meaningless for them.

**`CRÍTICO`, `MIN`, `SEG` and `SEGUNDOS` stay in Spanish even with `Config.Locale = 'en'`.**
The `critical` and `seconds_label` locale keys exist in all seven languages but `html/app.js` never applies them, so those four labels are hardcoded in `html/index.html`. Edit `index.html` directly — it ships unencrypted. The same is true of `death_title`: the `WASTED` title is literal HTML (intentionally, since it is the GTA term).

**The camera clips into walls when I die indoors.**
`orbit_free` already raycasts from the corpse to the camera and pulls the camera 0.12 m in front of whatever it hits. If it still looks wrong in a very tight space, reduce `Config.DeathCamera.orbitMinRadius` or `orbitMaxRadius`, or switch `mode` to `'static'` with a smaller `offset`. The `'orbit'` mode does **not** raycast and will clip.

**Can I disable the death camera entirely?**
Yes: `Config.DeathCamera.enabled = false`. The default game camera stays where it is and no scripted camera is created or destroyed.

**Blocking the mouse breaks the free camera, right?**
No. `orbit_free` reads mouse input with `GetDisabledControlNormal` and the scroll wheel with `IsDisabledControlJustPressed`, so `Config.Controls.dead = 'mouse'` still leaves the camera fully controllable while preventing the player from shooting or aiming.

**The paramedic call count does not reset between deaths.**
It does. `HideNui()` resets the dead-call counter and cooldown on every hide (respawn, revive or `/deathhide`). Note that the unconscious-call counter is reset at a different point in the lifecycle, so if a player goes down, is revived, and goes down again within the same session, verify the dot counter looks right before reporting a bug.

**Does this script save anything to the database?**
No. No tables, no `.sql` file, no oxmysql dependency. Everything is per-session client state. Death logging is something you add in `OnPlayerDeath`.

**The font looks wrong / everything is in a generic sans-serif.**
`html/index.html` loads Bebas Neue from Google Fonts. If the game client has no outbound internet access, or your players are behind a restrictive network, the font never arrives and the CSS falls back to the generic sans-serif stack. Self-host the font file and change the `<link>` if that matters to you.

**Is it safe to leave `Config.Debug = true` on a live server?**
No. `/deathtest` and `/deathhide` are registered with no permission check, so any player could open or close their own death screen at will. `Config.DebugAdminOnly` exists in the config but is not read by any file in this version, so it does not help.

### Before opening a ticket

- Make sure the resource folder is named exactly **`nexus_deathscreen`**. Any other name aborts the resource with a console error.
- Make sure you are on the latest version of the resource (**v2.1.0**, per `fxmanifest.lua`).
- Confirm that something in your server actually triggers `nexus_deathscreen:client:unconscious` / `nexus_deathscreen:client:death` — the resource never detects death on its own.
- Confirm your ambulance job's own death UI is disabled.
- Verify the screens in isolation with `Config.Debug = true` and `/deathtest dead` / `/deathtest unconscious` before blaming the integration, then set it back to `false`.
- Re-read this FAQ page.

## 📋 Changelog

**v2.1.0 — current release** (per `fxmanifest.lua`: `version '2.1.0'`, `description 'Custom death/unconscious screen for QBCore and ESX'`)

No version history file ships with the resource. What the code itself documents:

- The `switchToDead` NUI action exists specifically to replace a hide+show pair with a cross-fade, and the direct `nexus_deathscreen:client:death` handler carries an explicit guard against the duplicate event QBCore fires right after that transition — both are fixes layered onto an earlier unconscious → death flow.
- The input loop comments note that control 74 (`H`) is read with `IsDisabledControlJustReleased` because `qb-ambulancejob` calls `DisableAllControlActions` and because control 74 does not register reliably on foot with the non-disabled variant — a QBCore-specific compatibility fix.
- `locales/*.lua` is loaded with a glob and all seven target languages (`es`, `en`, `de`, `fr`, `it`, `pt`, `zh`) are present with matching key sets, so the localisation set is complete as of this release.
