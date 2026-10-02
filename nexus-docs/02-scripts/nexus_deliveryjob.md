# Nexus Delivery Job

A delivery job built as a company-management sim: an XP and skill tree that gates premium routes, a vehicle shop with owned vehicles, hireable employees that generate passive income into a company balance, and a server-wide earnings leaderboard — all behind a single sidebar-driven NUI.

## 📝 Description

Nexus Delivery Job starts where most delivery scripts stop. The basic loop is familiar: talk to the dispatcher at the depot, pick a route, a company van spawns in the alley, you follow the GPS to each delivery point, play a box-carry animation at each one, then drive back to the return point to get paid. What sits on top of that loop is a progression economy.

Every completed route awards cash *and* XP. XP converts to skill points at a configurable rate, and skill points buy levels in three skills — Long Routes, Valuable Goods and Hazardous Goods. Routes are tagged with a required skill, and a route is locked until the matching skill is at its **maximum** level, so a skill is a binary unlock gated behind several points rather than an incremental buff. Premium routes pay roughly double. Hazardous routes add real risk: with `Config.DangerousGoodsExplosion` on, a crash above a force threshold or a vehicle health drop below a percentage detonates the cargo and fails the run outright.

The second economy is the company. Players buy their own delivery vehicles from an in-NUI shop (five models by default, each with display stats and an image), which unlocks the "Owned Vehicle" route list. They also hire employees from a rotating hiring market — each candidate has an avatar, a star rating, trait tags, a one-time cost and a tier from 1 to 5. On a fixed interval every hired employee generates a random amount inside its tier's range, accumulated into a `company_balance` that the player withdraws as cash from the Bank page. Employees keep earning whether or not the player is driving, which is the point.

The NUI is a nine-page dashboard: Home (eight stat cards plus a top-5 leaderboard with medals), Quick Jobs and Owned Vehicle Jobs (route cards with payout and distance, plus a live countdown to the next route rotation), Skills (cards with level dots and an unlock/upgrade button), Shop (vehicle cards with colour-graded stat bars), My Vehicles (a list/detail split view with select-for-job and sell actions), Employees (my-employees and hiring-market columns with a rotation countdown), Bank (a vault graphic and a withdraw button) and Cloakroom (put on / take off the work uniform). Routes and the hiring market both rotate on server timers, and the NUI counts down to each rotation and auto-refreshes when it hits zero.

Supporting systems round it out: an optional uniform requirement that blocks starting a route out of uniform and restores the player's civilian outfit afterwards, a configurable depot marker or NPC with ox_target / qb-target / qtarget support, a ten-slot vehicle spawn queue that skips occupied bays, and an NPC/traffic suppression zone around the depot so spawning and returning vehicles are not blocked by ambient traffic.

## ✨ Features

- **Nine-page NUI dashboard** — a fixed sidebar (Home, Quick Jobs, Owned Vehicle, Skills, Shop, My Vehicles, Employees, Bank, Cloakroom, Close) with a sliding page area and a contextual footer bar that only appears on the two route pages.
- **Home overview** — eight stat cards (completed trips, money earned, kilometres travelled, XP, owned vehicles, employees, skill points, company balance) plus a server-wide top-5 leaderboard by total earnings with gold/silver/bronze medals for the top three.
- **Two route modes** — *Quick Jobs* spawns a free company van (`Config.DefaultVehicle`); *Owned Vehicle Jobs* uses a vehicle the player bought, which is how owned vehicles pay for themselves. Both lists show the same rotating route set.
- **Rotating route offers** — the server reshuffles the available route list every `Config.RouteRefreshRate` minutes and reports the remaining time to the NUI, which renders a live `Nuevos en: MM:SS` countdown and triggers a data refresh the moment it hits zero.
- **Skill-gated routes** — a route with `requiredSkill` is rejected unless the player's matching skill is at `maxLevel`, checked both in the NUI before the request and again on the client before the vehicle spawns.
- **Three-skill progression** — Long Routes, Valuable Goods and Hazardous Goods, each with a configurable `maxLevel` (3 by default, so three skill points each). Rendered as cards with filled level dots and a button that reads Unlock → Upgrade → Maxed, or Insufficient Points.
- **XP → skill points** — each route grants `xp`; crossing a `XP_PER_SKILL_POINT` boundary awards `SKILL_POINTS_AWARDED` points. The award is computed as the difference between the old and new tier, so a single large XP grant that crosses several boundaries correctly awards several points at once.
- **Vehicle shop** — five models by default with price, a product image and freeform stat bars; the stat bar fill colour is graded across five bands from cyan through green and yellow to orange and red depending on the value, so a 90 bar looks visibly different from a 40 bar. Owned vehicles are shown as purchased and cannot be re-bought.
- **My Vehicles split view** — a thumbnail list on the left, a detail panel on the right with the vehicle image, its stat bars, a toggleable "select for job" button and a sell button behind a confirmation modal showing the exact resale figure (50% of the purchase price).
- **Hireable employees** — a rotating market of `Config.NumAvailableEmployees` candidates drawn at random from a 20-strong pool, each with an avatar image, a 1–5 star rating, trait tags and a one-time hiring cost; the player is capped at `Config.MaxEmployees` simultaneous hires and cannot hire the same candidate twice.
- **Passive income ticker** — every `Config.EmployeePayoutInterval` minutes the server iterates online players and credits each hired employee's tier range (`Config.EmployeeTiers`) into that player's `company_balance`.
- **Company bank page** — a vault graphic, the current balance, a disabled-when-empty withdraw button, and a withdrawal that pays out as **cash** (`xPlayer.addMoney`) rather than into the bank account.
- **Hazardous cargo explosions** — on a `dangerous` route, a dedicated 500 ms watchdog thread detonates the vehicle (`AddExplosion`) and fails the mission when vehicle health drops below `Config.DangerousGoodsHealthPercent` of maximum, or when a single tick's health loss exceeds `Config.ExplosionCrashForce`.
- **Vehicle-proximity enforcement** — a delivery only counts if the mission vehicle is within `Config.MaxDistanceToDeliver` metres of the delivery point, so players cannot park the van and run the route in a sports car.
- **Box-carry animation** — at each delivery point the player plays `anim@heists@box_carry@` with a `prop_cs_cardbox_01` attached to the right hand for 5 seconds, with movement, sprint, jump and vehicle-exit controls disabled for the duration. The prop load is guarded by a 5-second timeout that aborts the animation with a notification rather than hanging.
- **Delivery-point markers and blips** — each point gets a routed GPS blip (`SetBlipRoute` in green) and a cylinder marker drawn within 20 m; after the last point a new routed blip guides the player to the return point.
- **Two interaction methods per point** — `target` creates a 2 m sphere/circle zone at each delivery point and a 5 m zone at the return point (ox_target, qb-target or qtarget, auto-detected); `textui` falls back to a distance check plus a TextUI prompt and `Config.KeyInteract`.
- **Depot marker or NPC** — `Config.UseNPC` switches between a configurable ped with a scenario (`cs_floyd` playing `WORLD_HUMAN_CLIPBOARD` by default) and a bare `Config.MarkerType` marker.
- **Optional job gate** — `Config.JobNeed` restricts the whole job to holders of `Config.JobName`, enforced in the target's `canInteract` and in the TextUI interaction path.
- **Uniform system** — with `Config.WorkOutfitRequired`, the player's civilian outfit is captured via `esx_skin:getPlayerSkin` the first time they open the menu, the Cloakroom applies a configurable per-gender work outfit (components and props, including `-1` to clear a prop), and starting a route is blocked while out of uniform. A per-component comparison (`GetPedDrawableVariation` / `GetPedTextureVariation`) does the checking.
- **Ten-bay spawn queue** — `Config.VehicleSpawns` holds ten alley positions; the script takes the first one with no vehicle within 3 m, and notifies the player if all ten are blocked.
- **Depot traffic suppression** — within `Config.ClearZoneRadius` of `Config.ClearZoneCenter`, vehicle and ped density are zeroed every frame and `RemoveVehiclesFromGeneratorsInArea` keeps the spawn alley and return point clear.
- **Mission-state guard** — reopening the menu mid-route shows a dedicated "Active Delivery" screen with a cancel button instead of the dashboard, so a route cannot be started on top of another.
- **Return-point handover** — finishing requires the player to be in the **driver's seat** of the mission vehicle; otherwise a notification explains why. On success the vehicle is deleted, the blip removed and the payout applied.
- **Seven languages** — `es`, `en`, `fr`, `pt`, `it`, `de`, `zh-CN`, all with identical 42-key sets, all outside the escrow.
- **Modular notify / TextUI / keys / fuel** — four independent Config switches routed through `shared/functions.lua`, which ships outside the escrow with a `custom` branch for each.
- **Resource-name guard** — `shared/_resource.lua` aborts the resource if the folder is not named `nexus_deliveryjob`, with a second redundant check at the top of `client.lua` and `server.lua`.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **es_extended (ESX)** | **Required** | Declared in `dependencies` and loaded as `@es_extended/imports.lua`. Both client and server call `exports['es_extended']:getSharedObject()` unconditionally at file scope. There is no QBCore branch anywhere in this resource — it is **ESX-only**. |
| **oxmysql** | **Required** | Declared in `dependencies` and loaded as `@oxmysql/lib/MySQL.lua`. The code uses the `MySQL.Async.*` compatibility API (`fetchAll`, `execute`) with `@named` parameters, which oxmysql provides. |
| **MySQL / MariaDB** | **Required** | Five tables, imported manually from `tables.sql`. They are **not** created automatically. |
| **illenium-appearance** | Declared required | Listed in `dependencies`. Only relevant when `Config.WorkOutfitRequired = true`. |
| **skinchanger** | Declared required | Listed in `dependencies`. `skinchanger:loadSkin` is the event used to restore the civilian outfit, and `esx_skin:getPlayerSkin` is the callback used to capture it. If you run a different clothing stack, those two calls in `client.lua` are the integration points — but note they are inside the escrow. |
| **ESX `users` table** | **Required** | The leaderboard query joins `delivery_history` against `users` on `identifier` and reads `firstname` / `lastname`. |
| **ox_target** / **qb-target** / **qtarget** | Optional | Only when `Config.InteractionMethod = 'target'`. Auto-detected in that order of preference. With no target resource present the script logs a warning and the depot becomes unreachable in `target` mode. |
| **A notification resource** | Optional | `nexus_notify`, `okokNotify`, `mythic_notify` or ESX natives, selected by `Config.NotifySystem`. With no match it prints to the F8 console. |
| **A TextUI resource** | Optional | `okokTextUI` or ESX's `ShowHelpNotification`, selected by `Config.TextUISystem`. With no match it prints to the F8 console. |
| **LegacyFuel** | Optional | Only when `Config.UseFuelSystem = 'legacyfuel'`. |
| **cd_garage** | Optional | Only when `Config.UseKeysSystem = 'cd_garage'`. |
| **A job in ESX** | Conditional | Only when `Config.JobNeed = true`; `Config.JobName` must match a real ESX job name. |
| **Internet access from the game client** | Soft | The NUI loads Font Awesome 6.2.0 from cdnjs and Poppins from Google Fonts. Without outbound access the icons and typography degrade. |

