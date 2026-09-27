# 🎯 Nexus Bounty

A bounty and player hunting system for your FiveM server, featuring a fully custom tablet-style app (custom NUI) and contracts targeting both players (PvP) and NPCs (PvE).

## 📝 Description

**Nexus Bounty** brings a complete bounty system to your server: any player can place a hit on another (or on law enforcement through exclusive police contracts) using an in-game tablet featuring a modern, custom interface.

The script automatically handles target tracking (whether the player is online or offline), payment deductions, target blip positioning on the map once a hunter accepts the contract, payouts upon target elimination, and a global leaderboard ranking the server's top bounty hunters.

Additionally, it features a parallel NPC contract system (PvE), fully configurable via templates, ensuring available bounties at all times even during low player counts.

## ✨ Features

* **PvP Contracts**: Any player can place a bounty on another player, whether they are online or offline.
* **Police Contracts**: An exclusive contract type that can only be created (and accepted) by members of the police department
* **PvE (NPC) Contracts**: Bounties targeting NPC objectives generated from configurable templates (model, location, police involvement/aggression chance, reward amount).
* **Custom NUI Tablet**: A fully tailored, custom-built interface (not a generic menu) featuring a home screen, contract creation tab, active contracts list, personal dashboard, and leaderboard.
* **Target Tracking**: Upon accepting a contract, the hunter receives a target blip on the map once within the configured range, complete with adjustable duration and cooldown settings
* **Personal Dashboard ("My Dashboard")**: Manage your active contracts and created bounties, with options to edit contract details/images or cancel them (with automated fee refunds)
* **Global Leaderboard**: Leaderboard tracking the top bounty hunters by completed contracts and total earnings.
* **Automatic Expiration**: Contracts automatically expire after the configured duration and are wiped from the database without requiring manual intervention.
* **Live Home Screen Stats**: Real-time metrics displaying online player counts and active contracts right on the tablet's home screen.
* **Integrated Testing System**: Included command to spawn a test NPC to validate the full contract workflow without waiting for a live target (active when `Config.DebugMode` is enabled).
* **Framework Compatible**: Native support for both ESX and QBCore, selectable directly via `Config.Framework`.

## 📋 Dependencies

* **Framework**: ESX or QBCore (selected on `Config.Framework`).
* **MySQL database** with the `MySQL.Async` wrapper available (provided by `mysql-async` or `oxmysql` via its compatibility layer). The script directly depends on `@mysql-async/lib/MySQL.lua`.
* A properly configured **police job** within your framework if you plan to use "Police" type contracts.

## ⚙️ Installation

{% stepper %}
{% step %}
### Download and Unzip the resource

Dowload (from the cfx.re portal) and extract the resource's folder and add it to your  `resources` folder.
{% endstep %}

{% step %}
### Import the database

Import the `nexus_bounty.sql` file into your database. Creating the `bounties_active` and `bounty_leaderboard` tables.
{% endstep %}

{% step %}
### Correct `server.cfg` positioning

Make sure your MySQL resource (`mysql-async` or `oxmysql`) is initialized before `nexus_bounty`.
{% endstep %}

{% step %}
### Add the resource to your server

Add it to your `server.cfg`:

```cfg
ensure "your MySQL resource"
--THEN you add nexus_bounty
ensure nexus_bounty
```
{% endstep %}

{% step %}
### Adapt the script to your server needs

Take a look of the `config.lua` and customize it to your needs.
{% endstep %}

{% step %}
### Restart the resource/server

Restart the resource or the server. We reccommend restarting the whole server.
{% endstep %}

{% step %}
### Start using your new script

Use the `Config.OpenCommand` (by default `/recompensas`) to open the tablet.
{% endstep %}
{% endstepper %}

## 🔧 Configuration

Every possible config resides in `shared/config.lua`:

