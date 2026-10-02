# Nexus Bounty

A player-hunting and contract system driven from a fully custom in-game tablet, with civilian and police-only contracts, offline-target support, map tracking, automatic expiry and a persistent server-wide leaderboard.

## 📝 Description

Nexus Bounty lets any player put a price on another player's head from a tablet-style NUI that looks and behaves like a real device — status bar with clock, battery and a fake `nexusbounty://` address bar that updates as you navigate, a home screen with four app cards, and sliding detail panels and modals on top. Four "apps" live inside it: **Crear Contrato** (create a contract), **Contratos Activos** (browse every live contract in the city), **Mi Panel** (your own stats, the contracts you created, and the ones you accepted), and **Clasificación** (the server leaderboard).

The contract flow is fully server-authoritative. When a contract is created, the server validates the amount against `Config.PVP.MinBountyAmount`, validates the contract type against the `Civil`/`Policial` enum, validates the requested duration against `Config.PVP.Durations` **before** touching the player's money, and only then withdraws from the bank account. It then resolves the target by name: first across online players (respecting `Config.UseApproximateNameSearch`, which switches between a substring match and an exact match), and if nothing is found online, against the framework's own player table — `users` for ESX, `players` with `JSON_EXTRACT` on `charinfo` for QBCore. This is what makes **offline targets** work: you can place a bounty on someone who is not even connected, and the contract will be waiting when they log in. Any failure after the withdrawal refunds the player automatically.

Payout is wired directly into the framework's death event (`esx:onPlayerDeath` or `QBCore:Server:OnPlayerDeath`). On every player death the server looks up a live contract on the victim's identifier, decodes the JSON list of hunters who accepted it, and only pays out if the killer is on that list — so random kills never collect a bounty, the hunter has to have formally accepted the contract first. The payout adds the money to the killer's bank, notifies both parties, deletes the contract, and does a single `INSERT … ON DUPLICATE KEY UPDATE` against `bounty_leaderboard` to increment the hunter's completed-contract count and total earnings.

Expiry is handled with real timers rather than polling. Every contract schedules a `Citizen.SetTimeout` for its exact expiration moment, which deletes the row and tells clients to drop any tracking blip. Two seconds after the resource starts, the server re-reads every non-expired contract from the database and re-schedules all of their timers, so a server restart does not leave immortal contracts behind. Tracking itself is a client-side loop: once you accept a contract against an online target, the client polls every second, and when you come within `Config.PVP.BlipRevealDistance` of them it drops a blip at their position for `Config.PVP.BlipDuration` milliseconds, then enters a 30-second cooldown — close enough to hunt, not close enough to be a permanent GPS.

> **Important scope note:** the PvE / NPC contract system is **not implemented in this version.** `Config.PVE` (including `Config.PVE.NPCTemplates`, `MaxActiveNPCBounties` and `Enabled`) is present in the config file but is **never read by any Lua or JS file in the resource**. The database and the NUI both support NPC contracts (`is_npc_bounty`, `npc_template_id`, `npc_bounty_name`), and there is a debug-only test NPC workflow, but no generator creates contracts from the templates. See the incident report.

## ✨ Features