## ⚙️ Installation

1. **Extract the resource.** Download it from the cfx.re (Keymaster) portal and place the folder in `resources`. The folder **must** be named exactly `nexus_deliveryjob` — `shared/_resource.lua` stops the resource otherwise.
2. **Import the database.** Run `tables.sql` against your database. It creates `delivery_players`, `delivery_history`, `delivery_owned_vehicles`, `delivery_player_skills` and `delivery_employees`. The script does **not** create them for you.
   > The shipped statements use plain `CREATE TABLE`, not `CREATE TABLE IF NOT EXISTS`, so re-importing the file on an existing database will error out on the first table. Import it once, or add `IF NOT EXISTS` yourself before re-running it.
3. **Order your `server.cfg`.** ESX, oxmysql, your clothing stack and your target resource all need to start first:
   ```cfg
   ensure oxmysql
   ensure es_extended
   ensure skinchanger
   ensure illenium-appearance
   # ensure ox_target        # if Config.InteractionMethod = 'target'
   # ensure okokNotify       # if Config.NotifySystem  = 'okok'
   # ensure okokTextUI       # if Config.TextUISystem  = 'okok'
   ensure nexus_deliveryjob
   ```
4. **Set the language.** `Config.Locale` in `shared/config.lua` (`es`, `en`, `fr`, `pt`, `it`, `de`, `zh-CN`). Note the warning in the file itself: the NUI is **not** covered by the locale files and has to be translated separately in `html/script.js` and `html/index.html`.
5. **Pick your external systems.** `Config.NotifySystem`, `Config.TextUISystem`, `Config.UseKeysSystem` and `Config.UseFuelSystem`. Each has a `custom` option with a marked block in `shared/functions.lua`. Defaults as shipped are `nexus` notify and `okok` TextUI — change them if you do not run those resources.
6. **Place the depot.** Set `Config.NPC.coords` (this single value drives both the NPC and the marker), then set `Config.ClearZoneCenter` to the same position, `Config.VehicleSpawns` to your spawn bays and `Config.ReturnPoint` to where the van is handed back.
7. **Choose the interaction method.** `Config.InteractionMethod = 'target'` (recommended) or `'textui'`. Note that in `target` mode the dispatcher NPC is always created regardless of `Config.UseNPC`.
8. **Build your routes.** Edit `Config.Rutas`. The commented template above the table documents every field. Three routes ship by default (`sandyshores`, `paletobay`, `beach1`); only one of them uses a skill (`distance`), so if you want the Valuable Goods and Hazardous Goods skills to be worth buying you need to add `expensive` and `dangerous` routes yourself.
9. **Review the economy.** `Config.EmployeeTiers`, `Config.AvailableEmployees` costs, `Config.ShopVehicles` prices, `Config.EmployeePayoutInterval`, `Config.XPSystem` and each route's `recompensa` range all feed the same economy — tune them together.
10. **Configure the uniform.** If `Config.WorkOutfitRequired = true`, set `Config.WorkOutfit.male` and `.female` to real component and texture ids for your clothing pack. Wrong values mean players can never satisfy the uniform check and can never start a route.
11. **Restart the server**, then walk to the depot and interact with the dispatcher (target option *"Abrir Menú de Repartos"*, or `E` in `textui` mode) to open the dashboard.

## 🔧 Configuration

Everything lives in `shared/config.lua`, which ships outside the escrow.

