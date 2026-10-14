---
icon: truck
---

# Nexus Deliveryjob

> A complete delivery job for ESX: dynamic routes, skill tree, vehicle shop, hireable employees with passive income, company bank, uniforms and dangerous cargo, all inside one custom panel.

**Version:** 1.4.1 · **Framework:** ESX · **Database:** oxmysql

***

## Description

Nexus Deliveryjob turns deliveries into a full career. Players visit the delivery HQ, pick a route from a rotating list, get a van (or use one they bought), follow the GPS through every drop-off, carry each package to the door and return to base to get paid.

On top of the basic loop the script adds long-term progression. Every route gives XP, XP turns into skill points, and skill points unlock better paid route types (long distance, valuable cargo, dangerous goods). Players can invest their earnings in their own delivery vehicles, hire employees that generate passive income into a company bank account, and climb a server leaderboard.

Everything happens inside one menu with nine sections: Home, Quick Jobs, Owned Vehicle Jobs, Skills, Shop, My Vehicles, Employees, Bank and Cloakroom.

***

## Features

### Routes

* **Rotating route board.** Every `Config.RouteRefreshRate` minutes the server picks up to 10 random routes from `Config.Rutas`. The menu shows a countdown until the next refresh.
* **Two ways to work.** Quick Jobs use the company vehicle (`Config.DefaultVehicle`). Owned Vehicle Jobs use one of the vehicles the player bought.
* **Multi-stop deliveries.** Each route has any number of delivery points. The player parks near the marker, gets out, takes the package (prop and animation) and delivers it at the door.
* **Delivery checks.** The job vehicle must be within `Config.MaxDistanceToDeliver` meters of the drop-off, and the player must be the driver to finish the route.
* **Server-side rewards.** The server records the route each player accepted and pays only the reward and XP configured for that route. The client never sends money or XP values. Rewards are paid to the player's bank account.
* **Route types.** `normal`, `distance`, `expensive` and `dangerous`. Non-normal routes require the matching skill at its maximum level.
* **Dangerous goods.** On `dangerous` routes the vehicle explodes on hard crashes or when its health drops below a configurable threshold, and the mission fails.

### Progression

* **XP to skill points.** Every `XP_PER_SKILL_POINT` XP grants `SKILL_POINTS_AWARDED` skill points.
* **Skill tree.** Three skills by default (Long Distance, Valuable Cargo, Dangerous Goods), each with its own number of levels. The maximum level is validated on the server.
* **Statistics.** Completed trips, money earned, kilometers, XP, owned vehicles, employees and company balance on the Home page.
* **Leaderboard.** Top 5 players by total earnings.

### Economy

* **Vehicle shop.** Buy delivery vehicles with stat bars (speed, braking, handling). Owned vehicles unlock Owned Vehicle Jobs. Purchases use bank money.
* **Vehicle selling.** Sell a vehicle for 50% of its price after a confirmation. The money is paid only after the database confirms the vehicle was removed.
* **Employees.** A rotating hiring market with `Config.NumAvailableEmployees` candidates every `Config.EmployeeRefreshRate` minutes. Each employee has a level from 1 to 5, a star rating, traits and a hire cost.
* **Passive income.** Every `Config.EmployeePayoutInterval` minutes each hired employee of an online player produces money according to its level. The money goes to the company balance.
* **Company bank.** Withdraw the accumulated company balance in cash at any time.

### Immersion

* **Uniform system.** Optional mandatory work outfit for male and female peds, with components and props. The civilian clothes are saved and restored from the Cloakroom.
* **NPC or marker.** Open the menu through an NPC (any ped model and scenario) or a marker.
* **Target or text prompt.** `ox_target`, `qb-target`, `qtarget` or a text prompt.
* **NPC-free zone.** Optionally removes pedestrians and traffic around the HQ so spawn and return points are never blocked.
* **In-menu tutorial.** First-time players see a short guide with a Do not show again option.

***

## Dependencies

