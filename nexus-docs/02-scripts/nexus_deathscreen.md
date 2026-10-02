# Nexus Deathscreen

## Description
Custom unconscious and death screens for ESX and QBCore. When a player goes down, an unconscious screen with a live heart monitor counts down, then switches to a WASTED screen with a cause of death, a respawn timer and a medic call system. Includes an optional orbiting death camera.

## Features
- Unconscious screen with countdown, heart animation, BPM and ECG that react to the remaining time
- WASTED screen in two phases: waiting for medical response, then forced or manual respawn
- Cause of death text based on weapon category and body zone
- Medic call system on both screens with max calls, cooldown bar and remaining-call indicators
- Configurable keys for help, respawn and medic call
- Death camera modes: `orbit`, `orbit_free` (mouse controlled with zoom) and `static`
- Visual effects: `blackout`, `minimal`, `vignette`
- Optional control blocking per state
- 7 languages: es, en, de, fr, it, pt, zh
- No database usage

## Dependencies
- ESX (`esx_ambulancejob`) or QBCore (`qb-core`, `qb-ambulancejob`), selected in the config
- No database required

## Installation
1. Drop the `nexus_deathscreen` folder into `resources`. The folder name must stay exactly `nexus_deathscreen`.
2. Add `ensure nexus_deathscreen` to `server.cfg` after your framework and ambulance resources.
3. Set `Config.Framework` and `Config.Locale` in `shared/config.lua`.
4. Make your ambulance script trigger the events listed in the Developer API.
5. Sync the timers (see Configuration) and restart.

## Configuration
`shared/config.lua`:

| Option | Default | Description |
|---|---|---|
| `Config.Framework` | `'esx'` | `'qb'` or `'esx'`. |
| `Config.Locale` | `'es'` | `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`. |
| `Config.HelpKey` | `'g'` | Respawn key on the death screen (phase 2). |
| `Config.UnconsciousTime` | `15` | Seconds of the unconscious screen. Match `Config.ReviveInterval` of qb-ambulancejob. |
| `Config.RespawnWaitTime` | `15` | Seconds of the WASTED waiting phase. Match `Config.DeathTime` of qb-ambulancejob. |
| `Config.RespawnForceTime` | `30` | Extra seconds until forced respawn. |
| `Config.VisualEffect` | `'minimal'` | `blackout`, `minimal` or `vignette`. |
| `Config.Controls` | `none` | Control blocking per state: `none`, `keyboard`, `mouse`, `keyboard+mouse`. |
| `Config.DeathCamera` | see file | Camera mode, blend times, FOV and orbit options. |
| `Config.UnconsciousHelpCall` | enabled, 3 calls, 20 s, key `g` | Medic call on the unconscious screen. |
| `Config.DeadHelpCall` | enabled, 3 calls, 30 s, key `h` | Medic call on the death screen. |
| `Config.DeathCauses` | see file | Weapon hash to category and bone to body zone mapping. |
| `Config.Debug` | `false` | Registers `/deathtest [dead|unconscious]` and `/deathhide`. |

Supported key names for the three key options: letters, digits, `F1` to `F11`, arrows, `SPACE`, `ENTER`, `TAB`, `BACKSPACE`, `LSHIFT`, `LCTRL`, `LALT`, `HOME`, `PAGEUP`, `PAGEDOWN`, `DELETE` and a few symbols. Invalid names print a console warning and fall back to the default (G for respawn and unconscious help, H for the death medic call).

## Locales & Editable Strings
Texts live in `locales/<lang>.lua` and are not encrypted. Every file has the same keys:

- Screen texts: `unconscious_title`, `unconscious_wait`, `death_title`, `death_reason_default`, `phase1_msg`, `phase2_msg`, `phase2_hint`, `help_hint`, `help_sent`
- Medic call texts: `u_call_hint`, `u_call_sent`, `u_call_cooldown`, `u_call_no_calls`, `dead_call_hint`, `dead_call_sent`, `dead_call_cooldown`, `dead_call_no_calls`
- Labels: `critical`, `seconds_label`, `min_label`, `sec_label`
- Ambulance alerts (QBCore): `ambulance_alert_unconscious`, `ambulance_alert_dead`
- `death` (cause-of-death phrases by category and zone) and `death_fallback`

`%s` is the key and `%d` is the seconds, so keep them in your edits. To add a language, copy `en.lua`, rename it and change `Locales['en']`.

## Compatibility
- ESX with `esx_ambulancejob`
- QBCore with `qb-ambulancejob`
- Escrow: `shared/config.lua`, `shared/functions.lua`, `locales.lua`, `locales/*.lua` and the NUI files are excluded. The client and server logic is encrypted.

## Developer API

### Client events (listened by this resource)
```lua
TriggerEvent('nexus_deathscreen:client:unconscious')
TriggerEvent('nexus_deathscreen:client:death')
TriggerEvent('nexus_deathscreen:client:revive')
```
- `unconscious` starts the unconscious screen. When its timer ends, the screen switches to WASTED by itself.
- `death` shows the WASTED screen directly.
- `revive` closes the screen.

### Events emitted by this resource
- `nexus_deathscreen:client:onTransitionToDead` (client, local): fired when the unconscious screen turns into the death screen.
- `nexus_deathscreen:server:onDeath` (server): the player is now officially dead.
- `nexus_deathscreen:server:doRespawn` (server): the player asked to respawn.
- `nexus_deathscreen:client:doSpawn` (client): resurrects the local player at `coords` (`x`, `y`, `z`, `heading`).

### Hooks in `shared/functions.lua`
`OnHelpKeyPressed`, `OnDeadHelpCallTriggered`, `OnRespawnTriggered`, `OnRespawnDone`, `OnPlayerDeath`, `OnPlayerRevived`, `DoServerRespawn`. Edit them to connect other ambulance or dispatch scripts. ESX uses `esx_ambulancejob:onPlayerDistress`; QBCore uses `hospital:server:ambulanceAlert` with the localized alert texts.

### Schema SQL
Not applicable.

### Integration Example
```lua
AddEventHandler('hospital:client:SetLaststand', function()
    TriggerEvent('nexus_deathscreen:client:unconscious')
end)

RegisterNetEvent('hospital:client:Revive', function()
    TriggerEvent('nexus_deathscreen:client:revive')
end)
```

## FAQ
**My keys do nothing.** Use a key name from the supported list. Check the console for an invalid-key warning.

**I was revived and went down again, but could not call for help.** Fixed in 2.2.0: the call counters now reset when the screen closes.

**The screen text is in the wrong language.** Set `Config.Locale` and make sure the locale file exists.

**How do I add a weapon to the death causes?** Add its hash and a category in `Config.DeathCauses.Weapons`. A new category needs a `death` entry with `_default` in every locale file.

**Does it need a database?** No.

## Changelog
### 2.2.0
- `Config.HelpKey`, `Config.UnconsciousHelpCall.key` and `Config.DeadHelpCall.key` now control the real keys through a key lookup table.
- Locale keys `critical` and `seconds_label` are now applied in the UI.
- Added locale keys `min_label` and `sec_label`, replacing the fixed MIN/SEG labels.
- Unconscious call count and cooldown now reset when the screen closes.
- QBCore ambulance alert messages moved to locales (`ambulance_alert_unconscious`, `ambulance_alert_dead`) and translated for all languages.

### 2.1.0
- Previous release.