### Framework

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Locale` | string | `'en'` | Active language for the Lua-side strings: `es`, `en`, `fr`, `pt`, `it`, `de`, `zh-CN`. Resolved by `Functions.Lang`. **Does not translate the NUI** — see Locales below. |
| `Config.JobNeed` | boolean | `false` | When `true`, only players whose ESX job matches `Config.JobName` can open the menu. Enforced in the target's `canInteract` and in the `textui` interaction. |
| `Config.JobName` | string | `'ambulance'` | The ESX job **name** (not label) required when `Config.JobNeed` is `true`. |
| `Config.DefaultVehicle` | string | `'boxville4'` | Vehicle model spawned for "Quick Jobs" routes. |
| `Config.TextUISystem` | string | `'okok'` | `'okok'` → `okokTextUI`, `'esx'` → `ESX.ShowHelpNotification`, `'custom'` → your own block in `shared/functions.lua`. Any other value prints to the F8 console. Note `HideText` only has an implementation for `okok` and `custom`. |
| `Config.NotifySystem` | string | `'nexus'` | `'nexus'` → `nexus_notify`, `'okok'` → `okokNotify`, `'mythic'` → `mythic_notify`, `'esx'` → ESX natives, `'custom'` → your own block. Any other value prints to the F8 console. |
| `Config.UseKeysSystem` | string | `'default'` | `'cd_garage'` → hands the player keys to the spawned van, `'custom'` → your own block, `'default'` → no key system (the van simply cannot be locked or unlocked). |
| `Config.UseFuelSystem` | boolean/string | `false` | `'legacyfuel'` → `exports['legacyfuel']:SetFuel`, `'custom'` → your own block, `'default'`/`false` → no fuel handling (the van never consumes fuel). |

### Interaction

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.InteractionMethod` | string | `'target'` | `'target'` → sphere/circle zones for the depot, the delivery points and the return point, using ox_target, qb-target or qtarget (auto-detected in that order). `'textui'` → distance checks plus a TextUI prompt and `Config.KeyInteract`. |
| `Config.KeyInteract` | number | `38` (`E`) | Control id used for every interaction in `textui` mode. Ignored in `target` mode. |
| `Config.UseNPC` | boolean | `false` | `true` → spawn the dispatcher ped at the depot; `false` → draw a marker instead. **Only honoured in `textui` mode** — in `target` mode `client/target.lua` always creates the ped, because a target zone needs an entity. |
| `Config.MarkerType` | number | `20` | Marker type drawn at the depot when `Config.UseNPC = false` in `textui` mode. See the FiveM marker reference. |
| `Config.NPC.model` | string | `'cs_floyd'` | Dispatcher ped model. |
| `Config.NPC.name` | string | `'Siro'` | Dispatcher name, substituted into the `interact_npc` locale string in `textui` + `UseNPC` mode. |
| `Config.NPC.coords` | vector4 | `vector4(-424.285706, -2789.868164, 6.515747, 323.149597)` | Depot position and heading. **Used by both the NPC and the marker** — change this to relocate the depot. The ped is created at `z - 1.0`. |
| `Config.NPC.anim` | string | `'WORLD_HUMAN_CLIPBOARD'` | Scenario played in place by the dispatcher. |
| `Config.VehicleSpawns` | vector4[] | 10 alley positions | Candidate spawn bays, tried in order; the first with no vehicle within 3 m is used. If all are occupied the player is notified and the route does not start. |
| `Config.ReturnPoint` | vector3 | `vector3(-408.210999, -2799.494385, 5.993408)` | Where the mission vehicle must be returned to finish the route. |

### Routes

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.RouteRefreshRate` | number (min) | `1` | How often the server reshuffles the available route list. The NUI counts down to this and auto-refreshes. |
| `Config.MaxDistanceToDeliver` | number (m) | `25.0` | Maximum distance the mission vehicle may be from a delivery point for the delivery to count. Prevents doing the route in a faster private car. |
| `Config.Rutas` | table | 3 routes | The route catalogue. See the structure below. |
| `Config.DangerousGoodsExplosion` | boolean | `true` | Enables the explosion watchdog on `dangerous`-type routes. |
| `Config.ExplosionCrashForce` | number | `40` | Maximum health loss in a single 500 ms tick the vehicle can absorb without detonating. Higher = harder to blow up. |
| `Config.DangerousGoodsHealthPercent` | number | `0.8` | Vehicle health floor as a fraction of maximum; below this the cargo detonates. `0.8` = 80%. Lower = easier to survive. |

Route structure — each key is the internal id the server uses:

```lua
Config.Rutas = {
    sandyshores = {                        -- internal id, used in delivery_history.route_id
        label       = "Delivery on Sandy", -- shown to the player
        description = "Long range delivery.",
        type        = "normal",            -- "normal" | "distance" | "expensive" | "dangerous"
        requiredSkill = nil,               -- nil, or a key of Config.Skills; must be at maxLevel
        deliveryPoints = {                 -- visited in order, one package each
            vector3(1978.88, 3819.35, 32.22),
            vector3(1738.15, 3719.77, 34.03),
        },
        recompensa  = { min = 3400, max = 3800 },  -- random payout in this range
        xp          = 50,                  -- XP awarded on completion
        distance    = 8.4                  -- display only, and logged to delivery_history.distance
    },
}
```

`type = "dangerous"` is what arms the explosion watchdog; `requiredSkill` is what locks the route. They are independent fields, so a route can be dangerous without requiring the Hazardous Goods skill, or vice versa. **As shipped, no route has `type = "dangerous"` and none requires `expensive` or `dangerous`**, so the explosion system and two of the three skills have nothing to act on until you add routes for them.

### Employees

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.EmployeePayoutInterval` | number (min) | `30` | How often the passive-income ticker runs. It only pays players who are online at that moment. |
| `Config.MaxEmployees` | number | `4` | Maximum simultaneous hires per player. Enforced server-side. |
| `Config.EmployeeTiers` | table | 5 tiers | Payout range per employee level, in cash per interval. |
| `Config.AvailableEmployees` | table | 20 candidates | The candidate pool the hiring market draws from. |
| `Config.EmployeeRefreshRate` | number (min) | `2` | How often the hiring market is reshuffled. The NUI counts down to this. |
| `Config.NumAvailableEmployees` | number | `4` | How many candidates the market shows at a time, drawn at random without repeats. |

```lua
Config.EmployeeTiers = {
    [1] = { min = 100, max = 200 },   -- paid per Config.EmployeePayoutInterval
    [2] = { min = 250, max = 400 },
    [3] = { min = 300, max = 600 },
    [4] = { min = 400, max = 700 },
    [5] = { min = 500, max = 800 },
}

Config.AvailableEmployees = {
    {
        id          = 'cand01',          -- internal id, stored in delivery_employees.employee_name
        name        = 'James',           -- shown to the player
        avatarImage = 'employee4.png',   -- file in html/img/employees/
        rating      = 1,                 -- 1-5 stars, display only
        traits      = {'Barato', 'Aprendiz'},  -- tag chips, display only
        cost        = 5000,              -- one-time hiring cost, paid from the bank account
        level       = 1,                 -- 1-5, indexes Config.EmployeeTiers
    },
    -- …19 more
}
```

The `rating` is cosmetic; `level` is what determines income. Keeping them aligned is a convention, not a rule. Note that the NUI renders a "Salario" (salary) figure on hired employee cards, but no candidate in the shipped config defines a `salary` field, so the server substitutes a flat `500` for every employee — the figure is cosmetic and does not come out of anyone's pocket.

### Skills

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Skills` | table | 3 skills | The skill catalogue. The key is the id referenced by a route's `requiredSkill`. |
| `Config.XPSystem.XP_PER_SKILL_POINT` | number | `1000` | XP needed per skill point. |
| `Config.XPSystem.SKILL_POINTS_AWARDED` | number | `1` | Points granted each time a threshold is crossed. |

```lua
Config.Skills = {
    distance = {
        name        = "Rutas Lejanas",
        description = "Desbloquea el acceso a entregas de larga distancia…",
        maxLevel    = 3,          -- 3 skill points to unlock; routes need maxLevel, not level 1
    },
    expensive = { name = "Mercancía Valiosa",  description = "…", maxLevel = 3 },
    dangerous = { name = "Mercancía Peligrosa", description = "…", maxLevel = 3 },
}
```

Adding a fourth skill also requires adding its Font Awesome icon to the `skillIcons` maps in `html/script.js` (two copies: one for route cards, one for skill cards), otherwise it renders with a question-mark icon.

### Outfit

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.WorkOutfitRequired` | boolean | `true` | When `true`, the uniform must be worn to start a route, checked component by component. When `false` the check always passes (but the Cloakroom page still works). |
| `Config.WorkOutfit.male` / `.female` | table | see below | Per-gender components and props. Gender is derived from the ped model (`mp_f_freemode_01` → female, anything else → male). |