- **Custom tablet NUI, not a menu** — a framed tablet with physical-looking home/volume buttons, a status bar (live clock updated every second, lock icon, battery), and a simulated URL bar that reflects the current screen (`nexusbounty://my-panel/accepted-bounties/bounty-details/42`).
- **Home screen with live metrics** — two badges in the header show the current online player count and the number of live contracts, pulled from the server on open and refreshed automatically every 30 seconds while the tablet is open.
- **Create-contract form** — target name, amount (with the configured minimum shown in the placeholder), optional image URL, free-text roleplay hints, a duration dropdown populated from `Config.PVP.Durations`, a `Civil`/`Policial` type selector, and an interactive 1–5 star difficulty rating with hover preview. A sidebar explains the rules.
- **Police-only contracts** — selecting `Policial` is validated server-side against `Config.PoliceJobName`; a non-officer who tries it is refused and refunded.
- **Offline target resolution** — if no online player matches the name, the server queries the framework's player table (`users` / `players`) with `LIKE` or `=` depending on `Config.UseApproximateNameSearch`, and refuses (with a refund) if the search is ambiguous — the match must resolve to exactly one person.
- **Active-contracts browser** — every live contract as a row with the target's photo (or a silhouette placeholder), target name, an online/offline status dot, contract type with a matching icon, star difficulty and the reward. Clicking a row slides in a detail panel.
- **Contract detail panel with image carousel** — `url_imagen` accepts several comma-separated URLs; with more than one the panel renders a scroll-snapping carousel with clickable dots. Bare `imgur.com/xxxx` links are automatically rewritten to `https://i.imgur.com/xxxx.png`. The panel shows reward, difficulty, type, contractor, exact expiry timestamp and the roleplay notes, and the Accept button only appears when you are browsing active contracts and have not already accepted this one.
- **Personal dashboard** — four stat cards (contracts created, total money offered, contracts completed, total earned) plus two tabs: the contracts you created (each with edit and delete buttons) and the contracts you accepted.
- **Edit your own contract** — an edit modal lets the creator change the image URL(s) and the roleplay notes after publication. The server re-verifies ownership via `contratador_identifier` before updating, and only those two columns can ever be changed.
- **Cancel with refund** — a two-step confirmation modal; on confirm the server re-verifies ownership, deletes the row and refunds the full `monto` to the creator's bank.
- **Target tracking blips** — per-second proximity check, blip sprite `161` in red at scale `1.5`, shown for `Config.PVP.BlipDuration` once you are inside `Config.PVP.BlipRevealDistance`, followed by a 30-second cooldown. Tracking stops automatically if the target goes offline, if the contract is claimed, or when it expires.
- **Automatic expiry** — one `Citizen.SetTimeout` per contract, re-scheduled for all live contracts 2 seconds after the resource starts. On fire, the row is deleted, clients drop the blip and open panels refresh.
- **Anti-self-contract** — you cannot accept a bounty you created; the server compares `contratador_identifier` against your identifier.
- **Server-wide leaderboard** — top 20 by total earned, from `bounty_leaderboard`, maintained with an upsert on each payout.
- **Death-driven payout** — hooks the framework's native death event; pays only if the killer is on the contract's accepted-hunters list; ignores suicides and self-kills (`victim == killer`).
- **Multi-hunter contracts** — any number of players can accept the same contract; whoever lands the kill gets the money.
- **Framework abstraction** — every framework touchpoint (identifier, name, job, add/remove money, player list, server callbacks, death event) goes through one wrapper function, selected by `Config.Framework`.
- **Pluggable notifications** — `Config.NotifySystem` selects between `nexus_notify`, `okokNotify`, native ESX, native QBCore, `mythic_notify` or your own code, in an unencrypted file.
- **Debug test NPC** — with `Config.DebugMode = true`, `/test_bounty_npc` spawns a standing ped 5 units in front of you, creates a $10,000 one-hour NPC contract on it, and watches for its death so you can validate the whole accept → kill → payout → leaderboard chain without a second player.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **ESX or QBCore** | **Required** | Selected with `Config.Framework` (`"ESX"` / `"QBCore"`). Used for identifiers, character names, jobs, bank money and the death event. There is no standalone mode. |
| **MySQL with the `MySQL.Async` API** | **Required** | `fxmanifest.lua` hard-depends on `@mysql-async/lib/MySQL.lua`. Provided by `mysql-async`, or by `oxmysql` through its mysql-async compatibility layer. |
| `nexus_bounty.sql` import | **Required** | Creates `bounties_active` and `bounty_leaderboard`. |
| A configured police job | Conditional | Only needed if you want `Policial` contracts. The job name must match `Config.PoliceJobName` exactly as stored by your framework. |
| `nexus_notify` | Optional | Only when `Config.NotifySystem = "nexus"` (the default — change it if you do not own it). |
| `okokNotify` / `mythic_notify` | Optional | Only for those `Config.NotifySystem` values. |
| Internet access from the game client | Optional | Only for loading contract images and the Font Awesome CDN stylesheet. Contracts work without images. |
| Target / TextUI / inventory | **Not used** | The tablet is a standalone NUI. No target system, no TextUI, no item required. |

**Load order matters:** your MySQL resource must be started before `nexus_bounty`, and your framework must be running — the resource only initialises inside the `esx:getSharedObject` callback or the `QBCore:Ready` event.

## ⚙️ Installation

1. **Unzip the resource.** Download from the cfx.re portal (Keymaster) and extract into `resources/`. The folder **must** be named exactly `nexus_bounty` — `shared/_resource.lua` raises a hard error otherwise.
2. **Import the SQL.** Run `nexus_bounty.sql` against your server's database. It creates `bounties_active` and `bounty_leaderboard`, and includes an idempotent `ALTER TABLE … ADD COLUMN IF NOT EXISTS contratador_identifier` migration for anyone upgrading from a build that predates that column (that statement needs MySQL 8.0.29+ / MariaDB 10.3+; on older servers, drop the `IF NOT EXISTS` and only run it if the column is actually missing).
3. **Check `server.cfg` order.** The MySQL resource and your framework must both come first:
   ```cfg
   ensure oxmysql          # or: ensure mysql-async
   ensure es_extended      # or: ensure qb-core
   ensure nexus_notify     # only if Config.NotifySystem = "nexus"
   ensure nexus_bounty
   ```
4. **Configure.** In `shared/config.lua` set at minimum:
   - `Config.Framework` — `"ESX"` or `"QBCore"`.
   - `Config.NotifySystem` — defaults to `"nexus"`; set it to `"esx"`, `"qb"`, `"okok"`, `"mythic"` or `"custom"` if you do not run `nexus_notify`.
   - `Config.PoliceJobName` — your actual police job name.
   - `Config.PVP.MinBountyAmount` and `Config.PVP.Durations` to suit your economy.
   - Translate `Config.Locales` — it ships in **Spanish only**.
5. **Remove the test duration.** `Config.PVP.Durations` includes a `"1 Minuto (TEST)"` entry (`hours = 0.0166`). Delete it before going live or players will use it.
6. **Restart.** A full server restart is recommended.
7. **Start using it.** Run `/recompensas` (or whatever you set `Config.OpenCommand` to) to open the tablet. Create a contract from the first app card; browse and accept from the second.

## 🔧 Configuration

Everything lives in `shared/config.lua`.