| Resource                                      | Required              | Notes                                                                |
| --------------------------------------------- | --------------------- | -------------------------------------------------------------------- |
| `es_extended`                                 | Yes                   | ESX Legacy.                                                          |
| `oxmysql`                                     | Yes                   | Database driver.                                                     |
| `skinchanger`                                 | Yes                   | Restores the civilian outfit.                                        |
| `illenium-appearance`                         | Yes                   | Declared in the manifest dependencies, so it must be started.        |
| `esx_skin`                                    | Yes for the Cloakroom | Provides `esx_skin:getPlayerSkin`, used to save the civilian outfit. |
| `ox_target`, `qb-target` or `qtarget`         | Optional              | Only when `Config.InteractionMethod = 'target'`.                     |
| `nexus_notify`, `okokNotify`, `mythic_notify` | Optional              | Notification system, selected in the configuration.                  |
| `okokTextUI`                                  | Optional              | Text prompt, selected in the configuration.                          |
| `cd_garage`                                   | Optional              | Vehicle keys.                                                        |
| `LegacyFuel`                                  | Optional              | Fuel.                                                                |

The resource folder must be named `nexus_deliveryjob`. The resource stops with an error otherwise.

***

## Installation

1. Place the `nexus_deliveryjob` folder in your `resources` directory.
2. Do not rename the folder.
3. Run `tables.sql` in your database. It uses `CREATE TABLE IF NOT EXISTS`, so it is safe to run more than once.
4. Add it to `server.cfg` after its dependencies:

```cfg
ensure oxmysql
ensure es_extended
ensure skinchanger
ensure illenium-appearance
ensure nexus_notify
ensure nexus_deliveryjob
```

5. Open `shared/config.lua` and set the language, notification, text prompt, target, keys and fuel systems, the HQ location, routes and prices.
6. Restart the server.

***

## Configuration

All options are in `shared/config.lua`, which is not encrypted.

### General

| Option                  | Default       | Description                                                      |
| ----------------------- | ------------- | ---------------------------------------------------------------- |
| `Config.Locale`         | `'en'`        | `es`, `en`, `fr`, `pt`, `it`, `de`, `zh-CN`.                     |
| `Config.JobNeed`        | `false`       | If `true`, only players with `Config.JobName` can open the menu. |
| `Config.JobName`        | `"ambulance"` | Job name (not label) required when `JobNeed` is enabled.         |
| `Config.DefaultVehicle` | `"boxville4"` | Vehicle model used for Quick Jobs.                               |
| `Config.TextUISystem`   | `"okok"`      | `okok`, `esx` or `custom`.                                       |
| `Config.NotifySystem`   | `"nexus"`     | `nexus`, `okok`, `mythic`, `esx` or `custom`.                    |
| `Config.UseKeysSystem`  | `"default"`   | `cd_garage`, `custom` or `default` (no vehicle keys).            |
| `Config.UseFuelSystem`  | `false`       | `legacyfuel`, `custom` or `false` (no fuel handling).            |
| `Config.DebugPrints`    | `false`       | Server console messages on route and employee refresh.           |

### Interaction and locations

| Option                                                                        | Default             | Description                                                                                              |
| ----------------------------------------------------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------- |
| `Config.InteractionMethod`                                                    | `'target'`          | `'target'` or `'textui'`. With `'target'`, an HQ NPC is spawned and the target option is attached to it. |
| `Config.KeyInteract`                                                          | `38`                | Control id for all interactions (`38` is E).                                                             |
| `Config.UseNPC`                                                               | `false`             | `true` spawns an NPC, `false` draws a marker.                                                            |
| `Config.MarkerType`                                                           | `20`                | Marker type when `UseNPC` is `false`.                                                                    |
| `Config.NPC`                                                                  | see file            | `model`, `name`, `coords` (vector4, also used by the marker) and `anim` scenario.                        |
| `Config.VehicleSpawns`                                                        | see file            | List of vector4 spawn points. The first free one is used.                                                |
| `Config.ReturnPoint`                                                          | see file            | Where the vehicle must be brought back to finish the route.                                              |
| `Config.ClearZoneEnabled`, `Config.ClearZoneCenter`, `Config.ClearZoneRadius` | `true`, HQ, `200.0` | NPC-free zone around the HQ.                                                                             |

### Routes

| Option                               | Default  | Description                                                                    |
| ------------------------------------ | -------- | ------------------------------------------------------------------------------ |
| `Config.RouteRefreshRate`            | `1`      | Minutes between route board refreshes.                                         |
| `Config.MaxDistanceToDeliver`        | `25.0`   | Maximum meters between the vehicle and the drop-off.                           |
| `Config.Rutas`                       | see file | Route table.                                                                   |
| `Config.DangerousGoodsExplosion`     | `true`   | Enables explosions on `dangerous` routes.                                      |
| `Config.ExplosionCrashForce`         | `40`     | Crash force the vehicle withstands. A higher value makes it harder to explode. |
| `Config.DangerousGoodsHealthPercent` | `0.8`    | The vehicle explodes when its health drops below this fraction.                |