```lua
Config.WorkOutfit = {
    male = {
        components = {
            ['mask']   = { drawable = 0,   texture = 0 },
            ['arms']   = { drawable = 0,   texture = 0 },
            ['pants']  = { drawable = 10,  texture = 0 },
            ['bags']   = { drawable = 0,   texture = 0 },
            ['shoes']  = { drawable = 36,  texture = 0 },
            ['tshirt'] = { drawable = 15,  texture = 0 },
            ['vest']   = { drawable = 250, texture = 0 },
            ['decals'] = { drawable = 0,   texture = 0 },
            ['bproof'] = { drawable = 0,   texture = 0 },
        },
        props = {
            -- ['hats'] = { drawable = -1, texture = 0 },  -- -1 clears the prop
        }
    },
    female = { components = { --[[ … ]] }, props = {} }
}
```

Supported component names: `face`, `mask`, `hair`, `torso`, `pants`, `bags`, `shoes`, `neck`, `tshirt`, `bproof`, `decals`, `vest`, `arms`. Supported prop names: `hats`, `glasses`, `ears`, `watches`, `bracelets`. Note that `torso` and `vest` both map to component id 11, so setting both is contradictory — set only `vest`.

### Shop

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.ShopVehicles` | table | 5 vehicles | The vehicle catalogue. The key is the internal id, stored in `delivery_owned_vehicles.vehicle_id` **and** used to resolve the image path. |

```lua
Config.ShopVehicles = {
    speedo = {                  -- key must match html/img/<key>.png
        name  = 'Vapid Speedo', -- shown to the player
        price = 40000,          -- paid from the bank account; resale is 50%
        model = 'speedo',       -- actual spawn model (may differ from the key)
        stats = {               -- freeform; each becomes a graded bar in the UI
            velocity = 70,
            braking  = 43,
            handling = 62,
        }
    },
}
```

The key and the `model` are independent — `boxville2` in the shipped config spawns `boxville4`, and `burrito` spawns `burrito3`. The **key** is what the image lookup uses (`html/img/<key>.png`), so a new vehicle needs a PNG named after its key added to `html/img/` and to the `files` list in `fxmanifest.lua`.

### Miscellaneous

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.ClearZoneEnabled` | boolean | `true` | Suppresses ambient vehicles and peds near the depot so spawn bays and the return point stay clear. Recommended on. |
| `Config.ClearZoneCenter` | vector3 | `vector3(-424.285706, -2789.868164, 6.515747)` | Centre of the suppression zone. Keep it in sync with `Config.NPC.coords`. |
| `Config.ClearZoneRadius` | number (m) | `200.0` | Radius of the suppression zone. Note that while the player is inside it the loop runs every frame (`Wait(0)`). |
| `Config.DebugPrints` | boolean | `false` | Prints the route and employee refresh lines to the server console. Leave `false` in production. |

## 🌐 Locales & Editable Strings

Seven languages ship, all in `escrow_ignore` and all loaded individually in `shared_scripts`: `locales/es.lua`, `en.lua`, `fr.lua`, `pt.lua`, `de.lua`, `it.lua`, `zh-CN.lua`. Key sets are **identical across all seven** — 42 keys each (verified).

Each file assigns `Locales['<lang>']`; `Functions.Lang(key, ...)` in `shared/functions.lua` resolves `Locales[Config.Locale][key]`, returns the key itself when missing, counts the `%s` placeholders in the string and pads missing arguments with the literal `"nil"` so a mismatched call never throws.

**The NUI is not localised.** `Config.Locale` says so explicitly in the config comment: *"YOU ALSO NEED TO TRANSLATE THE NUI (script.js, index.html, style.css)"*. There is no i18n layer in the NUI at all. As shipped the NUI text is a mix:

| Where | Language as shipped | Examples |
|---|---|---|
| `html/index.html` | **English** | page titles, stat card labels, sidebar, bank page, cloakroom, tutorial, confirmation modal |
| `html/script.js` (dynamic) | **Spanish** | `Comprar` / `Comprado`, `Desbloquear` / `Mejorar` / `Maximizado` / `Puntos Insuficientes`, `Mis Empleados (n/m)`, `Contratar` / `Contratado` / `Despedir`, `Seleccionar para Trabajo` / `Seleccionado`, `Vender Vehículo`, `Confirmar Venta`, `Salario:` / `Coste:`, `Nuevos en: MM:SS`, `Actualizando...`, `Desconocido` |

Both files ship as `files` in the manifest and are therefore never encrypted, so all of this is directly editable — it just has to be done by hand, twice if you want two languages.

**Two player-facing strings are duplicated in Spanish inside the NUI** even though a translated locale key already exists for them:

| NUI string (`html/script.js`) | Existing locale key |
|---|---|
| `Necesitas la habilidad '%s' al nivel máximo para esta ruta.` | `need_skill` |
| `Debes llevar puesto el uniforme de trabajo para empezar la ruta.` | `must_wear_outfit` |

Both are sent to the `notify` NUI callback, so they bypass `Functions.Lang` entirely and will stay Spanish on an English server.

**One string is hardcoded inside the escrow and cannot be changed.** In `client/client.lua` (which is **not** in `escrow_ignore`), the qb-target / qtarget branch for delivery points uses a literal label:

```lua
options = { { event = "delivery:client:deliverPackage", icon = "fa-solid fa-box-open",
              label = "Entregar Paquete" } }
```

The ox_target branch right above it correctly uses `Functions.Lang('deliver_package_target')`. So on a server running ox_target the label is translated; on qb-target or qtarget it is permanently Spanish. The same file also prints a Spanish console error when the box prop fails to load.

**Editable but not using the locale system:** `client/target.lua` *is* in `escrow_ignore`, and its three target labels are the hardcoded Spanish `"Abrir Menú de Repartos"` instead of a locale key. Easy to change, but it means the depot label does not follow `Config.Locale`.

**Config strings that are player-facing but only exist in one language:** `Config.Skills[*].name` and `.description` are Spanish; `Config.AvailableEmployees[*].traits` are Spanish (`'Barato'`, `'Aprendiz'`, `'Experto'`, `'Larga Distancia'`, `'Mercancía Peligrosa'`); `Config.Rutas[*].label` and `.description` are English. All three are in `config.lua` outside the escrow and are meant to be edited, but they are not covered by the locale files, so a multilingual server cannot serve both.

**Seven unused locale keys.** Present in all seven languages but never referenced by any Lua file: `already_job`, `get_out_to_deliver`, `must_be_in_vehicle`, `must_drive_vehicle`, `must_wear_outfit`, `need_to_pick_before`, `not_enough_skill_points`. The last one is notable — the server uses `Functions.Lang("not_enough_skill_points")` in `delivery:upgradeSkill`, so it *is* used; the other six are genuinely dead (two of them superseded by the hardcoded NUI strings above).

## 🔗 Compatibility