### Core

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Framework` | string | `"ESX"` | `"ESX"` or `"QBCore"`. Chosen branch is used for identifiers (`identifier` vs `citizenid`), names (`getName()` vs `charinfo.firstname/lastname`), jobs, bank money, the player list, server callbacks, the death event **and** which database table offline name lookups hit (`users` vs `players`). Any other value leaves the resource uninitialised. |
| `Config.DebugMode` | boolean | `false` | Prints `[nexus_bounty]` diagnostics to the server console, and enables the `/test_bounty_npc` command plus its client-side NPC spawner. Should be `false` in production. |
| `Config.NotifySystem` | string | `"nexus"` | Which notification resource `shared/functions.lua` routes through: `"nexus"` (`nexus_notify`), `"okok"` (`okokNotify`), `"esx"`, `"qb"`, `"mythic"` (`mythic_notify`), or `"custom"` (empty branch for your own code). An unrecognised value falls through to a plain `print`. |
| `Config.OpenCommand` | string | `"recompensas"` | The chat command registered client-side to toggle the tablet. Registered without a restriction, so every player can use it. |
| `Config.PoliceJobName` | string | `"police"` | Job name required to create a `Policial` contract. Compared with `==` against `xPlayer.job.name` (ESX) or `PlayerData.job.name` (QBCore), so it must match exactly, case included. |
| `Config.UseApproximateNameSearch` | boolean | `true` | `true`: online matching uses a case-insensitive `string.find` substring match, and the offline query uses `LIKE '%name%'`. `false`: both become exact, case-insensitive equality. Note the contract is still refused unless the search resolves to exactly one person. |

### PvP contracts — `Config.PVP`

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.PVP.Enabled` | boolean | `true` | **Currently not read by any code.** There is no PvP kill-switch in this version; setting it to `false` does not disable contract creation. See the incident report. |
| `Config.PVP.MinBountyAmount` | number | `5000` | Minimum accepted reward. Enforced server-side before any money moves, and shown to the player in the amount field's placeholder. |
| `Config.PVP.Durations` | table | 5 entries | The list of selectable contract lifetimes. Each entry is `{ label = <string shown in the dropdown>, hours = <number> }`. The server validates the submitted value against the `hours` of every entry, so a manipulated NUI cannot invent a duration. Fractional hours are allowed (`0.0166` ≈ 1 minute). Expiry is stored as `os.time() + floor(hours * 3600)`. |
| `Config.PVP.EnableTracking` | boolean | `true` | Master switch for the client-side target-blip loop. With `false`, accepted contracts never produce a blip (the contract itself still works normally). |
| `Config.PVP.BlipRevealDistance` | number | `200.0` | How close (game units) a hunter must get to an accepted target before the blip is revealed. |
| `Config.PVP.BlipDuration` | number (ms) | `10000` | How long the blip stays on the map once revealed. The 30-second cooldown that follows is **hardcoded** and not configurable. |

```lua
Config.PVP = {
    Enabled = true,
    MinBountyAmount = 5000,
    Durations = {
        { label = "1 Minuto (TEST)", hours = 0.0166 },  -- remove before production
        { label = "1 Hora",          hours = 1 },
        { label = "3 Horas",         hours = 3 },
        { label = "12 Horas",        hours = 12 },
        { label = "24 Horas",        hours = 24 },
    },
    EnableTracking     = true,
    BlipRevealDistance = 200.0,
    BlipDuration       = 10000
}
```

### PvE contracts — `Config.PVE`

> ⚠️ **None of these keys is read by the code in this version.** The structure is documented here because it exists in `config.lua` and because the database and NUI already carry the matching columns, but no generator consumes it. Editing these values has no effect.

| Config key | Type | Default | Description (intended) |
|---|---|---|---|
| `Config.PVE.Enabled` | boolean | `true` | *Unused.* Intended master switch for NPC contracts. |
| `Config.PVE.MaxActiveNPCBounties` | number | `5` | *Unused.* Intended cap on simultaneously active NPC contracts. |
| `Config.PVE.NPCTemplates` | table | 2 entries | *Unused.* Templates an NPC-contract generator would draw from. |

Structure of one `NPCTemplates` entry, as written in the shipped config:

```lua
{
    id_mision      = 1,                                 -- template id (maps to npc_template_id in the DB)
    nombre_mision  = "Eliminar al Skater de Grove St.", -- mission name (maps to npc_bounty_name)
    modelo_npc     = "a_m_y_skater_01",                 -- ped model to spawn
    monto          = 7500,                              -- reward
    prob_policia   = 20,                                -- % chance police get involved
    prob_agresivo  = 50,                                -- % chance the NPC fights back
    ubicacion_pool = {                                  -- one of these is picked at random
        vector3(247.2, -1363.8, 24.5),
        vector3(-14.9, -1442.2, 31.1),
        vector3(1189.7, -1328.6, 35.3)
    }
}
```

### Locales — `Config.Locales`

A flat table of all 40 translatable strings. The exhaustive list, grouped as in the file:

**Notifications (14 keys — all actually used by the server):**

| Key | Format args | Used when |
|---|---|---|
| `not_enough_money` | — | The bank withdrawal failed. |
| `player_not_found` | — | The name resolved to zero or more than one person. |
| `invalid_amount` | — | Amount is not a number or is below `MinBountyAmount`. |
| `bounty_created_successfully` | `%s` amount, `%s` target name | Contract created. |
| `you_have_a_bounty` | `%s` amount | Sent to the target, if online. |
| `bounty_accepted` | — | Contract accepted. |
| `target_eliminated` | `%s` amount | Sent to the hunter on payout. |
| `your_bounty_claimed` | — | Sent to the victim on payout. |
| `cant_accept_own_bounty` | — | You tried to accept your own contract. |
| `police_only_bounty` | — | A non-officer tried to create a `Policial` contract. |
| `bounty_expired` | `%s` target name | **Defined but never sent by any code.** |
| `must_be_police_to_accept` | — | **Defined but never sent by any code** — acceptance is not job-gated in this version. See the incident report. |

