---
icon: pickaxe
---

# Nexus Minerjob

> A complete mining career for ESX: persistent XP levels, upgradeable pickaxes and furnaces, tiered quarries, smelting recipes, a sell market, a leaderboard, server-side validation and random events, all in one panel.

**Version:** 2.8.0 · **Framework:** ESX · **Database:** oxmysql

***

## Description

Nexus Minerjob turns mining into a progression loop. The player talks to a supervisor NPC, puts on the work uniform from the panel, buys a pickaxe and starts a shift. While on shift, blips and ground markers show every quarry. Standing inside one and pressing the interaction key plays the pickaxe animation, fills a progress bar and asks the server for a reward. Level, XP, rocks mined, money earned, highest pickaxe and highest furnace are stored in one database row per player, so progress survives restarts.

Progression has three tracks:

* **Pickaxe level** decides which quarries can be mined. Each zone has a `requiredPick`, and the player's best pickaxe must reach it.
* **Miner level** decides which pickaxes can be bought. Each pickaxe has a `requiredLevel`, checked when `Config.Leveling.RequireLevelForPicks` is enabled.
* **Furnace level** decides which smelting recipes are available. Furnaces are bought with money only.

Raw ore is sold cheaply, refined items are worth more. Each furnace tier has a recipe list that converts raw items into refined items. Smelt All runs every recipe of the player's furnace against the whole inventory, and the Market tab sells raw and refined items in one action.

Random events run on top of the loop: a temporary rich vein, a global reward multiplier and a global mining speed bonus.

Every mining reward is validated on the server: the shift, the job, the position of the player, the pickaxe stored in the database and a minimum time between hits.

***

## Features

### Mining

* Supervisor NPC with a configurable model and map blip. Walking close shows a prompt and the interaction key opens the panel.
* Optional job requirement with `Config.JobRequired`. The job is checked on the server when the shift starts and on every hit.
* Zones by tier, each with its own location, radius, required pickaxe and weighted loot table.
* Shift system. Mining only works during a shift started through the panel.
* Configurable failure chance. A failed swing gives no item but still gives full XP.
* Pickaxe prop attached to the player and a mining animation for each swing, with movement and attack controls disabled while mining.
* Progress bar with the exact mining time, including speed bonuses.

### Progression

* XP per rock with a configurable curve. Surplus XP is carried over to the next level.
* Optional miner level requirement for each pickaxe.
* Top 10 leaderboard with first name, last name, rocks mined and money earned, ordered by the sum of rocks mined and money earned.

### Economy

* Panel pages: Home (statistics and leaderboard), Shop (pickaxes and furnaces), Furnace (smelting), Market (sell raw and refined items), Wardrobe and Info (in-game guide with General, Zones, Equipment and Events tabs).
* Home shows miner level, rocks mined, lifetime earnings and the names of the best pickaxe and furnace.
* Furnace cards include a Materials button that lists the recipes of that tier before buying.
* Purchases are validated on the server: type, level, money, ownership and, for pickaxes, miner level. Purchases are locked per player so repeated clicks cannot double charge.
* Rewards and smelting results are only given when the player can carry them. A full inventory shows a message and nothing is lost.

### Immersion

* Wardrobe to put on the work uniform or restore the civilian outfit saved the first time the player talks to the NPC.
* Optional uniform check before starting a shift.
* Seven languages and several notification and text prompt systems.

### Random events

* `ZONE`: a temporary quarry with its own blip, GPS route and reward table.
* `MULTIPLIER`: multiplies rewards of the regular zones.
* `SPEED`: divides the mining time and the server hit cooldown.
* Events are announced to everyone or only to nearby players. Players who join during an event receive it, and the stop message is always sent to everyone.

### Server validation

* Shift: the shift must have been started through the server.
* Job: checked when `Config.JobRequired` is enabled. Losing the job ends mining.
* Distance: the server reads the player position (OneSync required) and rejects hits farther than the zone radius plus 3 meters.
* Pickaxe: read from the database and compared with `requiredPick`.
* Cooldown: 90% of `Config.ProgressBarTime`, divided by the speed multiplier during a `SPEED` event.
* Input: zone ids, upgrade types and levels are validated.
* Rejected hits are printed in the server console with the player id and the reason.