| System | How it is selected | Notes |
|---|---|---|
| **Framework** | hard dependency | **ESX only.** `exports['es_extended']:getSharedObject()` is called at file scope in `client.lua`, `server.lua` and `target.lua`, `@es_extended/imports.lua` is loaded in `shared_scripts`, and the code uses `xPlayer.getAccount('bank')`, `addAccountMoney`, `removeAccountMoney`, `addMoney`, `ESX.RegisterServerCallback`, `ESX.TriggerServerCallback`, `ESX.Game.SpawnVehicle`, `ESX.Game.DeleteVehicle`, `ESX.GetPlayers` and the ESX `users` table. There is no QBCore path. |
| **Database** | hard dependency | oxmysql, via the `MySQL.Async.*` compatibility API with `@named` parameters. |
| **Notifications** | `Config.NotifySystem` | `'nexus'` (`nexus_notify`), `'okok'` (`okokNotify`), `'mythic'` (`mythic_notify`), `'esx'` (natives) or `'custom'`. Client and server variants are separate functions (`Functions.Notify` and `Functions.NotifyServer`) in `shared/functions.lua`. Unmatched values fall through to an F8 print. |
| **TextUI** | `Config.TextUISystem` | `'okok'` (`okokTextUI` with `Open`/`Close`), `'esx'` (`ShowHelpNotification`) or `'custom'`. Only used in `textui` interaction mode. `Functions.HideText` has no `esx` branch, which is harmless because ESX help notifications expire on their own. |
| **Target system** | `Config.InteractionMethod = 'target'` | Auto-detected at runtime via `GetResourceState`, preferring **ox_target**, then **qb-target**, then **qtarget**. ox_target uses the modern `addSphereZone` / `addLocalEntity` API; the other two use the legacy `AddCircleZone` / `AddTargetEntity` API. All three are removed with `removeZone`. |
| **Vehicle keys** | `Config.UseKeysSystem` | `'cd_garage'` (`cd_garage:AddKeys` with the plate from `exports['cd_garage']:GetPlate`), `'custom'` or `'default'` (no key handling). |
| **Fuel** | `Config.UseFuelSystem` | `'legacyfuel'` (`exports['legacyfuel']:SetFuel`), `'custom'` or `'default'`/`false` (no fuel handling). Note the shipped branches hardcode a fill of `100` and ignore the `amount` argument, so the van always spawns with a full tank. The function also contains a `trk_fueling` branch marked `--DONT USE THIS` — a developer leftover; do not select it. |
| **Clothing** | hardcoded | `esx_skin:getPlayerSkin` to capture the civilian outfit and `skinchanger:loadSkin` to restore it, both in `client/client.lua` (inside the escrow). `illenium-appearance` and `skinchanger` are both declared as dependencies. The uniform itself is applied with native `SetPedComponentVariation` / `SetPedPropIndex`, so it does not need a clothing resource — only the *restore* path does. |
| **Banking** | ESX accounts | Route payouts, vehicle purchases/sales and employee hires use the ESX `bank` account. The company-balance withdrawal pays out as **cash** (`addMoney`), by design — the Bank page states this. No external banking resource is used. |
| **Branding** | — | The NUI uses a cyan / green / pink / purple palette (`#8be9fd`, `#50fa7b`, `#ff79c6`, `#bd93f9`) and the Poppins typeface, which does not match the Nexus purple-and-amber identity used elsewhere in the catalogue. |

## 💻 Developer API

### Client Exports

**None.** `nexus_deliveryjob` registers no client export. Integrate through the events and NUI callbacks below, and through `shared/functions.lua`.

### Server Exports

**None.** `nexus_deliveryjob` registers no server export. There is also no server-side "is this player on a route" accessor — mission state lives only in the client's `enMision` local. Read the database tables directly if you need state from another resource.

### Events — Emitted

| Event | Side | Payload | When |
|---|---|---|---|
| `delivery:internal:handleOpenMenu` | client → client (local) | *none* | Fired by `client/target.lua` when the depot target option is selected, and the handler in `client.lua` saves the civilian outfit and opens the menu. This is the seam between the two client files. |
| `delivery:finishJob` | client → server | `ruta: table` — the full route object | The player finishes the route at the return point while in the driver's seat. The server reads `ruta.recompensa.min/max`, `ruta.xp`, `ruta.distance` and `ruta.label` from this payload. |
| `delivery:purchaseVehicle` | client → server | `{ id }` — a `Config.ShopVehicles` key | Buy button in the shop. |
| `delivery:sellOwnedVehicle` | client → server | `{ id }` | Sell confirmed in the My Vehicles modal. |
| `delivery:upgradeSkill` | client → server | `{ id }` — a `Config.Skills` key | Unlock/Upgrade button on a skill card. |
| `delivery:hireEmployee` | client → server | `{ id }` — a candidate id | Hire button in the hiring market. |
| `delivery:fireEmployee` | client → server | `{ id }` — the `delivery_employees.id` row id | Fire button on a hired employee card. |
| `delivery:withdrawCompanyBalance` | client → server | *none* | Withdraw button on the Bank page. |
| `skinchanger:loadSkin` | client → client | `civilianOutfit` | Restoring the civilian outfit from the Cloakroom. |
| `cd_garage:AddKeys` | client → client | plate string | `Functions.GiveKeys`, only on the `cd_garage` branch. |
| `okokNotify:Alert`, `esx:showNotification`, `mythic_notify:client:SendAlert` | server → client | per resource | `Functions.NotifyServer`, depending on `Config.NotifySystem`. |

### Events — Listened

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `delivery:client:openMenu` | server → client (`RegisterNetEvent`) | *none* | Public entry point. Relays to `delivery:internal:handleOpenMenu`, which saves the civilian outfit and opens the dashboard. **Use this one** to open the menu from another resource. |
| `delivery:client:deliverPackage` | server → client (`RegisterNetEvent`) | *none* | Confirms a package delivery at the current point. This is the event the target zones fire. Clears the client's `isWaitingForDelivery` flag, which lets the mission loop advance. |
| `delivery:client:finishJob` | server → client (`RegisterNetEvent`) | *none* | Confirms the route is finished. Checks the player is in the driver's seat of the mission vehicle, then triggers `delivery:finishJob` on the server and cleans up. This is the event the return-point target zone fires. |
| `esx:playerLoaded` | client | `xPlayer` | Caches player data. |
| `delivery:finishJob` | client → server (`RegisterNetEvent`) | `ruta: table` | Pays the route, writes history, awards XP and skill points. |
| `delivery:purchaseVehicle` / `sellOwnedVehicle` / `upgradeSkill` / `hireEmployee` / `fireEmployee` / `withdrawCompanyBalance` | client → server (`RegisterNetEvent`) | see above | Economy actions. |

### Server Callbacks

| Callback | Returns | Purpose |
|---|---|---|
| `delivery:getPlayerData` | one object, see below | The single source of truth for the whole NUI. Registered with `ESX.RegisterServerCallback`, requested by the client with `ESX.TriggerServerCallback`, and pushed to the NUI as the `update` message's `info`. |

Payload shape:

```lua
{
    name            = 'Firstname Lastname',  -- xPlayer.getName()
    xp              = 0,                     -- delivery_players.xp
    skillPoints     = 0,                     -- delivery_players.skill_points
    companyBalance  = 0,                     -- delivery_players.company_balance
    trips           = 0,                     -- COUNT(*)      from delivery_history
    money           = 0,                     -- SUM(payment)  from delivery_history
    km              = 0,                     -- SUM(distance) from delivery_history
    ownedVehicles   = { 'speedo', … },       -- delivery_owned_vehicles.vehicle_id
    leaderboard     = {                      -- top 5 by total earnings, server-wide
        { name = '…', rank = 1, total_earnings = 0, total_trips = 0 },
    },
    routes          = {                      -- current rotation
        { id = 'sandyshores', data = { … } },
    },
    shopVehicles    = Config.ShopVehicles,
    availableCandidates = { … },             -- current hiring market
    skills          = {                      -- every Config.Skills entry, merged with the player's level
        { id = 'distance', name = '…', description = '…', maxLevel = 3, currentLevel = 0 },
    },
    myEmployees     = {
        { id = 1, name = 'James', level = 1, salary = 500, rating = 1, traits = {…}, avatarImage = '…' },
    },
    maxEmployees    = Config.MaxEmployees,
    workOutfitRequired = Config.WorkOutfitRequired,
    routeTimeLeft    = 0,                    -- seconds until the next route rotation
    employeeTimeLeft = 0,                    -- seconds until the next market rotation
}
```

`salary` is always `500` because no shipped candidate defines the field — the server falls back to `template.salary or 500`.

### NUI Callbacks

Registered on the client with `RegisterNUICallback`.

| Callback | Payload | Returns | Effect |
|---|---|---|---|
| `closeMenu` | — | `{ ok = true }` | Release NUI focus. |
| `refreshData` | — | `{ ok = true }` | Re-run the `delivery:getPlayerData` callback and push `update`. |
| `isWearingOutfit` | — | `{ wearing: boolean }` | Component-by-component uniform check. Returns `true` unconditionally when `Config.WorkOutfitRequired` is `false`. |
| `cancelCurrentJob` | — | `{ ok = true }` | Abort the active route: delete the vehicle, remove the blip and the target zone, then return to the dashboard. |
| `notify` | `{ message, type }` | `{ ok = true }` | Lets the NUI raise a notification through `Functions.Notify` with `Functions.Lang('job_name')` as the title. The NUI uses this for its two hardcoded Spanish validation messages. |
| `startQuickJob` | `{ id }` | `{ ok: boolean }` | Start a route with a spawned company van. `ok = false` if the route is unknown, the skill is missing or all spawn bays are blocked. |
| `startOwnedVehicleJob` | `{ id, vehicle }` | `{ ok: boolean }` | Same, using an owned vehicle. `ok = false` if `vehicle` is nil. |
| `setWorkOutfit` | — | `{ ok = true }` | Apply the configured uniform with native component/prop calls. |
| `setCivilianOutfit` | — | `{ ok = true }` | Restore the cached civilian outfit via `skinchanger:loadSkin`. |
| `withdrawCompanyBalance` | — | `{ ok = true }` | Relay to the server event. |
| `upgradeSkill` / `purchaseVehicle` / `sellOwnedVehicle` / `hireEmployee` / `fireEmployee` | `{ id }` | `{ ok = true }` | Relay to the matching server event, then refresh after 500 ms. |

### NUI Messages (Lua → JS)

| Action | Payload | Effect |
|---|---|---|
| `show` | — | Reveal the dashboard and navigate to Home. |
| `update` | `{ info }` | Repaint every page from the `delivery:getPlayerData` payload and (re)start the rotation countdowns. |
| `inMission` | — | Hide the dashboard and show the "Active Delivery" cancel screen instead. |

### Editable Functions (functions.lua)

`shared/functions.lua` is in `escrow_ignore` and loaded as a `shared_script`, so all of the below exist on both client and server.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `Functions.Notify` | `Functions.Notify(title, msg, type, time)` | **Client** — everywhere a player needs feedback: mission start, cancel, explosion, vehicle too far, not the driver, prop error, missing skill, blocked spawn bays, outfit on/off/not-found, and the `notify` NUI callback. | Route a client-side notification. `type` is passed through as the notification type (`'success'`, `'error'`, `'warning'`, `'primary'`, `'info'`); `time` defaults to `5000`. Branches on `Config.NotifySystem`. Return value ignored. The `nexus` branch uses `exports['nexus_notify']:Alert(title, msg, time, type, true)`. |
| `Functions.NotifyServer` | `Functions.NotifyServer(player, title, msg, type, time)` | **Server** — every economy action: payout, XP points, purchase, sale, sale cooldown, skill upgrade, insufficient points/money/slots, hire, already hired, fire, withdrawal, nothing to withdraw. | Route a server→client notification. `player` is the server id. Branches on `Config.NotifySystem`. Return value ignored. |
| `Functions.ShowText` | `Functions.ShowText(msg)` | **Client** — `textui` mode only: near the depot, and near a delivery point or the return point. | Show a persistent TextUI prompt. Branches on `Config.TextUISystem`. Return value ignored. |
| `Functions.HideText` | `Functions.HideText()` | **Client** — when leaving a prompt radius, after an interaction, and on mission cancel. | Hide the prompt. Only `okok` and `custom` have an implementation; the ESX branch is intentionally absent because help notifications self-expire. Return value ignored. |
| `Functions.GiveKeys` | `Functions.GiveKeys(vehicle)` | **Client** — `spawnVehicle`, immediately after `ESX.Game.SpawnVehicle` returns. | Hand the player keys to the mission vehicle. `vehicle` is the entity handle. Branches on `Config.UseKeysSystem`. Return value ignored. |
| `Functions.SetFuel` | `Functions.SetFuel(vehicle, amount)` | **Client** — `iniciarMision`, called as `Functions.SetFuel(veh, 100.0)` after the player is warped in. | Set the mission vehicle's fuel. Branches on `Config.UseFuelSystem`. **The shipped `legacyfuel` and `trk_fueling` branches ignore `amount` and hardcode `100`** — honour `amount` in your own branch if you want partial tanks. Return value ignored. |
| `Functions.Lang` | `Functions.Lang(key, ...)` → `string` | **Client and server** — every notification title and body, and the ox_target zone labels. | Resolve a locale string. Returns `Locales[Config.Locale][key]`, or the key itself if missing. Counts `%s` placeholders in the string and pads missing arguments with the literal `"nil"`, so a call with too few arguments renders `nil` instead of erroring. Marked *"do not touch below here"* — treat it as internal. |

Example of wiring a custom notification stack:

```lua
-- shared/functions.lua
-- Config.NotifySystem = 'custom'

function Functions.Notify(title, msg, type, time)
    if Config.NotifySystem == 'custom' then
        exports['my_notify']:show({ title = title, body = msg, kind = type, ms = time or 5000 })
        return
    end
    -- …leave the shipped branches below intact
end

function Functions.NotifyServer(player, title, msg, type, time)
    if Config.NotifySystem == 'custom' then
        TriggerClientEvent('my_notify:show', player, title, msg, type, time or 5000)
        return
    end
end
```

### Database Schema

Five tables, imported manually from `tables.sql`. Not created at runtime.