**NUI strings (26 keys): `tablet_title`, `tab_create_bounty`, `tab_active_bounties`, `tab_leaderboard`, `form_title_pvp`, `label_target_name`, `placeholder_target_name`, `label_bounty_amount`, `placeholder_bounty_amount`, `label_image_url`, `placeholder_image_url`, `label_extra_info`, `placeholder_extra_info`, `label_duration`, `label_bounty_type`, `option_civil`, `option_police`, `button_create_bounty`, `header_target`, `header_bounty`, `header_expires`, `header_type`, `button_accept_bounty`, `button_view_info`, `no_active_bounties`, `info_popup_title`, `info_creator`, `info_anonymous`, `info_details`, `leaderboard_title`, `rank`, `hunter_name`, `contracts_completed`, `total_earned`, `close_button`.**

> ⚠️ **None of the NUI keys has any effect.** `Config` is sent to the NUI (`loadConfig`) and `script.js` does store it (`locales = data.Locales`), but the `locales` variable is never read — every string rendered inside the tablet is hardcoded Spanish in `nui/index.html` and `nui/script.js`, both of which are inside the escrow. Only `Config.PVP.MinBountyAmount` and `Config.PVP.Durations` are actually consumed from the config by the NUI. See the incident report.

## 🌐 Locales & Editable Strings

**Shipped languages: 1 (Spanish).** Unlike other Nexus resources, `nexus_bounty` has **no `locales/` folder**. All translatable text lives in `Config.Locales` inside `shared/config.lua`, which is in `escrow_ignore` and therefore editable — but it ships in Spanish only, so an English, German, French, Italian, Portuguese or Chinese server has to translate the table by hand.

What is editable (`escrow_ignore`): `shared/config.lua`, `shared/functions.lua`, `nexus_bounty.sql`.

### ⚠️ Hardcoded strings inside the escrow

`client/main.lua`, `server/main.lua` and the whole `nui/` folder are **not** in `escrow_ignore`, and all of them contain player-facing Spanish text that cannot be translated:

| Location | Strings |
|---|---|
| `client/main.lua` | `"Objetivo de Recompensa"` (the map blip name) |
| `server/main.lua` | `'Recompensas'` (the title on every single notification), `'Tipo de contrato inválido.'`, `'Duración de contrato inválida.'`, `'Error al crear la recompensa.'`, `'Recompensa no encontrada.'`, `'No puedes editar esta recompensa.'`, `'Recompensa actualizada correctamente.'`, `'Error al actualizar la recompensa.'`, `'No puedes eliminar esta recompensa.'`, `'Recompensa eliminada. Se te han devuelto $%s'`, `'Error al eliminar la recompensa.'`, `"Desconocido"` (fallback target name), plus the test-system strings |
| `nui/index.html` | `Sistema de Recompensas`, `Crear Contrato`, `Pon precio a la cabeza de alguien`, `Contratos Activos`, `Explora objetivos disponibles`, `Mi Panel`, `Gestiona tus recompensas`, `Clasificación`, `Top cazarrecompensas` |
| `nui/script.js` | Every app title, form label, placeholder, button caption, table header, empty-state message, modal title and confirmation text — roughly 60 strings |

This is logged in the incident report as the single largest localisation gap across the three resources reviewed.

## 🔗 Compatibility

| System | How it is selected | Notes |
|---|---|---|
| **Framework** | `Config.Framework` | `"ESX"` or `"QBCore"`. Every framework call is wrapped: `GetPlayerIdentifier`, `GetPlayerFromIdentifier`, `GetPlayerName`, `GetPlayerJob`, `RemoveMoney`, `AddMoney`, `GetAllPlayers`, `RegisterCallback`. ESX initialises from `esx:getSharedObject`; QBCore from the `QBCore:Ready` event. |
| **Database** | Fixed | Any wrapper exposing the `MySQL.Async` API — `mysql-async` natively, `oxmysql` via its compatibility layer. The manifest loads `@mysql-async/lib/MySQL.lua` directly, so that path must resolve. |
| **Money** | Framework-native | Always the **bank** account, both ways. ESX: `getAccount('bank')` + `removeAccountMoney`/`addAccountMoney`. QBCore: `Player.Functions.RemoveMoney('bank', …)` / `AddMoney('bank', …)`. Cash is never touched and the account is not configurable. |
| **Notifications** | `Config.NotifySystem` in `shared/functions.lua` | `nexus_notify` (`exports['nexus_notify']:Alert(title, text, duration, type, playSound)` client-side, `nexus_notify:Alert` event server-side), `okokNotify`, native ESX (`esx:showNotification`), native QBCore (`QBCore:Notify`), `mythic_notify`, or `custom`. Both a client helper (`Functions.Notify`) and a server helper (`Functions.NotifyServer`) are provided; the resource only ever calls the server one. |
| **Jobs** | `Config.PoliceJobName` | Only used for *creating* `Policial` contracts. |
| **Death detection** | Framework-native | ESX `esx:onPlayerDeath` (reads `data.killerId`), QBCore `QBCore:Server:OnPlayerDeath` (reads `deathData.killerId`). If you run a custom ambulance/death resource that does not fire these, payouts will not trigger. |
| **TextUI / target / keys / fuel / inventory** | **Not used** | The tablet is a standalone NUI opened by a chat command. No item is consumed, no target system, no interaction prompt. |
| **OneSync** | Compatible | Only used for per-player and broadcast client events; no entity state is managed except the debug test NPC. |