***

## Dependencies

| Dependency                                  | Required             | Notes                                                     |
| ------------------------------------------- | -------------------- | --------------------------------------------------------- |
| `es_extended`                               | Yes                  | ESX only. The resource is not compatible with QBCore.     |
| `oxmysql`                                   | Yes                  | Stores player progress.                                   |
| `esx_skin` and `skinchanger`                | Yes for the wardrobe | Used to save and restore the civilian outfit.             |
| OneSync                                     | Yes                  | The server reads player positions for validation.         |
| `nexus_notify`                              | Optional             | Default notification system. Other options are available. |
| `nexus_TextUI`                              | Optional             | World prompts when `Config.TextUISystem = 'nexus'`.       |
| `okokNotify`, `okokTextUI`, `mythic_notify` | Optional             | Alternative notification and text systems.                |

The panel loads Font Awesome from cdnjs and the Poppins font from Google Fonts, so clients need internet access for the icons and the font to display.

The resource folder must be named `nexus_minerjob`.

***

## Installation

1. Place the `nexus_minerjob` folder in your `resources` directory.
2. Do not rename the folder.
3. Import `nexus_minerjob.sql`. It creates the `nexus_miner_data` table.
4. Make sure every item used in `Config.Zones`, `Config.Recipes` and `Config.Market` exists in your ESX `items` table. The item names in the default configuration are examples and must be replaced with your own mining items.
5. Add it to `server.cfg`:

```cfg
ensure oxmysql
ensure es_extended
ensure nexus_notify
ensure nexus_TextUI
ensure nexus_minerjob
```

6. Open `shared/config.lua` and adjust the NPC location, zones, prices, recipes and notification systems.
7. Restart the server.
8. In game, talk to the supervisor NPC, buy a pickaxe, put on the uniform and start a shift.

***

## Configuration

Everything is in `shared/config.lua`, which is not encrypted.

### General