```sql
-- Per-player progression and company balance. One row per identifier, upserted.
CREATE TABLE `delivery_players` (
  `identifier`      VARCHAR(60) NOT NULL,
  `xp`              INT(11) NOT NULL DEFAULT 0,
  `skill_points`    INT(11) NOT NULL DEFAULT 0,
  `company_balance` INT(11) NOT NULL DEFAULT 0,
  PRIMARY KEY (`identifier`)
);

-- One row per completed route. Feeds the Home stats and the leaderboard.
CREATE TABLE `delivery_history` (
  `id`                INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `route_id`          VARCHAR(50) NOT NULL,   -- NOTE: the route LABEL is written here, not the key
  `payment`           INT(11) NOT NULL,
  `distance`          DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `completion_date`   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX `player_identifier_index` (`player_identifier`)
);

-- Vehicles bought from the in-NUI shop. vehicle_id is a Config.ShopVehicles key.
CREATE TABLE `delivery_owned_vehicles` (
  `id`                INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `vehicle_id`        VARCHAR(50) NOT NULL,
  PRIMARY KEY (`id`)
);

-- Skill levels. skill_id is a Config.Skills key.
CREATE TABLE `delivery_player_skills` (
  `id`                INT(11) NOT NULL AUTO_INCREMENT,
  `player_identifier` VARCHAR(60) NOT NULL,
  `skill_id`          VARCHAR(50) NOT NULL,
  `skill_level`       INT(11) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  UNIQUE KEY `player_skill_unique` (`player_identifier`, `skill_id`)
);

-- Hired employees. employee_name holds the candidate ID (e.g. 'cand01'), not the display name.
CREATE TABLE `delivery_employees` (
  `id`                INT(11) NOT NULL AUTO_INCREMENT,
  `owner_identifier`  VARCHAR(60) NOT NULL,
  `employee_name`     VARCHAR(50) NOT NULL,
  `level`             INT(11) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`)
);
```

Notes that matter when you query these tables yourself:

- `identifier` is the raw ESX `xPlayer.identifier` (`license:…` / `char1:…` depending on your setup), the same value as `users.identifier`, which is what the leaderboard join relies on.
- `delivery_history.route_id` receives `ruta.label` (e.g. `"Delivery on Sandy"`), **not** the `Config.Rutas` key. Renaming a route's label splits its history.
- `delivery_employees.employee_name` receives the candidate **id** (`cand01`), and `level` is copied from the candidate at hire time, so changing a candidate's `level` in config does not retroactively change existing hires.
- `delivery_owned_vehicles` has no unique key on `(player_identifier, vehicle_id)`; duplicate ownership is only prevented in application logic.
- `delivery_player_skills.skill_level` is incremented with `ON DUPLICATE KEY UPDATE skill_level = skill_level + 1` and is **not** capped at `maxLevel` server-side (the NUI disables the button at max, which is the only guard).

Useful queries:

```sql
-- Top earners
SELECT u.firstname, u.lastname, SUM(d.payment) AS earned, COUNT(*) AS trips
FROM delivery_history d JOIN users u ON u.identifier = d.player_identifier
GROUP BY d.player_identifier ORDER BY earned DESC LIMIT 10;

-- Everything one player owns
SELECT p.xp, p.skill_points, p.company_balance,
       (SELECT GROUP_CONCAT(vehicle_id) FROM delivery_owned_vehicles WHERE player_identifier = p.identifier) AS vehicles,
       (SELECT COUNT(*)                 FROM delivery_employees      WHERE owner_identifier  = p.identifier) AS staff
FROM delivery_players p WHERE p.identifier = 'license:xxxx';
```

### Integration Example

A third-party resource that gives a Discord-linked bonus on every completed route, keeps its own analytics, and opens the delivery menu from a phone app instead of the depot NPC. Nothing escrowed is touched.

```lua
-- ────────────────────────────────────────────────────────────
-- my_delivery_addon/server.lua
-- ────────────────────────────────────────────────────────────

-- The delivery job has no server-side "route completed" broadcast of its own,
-- so hook the same event it listens to. Our handler runs alongside theirs.
RegisterNetEvent('delivery:finishJob', function(ruta)
    local src     = source
    local xPlayer = ESX.GetPlayerFromId(src)
    if not xPlayer or not ruta then return end

    -- Loyalty bonus: +10% on every route, paid separately
    local base  = math.floor(((ruta.recompensa and ruta.recompensa.max) or 0))
    local bonus = math.floor(base * 0.10)
    if bonus > 0 then
        xPlayer.addAccountMoney('bank', bonus)
        TriggerClientEvent('esx:showNotification', src,
            ('Loyalty bonus: $%s'):format(bonus))
    end

    -- Our own analytics table, keyed on the route KEY rather than the label,
    -- so renaming a route later does not split the data.
    local routeKey = 'unknown'
    for id, cfg in pairs(Config.Rutas) do
        if cfg.label == ruta.label then routeKey = id break end
    end

    MySQL.Async.execute([[
        INSERT INTO my_delivery_stats (identifier, route_key, route_type, bonus, at)
        VALUES (@id, @key, @type, @bonus, @at)
    ]], {
        ['@id']    = xPlayer.identifier,
        ['@key']   = routeKey,
        ['@type']  = ruta.type or 'normal',
        ['@bonus'] = bonus,
        ['@at']    = os.time(),
    })
end)

-- Open the delivery dashboard from a phone app / another menu
RegisterNetEvent('my_delivery_addon:openRemotely', function()
    TriggerClientEvent('delivery:client:openMenu', source)
end)

-- Read a player's delivery standing for an external leaderboard or a bank app
exports('getDeliveryStanding', function(identifier)
    local p = MySQL.Sync.fetchAll(
        'SELECT xp, skill_points, company_balance FROM delivery_players WHERE identifier = @id',
        { ['@id'] = identifier })[1] or { xp = 0, skill_points = 0, company_balance = 0 }

    local h = MySQL.Sync.fetchAll([[
        SELECT COUNT(*) AS trips, IFNULL(SUM(payment),0) AS earned, IFNULL(SUM(distance),0) AS km
        FROM delivery_history WHERE player_identifier = @id
    ]], { ['@id'] = identifier })[1]

    local staff = MySQL.Sync.fetchAll(
        'SELECT COUNT(*) AS n FROM delivery_employees WHERE owner_identifier = @id',
        { ['@id'] = identifier })[1]

    return {
        xp        = p.xp,
        points    = p.skill_points,
        balance   = p.company_balance,
        trips     = h.trips,
        earned    = h.earned,
        km        = h.km,
        employees = staff.n,
    }
end)


-- ────────────────────────────────────────────────────────────
-- my_delivery_addon/client.lua  —  a radio call on hazardous routes
-- ────────────────────────────────────────────────────────────

-- Nexus fires this locally each time a package is delivered.
AddEventHandler('delivery:client:deliverPackage', function()
    TriggerEvent('my_hud:client:toast', 'Package delivered')
end)
```

To add an entirely new skill end-to-end:

```lua
-- 1) shared/config.lua — define the skill
Config.Skills.night = {
    name        = "Night Shifts",
    description = "Certifies you for after-hours runs.",
    maxLevel    = 2,
}

-- 2) shared/config.lua — add a route that requires it at max level
Config.Rutas.docks_night = {
    label = "Night run — Terminal",
    description = "After-hours delivery at the docks.",
    type = "expensive",
    requiredSkill = "night",
    deliveryPoints = { vector3(1206.0, -3113.0, 5.5) },
    recompensa = { min = 9000, max = 11000 },
    xp = 300,
    distance = 10.1,
}