## 💻 Developer API

### Client Exports

**None.** `nexus_bounty` declares no `exports` on the client.

### Server Exports

**None.** `nexus_bounty` declares no `exports` on the server. Everything is reached through events and framework callbacks.

### Framework Server Callbacks

Registered through `RegisterCallback`, which maps to `ESX.RegisterServerCallback` or `QBCore.Functions.CreateCallback` depending on `Config.Framework`. Any resource using the same framework can call them.

| Callback | Parameters | Returns | Description |
|---|---|---|---|
| `nexus_bounty:server:getActiveBounties` | — | array of `bounties_active` rows | Every contract where `fecha_caducidad > NOW()`. Each row is enriched with `target_name` (character name, the NPC mission name, or `"Desconocido"`) and `is_online` (boolean), and `dificultad` is coerced to a number. Offline target names are resolved with a second batched query against `users` / `players`. |
| `nexus_bounty:server:getLeaderboard` | — | array of `bounty_leaderboard` rows | Top 20 ordered by `total_cobrado DESC`. Raw rows: `steamid`, `nombre_personaje`, `total_cobrado`, `contratos_completados`. |
| `nexus_bounty:server:getMyBounties` | — | array of rows | Live contracts where `contratador_identifier` = the caller's identifier, enriched the same way as `getActiveBounties`. |
| `nexus_bounty:server:getAcceptedBounties` | — | array of rows | Live contracts whose `cazadores_activos` JSON array contains the caller's identifier, enriched the same way. |
| `nexus_bounty:server:getPlayerStats` | — | `{ bountiesCreated, totalBountyValue, bountiesCompleted, totalEarned }` | `bountiesCreated`/`totalBountyValue` are `COUNT(*)`/`SUM(monto)` over all rows the caller created (note: **no expiry filter**, so this counts expired-but-not-yet-deleted rows). `bountiesCompleted`/`totalEarned` come from `bounty_leaderboard`. |
| `nexus_bounty:server:getHomescreenStats` | — | `{ onlinePlayers, activeContracts }` | `onlinePlayers` is `#GetAllPlayers()`; `activeContracts` is a `COUNT(*)` of non-expired contracts. |

```lua
-- Reading the leaderboard from your own resource (ESX)
ESX.TriggerServerCallback('nexus_bounty:server:getLeaderboard', function(rows)
    for i, r in ipairs(rows) do
        print(('#%d %s — %d contracts, $%d'):format(
            i, r.nombre_personaje, r.contratos_completados, r.total_cobrado))
    end
end)
```

### Events — Emitted