| Key                       | Default   | Description                                                                   |
| ------------------------- | --------- | ----------------------------------------------------------------------------- |
| `Config.Locale`           | `'en'`    | Language file in `locales/`: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`.        |
| `Config.NotifySystem`     | `'nexus'` | `'nexus'`, `'okok'`, `'esx'`, `'mythic'` or `'custom'`.                       |
| `Config.TextUISystem`     | `'okok'`  | `'nexus'` (world prompts), `'okok'`, `'esx'` or `'custom'`.                   |
| `Config.TextUIKey`        | `'E'`     | Key name shown by `nexus_TextUI`. Keep it equal to `Config.Keys.interaction`. |
| `Config.Keys.interaction` | `38`      | Control id used by the `okok`, `esx` and `custom` text systems. `38` is E.    |
| `Config.JobRequired`      | `false`   | Requires the ESX job in `Config.JobName` to use the NPC and mine.             |
| `Config.JobName`          | `'miner'` | Job name used when `JobRequired` is enabled.                                  |
| `Config.ProgressBarTime`  | `5000`    | Milliseconds per hit. It is also the base of the server hit cooldown.         |

### NPC

```lua
Config.NPC = {
    coords = vector4(2946.52, 2747.07, 43.40, 283.46),
    model = 'cs_floyd',
    blip = { enabled = true, sprite = 47, display = 4, scale = 0.8, color = 2, text = 'Miner Job' }
}
```

The blip text is not part of the locale files and is edited here.

### Uniform

`Config.WorkOutfitRequired` requires the uniform before a shift can start. `Config.WorkOutfit` has a `male` and a `female` entry, each with `components` and `props`.

```lua
Config.WorkOutfit = {
    male = {
        components = {
            ['pants'] = { drawable = 10, texture = 0 },
            ['vest']  = { drawable = 250, texture = 0 },
        },
        props = {
            ['hats'] = { drawable = -1, texture = 0 },
        }
    },
    female = { components = {}, props = {} }
}
```

| Component names                                                                                                 | Prop names                                        |
| --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| `face`, `mask`, `hair`, `arms`, `pants`, `bags`, `shoes`, `neck`, `tshirt`, `bproof`, `decals`, `torso`, `vest` | `hats`, `glasses`, `ears`, `watches`, `bracelets` |

`vest` is an alias of `torso` (component 11), which is where jackets and vests are placed on freemode peds. A prop with `drawable = -1` removes that prop. When the uniform is required, every configured component and prop must match in drawable and texture.

### Leveling

| Key                                        | Default | Description                                                           |
| ------------------------------------------ | ------- | --------------------------------------------------------------------- |
| `Config.Leveling.Enabled`                  | `true`  | Enables XP and levels.                                                |
| `Config.Leveling.RequireLevelForPicks`     | `true`  | Requires `requiredLevel` to buy a pickaxe.                            |
| `Config.Leveling.XPPerRock`                | `15`    | XP given for every hit, including failed ones.                        |
| `Config.Leveling.BaseXP`                   | `100`   | Base of the level curve.                                              |
| `Config.Leveling.Multiplier`               | `1.1`   | XP needed for the next level is `floor(level ^ Multiplier * BaseXP)`. |
| `Config.Leveling.ChanceToFail.Enabled`     | `true`  | Enables failed hits.                                                  |
| `Config.Leveling.ChanceToFail.Probability` | `20`    | Percentage of hits that give no item.                                 |

### Pickaxes

```lua
Config.Picks = {
    { name = 'Stone Pickaxe', description = 'The basic pickaxe to get started.', price = 1500, level = 1, image = 'pico_piedra.png', requiredLevel = 1 },
}
```

`level` must be unique and sequential starting at 1. It is compared with the `requiredPick` of each zone. `image` is a file in `html/img/`. `requiredLevel` is the miner level needed to buy it.

### Furnaces

```lua
Config.Furnaces = {
    { name = 'Primitive Furnace', description = 'Built from clay and stone.', price = 10000, level = 1, image = 'horno_primitivo.png' },
}
```

`level` must be unique and sequential. It selects the recipe list in `Config.Recipes`. `image` is a file in `html/img/`.

### Zones

```lua
Config.Zones = {
    {
        name = 'Main Quarry',
        description = 'The starting zone.',
        image = 'zone_quarry.png',
        coords = vector3(2941.29, 2797.93, 41.0),
        radius = 2.0,
        requiredPick = 1,
        rewards = {
            { item = 'iron_ore', min = 3, max = 8, chance = 60 },
            { item = 'coal',     min = 1, max = 3, chance = 40 },
        }
    },
}
```

| Field              | Description                                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `coords`, `radius` | Position and mining radius. The ground marker uses the radius. The server accepts hits up to the radius plus 3 meters.                                                   |
| `requiredPick`     | Pickaxe `level` needed to mine here. Use a level that exists in `Config.Picks`.                                                                                          |
| `rewards`          | Weighted list. `chance` values are weights out of 100, so keep the total at 100. A single roll picks at most one entry and the amount is random between `min` and `max`. |
| `image`            | File in `html/img/` used by the Info page.                                                                                                                               |

### Recipes

```lua
Config.Recipes = {
    [1] = {
        { raw = 'iron_ore', refined = 'iron_ingot', rate = 1 },
    },
    [2] = {
        { raw = 'iron_ore', refined = 'iron_ingot', rate = 1 },
        { raw = 'gold_ore', refined = 'gold_ingot', rate = 2 },
    },
}
```

The key is the furnace level. A furnace uses the recipe list of the highest level that is defined and not above the furnace level, so higher tiers must repeat the recipes of the lower tiers. `rate` is the number of raw items consumed for each refined item.

### Random events

```lua
Config.RandomEvents = {
    Notification = { Type = 'server', Radius = 500.0 },
    EventTypes = { ... }
}
```

`Notification.Type` is `'server'` to announce to everyone or `'radius'` to announce only to players within `Radius` meters of the event coordinates.

| Field                         | Types        | Description                                                                           |
| ----------------------------- | ------------ | ------------------------------------------------------------------------------------- |
| `name`, `description`         | All          | Name and text of the event.                                                           |
| `type`                        | All          | `ZONE`, `MULTIPLIER` or `SPEED`. Do not change the value of the included events.      |
| `enabled`                     | All          | Enables the event.                                                                    |
| `duration`                    | All          | Duration in seconds.                                                                  |
| `timer.min`, `timer.max`      | All          | Random wait in seconds before the event can start. Only one event runs at a time.     |
| `blip`                        | All          | `{ sprite, color, scale, text }` or `nil`. Only shown for `ZONE`.                     |
| `coords`, `radius`, `rewards` | `ZONE`       | Location, radius and reward table of the temporary vein.                              |
| `anyPickaxe`, `requiredPick`  | `ZONE`       | With `anyPickaxe = true` any owned pickaxe works. Otherwise `requiredPick` is needed. |
| `rewardMultiplier`            | `MULTIPLIER` | Multiplies amounts in the regular zones.                                              |
| `speedMultiplier`             | `SPEED`      | `2.0` halves the mining time.                                                         |

### Market

```lua
Config.Market = {
    RawItems     = { { name = 'iron_ore',   image = 'ironore.png' } },
    RefinedItems = { { name = 'iron_ingot', image = 'iron_ingot.png' } },
}