| Config                            | Description                                                                                                                                                                                                                             |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Framework`                | `"ESX"` or `"QBCore"`.                                                                                                                                                                                                                  |
| `Config.DebugMode`                | Enables detailed console logging and unlocks the `/test_bounty_npc` command to test the complete contract workflow without waiting for a live target. **It is strongly recommended** to set this to `false` in production environments. |
| `Config.OpenCommand`              | Command used to open the tablet interface (default: `recompensas`).                                                                                                                                                                     |
| `Config.PoliceJobName`            | The police job name defined in your framework database, used to validate access for "Police" category contracts.                                                                                                                        |
| `Config.UseApproximateNameSearch` | Reservado para una futura búsqueda exacta/aproximada del objetivo por nombre.                                                                                                                                                           |
| `Config.PVP.Enabled`              | Enables or disables PvP contracts targeting other players.                                                                                                                                                                              |
| `Config.PVP.MinBountyAmount`      | Minimum permited bounty amount.                                                                                                                                                                                                         |
| `Config.PVP.Durations`            | Available bounty durations list.                                                                                                                                                                                                        |
| `Config.PVP.EnableTracking`       | Enables the target tracking blip on the map once a contract is accepted.                                                                                                                                                                |
| `Config.PVP.BlipRevealDistance`   | The distance (in game units/meters) within which the target blip becomes visible on the map.                                                                                                                                            |
| `Config.PVP.BlipDuration`         | The duration in milliseconds that the target blip remains visible on the map each time it is revealed.                                                                                                                                  |
| `Config.PVE.Enabled`              | Enables or disables PvE contracts targeting NPCs.                                                                                                                                                                                       |
| `Config.PVE.MaxActiveNPCBounties` | Active bounty contracts maximum number.                                                                                                                                                                                                 |
| `Config.PVE.NPCTemplates`         | Available NPC templates for PvE contracts (including model, reward amount, locations, and probabilities).                                                                                                                               |
| `Config.Locales`                  | Contains all script text, labels, and notification strings, ready for localization and translation.                                                                                                                                     |

{% hint style="info" %}
Be sure to also set `Config.NotifySystem` (see the following page) to select your preferred notification system.
{% endhint %}

## 🔗 Compatibility

* **Frameworks**: ESX and QBCore, with no need to dive into the code — just select it in  `Config.Framework`.
* **Database**: any wrapper comaptible with `MySQL.Async` (mysql-async / oxmysql).
* **Notifys**: nexus\_notify, okokNotify, mythic\_notify,or the natives from ESX/QBCore, selectable in `functions.lua` without modifying the protected code (see the following page).
* The script does not depend on any external TextUI or target system: the tablet features a standalone NUI interface, and contract rewards are handled directly through standard framework events (player death and payout processing).

## 💻 Notifys personalization (functions.lua)

The `functions.lua` file is provided unencrypted outside of the escrow system, allowing you to integrate the script with your server's existing notification system without modifying any protected code.

Just need to add it to `shared/config.lua`:

```lua
Config.NotifySystem = 'esx' -- 'nexus', 'okok', 'esx', 'qb', 'mythic', 'custom'
```

Natively supported systems: `nexus_notify`, `okokNotify`, `mythic_notify`, native ESX notifications, native QBCore notifications, or `'custom'` to connect any other notification resource by editing its designated block directly in `functions.lua`.

The string values in `Config.Locales` include native GTA color codes (`~r~`, `~g~`, `~o~`, etc.). `functions.lua` automatically detects these prefixes and maps them to standard notification types (`error`, `success`, `warning`, `info`), ensuring they display correctly even on custom notification systems that do not parse GTA color codes natively.

## 🆚 PvP Contracts vs. PvE Contracts

Nexus Bouunty combines two types of contracts within the same system:

* **PvP (Player vs. Player)**: Created by entering a target's full or partial name. The script first searches active online players and falls back to searching the database if no online match is found. Contracts can be issued standard (civilian) or marked as "Police" contracts if created by an officer (restricting acceptance strictly to police officers).
* **PvE (Player vs. Environment)**: Generated from the templates defined in `Config.PVE.NPCTemplates`, specifying model, location, and spawn probabilities. These ensure contracts are continuously available, even when player counts are low or PvP activity is quiet.

Both types share the exact same flow for contract acceptance, tracking, reward collection, and leaderboard classification.

## ❓ FAQ

{% hint style="info" %}
**Before opening a ticket:**

* Make sure the resources name is exactly `nexus_bounty`.
* Make sure you are using the latest version of the resource.
* Revise this FAQ page.
{% endhint %}

<details>

<summary>What happens if the objetive disconnects while being hunted?</summary>

The tracking system and blip dissappears until the objetive connects again.

</details>

<details>

<summary>Can i set a bounty on an offline player?</summary>

Yes. if the desired player is not online, the script searches for their data in the database.

</details>

<details>

<summary>What happens if a bounty is not accepted by anyone?</summary>

It's removed from the active bounties list and from the database; no return is made to the creator.

</details>

<details>

<summary>Can i cancel a bounty made by me?</summary>

Yes, through "Mi Panel" you can remove your created bounties; your money is returned inmediately.

</details>

<details>

<summary>What info do i need to include in the support ticket?</summary>

Exact name and version of the resource, used framework (ESX/QBCore), the EXACT error that appears in the F8/console/in-game, ideally with screenshots.

</details>