Server → client:

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_bounty:client:startTracking` | server → one client | `bountyInfo` = `{ bountyId = <number>, isNpc = false, targetServerId = <number> }` | A hunter accepted a contract whose target is currently online. Starts the proximity-blip loop and closes the tablet. |
| `nexus_bounty:client:stopTracking` | server → client | — | Clears tracking and removes the blip. Handler exists; not currently fired by the shipped server code. |
| `nexus_bounty:client:stopTrackingBounty` | server → all clients (`-1`) | `bountyId` (number) | A specific contract expired. Only clients tracking that exact `bountyId` clear their state. |
| `nexus_bounty:client:refreshMyPanel` | server → one client | — | After a successful edit or delete; tells the NUI to re-pull `getMyBounties`, `getAcceptedBounties` and `getPlayerStats`. |
| `nexus_bounty:client:refreshAllPanels` | server → all clients (`-1`) | — | A contract expired; every client with the tablet open refreshes. |
| `nexus_bounty:client:spawnTestNPC` | server → one client | — | `Config.DebugMode` only. Spawns the test ped and starts the death watcher. |

### Events — Listened

Client → server:

| Event | Payload | Purpose |
|---|---|---|
| `nexus_bounty:server:createBounty` | `data` = `{ targetName, amount, imageUrl, extraInfo, duration, type, difficulty }` | Full create flow: validate amount → validate `type` ∈ {`Civil`,`Policial`} → validate `duration` against `Config.PVP.Durations` → `RemoveMoney` → resolve target online, else in the DB → validate police job for `Policial` → `INSERT` → schedule expiry → notify creator and (if online) target. Every failure after the withdrawal calls `AddMoney` to refund. |
| `nexus_bounty:server:acceptBounty` | `bountyId` (number) | Loads the contract, refuses if the caller is the creator, appends the caller's identifier to the `cazadores_activos` JSON array, and starts tracking if the target is online. |
| `nexus_bounty:server:editBounty` | `data` = `{ id, imageUrl, extraInfo }` | Verifies `contratador_identifier` ownership, then `UPDATE`s **only** `url_imagen` and `info_extra`. |
| `nexus_bounty:server:deleteBounty` | `data` = `{ id }` | Verifies ownership, `DELETE`s the row and refunds the full `monto` via `AddMoney`. |
| `nexus_bounty:server:createTestBounty` | `npcIdentifier` (string), `coords` (vector3) | Inserts a hardcoded $10,000 / 1-hour / difficulty-3 NPC contract. ⚠️ **Registered unconditionally, outside the `Config.DebugMode` guard.** |
| `nexus_bounty:server:claimTestBounty` | `npcIdentifier` (string) | Pays `bounty.monto` if the caller is on the hunter list, deletes the row, upserts the leaderboard. ⚠️ **Also registered unconditionally.** See the incident report — this pair is exploitable in production. |

Framework events consumed: `esx:getSharedObject` / `QBCore:Ready` (init), `esx:onPlayerDeath` / `QBCore:Server:OnPlayerDeath` (payout).

### NUI callbacks (client ← NUI)

All registered inside `InitializeClientScript()` in `client/main.lua`, so they only exist once the framework has loaded.

| Callback | Body | Action |
|---|---|---|
| `close` | — | Hides the tablet, releases NUI focus. |
| `getActiveBounties` | — | Framework callback → `updateBounties` NUI message. |
| `getLeaderboard` | — | → `updateLeaderboard`. |
| `getMyBounties` | — | → `updateMyBounties`. |
| `getAcceptedBounties` | — | → `updateAcceptedBounties`. |
| `getPlayerStats` | — | → `updatePlayerStats`. |
| `getHomescreenStats` | — | → `updateHomescreenStats`. |
| `createBounty` | `{ targetName, amount, imageUrl, extraInfo, duration, type, difficulty }` | Fires `nexus_bounty:server:createBounty`. |
| `acceptBounty` | `{ id }` | Fires `nexus_bounty:server:acceptBounty` (only if `data.id` is truthy). |
| `editBounty` | `{ id, imageUrl, extraInfo }` | Fires `nexus_bounty:server:editBounty`. |
| `deleteBounty` | `{ id }` | Fires `nexus_bounty:server:deleteBounty`. |

### NUI messages (client → NUI)

Dispatched on `event.data.action`:

| `action` | Fields | Meaning |
|---|---|---|
| `loadConfig` | `data` = the whole `Config` table | Sent on every open, before `setVisible`. The NUI keeps `config` (used for `PVP.MinBountyAmount` and `PVP.Durations`) and `locales` (stored but never read). |
| `setVisible` | `status` (boolean) | Show/hide the tablet with the scale+fade transition. On `true` the NUI also pulls `getHomescreenStats`. |
| `updateBounties` | `data` = rows | Render the active-contracts list. |
| `updateLeaderboard` | `data` = rows | Render the leaderboard table. |
| `updateMyBounties` | `data` = rows | Render "Mis Recompensas" (also caches them so edit/delete modals can look a contract up by id). |
| `updateAcceptedBounties` | `data` = rows | Render "Contratos Aceptados" (also used to decide whether the Accept button shows). |
| `updatePlayerStats` | `data` = stats object | Fill the four stat cards. |
| `updateHomescreenStats` | `data` = `{ onlinePlayers, activeContracts }` | Update the two header badges. |
| `refreshMyPanel` | — | Re-pull all three panel datasets. |

> Note: `nexus_bounty:client:refreshAllPanels` also sends `{ action = 'getActiveBounties' }`, but the NUI's `action` switch has no case for it (its `getActiveBounties` handler is registered on `event.data.type`, not `action`), so that half of the refresh is a no-op. Logged in the incident report.

### Editable Functions (`shared/functions.lua`)

The only unencrypted code file. It defines the `Functions` table.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `Functions.Notify` | `Functions.Notify(title, msg, type, time)` | **Nowhere in the shipped code.** | The client-side notification helper. Provided for your own use if you extend the resource. Branches on `Config.NotifySystem`. ⚠️ Its `esx` branch calls a global `ESX`, which this resource never defines — it will error if you call it with `Config.NotifySystem = "esx"`. |
| `Functions.NotifyServer` | `Functions.NotifyServer(player, title, msg, type, time)` | Every notification in `server/main.lua` (≈20 call sites) | Sends a notification to one player from the server. `player` is a server ID, `type` is one of `error` / `success` / `warning` / `info` (mapped per notify system), `time` is milliseconds and defaults to `5000`. Add your own resource in the `custom` branch. |
| `Functions.TerminarContrato` | `Functions.TerminarContrato(hunterSource, bountyData)` | `ProcessBountyPayment`, as the **last** step of a successful payout (after the money, the notifications, the row deletion and the leaderboard upsert) | **The main integration hook.** Empty by default. `hunterSource` is the killer's server ID; `bountyData` is the complete `bounties_active` row that was just claimed (`id`, `objetivo_steamid`, `monto`, `contratador_nombre`, `contratador_identifier`, `tipo`, `dificultad`, `url_imagen`, `info_extra`, `cazadores_activos`, `fecha_caducidad`, `is_npc_bounty`, `npc_template_id`, `npc_bounty_name`). Put Discord webhooks, item rewards, XP, criminal-record entries or anything else here. No return value is used. ⚠️ Its body reads `Config.Debug`, which does not exist in `config.lua` (the real key is `Config.DebugMode`) — harmless while the function is empty, but fix it if you add code that relies on that guard. |

### Database Schema

Created by `nexus_bounty.sql`.

```sql
CREATE TABLE IF NOT EXISTS `bounties_active` (
  `id`                     INT(11) NOT NULL AUTO_INCREMENT,
  `objetivo_steamid`       VARCHAR(60)  DEFAULT NULL COMMENT 'Target player identifier (PvP contracts)',
  `monto`                  INT(11) NOT NULL,
  `contratador_nombre`     VARCHAR(255) NOT NULL,
  `contratador_identifier` VARCHAR(60)  DEFAULT NULL COMMENT 'Creator identifier (ownership checks)',
  `tipo`                   ENUM('Civil','Policial') NOT NULL,
  `dificultad`             TINYINT(1) NOT NULL DEFAULT 1,
  `url_imagen`             TEXT DEFAULT NULL COMMENT 'May hold several comma-separated URLs',
  `info_extra`             TEXT DEFAULT NULL,
  `cazadores_activos`      TEXT COMMENT 'JSON array of hunter identifiers',
  `fecha_caducidad`        TIMESTAMP NOT NULL,
  `is_npc_bounty`          BOOLEAN NOT NULL DEFAULT FALSE,
  `npc_template_id`        INT(11) DEFAULT NULL COMMENT 'Config.PVE template id',
  `npc_bounty_name`        VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  INDEX `idx_objetivo`  (`objetivo_steamid`),
  INDEX `idx_caducidad` (`fecha_caducidad`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS `bounty_leaderboard` (
  `steamid`               VARCHAR(60)  NOT NULL COMMENT 'Player identifier',
  `nombre_personaje`      VARCHAR(255) NOT NULL,
  `total_cobrado`         BIGINT(20) NOT NULL DEFAULT 0,
  `contratos_completados` INT(11)    NOT NULL DEFAULT 0,
  PRIMARY KEY (`steamid`),
  INDEX `idx_total_cobrado` (`total_cobrado` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

The file also ships an upgrade statement for installs predating `contratador_identifier`:

```sql
ALTER TABLE `bounties_active`
  ADD COLUMN IF NOT EXISTS `contratador_identifier` VARCHAR(60) DEFAULT NULL
  AFTER `contratador_nombre`;
```

Notes for integrators:
- `objetivo_steamid` holds an ESX `identifier` or a QBCore `citizenid`, not a Steam ID, despite the name. For NPC/test contracts it holds the generated NPC identifier instead.
- `cazadores_activos` is a JSON array of identifier strings; append to it rather than replacing it.
- Rows are **deleted** on payout and on expiry — there is no history table. If you want a permanent audit trail, write it from `Functions.TerminarContrato`.
- `bounty_leaderboard` is append-only in practice and is the only persistent record of completed contracts.

### Integration Example

A Discord-webhook logger plus an item reward on every claimed contract, written entirely in the unencrypted `shared/functions.lua`:

```lua
-- shared/functions.lua
local WEBHOOK = 'https://discord.com/api/webhooks/xxx/yyy'

function Functions.TerminarContrato(hunterSource, bountyData)
    if not IsDuplicityVersion() then return end   -- server only

    local hunterName = GetPlayerName(hunterSource)

    -- 1) Discord log
    PerformHttpRequest(WEBHOOK, function() end, 'POST', json.encode({
        embeds = {{
            title = 'Bounty claimed',
            color = 9643754,   -- Nexus purple
            fields = {
                { name = 'Hunter',     value = hunterName,                    inline = true },
                { name = 'Reward',     value = '$' .. bountyData.monto,       inline = true },
                { name = 'Type',       value = bountyData.tipo,               inline = true },
                { name = 'Contractor', value = bountyData.contratador_nombre, inline = true },
                { name = 'Target id',  value = bountyData.objetivo_steamid or 'NPC' },
                { name = 'Contract',   value = '#' .. bountyData.id,          inline = true },
            }
        }}
    }), { ['Content-Type'] = 'application/json' })

    -- 2) Bonus item for high-difficulty contracts
    if tonumber(bountyData.dificultad) and tonumber(bountyData.dificultad) >= 4 then
        exports.ox_inventory:AddItem(hunterSource, 'goldbar', 1)
    end

    -- 3) Your own permanent history table
    MySQL.Async.execute(
        'INSERT INTO bounty_history (bounty_id, hunter_identifier, amount, claimed_at) VALUES (@b, @h, @a, NOW())',
        { ['@b'] = bountyData.id, ['@h'] = GetPlayerIdentifier(hunterSource, 0), ['@a'] = bountyData.monto }
    )
end
```

Reading live contract state from an unrelated resource (e.g. a dispatch or MDT script):

```lua
-- server side of your resource, ESX
ESX.TriggerServerCallback('nexus_bounty:server:getActiveBounties', function(rows)
    for _, b in ipairs(rows) do
        if b.tipo == 'Policial' then
            print(('[MDT] Police contract #%d on %s — $%d, expires %s')
                :format(b.id, tostring(b.target_name), b.monto, tostring(b.fecha_caducidad)))
        end
    end
end)
```

Reacting to expiry on the client (e.g. to clear your own custom marker):

```lua
RegisterNetEvent('nexus_bounty:client:stopTrackingBounty', function(bountyId)
    print(('[addon] contract %s expired'):format(bountyId))
end)
```

## ❓ FAQ

**The tablet doesn't open and `/recompensas` does nothing.**
The command is registered inside `InitializeClientScript()`, which only runs once the framework reports ready. On ESX that is the `esx:getSharedObject` callback; on QBCore it is the `QBCore:Ready` **event** — which means if `nexus_bounty` starts *after* QBCore has already fired `QBCore:Ready`, the handler never runs and nothing initialises. Make sure `qb-core` is ensured before `nexus_bounty`, and check `Config.Framework` matches the framework you actually run.

**"Objetivo no encontrado. Sé más específico."** — but the player is right there.
The search must resolve to **exactly one** person. With `Config.UseApproximateNameSearch = true`, a partial name that matches two characters is rejected, not disambiguated. Type more of the name, or set `Config.UseApproximateNameSearch = false` and use the full exact name. Also remember the name compared is the **character** name (ESX `firstname lastname`, QBCore `charinfo.firstname/lastname`), not the Steam/Discord name.

**I killed the target but got no money.**
Three things must all be true: you **accepted** the contract from the tablet first (random kills never pay), the contract has not expired, and your framework fires the standard death event (`esx:onPlayerDeath` / `QBCore:Server:OnPlayerDeath`) with a valid `killerId`. A custom ambulance or death-handling resource that does not fire these will silently break every payout.

**Money is taken from the bank, not cash.**
By design and not configurable: both the charge and the payout use the bank account on both frameworks.

**Contracts never expire / expired contracts are still listed.**
Expiry uses `os.time()` on the OS clock and `NOW()` / `FROM_UNIXTIME()` on the MySQL clock. If your MySQL server's timezone differs from the game server's OS timezone, those two disagree and contracts appear to expire early or late. Make sure both are on the same timezone (UTC is the easy answer). Also note the list query filters on `fecha_caducidad > NOW()`, so a row whose timer has not fired yet is still correctly hidden.

**A contract on an offline player never notifies them when they log in.**
Correct — there is no login hook. The target is only notified if they were online at the moment the contract was created. The contract itself is perfectly valid and will pay out whenever they are killed by an accepted hunter.

**"Solo la policía puede crear contratos de este tipo."**
`Config.PoliceJobName` must match your job name exactly, including case, as your framework stores it. ESX typically uses `police`; QBCore servers often use `police` or `leo`.

**Non-police players can accept police contracts.**
Yes — in this version acceptance is not job-gated. The `must_be_police_to_accept` locale string exists but is never used. Creation is restricted; acceptance is not. See the incident report.

**The target's name shows as blank / `undefined` in the list.**
Report this with your framework and MySQL driver versions — there is a known type-coercion issue around the `is_npc_bounty` column (a MySQL `BOOLEAN` is a `TINYINT(1)`, and in Lua the number `0` is truthy, so a PvP row can be misread as an NPC row and pick up the empty `npc_bounty_name`). It is documented in the incident report.

**The whole tablet is in Spanish. How do I translate it?**
Only the notification strings in `Config.Locales` are usable — and those cover the server notifications, not the UI. Every label inside the tablet is hardcoded in `nui/index.html` and `nui/script.js`, which are inside the escrow. There is no supported translation path for the UI in this version.

**My NPC/PvE contracts never appear even with `Config.PVE.Enabled = true`.**
The PvE generator is not implemented in this version; `Config.PVE` is read by no code. The only way to get an NPC contract is the debug-only `/test_bounty_npc` command.

**`/test_bounty_npc` says "Unknown command".**
It is only registered when `Config.DebugMode = true`, and it requires a server restart of the resource after changing that. Note it has **no permission check** (the ace check is commented out in the source), so leave `DebugMode` off in production.

**Images don't load in the contract details.**
Use a direct image link (`https://i.imgur.com/xxxx.png`, `https://i.ibb.co/.../x.png`). A gallery or album link will not render. Bare `imgur.com/xxxx` links are auto-rewritten to the `i.imgur.com` form, but album and `/a/` links are not. Several comma-separated URLs become a carousel.

**Can two hunters accept the same contract?**
Yes, any number. The first one to kill the target collects; the contract is then deleted for everyone.

### Before opening a ticket

- Make sure the resource folder name is exactly **`nexus_bounty`** — it refuses to start otherwise and says so in the server console.
- Make sure you are on the latest version of the resource (**1.0.0** at the time of writing).
- Confirm `nexus_bounty.sql` was actually imported and both tables exist, `contratador_identifier` included.
- Confirm your MySQL resource and your framework both start **before** `nexus_bounty` in `server.cfg`.
- Re-read this FAQ page.
- Set `Config.DebugMode = true`, reproduce the problem, and attach the full server console output plus the F8 client log.

## 📋 Changelog

### 1.0.0 — Initial release
As declared in `fxmanifest.lua` (`version '1.0.0'`). No earlier version history ships with the resource, although `nexus_bounty.sql` contains an `ALTER TABLE` migration adding `contratador_identifier` for anyone upgrading from a pre-release build that lacked it.
- Custom tablet NUI with home screen, create-contract form, active-contracts browser, personal dashboard and leaderboard.
- Civilian and police-only PvP contracts, with police creation gated on `Config.PoliceJobName`.
- Online and offline target resolution against the framework player table, with exact or approximate name matching.
- Edit (image + notes) and cancel-with-refund for your own contracts, both ownership-verified server-side.
- Proximity-based target blips with reveal distance, duration and a fixed 30-second cooldown.
- Timer-based automatic expiry, re-scheduled for all live contracts on resource start.
- Death-event payout restricted to hunters who formally accepted the contract, with leaderboard upsert.
- Live home-screen metrics refreshed every 30 seconds.
- ESX and QBCore support behind a single wrapper layer; six notification backends via `Config.NotifySystem`.
- Debug-only `/test_bounty_npc` end-to-end test workflow.