Config.SellablePrices = {
    ['iron_ore']   = 8,
    ['iron_ingot'] = 40,
}
```

`RawItems` images are read from `html/items_images/raw/` and `RefinedItems` images from `html/items_images/refined/`. Only items that appear in the market lists and have a price are sold by Sell All. Items must exist in your ESX `items` table.

***

## Player flow

1. Talk to the supervisor NPC to open the panel.
2. Buy a pickaxe in the Shop page. Pickaxes require a miner level.
3. Put on the uniform in the Wardrobe page.
4. Press Start Shift. A shift needs at least one pickaxe.
5. Go to a quarry and press the interaction key to mine.
6. Buy a furnace and use Smelt All in the Furnace page.
7. Sell everything from the Market page.
8. Press Finish Shift when done. Civilian clothes can be restored in the Wardrobe page.

***

## Locales

Texts are in `locales/<lang>.lua`. Missing keys fall back to English.

| Group                | Keys                                                                                                                                                                                                                                                                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| General              | `job_name`                                                                                                                                                                                                                                                                                                                                   |
| Client notifications | `start_work`, `stop_work`, `uniform_on`, `uniform_off`, `civilian_outfit_not_found`, `uniform_required`, `job_required`, `pickaxe_too_weak`, `event_pickaxe_too_weak`, `sync_error`, `random_event_start`, `random_event_stop`, `speed_bonus_active`                                                                                         |
| Server notifications | `received_items`, `found_nothing`, `no_pickaxe_start`, `data_load_error`, `already_owned`, `bought_item`, `not_enough_money`, `materials_sold`, `no_materials_to_sell`, `need_furnace`, `no_furnace_recipes`, `smelted_items`, `not_enough_to_smelt`, `level_up`, `level_too_low`, `received_items_xp`, `found_nothing_xp`, `inventory_full` |
| Progress bar         | `mining`, `mining_vein`                                                                                                                                                                                                                                                                                                                      |
| Prompts              | `interact_npc`, `mine_rock`, `mine_event_rock`                                                                                                                                                                                                                                                                                               |

To add a language, copy `locales/en.lua` to `locales/<code>.lua`, change the first line to `Locales['<code>']`, translate the values and set `Config.Locale`.

The panel texts are in `html/index.html` and `html/script.js` and are in English. They are not tied to `Config.Locale` and can be edited directly. The NPC blip text is edited in `Config.NPC.blip.text`.

### Theming

`html/style.css` defines the colors in a `:root` block that can be edited freely:

```css
:root {
    --bg-dark: rgba(24, 23, 28, 0.95);
    --bg-card: rgba(68, 71, 90, 0.5);
    --text-light: #f8f8f2;
    --accent-red: #e74c3c;
    --accent-orange: #ffb86c;
    --accent-purple: #bd93f9;
}
```

The panel is a fixed 1300 x 800 box centered on the screen.

***

## Compatibility

| System        | Behavior                                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------------------------ |
| ESX           | Required. Uses inventory, money, job, `esx_skin` and server callbacks.                                             |
| QBCore        | Not supported.                                                                                                     |
| Notifications | `nexus_notify`, `okokNotify`, ESX native, `mythic_notify` or a custom function in `shared/functions.lua`.          |
| Text prompts  | `nexus_TextUI` world prompts, `okokTextUI`, ESX help notification or a custom function.                            |
| Inventory     | Uses the ESX inventory functions `addInventoryItem`, `removeInventoryItem`, `getInventoryItem` and `canCarryItem`. |

Using `nexus_TextUI` prompts: set `Config.TextUISystem = 'nexus'`, keep `Config.TextUIKey` equal to the key behind `Config.Keys.interaction`, and make sure `nexus_TextUI` starts before this resource. The prompts for the NPC, the zones and the event vein are created and removed automatically as the shift and events change.

***

## Developer API

### Exports

The resource declares no client or server exports. To read player statistics from your own server code, query the table:

```lua
local data = MySQL.single.await('SELECT level, xp, rocks_mined, money_earned, p_highest, f_highest FROM nexus_miner_data WHERE identifier = ?', { identifier })
```

### ESX server callback

| Callback                 | Returns        | Description                                                                                                              |
| ------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `nexus_minerjob:getData` | table or `nil` | Player data used by the panel. Creates the database row if it does not exist. Returns `nil` if the player is not loaded. |

```lua
ESX.TriggerServerCallback('nexus_minerjob:getData', function(data)
    if not data then return end
    print(data.minerData.level, data.minerData.xp, data.minerData.p_highest)
    for _, row in ipairs(data.leaderboards.rocks) do
        print(row.firstname, row.lastname, row.rocks_mined, row.money_earned)
    end
end)
```

Returned structure:

```lua
{
    minerData = { identifier, level, rocks_mined, money_earned, p_owned, f_owned, p_highest, f_highest, xp, firstname },
    leaderboards = { rocks = { { firstname, lastname, rocks_mined, money_earned } } },
    inventory = { ... }
}
```

### Events

| Event                                      | Direction        | Payload                                          | Description                                                                                                           |
| ------------------------------------------ | ---------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| `nexus_minerjob:server:startWork`          | Client to server | none                                             | Requests to start a shift. Checks the job and that the player owns a pickaxe.                                         |
| `nexus_minerjob:server:stopWork`           | Client to server | none                                             | Ends the shift on the server.                                                                                         |
| `nexus_minerjob:server:requestMineRewards` | Client to server | `zoneId`, `isEventZone`                          | Requests the reward of a hit. Fully validated on the server.                                                          |
| `nexus_minerjob:server:buyUpgrade`         | Client to server | `{ type = 'pick' or 'furnace', level = number }` | Buys a pickaxe or a furnace.                                                                                          |
| `nexus_minerjob:server:sellMaterials`      | Client to server | none                                             | Sells all market items.                                                                                               |
| `nexus_minerjob:server:smeltAll`           | Client to server | none                                             | Runs every recipe of the player's furnace.                                                                            |
| `nexus_minerjob:client:startWorkResponse`  | Server to client | `{ success, reason }`                            | Result of `startWork`.                                                                                                |
| `nexus_minerjob:client:refreshUI`          | Server to client | none                                             | Reloads the panel data if the panel is open.                                                                          |
| `nexus_minerjob:client:startRandomEvent`   | Server to client | event table                                      | Starts an event on the client.                                                                                        |
| `nexus_minerjob:client:stopRandomEvent`    | Server to client | none                                             | Ends the current event on the client.                                                                                 |
| `nexus_minerjob:client:tryMine`            | Local client     | `{ zoneId, isEvent }`                            | Starts a mining hit from another client script, for example a target system. Ignored when the player is not on shift. |

Server events that change player data always validate the player on the server. The client never sends rewards, prices or grades.

The server functions `StartRandomEvent(eventConfig)` and `StopRandomEvent()` are global and can be called from `shared/functions.lua` or other server scripts to trigger events manually, for example from an admin command.

### Editable functions

`shared/functions.lua` is not encrypted.

| Function                                                     | Description                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------------ |
| `Functions.Notify(title, msg, type, time)`                   | Client notification. Add your system in the `custom` branch.       |
| `Functions.NotifyServer(player, title, msg, type, time)`     | Server notification to a player.                                   |
| `Functions.ShowText(msg)` / `Functions.HideText()`           | Text prompt for the `okok`, `esx` and `custom` systems.            |
| `Functions.PromptsAvailable()`                               | Returns `true` when `nexus_TextUI` prompts are in use and running. |
| `Functions.CreatePrompt(...)` / `Functions.DeletePrompt(id)` | Create and delete `nexus_TextUI` prompts.                          |
| `Functions.Lang(key, ...)`                                   | Translation helper.                                                |

```lua
-- shared/functions.lua
elseif Config.NotifySystem == 'custom' then
    exports['my_notify']:Show(title, msg, type)