Route format:

```lua
Config.Rutas = {
    sandyshores = {
        label = "Delivery on Sandy",
        description = "Long range delivery.",
        type = "normal",
        requiredSkill = nil,
        deliveryPoints = {
            vector3(1978.88, 3819.35, 32.22),
            vector3(1738.15, 3719.77, 34.03),
        },
        recompensa = { min = 3400, max = 3800 },
        xp = 50,
        distance = 8.4
    },
}
```

| Field                  | Description                                                |
| ---------------------- | ---------------------------------------------------------- |
| Key (`sandyshores`)    | Unique route id used by the server.                        |
| `label`, `description` | Texts shown to the player.                                 |
| `type`                 | `normal`, `distance`, `expensive` or `dangerous`.          |
| `requiredSkill`        | `nil` or a key of `Config.Skills`.                         |
| `deliveryPoints`       | List of drop-off coordinates.                              |
| `recompensa`           | Random payout between `min` and `max`.                     |
| `xp`                   | XP given when finishing the route.                         |
| `distance`             | Kilometers shown in the menu and stored in the statistics. |

Reward, XP and distance are always read on the server from this table.

### Employees

| Option                          | Default       | Description                                                                                       |
| ------------------------------- | ------------- | ------------------------------------------------------------------------------------------------- |
| `Config.EmployeePayoutInterval` | `30`          | Minutes between passive payouts.                                                                  |
| `Config.MaxEmployees`           | `4`           | Maximum hired employees per player.                                                               |
| `Config.EmployeeTiers`          | see file      | `[level] = { min, max }` money produced per payout.                                               |
| `Config.AvailableEmployees`     | 20 candidates | `id`, `name`, `avatarImage` (file in `html/img/employees/`), `rating`, `traits`, `cost`, `level`. |
| `Config.EmployeeRefreshRate`    | `2`           | Minutes between hiring market refreshes.                                                          |
| `Config.NumAvailableEmployees`  | `4`           | Candidates shown at once.                                                                         |

The hire cost is paid once. Firing an employee requires paying the full cost again to hire them later.

### Skills and XP

```lua
Config.Skills = {
    distance  = { name = "Long Distance",   description = "...", maxLevel = 3 },
    expensive = { name = "Valuable Cargo",  description = "...", maxLevel = 3 },
    dangerous = { name = "Dangerous Goods", description = "...", maxLevel = 3 },
}

Config.XPSystem = {
    XP_PER_SKILL_POINT   = 1000,
    SKILL_POINTS_AWARDED = 1,
}
```

Each level costs one skill point. A route with `requiredSkill` needs that skill at its `maxLevel`. The icons of the three default skills are mapped in `html/script.js`, so a new skill id also needs an icon there.

### Outfit

| Option                      | Description                                                                       |
| --------------------------- | --------------------------------------------------------------------------------- |
| `Config.WorkOutfitRequired` | If `true`, the uniform is required to start a route.                              |
| `Config.WorkOutfit`         | `male` and `female` tables with `components` and `props` (`drawable`, `texture`). |

### Vehicle shop

```lua
Config.ShopVehicles = {
    speedo = {
        name = 'Vapid Speedo',
        price = 40000,
        model = 'speedo',
        stats = { velocity = 70, braking = 43, handling = 62 }
    },
}
```

The key (`speedo`) is also the image name: `html/img/speedo.png`. `stats` values go from 0 to 100 and are only used by the interface. Selling returns 50% of `price`.

***

## Locales

