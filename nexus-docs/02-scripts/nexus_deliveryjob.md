# 🚚 Nexus Deliveryjob

> The most complete delivery job for ESX: dynamic routes, skill tree, vehicle shop, hireable employees with passive income, company bank, uniforms and dangerous cargo — all inside one handcrafted NUI.

**Version:** 1.4.1 · **Framework:** ESX · **Database:** oxmysql

---

## 📝 Description

`nexus_deliveryjob` turns deliveries into a full career. Players visit the delivery HQ, pick a route from a rotating list, get a van (or use one they bought), follow the GPS through every drop-off, carry each package to the door and return to base to get paid.

On top of the basic loop the script adds long-term progression: every route gives XP, XP turns into skill points, and skill points unlock better-paid route types (long distance, valuable cargo, dangerous goods). Players can invest their earnings into their own delivery vehicles, hire NPC employees that generate passive income into a company bank account, and climb a server-wide leaderboard.

Everything happens inside a single menu with nine sections: Home, Quick Jobs, Owned Vehicle Jobs, Skills, Shop, My Vehicles, Employees, Bank and Cloakroom.

---

## ✨ Features

### Routes

- **Rotating route board** — every `Config.RouteRefreshRate` minutes the server picks up to 10 random routes from `Config.Rutas`. The menu shows a live countdown until the next refresh.
- **Two ways to work** — *Quick Jobs* (company vehicle, `Config.DefaultVehicle`) or *Owned Vehicle Jobs* (one of the player's purchased vehicles).
- **Multi-stop deliveries** — each route has any number of delivery points. The player parks near the marker, gets out, grabs the package (with prop + animation) and delivers it at the door.
- **Anti-abuse delivery checks** — the job vehicle must be within `Config.MaxDistanceToDeliver` metres of the drop-off, and the player must be the driver to finish the route.
- **Server-authoritative rewards** — the server records which route each player accepted and pays only the configured reward/XP for that route. The client never sends money or XP values.
- **Route types** — `normal`, `distance`, `expensive`, `dangerous`. Non-normal routes require the matching skill at max level.
- **Dangerous goods** — on `dangerous` routes the vehicle explodes on hard crashes or when its health drops below a configurable threshold, failing the mission.

### Progression

- **XP → skill points** — every `XP_PER_SKILL_POINT` XP grants `SKILL_POINTS_AWARDED` skill points.
- **Skill tree** — three skills by default (Long Distance, Valuable Cargo, Dangerous Goods), each with its own number of levels. Max level is validated on the server.
- **Statistics** — completed trips, money earned, km travelled, XP, owned vehicles, employees and company balance on the Home page.
- **Leaderboard** — top 5 deliverers of the server, by total earnings.

### Economy

- **Vehicle shop** — buy delivery vehicles with custom stats bars (speed, braking, handling). Owned vehicles unlock *Owned Vehicle Jobs*.
- **Vehicle selling** — sell an owned vehicle back for 50% of its price, with confirmation modal and a short cooldown. Money is only paid after the database confirms the vehicle was removed.
- **Employees** — a rotating hiring market (`Config.NumAvailableEmployees` candidates every `Config.EmployeeRefreshRate` minutes). Each employee has a level (1-5), star rating, traits and hire cost.
- **Passive income** — every `Config.EmployeePayoutInterval` minutes each hired employee of an online player produces money according to its tier. Earnings go to the company balance.
- **Company bank** — withdraw the accumulated company balance in cash at any time.

### Immersion

- **Uniform system** — optional mandatory work outfit (male/female) with components and props; the player's civilian clothes are saved and restored from the Cloakroom.
- **NPC or marker** — open the menu through an NPC (any ped model + scenario) or a classic marker.
- **Target or TextUI** — `ox_target`, `qb-target`, `qtarget` or a TextUI prompt.
- **NPC-free zone** — optionally removes NPC pedestrians and traffic around the HQ so spawn/return points are never blocked.
- **In-menu tutorial** — first-time players see a short "how it works" guide (with "don't show again").

---

## 📋 Dependencies

| Resource | Required | Notes |
|---|---|---|
| `es_extended` | Yes | ESX Legacy. |
| `oxmysql` | Yes | Database driver. |
| `illenium-appearance` | Yes | Listed in the manifest dependencies (uniform system). |
| `skinchanger` | Yes | Used to restore the civilian outfit. |
| `esx_skin` | Recommended | Provides `esx_skin:getPlayerSkin` used to save the civilian outfit. |
| `ox_target` / `qb-target` / `qtarget` | Optional | Only when `Config.InteractionMethod = 'target'`. |
| `nexus_notify` / `okokNotify` / `mythic_notify` | Optional | Notification system, selectable in config. |
| `okokTextUI` | Optional | TextUI, selectable in config. |
| `cd_garage` | Optional | Vehicle keys, selectable in config. |
| `LegacyFuel` | Optional | Fuel, selectable in config. |

---

## ⚙️ Installation

1. **Download** the resource from your Cfx.re Keymaster (Granted Assets).
2. **Extract** it into your resources folder. The folder **must** be named exactly `nexus_deliveryjob` — the script stops with an error otherwise.
3. **Import the SQL**: run `tables.sql` in your database. It uses `CREATE TABLE IF NOT EXISTS`, so it is safe to run again on updates.
4. **Add to `server.cfg`** after its dependencies:

```cfg
ensure oxmysql
ensure es_extended
ensure skinchanger
ensure illenium-appearance
# ensure ox_target / nexus_notify / ... (optional systems)
ensure nexus_deliveryjob
```

5. **Configure** `shared/config.lua` (language, notify/TextUI/target/keys/fuel systems, HQ location, routes, prices...).
6. Restart the server (or `refresh` + `ensure nexus_deliveryjob`).

### Updating from 1.4

Replace the resource folder, keeping your `shared/config.lua`, `shared/functions.lua`, `client/target.lua` and `locales/` if you customised them. Then add the new locale key `skill_max_level` to any custom locale file (see [Locales](#-locales--editable-strings)). Re-running `tables.sql` is harmless.

---

## 🔧 Configuration

All options live in `shared/config.lua` (open file, not escrowed).

### General

| Option | Default | Description |
|---|---|---|
| `Config.Locale` | `'en'` | `es`, `en`, `fr`, `pt`, `it`, `de`, `zh-CN`. |
| `Config.JobNeed` | `false` | If `true`, only players with `Config.JobName` can open the menu. |
| `Config.JobName` | `"ambulance"` | Job **name** (not label) required when `JobNeed = true`. |
| `Config.DefaultVehicle` | `"boxville4"` | Vehicle model used for Quick Jobs. |
| `Config.TextUISystem` | `"okok"` | `okok`, `esx`, `custom`. |
| `Config.NotifySystem` | `"nexus"` | `nexus`, `okok`, `mythic`, `esx`, `custom`. |
| `Config.UseKeysSystem` | `"default"` | `cd_garage`, `custom`, `default` (no keys). |
| `Config.UseFuelSystem` | `false` | `legacyfuel`, `custom`, or `false`/`default` (no fuel). |
| `Config.DebugPrints` | `false` | Server console prints on route/employee refresh. |

### Interaction & locations

| Option | Description |
|---|---|
| `Config.InteractionMethod` | `'target'` or `'textui'`. |
| `Config.KeyInteract` | Control ID for all interactions (default `38` = E). |
| `Config.UseNPC` | `true` = spawn an NPC, `false` = draw a marker. |
| `Config.MarkerType` | Marker type when `UseNPC = false` ([marker list](https://docs.fivem.net/docs/game-references/markers/)). |
| `Config.NPC` | `model`, `name`, `coords` (vector4, also used by the marker) and `anim` scenario. |
| `Config.VehicleSpawns` | List of vector4 spawn points. The first free one is used. |
| `Config.ReturnPoint` | Where the vehicle must be brought back to finish the route. |
| `Config.ClearZoneEnabled` / `ClearZoneCenter` / `ClearZoneRadius` | NPC-free zone around the HQ. |

### Routes

| Option | Default | Description |
|---|---|---|
| `Config.RouteRefreshRate` | `1` | Minutes between route board refreshes. |
| `Config.MaxDistanceToDeliver` | `25.0` | Max metres between the vehicle and the drop-off. |
| `Config.Rutas` | — | Route table (see below). |
| `Config.DangerousGoodsExplosion` | `true` | Enable explosions on `dangerous` routes. |
| `Config.ExplosionCrashForce` | `40` | Crash force the vehicle withstands. Higher = harder to explode. |
| `Config.DangerousGoodsHealthPercent` | `0.8` | Explodes when health drops below this fraction. |

Route format:

```lua
Config.Rutas = {
    sandyshores = {                          -- route id (unique key)
        label = "Delivery on Sandy",
        description = "Long range delivery.",
        type = "normal",                     -- normal | distance | expensive | dangerous
        requiredSkill = nil,                 -- nil or a key of Config.Skills
        deliveryPoints = {
            vector3(1978.88, 3819.35, 32.22),
            vector3(1738.15, 3719.77, 34.03),
        },
        recompensa = { min = 3400, max = 3800 }, -- random payout range
        xp = 50,
        distance = 8.4                       -- km, shown in the UI and stored in stats
    },
}
```

> The route **key** (`sandyshores`) is the route id. Rewards, XP and distance are always read on the server from this table.

### Employees

| Option | Default | Description |
|---|---|---|
| `Config.EmployeePayoutInterval` | `30` | Minutes between passive payouts. |
| `Config.MaxEmployees` | `4` | Max hired employees per player. |
| `Config.EmployeeTiers` | — | `[level] = { min, max }` money produced per payout. |
| `Config.AvailableEmployees` | 20 candidates | `id`, `name`, `avatarImage` (`html/img/employees/`), `rating`, `traits`, `cost`, `level`. |
| `Config.EmployeeRefreshRate` | `2` | Minutes between hiring market refreshes. |
| `Config.NumAvailableEmployees` | `4` | Candidates shown at once. |

### Skills & XP

```lua
Config.Skills = {
    distance  = { name = "Long Distance",  description = "...", maxLevel = 3 },
    expensive = { name = "Valuable Cargo", description = "...", maxLevel = 3 },
    dangerous = { name = "Dangerous Goods", description = "...", maxLevel = 3 },
}

Config.XPSystem = {
    XP_PER_SKILL_POINT   = 1000,
    SKILL_POINTS_AWARDED = 1,
}
```

Each level costs one skill point. A route with `requiredSkill` needs that skill at `maxLevel`. Icons for the three default skills are mapped in `html/script.js`.

### Outfit

| Option | Description |
|---|---|
| `Config.WorkOutfitRequired` | If `true`, the uniform is required to start a route. |
| `Config.WorkOutfit` | `male` / `female` tables with `components` and `props` (`drawable`, `texture`). |

### Vehicle shop

```lua
Config.ShopVehicles = {
    speedo = {
        name = 'Vapid Speedo',
        price = 40000,
        model = 'speedo',
        stats = { velocity = 70, braking = 43, handling = 62 } -- 0-100, UI only
    },
}
```

The key (`speedo`) is also the image name: `html/img/speedo.png`. Selling returns 50% of `price`.

---

## 🌐 Locales & Editable Strings

Everything a server owner may want to change is outside the escrow:

| File | Contents |
|---|---|
| `shared/config.lua` | All options, route/skill/employee/vehicle names and descriptions. |
| `shared/functions.lua` | Notify, TextUI, keys and fuel bridges + the `Functions.Lang` helper. |
| `client/target.lua` | Target integration for the HQ NPC. |
| `locales/*.lua` | In-game texts in 7 languages: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh-CN`. |
| `html/index.html`, `html/script.js`, `html/style.css` | The NUI (English by default). |

`Config.Locale` selects the language used by notifications and prompts. If a key is missing in the selected file, the key name itself is shown, so keep all files in sync.

**New in 1.4.1:** `skill_max_level` — shown when a player tries to upgrade a skill that is already maxed.

```lua
Locales['en'] = {
    -- ...
    skill_max_level = "This skill is already at its maximum level",
}
```

The NUI ships in English. To translate it, edit the texts in `html/index.html` and `html/script.js`.

---

## 🔗 Compatibility

| System | Supported |
|---|---|
| Framework | ESX Legacy (`es_extended`) |
| Database | oxmysql |
| Notifications | nexus_notify, okokNotify, mythic_notify, ESX, custom |
| TextUI | okokTextUI, ESX help notification, custom |
| Target | ox_target, qb-target, qtarget (auto-detected) |
| Vehicle keys | cd_garage, custom, none |
| Fuel | LegacyFuel, custom, none |
| Clothing | illenium-appearance / skinchanger + esx_skin |

QBCore is **not** supported by this script.

---

## 💻 Developer API

### Server events (net)

All are triggered by the script's own client. The server validates every one of them.

| Event | Payload | Behaviour |
|---|---|---|
| `delivery:startJob` | `routeId` (string) | Stores the accepted route for that player. Ignored if the id is not in `Config.Rutas`. |
| `delivery:cancelJob` | — | Clears the stored route. |
| `delivery:finishJob` | `routeId` (string) | Pays only if `routeId` matches the stored route; reward/XP/distance are read from `Config.Rutas[routeId]`. The stored route is cleared either way. |
| `delivery:purchaseVehicle` | `{ id }` | Buys `Config.ShopVehicles[id]` with bank money. |
| `delivery:sellOwnedVehicle` | `{ id }` | Deletes the vehicle from the DB and, only if a row was removed, pays 50%. |
| `delivery:upgradeSkill` | `{ id }` | Spends 1 skill point; refused if already at `maxLevel`. |
| `delivery:hireEmployee` | `{ id }` | Hires a candidate (bank money, slot and duplicate checks). |
| `delivery:fireEmployee` | `{ id }` | Fires one of the player's employees. |
| `delivery:withdrawCompanyBalance` | — | Moves the company balance to cash. |

> **Breaking change in 1.4.1:** `delivery:finishJob` no longer accepts a route table. Before 1.4.1 the client sent the whole route table (reward, XP and distance included) directly to the server, which trusted it as-is — a player could tamper with that payload to claim an inflated reward. Since 1.4.1 the client only sends the route id; the server stores the route accepted at start (`delivery:startJob`), checks it matches on finish, and reads reward/XP/distance itself from `Config.Rutas`. If you built anything that triggered `delivery:finishJob` manually, send the route id instead of a table and make sure `delivery:startJob` was sent first.

### Server callback

| Callback | Returns |
|---|---|
| `delivery:getPlayerData` (ESX) | Player stats, skills, employees, owned vehicles, leaderboard, available routes/candidates and refresh timers. |

### Client events

| Event | Description |
|---|---|
| `delivery:client:openMenu` | Opens the delivery menu (used by the target). |
| `delivery:client:finishJob` | Finishes the active route at the return point (used by the target zone). |
| `delivery:client:deliverPackage` | Internal delivery step. |

### Editable functions (`functions.lua`)

| Function | Side | Purpose |
|---|---|---|
| `Functions.Notify(title, msg, type, time)` | Client | Notification bridge (`Config.NotifySystem`). |
| `Functions.NotifyServer(player, title, msg, type, time)` | Server | Notification bridge from server. |
| `Functions.ShowText(msg)` / `Functions.HideText()` | Client | TextUI bridge. |
| `Functions.GiveKeys(vehicle)` | Client | Keys bridge, called when the job vehicle spawns. |
| `Functions.SetFuel(vehicle, amount)` | Client | Fuel bridge, called when the job vehicle spawns. |
| `Functions.Lang(key, ...)` | Shared | Locale lookup with `%s` formatting. |

Add your own system by filling the `custom` branch of each function and setting the matching config value to `"custom"`.

### Database Schema

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
  `route_id` VARCHAR(50) NOT NULL,          -- route label
  `payment` INT(11) NOT NULL,
  `distance` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `completion_date` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX `player_identifier_index` (`player_identifier`)
);

CREATE TABLE IF NOT EXISTS `delivery_owned_vehicles` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `vehicle_id` VARCHAR(50) NOT NULL,        -- key of Config.ShopVehicles
  PRIMARY KEY (`id`)
);

CREATE TABLE IF NOT EXISTS `delivery_player_skills` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `skill_id` VARCHAR(50) NOT NULL,          -- key of Config.Skills
  `skill_level` INT(11) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  UNIQUE KEY `player_skill_unique` (`player_identifier`, `skill_id`)
);

CREATE TABLE IF NOT EXISTS `delivery_employees` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `owner_identifier` VARCHAR(60) NOT NULL,
  `employee_name` VARCHAR(50) NOT NULL,     -- id of Config.AvailableEmployees
  `level` INT(11) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`)
);
```

The leaderboard joins `delivery_history` with the ESX `users` table (`firstname`, `lastname`).

### Integration Example

Read a player's delivery stats from another server resource:

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

Adding a custom notification system:

```lua
-- shared/config.lua
Config.NotifySystem = "custom"

-- shared/functions.lua
elseif Config.NotifySystem == "custom" then
    exports['my_notify']:Send(msg, type, time or 5000)
```

---

## ❓ FAQ

**The script prints "the folder must be named nexus_deliveryjob".**
Rename the resource folder to exactly `nexus_deliveryjob`.

**Players can't open the menu.**
If `Config.JobNeed = true`, they need the job in `Config.JobName`. With `InteractionMethod = 'target'` make sure ox_target, qb-target or qtarget is started *before* the script.

**"You must wear the work uniform" even with the uniform on.**
The check compares every component/prop of `Config.WorkOutfit` exactly. Put the uniform on from the Cloakroom, or set `Config.WorkOutfitRequired = false`.

**"All spawn points are occupied".**
Add more points to `Config.VehicleSpawns` or enable `Config.ClearZoneEnabled`.

**A route with a required skill can't be started.**
The skill must be at its `maxLevel`, not just unlocked.

**I finished a route but didn't get paid.**
Payment requires the route to have been started normally from the menu. Restarting the resource mid-route clears the active route and that run is not paid.

**How do I add a new vehicle to the shop?**
Add an entry to `Config.ShopVehicles` and a PNG with the same key in `html/img/`.

**How do I translate the menu?**
Notifications use `Config.Locale`. The NUI is in English; edit `html/index.html` and `html/script.js` to translate it.

**Does it work with QBCore?**
No, this script is ESX only.

---

## 📋 Changelog

### 1.4.1

- **Security:** `delivery:finishJob` is now server-authoritative. Previously the client sent the whole route table (reward, XP, distance) to the server, which trusted it as-is. Now the client only sends the route id; the server stores the route accepted at start (`delivery:startJob`), checks it matches on finish, and reads reward/XP/distance itself from `Config.Rutas`.
- **Security:** fixed vehicle selling paying out before the database confirmed the deletion. The payout now only happens after the database confirms the vehicle row was removed.
- **Security:** skill upgrades now validate `maxLevel` on the server (previously relied on client-side checks).
- **SQL:** `tables.sql` uses `CREATE TABLE IF NOT EXISTS` (safe to re-import).
- **Locales:** new key `skill_max_level` added in all 7 languages.
- **Language:** English is now the default everywhere (NUI texts, default skill names/descriptions, employee traits, target label, console messages).
- **Escrow:** NUI files listed in `escrow_ignore`.

### 1.4

- Previous release.
