# Nexus Bounty

> A bounty-hunter tablet for FiveM: put a price on a player's head, let hunters take the contract, track the target and cash in. For ESX and QBCore.

**Version:** 1.1.0 · **Frameworks:** ESX, QBCore · **Database:** oxmysql

---

## Description

`nexus_bounty` adds an in-game tablet (opened with `/bounty` by default) where players create, browse and manage player-vs-player bounty contracts.

A player picks a target by name, sets a reward (paid upfront from their bank account), a duration, a contract type (Civil or Police), an optional picture and some roleplay clues. The contract shows up on the tablet for every hunter in the city. Hunters accept it, get a short-lived blip on the target whenever they get close, and when they kill the target the reward is paid straight into their bank account and the hunter climbs the server leaderboard.

Contract creators can edit the picture and clues of their contracts or cancel them for a full refund. Police contracts can only be created and accepted by officers. Contracts expire automatically when their time runs out.

---

## Features

- **Tablet UI** — homescreen with live counters (players online, active contracts) and four apps: Create Contract, Active Contracts, My Panel, Leaderboard.
- **Name search with anti-metagaming** — targets are searched by character name, online first and then in the database (offline players). Approximate (partial) or exact matching via `Config.UseApproximateNameSearch`.
- **Upfront payment** — the reward is taken from the creator's bank account when the contract is created and refunded if the target can't be found or the contract is cancelled.
- **Configurable durations** — the list of durations players can choose from is defined in `Config.PVP.Durations`.
- **Civil and Police contracts** — Police contracts can only be created **and accepted** by players with `Config.PoliceJobName`.
- **Difficulty rating** — the creator rates the target from 1 to 5 stars as a guide for hunters.
- **Pictures and clues** — one or more image URLs (comma-separated, Imgur links are normalised) and free-text clues, shown with a carousel in the contract details.
- **Proximity tracking** — after accepting, the hunter gets a blip on the target for `Config.PVP.BlipDuration` whenever they are within `Config.PVP.BlipRevealDistance`, with a 30-second cooldown between reveals.
- **Automatic payout** — kills are detected by the script itself (no death-script integration needed). When a hunter who accepted the contract kills the target, the reward is paid, both players are notified, the contract is closed and the leaderboard is updated. If the target had several contracts accepted by that hunter, all of them are paid, and each contract can only ever be paid once.
- **My Panel** — personal stats (bounties created, total offered, contracts completed, total earned), list of own contracts with edit / delete, and list of accepted contracts.
- **Leaderboard** — top 20 hunters by total earned.
- **Automatic expiry** — contracts are removed when they expire, also after a server restart.
- **6 notification systems** — nexus_notify, okokNotify, ESX, QBCore, mythic_notify or your own.
- **7 languages** — `en`, `es`, `de`, `fr`, `it`, `pt`, `zh` for both notifications and the tablet (English by default).
- **Secure by design** — server-side validation of amount, type, duration, ownership, police job and contract expiry; tablet output is escaped; all SQL is parameterised.

---

## Dependencies

### Required

| Resource | Notes |
|---|---|
| `es_extended` **or** `qb-core` | Selected with `Config.Framework`. |
| `oxmysql` | Database driver. |

### Optional

| Resource | Notes |
|---|---|
| `nexus_notify` | Default notification system (`Config.NotifySystem = "nexus"`). |
| `okokNotify` / `mythic_notify` | Only if selected in `Config.NotifySystem`. |

### Server requirements