| File                                                  | Contents                                                                           |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `shared/config.lua`                                   | All options, route, skill, employee and vehicle names and descriptions.            |
| `shared/functions.lua`                                | Notification, text prompt, keys and fuel bridges, and the `Functions.Lang` helper. |
| `client/target.lua`                                   | Target integration for the HQ NPC.                                                 |
| `locales/*.lua`                                       | In-game texts in 7 languages: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh-CN`.         |
| `html/index.html`, `html/script.js`, `html/style.css` | The panel, in English.                                                             |

`Config.Locale` selects the language of notifications and prompts. If a key is missing in the selected file, the key name is shown, so keep all language files complete. Money messages use the currency symbol written in the locale files, which can be edited.

The panel is in English and is not tied to `Config.Locale`. To translate it, edit `html/index.html` and `html/script.js`. The target label in `client/target.lua` can be edited in the same way.

To add a language, copy `locales/en.lua` to `locales/<code>.lua`, change the key on the first line, translate the values, add the file to the `shared_scripts` list of `fxmanifest.lua` and set `Config.Locale`.

***

## Compatibility

| System          | Supported                                               |
| --------------- | ------------------------------------------------------- |
| Framework       | ESX Legacy (`es_extended`). QBCore is not supported.    |
| Database        | oxmysql                                                 |
| Notifications   | nexus\_notify, okokNotify, mythic\_notify, ESX, custom  |
| Text prompt     | okokTextUI, ESX help notification, custom               |
| Target          | ox\_target, qb-target, qtarget (detected automatically) |
| Vehicle keys    | cd\_garage, custom, none                                |
| Fuel            | LegacyFuel, custom, none                                |
| Civilian outfit | esx\_skin and skinchanger                               |

***

## Developer API

### Server events

All of them are triggered by the script's own client. The server validates each one.

| Event                             | Payload            | Behavior                                                                                                                                                |
| --------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `delivery:startJob`               | `routeId` (string) | Stores the accepted route for the player. Ignored if the id is not in `Config.Rutas`.                                                                   |
| `delivery:cancelJob`              | none               | Clears the stored route.                                                                                                                                |
| `delivery:finishJob`              | `routeId` (string) | Pays only if `routeId` matches the stored route. Reward, XP and distance are read from `Config.Rutas[routeId]`. The stored route is cleared either way. |
| `delivery:purchaseVehicle`        | `{ id }`           | Buys `Config.ShopVehicles[id]` with bank money.                                                                                                         |
| `delivery:sellOwnedVehicle`       | `{ id }`           | Deletes the vehicle from the database and, only if a row was removed, pays 50%.                                                                         |
| `delivery:upgradeSkill`           | `{ id }`           | Spends 1 skill point. Refused if the skill is at `maxLevel`.                                                                                            |
| `delivery:hireEmployee`           | `{ id }`           | Hires a candidate, checking bank money, free slots and duplicates.                                                                                      |
| `delivery:fireEmployee`           | `{ id }`           | Fires one of the player's employees.                                                                                                                    |
| `delivery:withdrawCompanyBalance` | none               | Moves the company balance to cash.                                                                                                                      |

To finish a route from your own code, send the route id, not a table, and make sure `delivery:startJob` was sent first by the same player.

### Server callback

| Callback                 | Returns                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `delivery:getPlayerData` | Player stats, skills, employees, owned vehicles, leaderboard, available routes and candidates, and refresh timers. |

### Client events

| Event                            | Description                                    |
| -------------------------------- | ---------------------------------------------- |
| `delivery:client:openMenu`       | Opens the delivery menu. Used by the target.   |
| `delivery:client:finishJob`      | Finishes the active route at the return point. |
| `delivery:client:deliverPackage` | Internal delivery step.                        |

### Editable functions

These are in `shared/functions.lua`.

| Function                                                 | Side   | Purpose                                          |
| -------------------------------------------------------- | ------ | ------------------------------------------------ |
| `Functions.Notify(title, msg, type, time)`               | Client | Notification bridge (`Config.NotifySystem`).     |
| `Functions.NotifyServer(player, title, msg, type, time)` | Server | Notification bridge from the server.             |
| `Functions.ShowText(msg)` and `Functions.HideText()`     | Client | Text prompt bridge (`Config.TextUISystem`).      |
| `Functions.GiveKeys(vehicle)`                            | Client | Keys bridge, called when the job vehicle spawns. |
| `Functions.SetFuel(vehicle, amount)`                     | Client | Fuel bridge, called when the job vehicle spawns. |
| `Functions.Lang(key, ...)`                               | Shared | Locale lookup with `%s` formatting.              |

Add your own system by filling the `custom` branch of each function and setting the matching option to `"custom"`.

```lua
-- shared/config.lua
Config.NotifySystem = "custom"

