---
icon: crosshairs-simple
---

# Nexus Bounty

> A bounty-hunter tablet for FiveM: put a price on a player's head, let hunters take the contract, track the target and cash in. For ESX and QBCore.

**Version:** 1.1.0 · **Framework(s):** ESX, QBCore · **Database:** oxmysql

***

## Description

`nexus_bounty` adds an in-game tablet (opened with `/bounty` by default) where players create, browse and manage player-versus-player bounty contracts.

A player picks a target by name, sets a reward (paid upfront from their bank account), a duration, a contract type (Civil or Police), an optional picture and some roleplay clues. The contract shows up on the tablet for every hunter in the city. Hunters accept it, get a map blip on the target the moment they accept and again whenever they get close, and when they kill the target the reward is paid straight into their bank account and the hunter climbs the server leaderboard.

Contract creators can edit the picture and clues of their contracts or cancel them for a full refund. Police contracts can only be created and accepted by officers. Contracts expire automatically when their time runs out, also after a server restart.

***

## Features

* **Tablet UI**: homescreen with live counters (players online, active contracts) and four apps: Create Contract, Active Contracts, My Panel and Leaderboard.
* **Name search with anti-metagaming**: targets are searched by character name, online players first and then the database (offline players). Partial or exact matching with `Config.UseApproximateNameSearch`.
* **Upfront payment**: the reward is taken from the creator's bank account when the contract is created and refunded if the target cannot be found, the contract type is not allowed or the contract is cancelled.
* **Configurable durations**: the list of durations players can choose from is defined in `Config.PVP.Durations`.
* **Civil and Police contracts**: Police contracts can only be created **and accepted** by players with the job in `Config.PoliceJobName`.
* **Difficulty rating**: the creator rates the target from 1 to 5 stars as a guide for hunters.
* **Pictures and clues**: one or more image URLs (comma separated, Imgur links are normalised) and free-text clues, shown in a carousel in the contract details.
* **Target tracking**: when a hunter accepts a contract whose target is online, a red flashing blip with a route appears on the target for `Config.PVP.BlipDuration`. While the contract is accepted, the blip is shown again whenever the hunter is within `Config.PVP.BlipRevealDistance` of the target, with a 30 second cooldown between reveals.
* **Automatic payout**: kills are detected by the script itself, with no death-script integration needed. When a hunter who accepted the contract kills the target, the reward is paid, both players are notified, the contract is closed and the leaderboard is updated. If the target had several contracts accepted by that hunter, all of them are paid, and each contract can only be paid once.
* **My Panel**: personal stats (bounties created, total offered, contracts completed, total earned), the list of your own contracts with edit and delete, and the list of contracts you accepted.
* **Leaderboard**: top 20 hunters by total earned.
* **Automatic expiry**: contracts are removed when they expire.
* **6 notification systems**: nexus\_notify, okokNotify, ESX, QBCore, mythic\_notify or your own.
* **7 languages**: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh` for both notifications and the tablet. English is the default.
* **Secure by design**: server-side validation of amount, type, duration, ownership, police job and expiry; tablet output is escaped; every SQL query is parameterised.

***

## Dependencies

### Required

| Resource                       | Notes                             |
| ------------------------------ | --------------------------------- |
| `es_extended` **or** `qb-core` | Selected with `Config.Framework`. |
| `oxmysql`                      | Database driver.                  |

### Optional

| Resource                       | Notes                                                          |
| ------------------------------ | -------------------------------------------------------------- |
| `nexus_notify`                 | Default notification system (`Config.NotifySystem = "nexus"`). |
| `okokNotify` / `mythic_notify` | Only if selected in `Config.NotifySystem`.                     |

### Server requirements

| Requirement                   | Notes                                                                                  |
| ----------------------------- | -------------------------------------------------------------------------------------- |
| MySQL 5.7+ / MariaDB 10.2.3+  | JSON functions are used to register hunters on a contract.                             |
| MySQL 8.0.29+ / MariaDB 10.3+ | Only for the optional `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` line of the SQL file. |
| OneSync                       | Required for target tracking, which uses the position and server id of the target.     |

***

## Installation

1. **Download** the resource from your Cfx.re Keymaster (Granted Assets).
2. **Extract** it into your resources folder. The folder **must** be named exactly `nexus_bounty`.
3. **Import the SQL**: run `nexus_bounty.sql` in your database. It uses `CREATE TABLE IF NOT EXISTS`, so it is safe to run again.
4.  **Add it to `server.cfg`** after your framework and oxmysql:

    ```cfg
    ensure oxmysql
    ensure es_extended   # or qb-core
    ensure nexus_notify  # if you use it
    ensure nexus_bounty
    ```
5. **Configure** `shared/config.lua` (framework, language, police job, minimum reward, durations).
6. Restart the server. If you change files while the server is running, run `refresh` before `ensure nexus_bounty`, otherwise FiveM keeps using the old file list.

***

## Configuration

All options are in `shared/config.lua`, which is not encrypted.

| Option                            | Default              | Description                                                                                                           |
| --------------------------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `Config.Framework`                | `"ESX"`              | `"ESX"` or `"QBCore"`.                                                                                                |
| `Config.Locale`                   | `'en'`               | `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`. Missing keys fall back to English.                                          |
| `Config.DebugMode`                | `false`              | Prints debug information in the server console and registers the developer test tools. Keep it `false` in production. |
| `Config.NotifySystem`             | `"nexus"`            | `nexus`, `okok`, `esx`, `qb`, `mythic` or `custom` (write your own in `shared/functions.lua`).                        |
| `Config.OpenCommand`              | `"bounty"`           | Chat command that opens and closes the tablet.                                                                        |
| `Config.PoliceJobName`            | `"police"`           | Job required to create and accept Police contracts.                                                                   |
| `Config.UseApproximateNameSearch` | `true`               | `true`: partial name match. `false`: exact full name. A contract is only created when exactly one character matches.  |
| `Config.PVP.Enabled`              | `true`               | Reserved. Keep it `true`.                                                                                             |
| `Config.PVP.MinBountyAmount`      | `5000`               | Minimum reward per contract.                                                                                          |
| `Config.PVP.Durations`            | 1 h, 3 h, 12 h, 24 h | `{ label = "...", hours = n }` entries offered in the tablet. The server only accepts these values.                   |
| `Config.PVP.EnableTracking`       | `true`               | Enables the target blip for hunters.                                                                                  |
| `Config.PVP.BlipRevealDistance`   | `200.0`              | Distance in game units at which the target is revealed again after the first reveal.                                  |
| `Config.PVP.BlipDuration`         | `10000`              | Time in ms the blip stays on the map per reveal.                                                                      |

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

### Police job names

`Config.PoliceJobName` is compared with the exact job name of the player (`xPlayer.job.name` on ESX, `PlayerData.job.name` on QBCore). Only one job name is accepted. If your server uses another name for the police job, such as `lspd` or `leo`, set it here.

***

## How it works

1. **Create**: the creator types the name of the target, the reward, the type, the duration and the difficulty. The server validates everything, charges the bank account and searches the target. If exactly one character matches, the contract is inserted and the target is notified when online. In any other case the reward is refunded.
2. **Browse**: the Active Contracts app lists every contract that has not expired, with the target name, reward, type, difficulty and whether the target is online.
3. **Accept**: a hunter opens a contract and accepts it. The server registers the hunter on the contract. If the target is online, the hunter receives a blip with a route on the target, the tablet closes and tracking starts.
4. **Track**: the blip stays for `Config.PVP.BlipDuration`. Afterwards it is shown again each time the hunter gets within `Config.PVP.BlipRevealDistance` of the target, with a 30 second cooldown. Tracking stops when the target disconnects, when the contract expires, is cancelled or paid.
5. **Pay**: when the target dies by the hand of a hunter who accepted the contract, the contract is closed and the reward is paid to the hunter's bank account.

A hunter tracks one contract at a time. Accepting another contract replaces the current tracking.

***

## Locales and editable strings

| File                                               | Contents                                                                                                   |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `shared/config.lua`                                | All options and duration labels.                                                                           |
| `locales/*.lua`                                    | Notifications and every tablet text, in 7 languages.                                                       |
| `shared/locales.lua`                               | The `_T(key, ...)` translator (fallback: English) and the helper that sends tablet texts to the interface. |
| `shared/functions.lua`                             | Notification bridge and the payout hook.                                                                   |
| `nui/index.html`, `nui/script.js`, `nui/style.css` | The tablet. Texts are loaded from the active locale; in JavaScript use `_t('key')`.                        |

Each locale file has two parts: top-level keys (notifications and the map blip name) and a `ui` table (tablet texts).

**Notification keys**

| Key                           | English                                                  |
| ----------------------------- | -------------------------------------------------------- |
| `notify_title`                | Bounties                                                 |
| `not_enough_money`            | You do not have enough money.                            |
| `player_not_found`            | Target not found. Be more specific.                      |
| `invalid_amount`              | The bounty amount is invalid.                            |
| `invalid_type`                | Invalid contract type.                                   |
| `invalid_duration`            | Invalid contract duration.                               |
| `bounty_created_successfully` | You placed a $%s bounty on %s.                           |
| `bounty_create_error`         | The bounty could not be created.                         |
| `you_have_a_bounty`           | Watch out! Someone has placed a $%s bounty on your head. |
| `bounty_accepted`             | Contract accepted. The target has been marked.           |
| `bounty_unavailable`          | This contract is no longer available.                    |
| `already_accepted`            | You have already accepted this contract.                 |
| `cant_accept_own_bounty`      | You cannot accept a bounty you created.                  |
| `police_only_bounty`          | Only police officers can create this type of contract.   |
| `must_be_police_to_accept`    | This is a police contract. Only officers can accept it.  |
| `target_eliminated`           | Target neutralised! You earned $%s.                      |
| `your_bounty_claimed`         | A hunter has claimed the bounty on your head.            |
| `bounty_not_found`            | Bounty not found.                                        |
| `cant_edit_bounty`            | You cannot edit this bounty.                             |
| `bounty_updated`              | Bounty updated successfully.                             |
| `bounty_update_error`         | The bounty could not be updated.                         |
| `cant_delete_bounty`          | You cannot delete this bounty.                           |
| `bounty_deleted_refund`       | Bounty deleted. $%s has been refunded.                   |
| `bounty_delete_error`         | The bounty could not be deleted.                         |
| `unknown_target`              | Unknown                                                  |
| `blip_target`                 | Bounty Target (map blip name)                            |

**Tablet keys (`ui`)**: 73 keys grouped by screen: homescreen (`app_*`, `card_*`), create (`create_title`, `ph_*`, `type_civil`, `type_police`, `difficulty_label`, `btn_create`, `sidebar_*`, `tip_1` to `tip_3`), active list (`active_*`, `col_*`, `status_*`, `no_active`), My Panel (`panel_title`, `stat_*`, `tab_*`, `no_my`, `no_accepted`), leaderboard (`leaderboard_*`, `col_rank`, `col_hunter`, `col_contracts`, `col_earned`), contract details (`detail_title`, `lbl_*`, `no_info`, `btn_accept`, `accepted_*`) and the edit and delete windows (`edit_title`, `ph_edit_info`, `btn_cancel`, `btn_save`, `delete_*`, `btn_delete`). See `locales/en.lua` for the full list.

`%s` placeholders are filled in order. If the selected language is missing a key, the English text is used.

### Adding a language

1. Copy `locales/en.lua` to `locales/<code>.lua` and change the first line to `Locales['<code>'] = {`.
2. Translate the values, including the `ui` table.
3. Set `Config.Locale = '<code>'` and run `refresh` and `ensure nexus_bounty`.

***

## Compatibility

| System                      | Supported                                                                                      |
| --------------------------- | ---------------------------------------------------------------------------------------------- |
| Frameworks                  | ESX Legacy (`getSharedObject` export, with fallback to the old shared-object event) and QBCore |
| Death and ambulance scripts | Any. Kills are detected by the script itself                                                   |
| Database                    | oxmysql                                                                                        |
| Notifications               | nexus\_notify, okokNotify, ESX, QBCore, mythic\_notify, custom                                 |
| Money                       | Bank account (`bank`) on both frameworks                                                       |

**Identifiers**

| Data                | ESX                                    | QBCore                    |
| ------------------- | -------------------------------------- | ------------------------- |
| Player identifier   | `xPlayer.identifier`                   | `PlayerData.citizenid`    |
| Offline name lookup | `users.firstname` and `users.lastname` | `players.charinfo` (JSON) |
| Job check           | `xPlayer.job.name`                     | `PlayerData.job.name`     |

Escrow: `shared/config.lua`, `shared/functions.lua`, `shared/locales.lua`, `locales/*.lua`, the `nui/` files and `nexus_bounty.sql` are open and editable. The client and server logic is encrypted.

***

## Developer API

### Commands

| Command                          | Who                                             | Description                                                                                |
| -------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `/bounty` (`Config.OpenCommand`) | Player                                          | Opens and closes the tablet.                                                               |
| `/test_bounty_npc`               | Developers, only when `Config.DebugMode = true` | Creates a test contract on a static ped so you can try the accept and tracking flow alone. |

### Client exports

None.

### Server exports

None. Use the events, the payout hook and the database tables below to integrate.

### Events emitted

**Client events (server to client)**

| Event                                    | Arguments                                     | Description                                                                                                                                                                                                |
| ---------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_bounty:client:startTracking`      | `{ bountyId, isNpc, targetServerId, coords }` | Sent to a hunter after accepting a contract whose target is online. `coords` is the position of the target at that moment. Shows the blip with its route, starts proximity tracking and closes the tablet. |
| `nexus_bounty:client:targetLeft`         | `serverId`                                    | Sent to everyone when a player disconnects. A hunter tracking that player stops tracking and the blip is removed.                                                                                          |
| `nexus_bounty:client:stopTracking`       | none                                          | Stops tracking and removes the blip.                                                                                                                                                                       |
| `nexus_bounty:client:stopTrackingBounty` | `bountyId`                                    | Sent to everyone when a contract expires, is cancelled or is paid. Stops tracking if it matches the contract being tracked.                                                                                |
| `nexus_bounty:client:refreshMyPanel`     | none                                          | Refreshes My Panel after an edit or delete.                                                                                                                                                                |
| `nexus_bounty:client:refreshAllPanels`   | none                                          | Sent to everyone when a contract expires, is cancelled or is paid. Refreshes the open tablets.                                                                                                             |

**Server callbacks** (ESX `RegisterServerCallback`, QBCore `CreateCallback`)

| Callback                                  | Returns                                                                   |
| ----------------------------------------- | ------------------------------------------------------------------------- |
| `nexus_bounty:server:getHomescreenStats`  | `{ onlinePlayers, activeContracts }`                                      |
| `nexus_bounty:server:getActiveBounties`   | Array of active `bounties_active` rows plus `target_name` and `is_online` |
| `nexus_bounty:server:getMyBounties`       | Same shape, only the contracts created by the caller                      |
| `nexus_bounty:server:getAcceptedBounties` | Same shape, only the contracts the caller accepted                        |
| `nexus_bounty:server:getLeaderboard`      | Top 20 `bounty_leaderboard` rows                                          |
| `nexus_bounty:server:getPlayerStats`      | `{ bountiesCreated, totalBountyValue, bountiesCompleted, totalEarned }`   |

### Events listened

**Server events (from the tablet of the script itself)**. Every field is validated on the server.

| Event                              | Payload                                                                   | Behaviour                                                                                                                                                                                                                                                                                          |
| ---------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_bounty:server:createBounty` | `{ targetName, amount, imageUrl, extraInfo, duration, type, difficulty }` | Validates the amount (at least `MinBountyAmount`), the `type` (`Civil` or `Policial`), the `duration` (must match `Config.PVP.Durations`) and the police job for Police contracts. Charges the bank, finds exactly one matching character, inserts the contract and notifies the target if online. |
| `nexus_bounty:server:acceptBounty` | `bountyId`                                                                | Rejects expired contracts, own contracts, Police contracts for non-officers and hunters already registered. Registers the hunter atomically and starts tracking when the target is online.                                                                                                         |
| `nexus_bounty:server:editBounty`   | `{ id, imageUrl, extraInfo }`                                             | Creator only.                                                                                                                                                                                                                                                                                      |
| `nexus_bounty:server:deleteBounty` | `{ id }`                                                                  | Creator only. Refunds the full reward and stops tracking for everyone.                                                                                                                                                                                                                             |
| `nexus_bounty:server:playerKilled` | `killerServerId`                                                          | Sent by the client of the victim when it dies from another player or from a vehicle driven by another player. The server checks that both players exist and pays every active contract on the victim that the killer accepted.                                                                     |

**Framework hooks (server)**

| Event                                  | Arguments used                                              | Description                                                                       |
| -------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `esx:onPlayerDeath` (ESX)              | `source` = victim, `data.killerServerId` or `data.killerId` | Additional payout source on ESX.                                                  |
| `QBCore:Server:OnPlayerDeath` (QBCore) | `victimPlayer.source`, `deathData.killerId`                 | Optional additional payout source. You can trigger it from your own death script. |
| `QBCore:Ready` (QBCore)                | none                                                        | Optional. The script also initialises as soon as `qb-core` is started.            |

All payout sources share the same claim: the contract row is deleted first and the reward is only paid if that delete succeeded, so a kill reported twice is never paid twice.

### Editable functions

`shared/functions.lua` is loaded on the client and on the server.

| Function                                                 | Side   | Description                                                                                                                                                                                                              |
| -------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Functions.Notify(title, msg, type, time)`               | Client | Shows a notification with `Config.NotifySystem`. Add your system in the `custom` branch.                                                                                                                                 |
| `Functions.NotifyServer(player, title, msg, type, time)` | Server | Sends a notification to a player with `Config.NotifySystem`.                                                                                                                                                             |
| `Functions.TerminarContrato(hunterSource, bountyData)`   | Server | Hook called after a contract is paid. `bountyData` is the full `bounties_active` row (`id`, `monto`, `objetivo_steamid`, `contratador_identifier`, `tipo` and the rest). Use it for logs, items, experience and similar. |

```lua
function Functions.TerminarContrato(hunterSource, bountyData)
    print(('Hunter %s claimed bounty %s for %s'):format(hunterSource, bountyData.id, bountyData.monto))
end
```

### Database schema

The file `nexus_bounty.sql` creates both tables.

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
  `is_npc_bounty` BOOLEAN NOT NULL DEFAULT FALSE,
  `npc_template_id` INT(11) DEFAULT NULL,
  `npc_bounty_name` VARCHAR(255) DEFAULT NULL,
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

| Column                  | Meaning                                                          |
| ----------------------- | ---------------------------------------------------------------- |
| `objetivo_steamid`      | Target identifier (ESX identifier or QBCore citizenid).          |
| `monto`                 | Reward.                                                          |
| `tipo`                  | `Civil` or `Policial` (shown as Police in English).              |
| `dificultad`            | 1 to 5 stars.                                                    |
| `cazadores_activos`     | JSON array of the hunter identifiers that accepted the contract. |
| `fecha_caducidad`       | Expiry time.                                                     |
| `steamid` (leaderboard) | Hunter identifier (ESX identifier or QBCore citizenid).          |

The columns `is_npc_bounty`, `npc_template_id` and `npc_bounty_name` are used by the developer test tools only.

### Integration examples

**Report a kill from your own death script (QBCore)**. Kills are already detected automatically, so this is only needed if your server handles deaths in a special way.

```lua
local QBCore = exports['qb-core']:GetCoreObject()
local victim = QBCore.Functions.GetPlayer(victimSrc)
TriggerEvent('QBCore:Server:OnPlayerDeath', victim, { killerId = killerSrc })
```

**Check from another resource whether a player has a bounty on their head**

```lua
local function GetActiveBounty(identifier)
    return MySQL.single.await(
        'SELECT id, monto, tipo FROM bounties_active WHERE objetivo_steamid = ? AND fecha_caducidad > NOW()',
        { identifier }
    )
end
```

**Give an item to the hunter on every completed contract (`shared/functions.lua`)**

```lua
function Functions.TerminarContrato(hunterSource, bountyData)
    exports.ox_inventory:AddItem(hunterSource, 'bounty_badge', 1)
end
```

**Send a log on every payout**

```lua
function Functions.TerminarContrato(hunterSource, bountyData)
    TriggerEvent('my_logs:server:send', 'bounty', {
        hunter = hunterSource,
        amount = bountyData.monto,
        type = bountyData.tipo,
    })
end
```

**Use a custom notification system**

```lua
-- shared/functions.lua, branch custom
elseif Config.NotifySystem == 'custom' then
    exports.my_notify:Send(msg, type)
```

***

## FAQ

**The tablet does not open.** Check the command in `Config.OpenCommand` (default `/bounty`) and that the framework in `Config.Framework` matches your server.

**The hunter killed the target but was not paid.** The hunter must have accepted the contract before the kill, the contract must not be expired and the kill must be done by the character of the hunter, on foot or as the driver of a vehicle. Kills by NPCs, fire or falls do not count.

**The console shows `attempt to call a nil value (global 'GetNuiLocales')` or the tablet shows raw keys.** FiveM is using an old file list. Run `refresh` and then `ensure nexus_bounty`, or restart the server, and make sure the `locales/` folder and `shared/locales.lua` are present.

**"Target not found" when creating a contract.** The search must match exactly one character. Type more of the name, or set `Config.UseApproximateNameSearch = false` to require the exact full name. The money is refunded automatically.

**Can I target offline players?** Yes. If nobody online matches, the database is searched. The target is notified only if online at creation time, and the blip is only shown when the hunter accepts the contract while the target is online.

**Who can create and accept Police contracts?** Only players whose job is `Config.PoliceJobName`.

**Can the same hunter accept a contract twice?** No. Each hunter is registered once per contract. Several different hunters can compete for the same contract.

**What happens to the money when a contract expires?** The contract is closed and removed. The reward is refunded only when the creator cancels the contract from My Panel.

**How long does the blip stay on the map?** For `Config.PVP.BlipDuration` milliseconds each time it is revealed. Disable it completely with `Config.PVP.EnableTracking = false`.

**Why does the blip not appear again?** After each reveal there is a 30 second cooldown, and the target has to be within `Config.PVP.BlipRevealDistance` of the hunter. Tracking also ends when the target disconnects.

**How do I change the language or the texts?** Set `Config.Locale` and edit `locales/<lang>.lua`. Both notifications and the tablet use these files.