| Requirement | Notes |
|---|---|
| MySQL 5.7+ / MariaDB 10.2.3+ | JSON functions are used to register hunters on a contract. |
| MySQL 8.0.29+ / MariaDB 10.3+ | Only for the optional `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` upgrade line in the SQL file. |
| OneSync | Recommended (target tracking uses the target's server ID). |

---

## Installation

1. **Download** the resource from your Cfx.re Keymaster (Granted Assets).
2. **Extract** it into your resources folder. The folder **must** be named exactly `nexus_bounty`.
3. **Import the SQL**: run `nexus_bounty.sql` in your database (`CREATE TABLE IF NOT EXISTS`, safe to re-run).
4. **Add to `server.cfg`** after your framework and oxmysql:

```cfg
ensure oxmysql
ensure es_extended   # or qb-core
ensure nexus_notify  # if you use it
ensure nexus_bounty
```

5. **Configure** `shared/config.lua` (framework, language, police job, minimum reward, durations...).
6. Restart the server. If you update the files while the server is running, run `refresh` before `ensure nexus_bounty`, otherwise FiveM keeps using the old file list.

### Updating from 1.0.0

- Replace the resource, keeping your `shared/config.lua` and `shared/functions.lua` if customised.
- `mysql-async` is no longer used: make sure `oxmysql` is started.
- `Config.Locales` has been removed. All texts now live in `locales/*.lua`; move any custom wording there and set `Config.Locale`.
- `Config.PVE` has been removed.
- The default language is now English and the default command is `/bounty`. Set `Config.Locale = 'es'` and `Config.OpenCommand = "recompensas"` to keep the previous behaviour.
- The 1-minute TEST duration was removed from the default `Config.PVP.Durations`.
- No database changes are required.
- Run `refresh` before `ensure nexus_bounty` (new files were added).

---

## Configuration

All options are in `shared/config.lua` (open file).

| Option | Default | Description |
|---|---|---|
| `Config.Framework` | `"ESX"` | `"ESX"` or `"QBCore"`. |
| `Config.Locale` | `'en'` | `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`. Missing keys fall back to English. |
| `Config.DebugMode` | `false` | Prints debug info in the server console and enables the test command `/test_bounty_npc` and its test events. **Keep `false` in production.** |
| `Config.NotifySystem` | `"nexus"` | `nexus`, `okok`, `esx`, `qb`, `mythic`, `custom` (write your own in `shared/functions.lua`). |
| `Config.OpenCommand` | `"bounty"` | Chat command that opens / closes the tablet. |
| `Config.PoliceJobName` | `"police"` | Job required to create and accept Police contracts. |
| `Config.UseApproximateNameSearch` | `true` | `true`: partial name match (`LIKE`). `false`: exact full name. A contract is only created when exactly one character matches. |
| `Config.PVP.Enabled` | `true` | Reserved. |
| `Config.PVP.MinBountyAmount` | `5000` | Minimum reward per contract. |
| `Config.PVP.Durations` | 1 h, 3 h, 12 h, 24 h | `{ label = "...", hours = n }` entries offered in the tablet. Only these values are accepted by the server. |
| `Config.PVP.EnableTracking` | `true` | Enables the proximity blip for hunters. |
| `Config.PVP.BlipRevealDistance` | `200.0` | Distance (game units) at which the target is revealed. |
| `Config.PVP.BlipDuration` | `10000` | Time in ms the blip stays on the map per reveal. |

### Example

```lua
Config.Framework = "QBCore"
Config.Locale = 'en'
Config.NotifySystem = "qb"
Config.OpenCommand = "bounty"
Config.PoliceJobName = "police"

Config.PVP = {
    Enabled = true,
    MinBountyAmount = 10000,
    Durations = {
        { label = "2 Hours", hours = 2 },
        { label = "6 Hours", hours = 6 },
        { label = "24 Hours", hours = 24 },
    },
    EnableTracking = true,
    BlipRevealDistance = 150.0,
    BlipDuration = 8000
}
```

---

## Locales & Editable Strings

| File | Contents |
|---|---|
| `shared/config.lua` | All options and duration labels. |
| `locales/*.lua` | Notifications and every tablet text, in 7 languages. |
| `shared/locales.lua` | The `_T(key, ...)` translator (fallback: English) and the helper that sends tablet texts to the NUI. |
| `shared/functions.lua` | Notification bridge and the payout hook. |
| `nui/index.html`, `nui/script.js`, `nui/style.css` | The tablet. Texts are loaded from the active locale; in JS use `_t('key')`. |

Each locale file has two parts: top-level keys (notifications, map blip) and a `ui` table (tablet texts).

**Notification keys**

| Key | English |
|---|---|
| `notify_title` | Bounties |
| `not_enough_money` | You do not have enough money. |
| `player_not_found` | Target not found. Be more specific. |
| `invalid_amount` | The bounty amount is invalid. |
| `invalid_type` | Invalid contract type. |
| `invalid_duration` | Invalid contract duration. |
| `bounty_created_successfully` | You placed a $%s bounty on %s. |
| `bounty_create_error` | The bounty could not be created. |
| `you_have_a_bounty` | Watch out! Someone has placed a $%s bounty on your head. |
| `bounty_accepted` | Contract accepted. The target has been marked. |
| `bounty_unavailable` | This contract is no longer available. |
| `already_accepted` | You have already accepted this contract. |
| `cant_accept_own_bounty` | You cannot accept a bounty you created. |
| `police_only_bounty` | Only police officers can create this type of contract. |
| `must_be_police_to_accept` | This is a police contract. Only officers can accept it. |
| `target_eliminated` | Target neutralised! You earned $%s. |
| `your_bounty_claimed` | A hunter has claimed the bounty on your head. |
| `bounty_not_found` | Bounty not found. |
| `cant_edit_bounty` | You cannot edit this bounty. |
| `bounty_updated` | Bounty updated successfully. |
| `bounty_update_error` | The bounty could not be updated. |
| `cant_delete_bounty` | You cannot delete this bounty. |
| `bounty_deleted_refund` | Bounty deleted. $%s has been refunded. |
| `bounty_delete_error` | The bounty could not be deleted. |
| `unknown_target` | Unknown |
| `blip_target` | Bounty Target *(map blip name)* |

**Tablet keys (`ui`)** — 73 keys grouped by screen: homescreen (`app_*`, `card_*`), create (`create_title`, `ph_*`, `type_civil`, `type_police`, `difficulty_label`, `btn_create`, `sidebar_*`, `tip_1..3`), active list (`active_*`, `col_*`, `status_*`, `no_active`), My Panel (`panel_title`, `stat_*`, `tab_*`, `no_my`, `no_accepted`), leaderboard (`leaderboard_*`, `col_rank`, `col_hunter`, `col_contracts`, `col_earned`), contract details (`detail_title`, `lbl_*`, `no_info`, `btn_accept`, `accepted_*`), edit / delete modals (`edit_title`, `ph_edit_info`, `btn_cancel`, `btn_save`, `delete_*`, `btn_delete`). See `locales/en.lua` for the full list.

`%s` placeholders are filled in order. If the selected language is missing a key, the English text is used.

---

## Compatibility

| System | Supported |
|---|---|
| Frameworks | ESX Legacy (`getSharedObject` export, falls back to the old shared-object event), QBCore |
| Death / ambulance scripts | Any — kills are detected by the script itself |
| Database | oxmysql |
| Notifications | nexus_notify, okokNotify, ESX, QBCore, mythic_notify, custom |
| Money | Bank account (`bank`) on both frameworks |

**Identifiers**

| Data | ESX | QBCore |
|---|---|---|
| Player identifier | `xPlayer.identifier` | `PlayerData.citizenid` |
| Offline name lookup | `users.firstname` + `lastname` | `players.charinfo` (JSON) |
| Job check | `xPlayer.job.name` | `PlayerData.job.name` |

---

## Developer API

### Commands

| Command | Who | Description |
|---|---|---|
| `/bounty` (`Config.OpenCommand`) | Player | Opens / closes the tablet. |
| `/test_bounty_npc` | Anyone, **only when `Config.DebugMode = true`** | Spawns a static test ped in front of you and creates a $10,000 test contract on it. Development use only. |

### Client Exports

None.

### Server Exports

None. Use the events and the database tables below to integrate.

### Events Emitted

**Client events (server → client)**

| Event | Arguments | Description |
|---|---|---|
| `nexus_bounty:client:startTracking` | `{ bountyId, isNpc, targetServerId }` | Sent to a hunter after accepting a contract whose target is online. Starts proximity tracking and closes the tablet. |
| `nexus_bounty:client:stopTracking` | — | Stops tracking and removes the blip. |
| `nexus_bounty:client:stopTrackingBounty` | `bountyId` | Sent to everyone when a contract expires or is paid; stops tracking if it matches. |
| `nexus_bounty:client:refreshMyPanel` | — | Refreshes My Panel after an edit or delete. |
| `nexus_bounty:client:refreshAllPanels` | — | Sent to everyone when a contract expires or is paid; refreshes open tablets. |

**Server callbacks** (ESX `RegisterServerCallback` / QBCore `CreateCallback`)

| Callback | Returns |
|---|---|
| `nexus_bounty:server:getHomescreenStats` | `{ onlinePlayers, activeContracts }` |
| `nexus_bounty:server:getActiveBounties` | Array of active `bounties_active` rows + `target_name`, `is_online` |
| `nexus_bounty:server:getMyBounties` | Same shape, only contracts created by the caller |
| `nexus_bounty:server:getAcceptedBounties` | Same shape, only contracts the caller accepted |
| `nexus_bounty:server:getLeaderboard` | Top 20 `bounty_leaderboard` rows |
| `nexus_bounty:server:getPlayerStats` | `{ bountiesCreated, totalBountyValue, bountiesCompleted, totalEarned }` |

### Events Listened

**Server events (from the script's own tablet)** — every field is validated server-side.

| Event | Payload | Behaviour |
|---|---|---|
| `nexus_bounty:server:createBounty` | `{ targetName, amount, imageUrl, extraInfo, duration, type, difficulty }` | Validates amount ≥ `MinBountyAmount`, `type` (`Civil` / `Policial`), `duration` (must match `Config.PVP.Durations`) and police job for Police contracts; charges the bank; finds exactly one matching character; inserts the contract; notifies the target if online. |
| `nexus_bounty:server:acceptBounty` | `bountyId` | Rejects expired contracts, own contracts, Police contracts for non-officers and hunters already on the contract. Registers the hunter atomically. |
| `nexus_bounty:server:editBounty` | `{ id, imageUrl, extraInfo }` | Creator only. |
| `nexus_bounty:server:deleteBounty` | `{ id }` | Creator only. Refunds the full reward. |
| `nexus_bounty:server:playerKilled` | `killerServerId` | Sent by the victim's own client when it dies from another player (`CEventNetworkEntityDamage`). The server checks both players exist and pays every active contract on the victim that the killer accepted. |
| `nexus_bounty:server:createTestBounty` / `claimTestBounty` | — | **Only registered when `Config.DebugMode = true`.** |

**Framework hooks (server)**

| Event | Arguments used | Description |
|---|---|---|
| `esx:onPlayerDeath` (ESX) | `source` = victim, `data.killerServerId` (or `data.killerId`) | Extra payout source on ESX. |
| `QBCore:Server:OnPlayerDeath` (QBCore) | `victimPlayer.source`, `deathData.killerId` | Optional extra payout source. Not emitted by default qb-core and not needed. |

All payout sources share the same claim: the contract row is deleted first and the reward is only paid if that delete succeeded, so a kill reported twice is never paid twice.
| `QBCore:Ready` (QBCore) | — | Optional; the script also initialises as soon as `qb-core` is started. |

### Editable Functions

`shared/functions.lua` — loaded on client and server, outside the escrow.

| Function | Side | Description |
|---|---|---|
| `Functions.Notify(title, msg, type, time)` | Client | Shows a notification with `Config.NotifySystem`. Add your system in the `custom` branch. |
| `Functions.NotifyServer(player, title, msg, type, time)` | Server | Sends a notification to a player with `Config.NotifySystem`. |
| `Functions.TerminarContrato(hunterSource, bountyData)` | Server | Hook called after a contract is paid. `bountyData` is the full `bounties_active` row (`id`, `monto`, `objetivo_steamid`, `contratador_identifier`, `tipo`...). Use it for logs, items, XP... |

```lua
function Functions.TerminarContrato(hunterSource, bountyData)
    print(('Hunter %s claimed bounty #%s for $%s'):format(hunterSource, bountyData.id, bountyData.monto))
end
```

### Database Schema (SQL)

```sql
CREATE TABLE IF NOT EXISTS `bounties_active` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `objetivo_steamid` VARCHAR(60) DEFAULT NULL COMMENT 'Target player identifier',
  `monto` INT(11) NOT NULL,
  `contratador_nombre` VARCHAR(255) NOT NULL,
  `contratador_identifier` VARCHAR(60) DEFAULT NULL COMMENT 'Creator identifier (ownership check)',
  `tipo` ENUM('Civil', 'Policial') NOT NULL,
  `dificultad` TINYINT(1) NOT NULL DEFAULT 1,
  `url_imagen` TEXT DEFAULT NULL COMMENT 'One or more comma-separated image URLs',
  `info_extra` TEXT DEFAULT NULL,
  `cazadores_activos` TEXT COMMENT 'JSON array of hunter identifiers',
  `fecha_caducidad` TIMESTAMP NOT NULL,
  `is_npc_bounty` BOOLEAN NOT NULL DEFAULT FALSE COMMENT 'Reserved (debug test contracts only)',
  `npc_template_id` INT(11) DEFAULT NULL COMMENT 'Reserved',
  `npc_bounty_name` VARCHAR(255) DEFAULT NULL COMMENT 'Reserved (debug test contracts only)',
  PRIMARY KEY (`id`),
  INDEX `idx_objetivo` (`objetivo_steamid`),
  INDEX `idx_caducidad` (`fecha_caducidad`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS `bounty_leaderboard` (
  `steamid` VARCHAR(60) NOT NULL COMMENT 'Hunter identifier',
  `nombre_personaje` VARCHAR(255) NOT NULL,
  `total_cobrado` BIGINT(20) NOT NULL DEFAULT 0 COMMENT 'Total earned',
  `contratos_completados` INT(11) NOT NULL DEFAULT 0,
  PRIMARY KEY (`steamid`),
  INDEX `idx_total_cobrado` (`total_cobrado` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| Column | Meaning |
|---|---|
| `objetivo_steamid` | Target identifier (ESX identifier / QBCore citizenid). |
| `monto` | Reward. |
| `tipo` | `Civil` or `Policial` (shown as "Police" in English). |
| `dificultad` | 1–5 stars. |
| `cazadores_activos` | JSON array of hunter identifiers who accepted the contract. |
| `fecha_caducidad` | Expiry time. |
| `steamid` (leaderboard) | Hunter identifier (ESX identifier / QBCore citizenid). |

### Integration Example

**QBCore — report a kill manually from a custom server script (optional, kills are already detected automatically):**

```lua
-- when a player dies and you know the killer's server ID
local Victim = exports['qb-core']:GetCoreObject().Functions.GetPlayer(victimSrc)
TriggerEvent('QBCore:Server:OnPlayerDeath', Victim, { killerId = killerSrc })
```

**Check from another resource whether a player has a bounty on their head:**

```lua
local function GetActiveBounty(identifier)
    return MySQL.single.await('SELECT id, monto, tipo FROM bounties_active WHERE objetivo_steamid = ? AND fecha_caducidad > NOW()', { identifier })
end
```

**Give an item to the hunter on every completed contract (`shared/functions.lua`):**

```lua
function Functions.TerminarContrato(hunterSource, bountyData)
    exports.ox_inventory:AddItem(hunterSource, 'bounty_badge', 1)
end
```

---

## FAQ

**The tablet doesn't open.**
Check the command in `Config.OpenCommand` (default `/bounty`) and that the framework in `Config.Framework` matches your server.

**The hunter killed the target but wasn't paid.**
The hunter must have accepted the contract **before** the kill, the contract must not be expired, and the kill must be done by the hunter's own character (on foot or as a vehicle driver). Kills by NPCs, fire or falls don't count.

**F8 shows `attempt to call a nil value (global 'GetNuiLocales')` or the tablet shows raw keys.**
FiveM is still using the old file list. Run `refresh` and then `ensure nexus_bounty` (or restart the server). Make sure the `locales/` folder and `shared/locales.lua` were copied too.

**"Target not found" when creating a contract.**
The search must match exactly one character. Type more of the name, or set `Config.UseApproximateNameSearch = false` to require the exact full name. The money is refunded automatically.

**Can I target offline players?**
Yes. If nobody online matches, the database is searched. The target is notified only if online at creation time; tracking starts once you accept the contract while the target is online.

**Who can create and accept Police contracts?**
Only players whose job is `Config.PoliceJobName`.

**Can the same hunter accept a contract twice?**
No. Each hunter is registered once per contract; several different hunters can compete for the same contract.

**What happens to the money when a contract expires?**
The contract is closed and removed. The reward is refunded only when the creator cancels the contract from My Panel.

**How do I change the language or texts?**
Set `Config.Locale` and edit `locales/<lang>.lua`. Both notifications and the tablet use these files.

**Is there a PVE / NPC bounty mode?**
No. The only NPC-related code is the developer test command, available only with `Config.DebugMode = true`.

---

## Changelog

### 1.1.0
- **Fix:** payouts now work out of the box on ESX and QBCore. Kills are detected by the script itself; ESX's `killerServerId` is also read (the old code read a field ESX never sends).
- **Fix:** a contract can no longer be paid twice when the same death is reported more than once, and every contract on the target accepted by the killer is paid (not only the first one).
- **Fix:** ESX now uses the `getSharedObject` export (ESX Legacy), with fallback to the old event.
- **Fix:** hunters stop tracking and open tablets refresh when a contract is paid.
- **Fix:** `Functions.TerminarContrato` checked a non-existent `Config.Debug`; it now uses `Config.DebugMode`.
- **Fix:** clear console message (and no crash) if the locale files are not loaded.
- **Config:** removed the 1-minute TEST duration from the defaults.
- **Security:** the debug test events (`createTestBounty`, `claimTestBounty`) are now only registered when `Config.DebugMode = true`. Previously any client could create and claim test contracts in production.
- **Security:** all tablet output coming from players or the server (names, clues, image URLs, leaderboard names) is escaped to prevent HTML injection.
- **Security:** the offline-name lookups now use parameterised `IN (?, ?, ...)` queries.
- **Fix:** accepting a contract now checks the police job for Police contracts, rejects expired contracts and prevents the same hunter from being registered twice (atomic update).
- **Fix:** `is_npc_bounty` is compared explicitly, so player contracts are no longer treated as NPC contracts.
- **Fix:** QBCore initialisation no longer depends only on the `QBCore:Ready` event; the script starts as soon as `qb-core` is running.
- **Database:** migrated from `mysql-async` to `oxmysql`.
- **Locales:** new locale system (`locales/*.lua` + `_T()`), 7 languages for notifications and the whole tablet. The unused `Config.Locales` table was removed.
- **Language:** English is the default (config, server, client and tablet). Default command is now `/bounty`.
- **Config:** removed the unused `Config.PVE` block.
- **Escrow:** `shared/locales.lua`, `locales/*.lua` and NUI files added to `escrow_ignore`.

### 1.0.0
- Initial release.