-- 3) html/script.js — give it an icon in BOTH skillIcons maps,
--    otherwise it renders with a question mark:
--    const skillIcons = { distance: '…', expensive: '…', dangerous: '…',
--                         night: 'fa-solid fa-moon' };
```

No database change is needed — `delivery_player_skills` stores the skill id as a string.

## ❓ FAQ

**Nothing happens at the depot — no NPC, no marker, no target option.**
Check `Config.InteractionMethod` first. In `target` mode the dispatcher is created by `client/target.lua`, which requires ox_target, qb-target or qtarget to be present; with none of them the console prints `[DeliveryJob] ADVERTENCIA: No se encontró un sistema de target compatible.` and nothing is created. In `textui` mode the NPC only spawns if `Config.UseNPC = true`, otherwise you get a marker. Also confirm `Config.NPC.coords` is where you are standing.

**I set `Config.UseNPC = false` but an NPC still appears.**
`Config.UseNPC` is only honoured in `textui` mode. In `target` mode the ped is always created, because a target zone needs an entity to attach to. If you want a marker instead, switch to `textui`.

**The target option is in Spanish ("Abrir Menú de Repartos") even with `Config.Locale = 'en'`.**
Those labels are hardcoded Spanish in `client/target.lua`. That file is **not** escrowed, so edit it directly — ideally replacing the string with `Functions.Lang('open_delivery_menu')`.

**Delivering a package shows "Entregar Paquete" in Spanish, but only on some servers.**
The ox_target branch uses the locale key; the qb-target and qtarget branches use a hardcoded Spanish literal in `client/client.lua`, which **is** inside the escrow and cannot be edited. Switching to ox_target is the only workaround.

**Half the NUI is English and half is Spanish.**
That is how it ships, and `Config.Locale` does not affect the NUI at all — the config comment says as much. `html/index.html` is English and the dynamic strings in `html/script.js` are Spanish. Both files are unencrypted, so translate them directly.

**"You must select one of your vehicles" even though I own one.**
Owning a vehicle is not the same as selecting it. Go to **My Vehicles**, click the vehicle in the list, then press **Seleccionar para Trabajo** in the detail panel — the button should read **Seleccionado**. Only then will an Owned Vehicle route start.

**A route is greyed out / "You need the '…' skill at maximum level".**
A route with `requiredSkill` needs that skill at its **full `maxLevel`**, not level 1. With the default `maxLevel = 3` that is three skill points, which is 3000 XP at the default conversion rate.

**I have XP but no skill points.**
Points are awarded only when you cross a `XP_PER_SKILL_POINT` boundary on completing a route. At 999 XP you still have zero points. The award is computed from the tier difference, so a single route that takes you from 900 to 2100 XP correctly grants two points at once.

**My employees are not paying anything.**
Three conditions. The ticker runs every `Config.EmployeePayoutInterval` minutes (30 by default) — so nothing happens for the first half hour. It only pays players who are **online** when it fires. And the money goes into `company_balance`, not into your pocket: go to the **Bank** page and press **Withdraw Earnings** (it pays out as cash, not into your bank account).

**A hired employee shows "Salario: $500" but nothing is ever deducted.**
There is no salary mechanic. No candidate in `Config.AvailableEmployees` defines a `salary` field, so the server substitutes a flat `500` purely for display. Hiring is a one-time cost (`cost`) and employees only ever generate income.

**I fired an employee and lost the hiring fee.**
By design — `cost` is a one-time payment and firing refunds nothing. Re-hiring the same candidate costs the full amount again.

**The vehicle never spawns and I get "All spawn points are occupied".**
All ten `Config.VehicleSpawns` positions had a vehicle within 3 m. Either clear the alley or add more positions. Keeping `Config.ClearZoneEnabled = true` with a sensible radius is what normally prevents this, since it suppresses ambient traffic around the depot.

**"The delivery vehicle is too far away" even though I am standing on the marker.**
`Config.MaxDistanceToDeliver` (25 m by default) measures the distance from the **vehicle** to the delivery point, not from you. Park closer, or raise the value. This check is what stops players from doing the route in a faster private car.

**The cargo keeps exploding.**
Only `type = "dangerous"` routes arm the watchdog, and only when `Config.DangerousGoodsExplosion = true`. Raise `Config.ExplosionCrashForce` to tolerate harder impacts and lower `Config.DangerousGoodsHealthPercent` (e.g. to `0.5`) to tolerate more total damage. Note that **no shipped route is `dangerous`**, so if cargo is exploding, a route you added is the cause.

**"You must be in the driver's seat to finish the route."**
Exactly that: the completion check requires `GetPedInVehicleSeat(vehicle, -1) == PlayerPedId()`. Finishing from the passenger seat, or on foot next to the van, is not allowed.

**I cannot start a route — it says I need the uniform, but I am wearing it.**
The check compares every component and prop in `Config.WorkOutfit` against your ped, drawable *and* texture. If any single one differs the check fails. The usual cause is `Config.WorkOutfit` holding drawable ids from a different clothing pack than the one you actually run. Press **Put On** in the Cloakroom first; if that still does not satisfy it, your config values do not match your clothing pack. Note also that `torso` and `vest` both map to component 11, so defining both is self-contradictory.

**My civilian clothes were not restored ("Your saved civilian clothes were not found").**
The outfit is captured with `esx_skin:getPlayerSkin` the first time you open the menu in a session. If you put the uniform on through some other route, or `esx_skin` / `skinchanger` is not running, there is nothing cached to restore.

**Can I run this on QBCore?**
No. The resource is ESX-only: `getSharedObject()` is called at file scope in three files, `@es_extended/imports.lua` is a shared script, and the server uses ESX accounts, ESX server callbacks, `ESX.Game.SpawnVehicle` and the ESX `users` table for the leaderboard. There is no QBCore branch anywhere.

**Importing `tables.sql` fails with "table already exists".**
The statements use plain `CREATE TABLE`, not `CREATE TABLE IF NOT EXISTS`. Import it once on a fresh database, or add `IF NOT EXISTS` to each statement before re-running it.

**My shop vehicle shows a broken image.**
The image path is `html/img/<key>.png`, where `<key>` is the **key** in `Config.ShopVehicles`, not the `model`. Add a PNG named after the key to `html/img/` and add it to the `files` block in `fxmanifest.lua` (the shipped glob `html/img/*.png` already covers it).

**The leaderboard is empty.**
It joins `delivery_history` against the ESX `users` table on `identifier`. If your framework stores identifiers differently in the two tables — which happens after a multicharacter migration — the join returns nothing. Verify with the "Top earners" query in the Database Schema section.

**Icons are missing / the font looks wrong.**
`html/index.html` loads Font Awesome 6.2.0 from cdnjs and Poppins from Google Fonts. Without outbound internet from the game client both fail and the UI degrades to boxes and a generic sans-serif. Self-host both and change the two `<link>` tags.

**There is a tutorial overlay in the HTML but I never see it.**
The tutorial markup exists in `html/index.html`, including Okay and "Don't show again" buttons, but `html/script.js` never shows it and never wires those buttons up. It is unfinished UI in this version.

### Before opening a ticket

- Make sure the resource folder is named exactly **`nexus_deliveryjob`**. Any other name aborts the resource.
- Make sure you are on the latest version of the resource (**v1.4**, per `fxmanifest.lua`).
- Confirm you imported `tables.sql` — the tables are **not** created automatically, and almost every "nothing saves" report traces back to this.
- Confirm `ensure oxmysql` and `ensure es_extended` come **before** `ensure nexus_deliveryjob`, along with your target, notify and clothing resources.
- Set `Config.DebugPrints = true` and check the server console for the route and employee refresh lines, then set it back to `false`.
- Re-read this FAQ page.

## 📋 Changelog

**v1.4 — current release** (per `fxmanifest.lua`: `version "1.4"`, `description 'The most advanced and unique delivery job script'`)

No version history file ships with the resource. What the code itself documents about the current state:

- All seven target languages (`es`, `en`, `fr`, `pt`, `it`, `de`, `zh-CN`) are present with matching 42-key sets, so the Lua-side localisation set is complete as of this release. The NUI is not covered by it, which the config comment acknowledges explicitly.
- `shared/_resource.lua` carries the standard Nexus resource-name guard, which duplicates the older inline checks still present at the top of `client/client.lua` and `server/server.lua`.
- `Functions.SetFuel` still contains a `trk_fueling` branch marked `--DONT USE THIS`, a leftover from development that should not be selected.
- The explosion system, the `expensive` skill and the `dangerous` skill have no shipped route that exercises them, so those features are present but unused in the default configuration.