```

### Database

```sql
CREATE TABLE IF NOT EXISTS `nexus_miner_data` (
  `identifier` VARCHAR(60) NOT NULL,
  `level` INT(11) NOT NULL DEFAULT 1,
  `rocks_mined` INT(11) NOT NULL DEFAULT 0,
  `money_earned` BIGINT(20) NOT NULL DEFAULT 0,
  `p_owned` INT(11) NOT NULL DEFAULT 0,
  `f_owned` INT(11) NOT NULL DEFAULT 0,
  `p_highest` INT(11) NOT NULL DEFAULT 0,
  `f_highest` INT(11) NOT NULL DEFAULT 0,
  `xp` INT(11) NOT NULL DEFAULT 0,
  PRIMARY KEY (`identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

The leaderboard joins this table with the ESX `users` table to read names.

***

## FAQ

**I cannot start a shift.** A pickaxe is required, and the uniform must be worn when `Config.WorkOutfitRequired` is enabled. If `Config.JobRequired` is enabled, the player also needs the job in `Config.JobName`.

**The uniform is not accepted.** Every component and prop in `Config.WorkOutfit` must match in drawable and texture. Use the Put On Uniform button of the Wardrobe page, or set the outfit to match what your players wear.

**Mining does nothing or hits are rejected.** Check the server console. Rejected hits print the player id and the reason: not on shift, job mismatch, too far from the zone, pickaxe too low. OneSync must be enabled for the distance check.

**The panel shows empty images or the items do not exist.** The images in the default configuration are examples. Place your images in `html/img/` and `html/items_images/` and use item names that exist in your ESX `items` table.

**The icons or the font do not show.** The panel loads Font Awesome and the Poppins font from the internet.

**Smelting does nothing.** The player needs a furnace, enough raw items for at least one recipe of that furnace tier, and room in the inventory. Remember that each tier uses its own recipe list, which must repeat the lower tiers.

**A zone cannot be mined by anyone.** `requiredPick` must match a pickaxe `level` that exists in `Config.Picks`.

**How do I change the interaction key?** For `nexus_TextUI` change `Config.TextUIKey`. For the other systems change `Config.Keys.interaction` (a control id). Keep both equal.

**How do I reset a player's progress?** Delete or edit the player's row in `nexus_miner_data`.

**How do I disable random events?** Set `enabled = false` on each event in `Config.RandomEvents.EventTypes`.

### Before opening a ticket

* Confirm the folder is named `nexus_minerjob` and the SQL was imported.
* Confirm every configured item exists in your ESX `items` table.
* Confirm `oxmysql`, `es_extended` and your notification resources start before this one.
* Read the server console for `mine rejected` lines.