-- shared/functions.lua
elseif Config.NotifySystem == "custom" then
    exports['my_notify']:Send(msg, type, time or 5000)
```

### Database

```sql
CREATE TABLE IF NOT EXISTS `delivery_players` (
  `identifier` VARCHAR(60) NOT NULL,
  `xp` INT(11) NOT NULL DEFAULT 0,
  `skill_points` INT(11) NOT NULL DEFAULT 0,
  `company_balance` INT(11) NOT NULL DEFAULT 0,
  PRIMARY KEY (`identifier`)
);

CREATE TABLE IF NOT EXISTS `delivery_history` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `route_id` VARCHAR(50) NOT NULL,
  `payment` INT(11) NOT NULL,
  `distance` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `completion_date` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX `player_identifier_index` (`player_identifier`)
);

CREATE TABLE IF NOT EXISTS `delivery_owned_vehicles` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `vehicle_id` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`id`)
);

CREATE TABLE IF NOT EXISTS `delivery_player_skills` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `skill_id` VARCHAR(50) NOT NULL,
  `skill_level` INT(11) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  UNIQUE KEY `player_skill_unique` (`player_identifier`, `skill_id`)
);

CREATE TABLE IF NOT EXISTS `delivery_employees` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `owner_identifier` VARCHAR(60) NOT NULL,
  `employee_name` VARCHAR(50) NOT NULL,
  `level` INT(11) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`)
);
```

`delivery_history.route_id` stores the route label. `delivery_owned_vehicles.vehicle_id` is a key of `Config.ShopVehicles`. `delivery_player_skills.skill_id` is a key of `Config.Skills`. `delivery_employees.employee_name` is the `id` of an entry in `Config.AvailableEmployees`. The leaderboard joins `delivery_history` with the ESX `users` table.

### Integration example

Read a player's delivery statistics from another server resource:

```lua
local function GetDeliveryStats(identifier)
    local row = MySQL.single.await(
        'SELECT COUNT(*) AS trips, IFNULL(SUM(payment),0) AS earned, IFNULL(SUM(distance),0) AS km FROM delivery_history WHERE player_identifier = ?',
        { identifier }
    )
    local xp = MySQL.scalar.await('SELECT xp FROM delivery_players WHERE identifier = ?', { identifier }) or 0
    return { trips = row.trips, earned = row.earned, km = row.km, xp = xp }
end

RegisterCommand('deliverystats', function(source)
    local xPlayer = ESX.GetPlayerFromId(source)
    if not xPlayer then return end
    local s = GetDeliveryStats(xPlayer.identifier)
    xPlayer.showNotification(('Trips: %s | Earned: $%s | XP: %s'):format(s.trips, s.earned, s.xp))
end)
```

***

## FAQ

**The console says the folder must be named nexus\_deliveryjob.** Rename the resource folder to exactly `nexus_deliveryjob`.

**Players cannot open the menu.** If `Config.JobNeed` is enabled, they need the job in `Config.JobName`. With `InteractionMethod = 'target'`, make sure `ox_target`, `qb-target` or `qtarget` starts before this resource.

**The uniform is not accepted even when worn.** The check compares every component and prop of `Config.WorkOutfit` exactly. Put the uniform on from the Cloakroom, or set `Config.WorkOutfitRequired = false`.

**All spawn points are occupied.** Add more points to `Config.VehicleSpawns` or enable `Config.ClearZoneEnabled`.

**A route with a required skill cannot be started.** The skill must be at its `maxLevel`, not just unlocked.

**I finished a route but did not get paid.** The route must have been started from the menu, and the player must finish it with the same route. Restarting the resource during a route clears the active route and that run is not paid. Rewards are paid to the bank account.

**How do I add a vehicle to the shop?** Add an entry to `Config.ShopVehicles` and a PNG with the same key in `html/img/`.

**How do I change the currency symbol in messages?** Edit the money messages in the files of `locales/`.

**How do I translate the menu?** Notifications use `Config.Locale`. The panel texts are in `html/index.html` and `html/script.js`.

**Does it work with QBCore?** No. This script is ESX only.

### Before opening a ticket

* Confirm the folder is named `nexus_deliveryjob` and `tables.sql` was imported.
* Confirm the dependencies start before this resource.
* Confirm the systems selected in the configuration are installed and started.
* Check the server and client consoles for errors.
