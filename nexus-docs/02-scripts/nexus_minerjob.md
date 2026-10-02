# Nexus Miner Job

A complete mining career for ESX: persistent XP levels, upgradeable pickaxes and furnaces, gated quarry zones, a smelting recipe tree, a sell market, a server-wide leaderboard and three automatic random events — all driven from one custom NUI panel.

## 📝 Description

Nexus Miner Job turns mining into a progression loop instead of a single repeatable action. A player talks to a supervisor NPC, puts on the work uniform from the in-panel wardrobe, buys a pickaxe, and starts a shift. While on shift, blips and cylindrical ground markers appear over every configured quarry. Standing inside one and pressing the interaction key plays the pickaxe animation, runs a progress bar, and asks the server for a reward roll. Everything that matters — level, XP, rocks mined, money earned, the highest pickaxe owned and the highest furnace owned — lives in a single MySQL row keyed by the player's ESX identifier, so progress survives restarts and reconnects.

The progression has two independent gates. **Pickaxe level** gates which quarries you can mine: each zone declares a `requiredPick`, and the server-stored `p_highest` must meet or exceed it. **Miner level** gates which pickaxes you can buy: each pickaxe declares a `requiredLevel`, checked against the player's XP level when `Config.Leveling.RequireLevelForPicks` is on. The result is that you cannot skip ahead by buying your way in — you have to actually mine to unlock the ability to buy the better tool, which then unlocks the better quarry. **Furnace level** is a third, parallel track that gates the smelting recipe tier, and it is bought with money alone.

Raw ore is near-worthless on its own; the money is in refining. Every furnace tier exposes a recipe list (`Config.Recipes[tier]`) mapping a raw item to a refined item at a configurable consumption rate. Pressing "Fundir Minerales" runs **every** recipe the player's furnace tier knows against their whole inventory in one pass, consuming raw stock and granting refined stock. The Mercado tab then sells everything — raw and refined — in a single "Vender Todo" action at `Config.SellablePrices`, adding the total to the player's money and to their lifetime `money_earned` stat.

Running on top of that loop is an automatic event system. Each event type gets its own server-side thread that sleeps a random interval between `timer.min` and `timer.max`, then fires if no other event is active, runs for `duration` seconds, and stops. Three event kinds are implemented and each has distinct mechanics: **`ZONE`** spawns a temporary high-value quarry with its own blip and route, **`MULTIPLIER`** doubles (or whatever you set) the ore yield of every normal quarry server-wide, and **`SPEED`** divides the mining progress-bar time, making every swing faster. Events announce themselves either to the whole server or only to players within a radius, configurable per deployment.

## ✨ Features

- **Supervisor NPC with blip** — a frozen, invincible, event-blocking ped spawned at `Config.NPC.coords` with a configurable model and map blip. Walking within 2.0 units shows a TextUI prompt; the interaction key opens the panel.
- **Optional job requirement** — with `Config.JobRequired = true` the panel only opens for players whose ESX job matches `Config.JobName`; everyone else gets a "wrong job" notification. With it off, the NPC serves anybody.
- **Six-page custom NUI** — a 1300×800 glassmorphic panel with a sidebar: **Inicio** (stats + leaderboard), **Tienda** (pickaxes/furnaces tabs), **Horno** (smelting), **Mercado** (sell, raw/refined tabs), **Ropero** (wardrobe) and **Información** (a four-tab in-game guide). Start/Finish shift buttons live permanently in the sidebar footer.
- **Home dashboard** — five stat cards (Miner Level, Rocks Mined, lifetime Earnings, Best Pickaxe by name, Best Furnace by name) resolved from config so they read `"Pico de Acero"` rather than `3`.
- **Server-wide leaderboard** — a top-10 table joined against the ESX `users` table showing first name, last name, rocks mined and money earned, ordered by `money_earned + rocks_mined` descending.
- **Upgrade shop with dual gating** — pickaxe cards show price, description and image, and go to a disabled "Adquirido" state once owned. The server independently re-checks money, ownership and (for pickaxes) miner level before charging, so a tampered NUI cannot grant an upgrade.
- **Furnace recipe preview** — a "Minerales" button on every furnace card opens a modal listing that tier's full recipe tree (`rate`× raw → 1× refined) *before* you spend the money, so players can see what a 75 000 furnace actually unlocks.
- **Smelting page with live feasibility** — reads your actual inventory, lists every raw ore you hold, marks each as smeltable or not for your current furnace tier, shows the required rate, and keeps the "Fundir Minerales" button disabled until at least one recipe is satisfiable.
- **Market with live valuation** — raw and refined tabs, each card showing icon, label, quantity held, unit price and total value; a footer running total across both tabs; "Vender Todo" disabled at zero value.
- **In-panel wardrobe** — "Poner Uniforme" applies `Config.WorkOutfit` per gender (components and props, with `drawable = -1` supported to *remove* a prop); "Poner Ropa de Civil" restores the outfit captured through `esx_skin` the first time the player interacted with the NPC.
- **Uniform enforcement** — when `Config.WorkOutfitRequired` is on, the client verifies every configured component drawable *and* texture on the ped before letting the shift start, and refuses otherwise.
- **XP levelling with a geometric curve** — `XPPerRock` per swing; level-up at `floor((level ^ Multiplier) * BaseXP)` XP, with surplus XP carried into the next level rather than discarded.
- **Configurable failure chance** — `Config.Leveling.ChanceToFail` lets a swing yield no ore while still granting full XP, so grinding still progresses even on a bad roll.
- **Weighted reward tables** — each zone's `rewards` use cumulative `chance` weights; a single roll of 1–100 picks at most one entry, and the amount is a random integer between `min` and `max`.
- **Three random event types** — `ZONE` (temporary quarry with blip + GPS route and its own reward table), `MULTIPLIER` (global yield multiplier), `SPEED` (global mining-speed multiplier). Each has its own independent timer, duration and enable flag.
- **Event announcement modes** — `Notification.Type = 'server'` broadcasts to everyone, `'radius'` only notifies players within `Notification.Radius` metres of the event coordinates.
- **Attached pickaxe prop and animation** — `prop_tool_pickaxe` is created and bone-attached to the right hand for the duration of the shift, with the `melee@large_wpn@streamed_core` / `ground_attack_on_spot` animation played per swing.
- **Movement lock while mining** — forward/back/strafe and attack controls are disabled for the duration of a swing so the animation cannot be cancelled.
- **NUI progress bar** — a bottom-of-screen labelled bar animated with `requestAnimationFrame`, duration-matched to the actual mining time including any speed bonus.
- **Seven bundled languages** — `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`, selected with `Config.Locale`, all with identical key coverage.
- **Pluggable notifications and TextUI** — `shared/functions.lua` ships outside the escrow with branches for `nexus_notify`, `okokNotify`, ESX natives, `mythic_notify`, `okokTextUI` and a `custom` slot.
- **Resource name validation** — `shared/_resource.lua` hard-errors at startup if the folder is not named exactly `nexus_minerjob`.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **ESX** | **Required** | Hard requirement. The script calls `esx:getSharedObject`, `ESX.RegisterServerCallback`, `ESX.TriggerServerCallback`, `ESX.GetPlayerFromId`, `ESX.GetPlayers`, `ESX.GetItemLabel`, `xPlayer.getMoney/removeMoney/addMoney`, `xPlayer.getInventoryItem/addInventoryItem/removeInventoryItem`, `xPlayer.identifier` and `xPlayer.getName()`. **There is no QBCore branch.** |
| **oxmysql** | **Required** | Declared in `dependencies {}`. The server script loads `@oxmysql/lib/MySQL.lua` and uses the `MySQL.Async.fetchAll` / `MySQL.Async.execute` compatibility API. `mysql-async` would also satisfy the API but the manifest names `oxmysql`. |
| **A MySQL database** | **Required** | One table, `nexus_miner_data`, created by the bundled `nexus_minerjob.sql`. The leaderboard also reads the ESX `users` table. |
| **`esx_skin` + `skinchanger`** | **Required if** `Config.WorkOutfitRequired = true` *or* players will use the wardrobe | The civilian-outfit save/restore uses the `esx_skin:getPlayerSkin` server callback and the `skinchanger:loadSkin` client event. `illenium-appearance` provides both and is the tested setup. Without them the uniform can still be applied, but "Poner Ropa de Civil" will fail with `civilian_outfit_not_found`. |
| **The items in your config must exist in your inventory system** | **Required** | Every `item =` in `Config.Zones[].rewards`, every `raw`/`refined` in `Config.Recipes`, and every `name` in `Config.Market` must be a real registered item. See the FAQ — the shipped config is demo data using `bread`, `clothe`, `fish`, `phone`, `wool`. |
| **A notification resource** | Recommended | `nexus_notify` (default), `okokNotify`, `mythic_notify`, or ESX natives. Set `Config.NotifySystem`. With `custom`/unknown the script falls back to `print()` to console and the player sees nothing. |
| **A TextUI resource** | Recommended | `okokTextUI` (default) or ESX `ShowHelpNotification`. Set `Config.TextUISystem`. Without one, the "[E] Talk to the supervisor" prompts go to console only. |
| **`lua54`** | **Required** | The manifest declares `lua54 'yes'`. |
| **Internet access from the client** | Optional | The NUI loads Font Awesome 6.2.0 from cdnjs and Poppins from Google Fonts. Without it, icons and the font fall back; layout still works. |

## ⚙️ Installation

1. **Unzip the resource** into your `resources` folder. The folder **must** be named exactly `nexus_minerjob` — `shared/_resource.lua` throws a startup error otherwise.

2. **Import the SQL.** Run `nexus_minerjob.sql` against your database. It creates one table:
   ```sql
   CREATE TABLE IF NOT EXISTS `nexus_miner_data` ( ... );
   ```
   It is `CREATE TABLE IF NOT EXISTS`, so re-running it on an existing install is safe.

3. **Check load order in `server.cfg`.** `oxmysql` and ESX must both be started before this resource:
   ```cfg
   ensure oxmysql
   ensure es_extended
   ensure esx_skin          # or illenium-appearance
   ensure skinchanger       # if your appearance resource needs it
   ensure nexus_notify      # or your chosen notify resource
   ensure okokTextUI        # or your chosen TextUI resource

   # THEN:
   ensure nexus_minerjob
   ```

4. **Register your items.** This is the step people skip. The shipped config uses placeholder item names (`bread`, `clothe`, `diamond`, `fish`, `gold`, `phone`, `washed_stone`, `wool`, `cannabis`, `copper`, `fabric`, `iron`, `marijuana`, `stone`, `wood`, `bandage`). Replace every one of them with real items from **your** `items` table / `ox_inventory` item list, in all four places: `Config.Zones[].rewards`, `Config.Recipes`, `Config.Market.RawItems`, `Config.Market.RefinedItems` and `Config.SellablePrices`.

5. **Add the missing images.** Only `html/img/horno_forja.png` ships. You must supply the other nine PNGs referenced by the config into `html/img/`: `pico_piedra.png`, `pico_hierro.png`, `pico_acero.png`, `pico_diamante.png`, `horno_primitivo.png`, `horno_alto.png`, `zone_quarry.png`, `zone_mine.png`, `zone_cave.png` — or change the `image` fields in `Config.Picks`, `Config.Furnaces` and `Config.Zones` to filenames you do have. Item icons live in `html/items_images/raw/` and `html/items_images/refined/`; the folder an icon is loaded from is decided by which market list the item is in, so a raw item must have its icon in `raw/`.

6. **Set your coordinates.** Move `Config.NPC.coords` (a `vector4`, the `w` component is the heading) and each `Config.Zones[].coords` (`vector3`) to locations on your map. The NPC is spawned at `z - 1.0` and zone markers are drawn at `z - 0.95`, so take your coordinates standing on the ground.

7. **Pick your integrations.** Set `Config.Locale`, `Config.NotifySystem` and `Config.TextUISystem`. If you use something not on the list, set it to `'custom'` and fill the marked block in `shared/functions.lua` — it is outside the escrow.

8. **Restart.** A full server restart is recommended on first install.

9. **Start using it.** Walk to the supervisor NPC, press **E** (`Config.Keys.interaction = 38`), buy a pickaxe in **Tienda**, put the uniform on in **Ropero**, press **Empezar Turno**, then walk to a quarry blip and press **E** on the marker.

## 🔧 Configuration

All of this lives in `shared/config.lua`, which is in `escrow_ignore`.

### General

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Locale` | `string` | `'en'` | Which `Locales[...]` table to use. Must match a file in `locales/`. Bundled: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`. If a key is missing in the chosen language, `Functions.Lang` falls back to `Locales['es']`, then to the raw key name. |
| `Config.NotifySystem` | `string` | `'nexus'` | Notification backend. Recognised: `'nexus'`, `'okok'`, `'esx'`, `'mythic'`, `'custom'`. Anything else (including a typo) silently falls through to a `print()` and the player sees nothing. |
| `Config.TextUISystem` | `string` | `'okok'` | TextUI backend for the "[E] …" prompts. Recognised: `'okok'`, `'esx'`, `'custom'`, and `'nexus'` (see the warning below). Anything else prints to console. |
| `Config.Keys.interaction` | `number` | `38` (E) | GTA control index checked with `IsControlJustReleased(0, …)`. Used for both the NPC and the mining prompt. |
| `Config.JobRequired` | `boolean` | `false` | `true` restricts the panel to players whose ESX job name equals `Config.JobName`. |
| `Config.JobName` | `string` | `'miner'` | The ESX job name checked when `JobRequired` is on. Must match the `name` column in your `jobs` table. |
| `Config.ProgressBarTime` | `number` (ms) | `5000` | Base duration of one mining swing. Divided by `speedMultiplier` while a `SPEED` event is active. |

> ⚠️ **`Config.TextUISystem = 'nexus'` does not call nexus_TextUI.** In the shipped `shared/functions.lua`, the `'nexus'` branch of `Functions.ShowText` calls `exports['okokTextUI']:Open(...)`, and `Functions.HideText` has no `'nexus'` branch at all — so with `'nexus'` selected, prompts route to okokTextUI and are never closed. Until this is corrected in `functions.lua`, use `'okok'` or `'custom'`. This is logged in the incident report.

### NPC

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.NPC.coords` | `vector4` | `vector4(2946.52, 2747.07, 43.40, 283.46)` | Where the supervisor stands. `x, y, z` position (the ped is spawned at `z - 1.0`) and `w` as heading in degrees. |
| `Config.NPC.model` | `string` | `"cs_floyd"` | Ped model name. Must be a valid, streamed model; the script `RequestModel`s it and waits. |
| `Config.NPC.blip.enabled` | `boolean` | `true` | Whether to draw a map blip at the NPC. |
| `Config.NPC.blip.sprite` | `number` | `47` | Blip sprite ID. |
| `Config.NPC.blip.display` | `number` | `4` | Blip display mode (`4` = visible on both main map and minimap). |
| `Config.NPC.blip.scale` | `number` | `0.8` | Blip size. |
| `Config.NPC.blip.color` | `number` | `2` | Blip colour ID (`2` = green). |
| `Config.NPC.blip.text` | `string` | `'Miner Job'` | Blip label on the map. **Not** routed through the locale system — it is a raw config string, so translate it by hand. |

### Uniform

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.WorkOutfitRequired` | `boolean` | `true` | When `true`, `IsWearingWorkOutfit()` verifies every configured component's drawable **and** texture before a shift may start. When `false`, the check returns `true` unconditionally and the wardrobe becomes purely cosmetic. |
| `Config.WorkOutfit` | `table` | see below | Per-gender outfit. Gender is detected from the ped model (`mp_f_freemode_01` → `female`, anything else → `male`). |

```lua
Config.WorkOutfit = {
    male = {
        components = {
            -- key names are mapped internally:
            -- face=0, mask=1, hair=2, arms=3, pants=4, bags=5,
            -- shoes=6, neck=7, tshirt=8, bproof=9, decals=10, torso=11
            ['mask']   = { drawable = 0,   texture = 0 },
            ['arms']   = { drawable = 0,   texture = 0 },
            ['pants']  = { drawable = 10,  texture = 0 },
            ['bags']   = { drawable = 0,   texture = 0 },
            ['shoes']  = { drawable = 36,  texture = 0 },
            ['tshirt'] = { drawable = 15,  texture = 0 },
            ['vest']   = { drawable = 250, texture = 0 },  -- see warning
            ['decals'] = { drawable = 0,   texture = 0 },
            ['bproof'] = { drawable = 0,   texture = 0 },
        },
        props = {
            -- hats=0, glasses=1, ears=2, watches=6, bracelets=7
            -- drawable = -1 REMOVES the prop instead of setting it
            -- ['hats'] = { drawable = -1, texture = 0 },
        },
    },
    female = { components = { … }, props = {} },
}
```

> ⚠️ The shipped config uses the component key **`vest`**, which is **not** in the internal component map. Unmapped keys are silently skipped — both when applying the uniform and when verifying it — so the `vest` entry currently does nothing. The torso slot is called **`torso`** (component 11) in this script, and the slot usually labelled "vest"/"body armour" in appearance menus is **`bproof`** (component 9). Rename `vest` → `torso` (or `bproof`) to make it take effect.

### Levelling

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Leveling.Enabled` | `boolean` | `true` | Master switch. When `false`, XP and levels are not touched; only `rocks_mined` is incremented. **Do not ship with this off** — see the incident report, the disabled path emits untranslated locale keys as notification text. |
| `Config.Leveling.RequireLevelForPicks` | `boolean` | `true` | When `true`, buying a pickaxe also requires `minerData.level >= pick.requiredLevel`. When `false`, only money and ownership are checked. |
| `Config.Leveling.XPPerRock` | `number` | `15` | XP granted per swing — granted **whether or not** ore was found. |
| `Config.Leveling.BaseXP` | `number` | `100` | Base of the level curve. |
| `Config.Leveling.Multiplier` | `number` | `1.1` | **Exponent**, not a factor. The XP needed to leave level `L` is `floor((L ^ Multiplier) * BaseXP)`. |
| `Config.Leveling.ChanceToFail.Enabled` | `boolean` | `true` | Whether a swing can come up empty. |
| `Config.Leveling.ChanceToFail.Probability` | `number` (%) | `20` | Percentage chance of finding nothing. Rolled **before** the reward table, so it is a flat 20% whiff rate independent of zone weights. |

With the defaults, levelling up costs: L1→2 = 100 XP (7 rocks), L2→3 = 214 (15), L3→4 = 334 (23), L5→6 = 585 (39), L10→11 = 1258 (84), L20→21 = 2702 (181). Note `BaseXP` scales the whole curve linearly while `Multiplier` changes its steepness — raise `Multiplier` to 1.5 and L20 costs 8944 XP instead of 2702.

### `Config.Picks`

A **positional array** — `Config.Picks[n]` is the pickaxe bought when the NUI sends `level = n`. Keep `level` equal to the array index.

```lua
Config.Picks = {
    {
        name          = 'Pico de Piedra',
        description   = 'El pico básico para empezar...',
        price         = 1500,
        level         = 1,   -- must equal the array index
        image         = 'pico_piedra.png',  -- resolved as html/img/<image>
        requiredLevel = 1,   -- miner level needed to buy it
    },
}
```

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Shown on the shop card, in the "Mejor Pico" stat, and in the purchase notification. |
| `description` | `string` | Shop card body text. |
| `price` | `number` | Cost in cash. Checked against `xPlayer.getMoney()` — **cash only**, bank is not consulted. |
| `level` | `number` | The pickaxe tier. Stored in `p_highest`. Compared against each zone's `requiredPick`. **Must be unique and sequential starting at 1.** |
| `image` | `string` | Filename inside `html/img/`. |
| `requiredLevel` | `number` | Miner level gate, enforced only when `Config.Leveling.RequireLevelForPicks` is `true`. |

Defaults: Stone (1 500 / tier 1 / lvl 1), Iron (5 000 / tier 2 / lvl 5), Steel (12 000 / tier 3 / lvl 10), Diamond (25 000 / tier 4 / lvl 20).

### `Config.Furnaces`

Same positional-array rule. There is **no** `requiredLevel` on furnaces — money alone.

```lua
Config.Furnaces = {
    {
        name        = 'Horno Primitivo',
        description = 'Construido con arcilla y piedra...',
        price       = 10000,
        level       = 1,   -- must equal the array index; keys Config.Recipes
        image       = 'horno_primitivo.png',
    },
}
```

The furnace `level` is what indexes `Config.Recipes`, so a tier-2 furnace uses `Config.Recipes[2]`.

Defaults: Primitive (10 000 / tier 1), Forge (30 000 / tier 2), Blast Furnace (75 000 / tier 3).

### `Config.Zones`

```lua
Config.Zones = {
    {
        name         = 'Cantera Principal',
        description  = 'La zona de inicio...',
        image        = 'zone_quarry.png',  -- html/img/<image>, used in the guide tab
        coords       = vector3(2941.29, 2797.93, 41.0),
        radius       = 2.0,
        requiredPick = 1,
        rewards = {
            { item = 'stone',   min = 3, max = 8, chance = 33 },
            { item = 'ironore', min = 1, max = 3, chance = 33 },
            { item = 'diamond', min = 1, max = 3, chance = 34 },
        },
    },
}
```

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Blip label (created while on shift) and guide-tab heading. |
| `description` | `string` | Guide-tab body text. |
| `image` | `string` | Filename in `html/img/`, used as the guide card's background image. |
| `coords` | `vector3` | Centre of the zone. The ground marker is drawn at `z - 0.95`. |
| `radius` | `number` | Interaction radius in units. The marker is drawn at `radius * 2.0` wide, so `2.0` gives a 4-unit-wide circle. |
| `requiredPick` | `number` | Minimum `p_highest` to mine here. Checked **client-side before** the animation and **again server-side** is *not* done — see the security note in the Developer API. |
| `rewards` | `table[]` | Weighted table, below. |

**How the reward roll works.** One `math.random(1, 100)` is drawn. The script walks `rewards` in order accumulating `chance`; the first entry whose cumulative total is `>= roll` wins, and the loop breaks. So:

- `chance` values are **weights out of 100**, not independent probabilities.
- Only **one** entry can win per swing.
- If your weights sum to **less than 100**, the remainder is a silent miss on top of `ChanceToFail`. Zone 3 in the shipped config sums to 100 (50+50); zones 1 and 2 also sum to 100.
- If they sum to **more than 100**, the later entries become unreachable.
- Order matters: put the rare, low-weight entries first if you want them reachable at all — with `{A, chance=90}, {B, chance=10}`, B only wins on rolls 91–100, which is correct; but with `{A, chance=100}, {B, chance=10}`, B can never win.
- `min`/`max` are inclusive bounds on the amount, then multiplied by any active `MULTIPLIER` event and floored.

Defaults: Main Quarry (pick 1), Abandoned Mine (pick 2), Crystal Cave (pick 4). **Note there is no zone requiring pick 3**, so the Steel pickaxe unlocks nothing new in the stock config — add a zone with `requiredPick = 3` or lower the Crystal Cave to `3`.

### `Config.Recipes`

Keyed by furnace tier. **A higher tier must repeat the lower tiers' recipes** — the lookup is `Config.Recipes[f_highest]` exactly, not a merge.

```lua
Config.Recipes = {
    [1] = {
        { raw = 'ironore', refined = 'iron_ingot', rate = 1 },
    },
    [2] = {
        { raw = 'ironore', refined = 'iron_ingot', rate = 1 },  -- repeated
        { raw = 'goldore', refined = 'gold_ingot', rate = 2 },  -- new
    },
}
```

| Field | Type | Description |
|---|---|---|
| `raw` | `string` | Item consumed. Must also appear in `Config.Market.RawItems` to show up on the smelting page. |
| `refined` | `string` | Item produced. |
| `rate` | `number` | How many `raw` are consumed per **1** `refined`. `rate = 3` means 3 ore → 1 ingot. Smelting computes `floor(count / rate)` ingots and consumes `ingots * rate` ore, leaving the remainder. |

> ⚠️ **`Config.Recipes` must be a contiguous array starting at `[1]`.** It is serialised to JSON and sent to the NUI, which reads it as a JavaScript array (`Recipes[tier - 1]`). A contiguous `{[1]=…, [2]=…, [3]=…}` serialises as a JSON array and works. A sparse table such as `{[2]=…, [3]=…}` serialises as a JSON **object** and the NUI's recipe list and preview modal will come up empty — even though the server-side smelt still works. Always define tier 1 first and leave no gaps.

### `Config.RandomEvents`

```lua
Config.RandomEvents = {
    Notification = {
        Type   = 'server',  -- 'server' | 'radius'
        Radius = 500.0,
    },
    EventTypes = { … },
}
```

| Config key | Type | Default | Description |
|---|---|---|---|
| `Notification.Type` | `string` | `'server'` | `'server'` sends the event to all players (`-1`). `'radius'` sends it only to players within `Radius` of the event coordinates. **`'server'` is also forced for any event with no `coords`** (i.e. `MULTIPLIER` and `SPEED`), since a radius is meaningless for them. |
| `Notification.Radius` | `number` | `500.0` | Metres, used only when `Type = 'radius'`. |

> ⚠️ With `Type = 'radius'`, the event payload itself is only delivered to nearby players. A player who was 600 m away when the event fired never receives `CurrentClientEvent`, so even if they walk into the temporary quarry they cannot mine it and get no speed/announcement for the whole duration. If you want the mechanics to apply server-wide, keep `Type = 'server'`.

Each entry in `EventTypes` shares these fields:

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Shown in the announcement notification and in the guide tab. Passed through `random_event_start` as `%s`. |
| `type` | `string` | **Do not change.** One of `'ZONE'`, `'MULTIPLIER'`, `'SPEED'`. Selects the mechanic. |
| `description` | `string` | Guide-tab text only. |
| `enabled` | `boolean` | When `false`, no thread is created for this event at all. |
| `duration` | `number` (seconds) | How long the event stays active. |
| `timer.min` / `timer.max` | `number` (seconds) | Random wait between this event's occurrences. Each event type has its own independent timer thread. |
| `blip` | `table` or `nil` | `{ sprite, color, scale, text }`. Only meaningful for `ZONE`; set `nil` for global events. A `ZONE` blip also gets `SetBlipRoute(true)`, i.e. a GPS line. |

`ZONE`-only fields:

| Field | Type | Description |
|---|---|---|
| `coords` | `vector3` | Centre of the temporary quarry. |
| `radius` | `number` | Interaction radius; the marker is gold and drawn 2.0 tall. |
| `anyPickaxe` | `boolean` | `true` lets any pickaxe tier mine it. **When `true` it overrides `requiredPick` entirely.** |
| `requiredPick` | `number` | Minimum `p_highest`, only consulted when `anyPickaxe` is falsy. |
| `rewards` | `table[]` | Same weighted format as `Config.Zones[].rewards`. |

`MULTIPLIER`-only: `rewardMultiplier` (`number`, `2.0` = double ore; applied as `floor(amount * multiplier)` server-side).
`SPEED`-only: `speedMultiplier` (`number`, `2.0` = half the progress-bar time; applied client-side as `ProgressBarTime / speedMultiplier`).

Shipped defaults: *Enriched vein* (`ZONE`, 600 s, every 30–60 min, `anyPickaxe = true`), *Gold Fever* (`MULTIPLIER` ×2, 300 s, every 30–60 min), *Speed Cola* (`SPEED` ×2, 480 s, every 30–60 min). All three threads start after a 15-second boot delay, and only one event can be active at a time — whichever timer fires first wins and the others skip their turn.

### `Config.Market` and `Config.SellablePrices`

```lua
Config.Market = {
    RawItems = {
        { name = 'ironore', image = 'ironore.png' },  -- html/items_images/raw/<image>
    },
    RefinedItems = {
        { name = 'iron_ingot', image = 'iron_ingot.png' },  -- html/items_images/refined/<image>
    },
}

Config.SellablePrices = {
    ['ironore']    = 15,
    ['iron_ingot'] = 120,
}
```

| Config key | Type | Description |
|---|---|---|
| `Config.Market.RawItems` | `{name, image}[]` | Items shown on the market's "Minerales sin Refinar" tab **and** on the smelting page. Membership in this list is also what makes an item count as "raw" for icon-path resolution — a raw item's icon is loaded from `html/items_images/raw/`. |
| `Config.Market.RefinedItems` | `{name, image}[]` | Items on the "Minerales Refinados" tab. Icons load from `html/items_images/refined/`. |
| `Config.SellablePrices` | `table<string, number>` | Price per unit. **An item missing from this table shows `$0/u` in the NUI and is skipped entirely by the sell handler** — so adding an item to `Market` without a price makes it unsellable. |

"Vender Todo" iterates `RawItems` then `RefinedItems`, sells the player's entire stock of every priced item at once, credits `xPlayer.addMoney(total)` and adds the total to lifetime `money_earned`. There is no partial-sell UI and no per-item sell button.

### `Config.Keys`

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Keys.interaction` | `number` | `38` | GTA control index. `38` is `E`. Others you might use: `47` = `G`, `74` = `H`, `246` = `Y`, `182` = `L`. This is a raw control check, **not** a `RegisterKeyMapping`, so players cannot rebind it. |

## 🌐 Locales & Editable Strings

**Seven languages ship, all complete.** `locales/en.lua`, `es.lua`, `de.lua`, `fr.lua`, `it.lua`, `pt.lua`, `zh.lua` — every one defines the same **34 keys**, with no gaps. `locales/_init.lua` is a one-liner (`Locales = Locales or {}`) loaded first so the per-language files can append.

Selected with `Config.Locale`. `Functions.Lang(key, ...)` resolves in this order: `Locales[Config.Locale][key]` → `Locales['es'][key]` → the literal key string. When extra arguments are passed they go through `string.format`, so the `%s` placeholders must be preserved in every translation.

### Locale key reference

| Key | Placeholders | Used by |
|---|---|---|
| `job_name` | — | The title of every notification in the script. |
| `start_work` / `stop_work` | — | Shift start/end notifications. |
| `uniform_on` / `uniform_off` | — | Wardrobe confirmations. |
| `civilian_outfit_not_found` | — | When no civilian skin was captured. |
| `uniform_required` | — | Shift refused because the uniform is not worn. |
| `job_required` | — | NPC refused because of `Config.JobRequired`. |
| `pickaxe_too_weak` | — | Zone `requiredPick` not met. |
| `event_pickaxe_too_weak` | — | Event-zone pick requirement not met. |
| `sync_error` | — | `getData` returned no `minerData` while starting a shift. |
| `data_load_error` | — | `getData` returned nothing while opening the menu. |
| `random_event_start` | `%s` = event name | Event announcement. |
| `random_event_stop` | — | Event end announcement. |
| `speed_bonus_active` | — | Speed event feedback. |
| `no_pickaxe_start` | — | Shift refused, `p_highest < 1`. |
| `received_items` | `%s` ×2 (amount, label) | Reward notification with levelling **disabled**. |
| `found_nothing` | — | Empty swing with levelling **disabled**. |
| `received_items_xp` | `%s` ×3 (amount, label, XP) | Reward notification with levelling enabled. |
| `found_nothing_xp` | `%s` (XP) | Empty swing with levelling enabled. |
| `level_up` | `%s` = new level | Level-up notification. |
| `level_too_low` | `%s` = required level | Pickaxe purchase blocked by miner level. |
| `already_owned` | — | Upgrade already owned at that tier or better. |
| `bought_item` | `%s` = item name | Purchase confirmation. |
| `not_enough_money` | — | Insufficient cash. |
| `materials_sold` | `%s` = total | Sell confirmation. |
| `no_materials_to_sell` | — | Nothing priced in inventory. |
| `need_furnace` | — | Smelt attempted with `f_highest < 1`. |
| `no_furnace_recipes` | — | No `Config.Recipes` entry for that tier. |
| `smelted_items` | `%s` = comma list | Smelt confirmation. |
| `not_enough_to_smelt` | — | No recipe satisfiable. |
| `interact_npc` | — | TextUI at the supervisor. |
| `mine_rock` | — | TextUI in a normal quarry. |
| `mine_event_rock` | — | TextUI in an event quarry. |

### Editable outside the escrow

`escrow_ignore` covers:

```lua
escrow_ignore {
    'shared/config.lua',
    'shared/functions.lua',
    'locales/*.lua',
    'nexus_minerjob.sql',
}
```

So you can freely edit: all config, all 34 locale strings in all 7 languages, the notify/TextUI integration branches, and the SQL.

### Strings hardcoded **inside** the escrow

These are **not** translatable and will stay Spanish regardless of `Config.Locale`:

- **The entire NUI.** `html/index.html` and `html/script.js` are in `files {}` and therefore protected. That means every sidebar label (`Minería`, `Inicio`, `Tienda`, `Horno`, `Mercado`, `Ropero`, `Información`), both shift buttons (`Empezar Turno`, `Finalizar Turno`), `Cerrar Menú`, every page heading and subtitle, all five stat card titles, the leaderboard headings (`#`, `Nombre`, `Rocas Picadas`, `Ganancias`), `Bienvenido, {name}`, `Ninguno`, `Comprar` / `Adquirido`, `Minerales`, `Valor total a vender: $N`, `Vender Todo`, `Fundir Minerales`, `No tienes minerales brutos en tu inventario.`, `(Necesitas N)`, `(No se puede fundir)`, `No tienes un horno`, `Ir a la Tienda`, `Recetas para …`, `No hay recetas disponibles para este horno.`, the four wardrobe strings, and **all of the Información guide copy** across its four tabs (General, Canteras, Equipamiento, Eventos) — several paragraphs of Spanish prose.
- **The two progress-bar labels** `'Minando...'` and `'Extrayendo veta...'`, sent from `client/client.lua` as literals, plus the NUI fallback `'Procesando...'`.
- **`Config.NPC.blip.text`** is in config and therefore editable, but note it is *not* wired to the locale system, so it must be translated separately from the rest.
- Developer diagnostics: `"Outfit not defined for this gender in config.lua"`, `[nexus_minerjob] Refrescando datos del menú...`, `[nexus_minerjob] ERROR: …`, `[nexus_minerjob] Evento aleatorio iniciado/finalizado: …`.

**Net effect: notifications and TextUI prompts are fully localised in 7 languages; the panel itself is Spanish-only.** This is the main localisation gap and is logged in the incident report.

### Theming

`html/style.css` exposes a `:root` block you can read but not edit (it is inside the escrow):

```css
:root {
    --bg-dark: rgba(24, 23, 28, 0.95);
    --bg-card: rgba(68, 71, 90, 0.5);
    --border-color: rgba(255, 255, 255, 0.1);
    --text-light: #f8f8f2;
    --text-medium: #c7cce1;
    --text-dark: #a0a8b4;
    --accent-red-gradient: linear-gradient(90deg, #ff5555, #e74c3c);
    --accent-red: #e74c3c;
    --accent-orange: #ffb86c;
    --accent-purple: #bd93f9;
    --glow-color: rgba(255, 255, 255, 0.5);
}
```

The panel is a fixed `1300 × 800` centred box, so it does not scale with resolution.

## 🔗 Compatibility

| System | How it is handled |
|---|---|
| **Framework** | **ESX only.** There is no `Config.Framework` key and no QBCore branch anywhere in the three Lua files. The leaderboard query additionally hardcodes `JOIN users u ON nmd.identifier = u.identifier` with `u.firstname` / `u.lastname`, which is the ESX schema. Running this on QBCore requires code changes, not config. |
| **Database** | `oxmysql` (declared dependency) via the `MySQL.Async` compatibility wrapper loaded from `@oxmysql/lib/MySQL.lua`. `mysql-async` exposes the same API and would also work. |
| **Inventory** | Uses the ESX `xPlayer` inventory API (`getInventoryItem`, `addInventoryItem`, `removeInventoryItem`) and reads `xPlayer.inventory` to build the market and smelting views. This works with the stock ESX inventory. If you run **`ox_inventory`**, `xPlayer.inventory` is not populated the same way, so the market and furnace pages may show every item at `0` even when the player is holding ore — verify on a test character before going live. |
| **Money** | Cash only: `xPlayer.getMoney()` / `removeMoney()` / `addMoney()`. Bank balance is never checked and no banking resource is integrated, so a player with 50 000 in the bank and 0 cash cannot buy a pickaxe. |
| **Item labels** | `ESX.GetItemLabel(item)` for the names shown in reward and smelt notifications. An unregistered item will produce `nil` here. |
| **Notifications** | Selected by `Config.NotifySystem` in `shared/functions.lua` (outside the escrow). Branches: `nexus` → `exports['nexus_notify']:Alert(title, msg, time, type, true)` client-side and `exports['nexus_notify']:Alert(player, title, msg, time, type, true)` server-side; `okok` → `exports['okokNotify']:Alert` / `okokNotify:Alert` event; `esx` → `ESX.ShowNotification` / `esx:showNotification`; `mythic` → `mythic_notify:DoHudText` / `mythic_notify:client:SendAlert`; `custom` → an empty block for you to fill. |
| **TextUI** | Selected by `Config.TextUISystem`. `okok` → `exports['okokTextUI']:Open(msg, 'darkblue', 'right', true)` / `:Close()`; `esx` → `ESX.ShowHelpNotification` (self-closing, so `HideText` is a no-op); `custom` → empty block. The `nexus` branch is miswired — see the warning in Configuration. |
| **Clothing / appearance** | `esx_skin` (`esx_skin:getPlayerSkin` server callback) and `skinchanger` (`skinchanger:loadSkin` client event). `illenium-appearance` provides both. The uniform itself is applied with raw natives (`SetPedComponentVariation`, `SetPedPropIndex`, `ClearPedProp`), so it does not depend on an appearance resource — only the *civilian restore* does. The config comment notes more clothing systems are planned. |
| **Target systems** | Not used. Interaction is proximity + raw key check, not `ox_target` / `qb-target`. |
| **Keys / fuel / vehicles** | Not used. No vehicle is spawned and no keys or fuel resource is touched. |
| **Progress bars** | Self-contained. The script draws its own NUI progress bar; it does not call `ox_lib`, `progressBars` or `mythic_progbar`. |
| **Other mining scripts** | No conflict beyond overlapping coordinates and item names. Change `Config.NPC.coords` and `Config.Zones[].coords` if you run another mining job nearby. |

## 💻 Developer API

Everything below was taken from `server/server.lua`, `client/client.lua` and `shared/functions.lua` as shipped.

### Client Exports

**None.** `client/client.lua` declares no `exports(...)` and no `exports['nexus_minerjob']` handlers.

### Server Exports

**None.** `server/server.lua` declares no exports either.

The public surface of this resource is: **one ESX server callback**, **six network events**, and **the `Functions` table** in `shared/functions.lua` (which, being a shared script outside the escrow, you can read and extend).

### ESX Server Callback

| Callback | Parameters | Returns | Description |
|---|---|---|---|
| `nexus_minerjob:getData` | *(none)* | `nil`, or a table — see below | The single read path for all player state. Called from the client when the menu opens and whenever a refresh is requested. Returns `nil` if `ESX.GetPlayerFromId(source)` fails. **Creates the player's DB row on demand** if it does not exist yet. |

```lua
-- From any client-side resource:
ESX.TriggerServerCallback('nexus_minerjob:getData', function(data)
    if not data then return end

    print(data.minerData.level)        -- miner level
    print(data.minerData.xp)           -- current XP toward the next level
    print(data.minerData.rocks_mined)  -- lifetime rocks
    print(data.minerData.money_earned) -- lifetime earnings
    print(data.minerData.p_highest)    -- highest pickaxe tier owned (0 = none)
    print(data.minerData.f_highest)    -- highest furnace tier owned (0 = none)
    print(data.minerData.firstname)    -- xPlayer.getName(), overwritten server-side

    for _, row in ipairs(data.leaderboards.rocks) do
        print(row.firstname, row.lastname, row.rocks_mined, row.money_earned)
    end

    -- data.inventory is xPlayer.inventory (or {} if absent)
end)
```

Returned shape:

```lua
{
    minerData = {
        identifier, level, rocks_mined, money_earned,
        p_owned, f_owned, p_highest, f_highest, xp,
        firstname,  -- added server-side from xPlayer.getName()
    },
    leaderboards = {
        rocks = { { firstname, lastname, rocks_mined, money_earned }, … }  -- top 10
    },
    inventory = { … },  -- xPlayer.inventory
}
```

> Note that on the **first** call for a brand-new player the function inserts the row and returns a *synthetic* table that omits `xp`, `p_owned` and `f_owned`. Guard with `or 0` if you read those fields.

### Events — Emitted

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_minerjob:server:startWork` | Client → Server | *(none)* | Player pressed "Empezar Turno" **and** passed the uniform check. |
| `nexus_minerjob:server:requestMineRewards` | Client → Server | `zoneId` *(number\|nil)*, `isEventZone` *(boolean)* | A mining swing finished. `zoneId` is the 1-based index into `Config.Zones`; for an event zone it is `nil` and `isEventZone` is `true`. |
| `nexus_minerjob:server:buyUpgrade` | Client → Server | `data` = `{ type = 'pick'\|'furnace', level = number }` | Player clicked a "Comprar" button. |
| `nexus_minerjob:server:sellMaterials` | Client → Server | *(none)* | Player clicked "Vender Todo". |
| `nexus_minerjob:server:smeltAll` | Client → Server | *(none)* | Player clicked "Fundir Minerales". |
| `nexus_minerjob:client:startWorkResponse` | Server → Client | `{ success = boolean, reason = string? }` | Answer to `startWork`. `success = false` carries a localised `reason` (currently always `no_pickaxe_start`). |
| `nexus_minerjob:client:refreshUI` | Server → Client | *(none)* | After a successful purchase, sale or smelt, to make the open panel re-fetch its data. |
| `nexus_minerjob:client:startRandomEvent` | Server → Client (`-1` or targeted) | `CurrentEvent` table — `{ active, type, name, rewardMultiplier, speedMultiplier, coords, radius, anyPickaxe, rewards, blip }` | An event started. Broadcast to `-1` when `Notification.Type = 'server'` or the event has no `coords`; otherwise sent only to players within `Notification.Radius`. |
| `nexus_minerjob:client:stopRandomEvent` | Server → Client (`-1`) | *(none)* | An event ended. Always broadcast to everyone, even if the start was radius-limited. |
| `esx:getSharedObject` | Both (`TriggerEvent`) | callback | Standard ESX bootstrap, polled on the client until it resolves. |

### Events — Listened

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `esx:playerLoaded` | Server (`AddEventHandler`) | `playerId`, `xPlayer` | Ensures a `nexus_miner_data` row exists for the identifier, inserting one if not. |
| `nexus_minerjob:server:startWork` | Server | — | Checks `p_highest >= 1` and replies with `startWorkResponse`. |
| `nexus_minerjob:server:requestMineRewards` | Server | `zoneId`, `isEventZone` | Rolls failure chance, rolls the weighted reward table, grants the item, applies the multiplier, grants XP, handles level-up, increments `rocks_mined`, notifies. |
| `nexus_minerjob:server:buyUpgrade` | Server | `{type, level}` | Validates miner level (pickaxes only), ownership and cash; charges and updates `p_highest`/`f_highest`. |
| `nexus_minerjob:server:sellMaterials` | Server | — | Sells all priced items from both market lists. |
| `nexus_minerjob:server:smeltAll` | Server | — | Runs every recipe of the player's furnace tier against their inventory. |
| `nexus_minerjob:client:startWorkResponse` | Client | `{success, reason}` | On success, re-fetches `getData` to prime `PlayerData.minerData`, then calls `StartWork()`. |
| `nexus_minerjob:client:refreshUI` | Client | — | Calls `UpdateMenuData()` if the panel is open. |
| `nexus_minerjob:client:startRandomEvent` | Client | event table | Stores `CurrentClientEvent`, notifies, creates the routed blip. |
| `nexus_minerjob:client:stopRandomEvent` | Client | — | Clears `CurrentClientEvent` and removes the blip. |
| `nexus_minerjob:client:tryMine` | Client | `{ isEvent = boolean, zoneId = number }` | **Registered but never triggered by this resource.** A complete alternative mining entry point: it runs the pick check, animation, progress bar and reward request. It exists so an external resource (a `ox_target` zone, a custom prop interaction) can drive a mining swing instead of the built-in proximity loop. See the integration example. |

### Security notes on these events

Worth knowing before you build on them:

- `nexus_minerjob:server:requestMineRewards` performs **no server-side validation** that the player is on shift, is near the zone, owns a sufficient pickaxe, or waited the progress-bar duration. The `requiredPick` and distance checks are client-side only. A player able to trigger this event in a loop can farm the reward table at will. The only limiting factor is that `zoneId` must index a real `Config.Zones` entry.
- `nexus_minerjob:server:startWork` does **not** re-check `Config.JobRequired` / `Config.JobName`; the job gate is client-side only.
- `nexus_minerjob:server:buyUpgrade` **is** properly validated server-side (level, ownership, funds), and `sellMaterials` / `smeltAll` read real inventory counts server-side, so those three cannot be abused through the NUI.
- The uniform check is client-side only.

If you need these hardened, the fix is in `server/server.lua`, which is escrowed — raise it with the author rather than patching around it.

### NUI Callbacks

Internal to the resource (NUI callbacks are scoped to the owning frame) but documented for completeness:

| Callback | Body | Response | Purpose |
|---|---|---|---|
| `startWork` | `{}` | `'ok'` | Runs `IsWearingWorkOutfit()`, then triggers `server:startWork` and closes the panel. |
| `stopWork` | `{}` | `'ok'` | Calls `StopWork()` locally (deletes the prop, removes zone blips) and closes the panel. **No server event is sent** — ending a shift is purely client-side state. |
| `buyUpgrade` | `{ type, level }` | `'ok'` | Forwards to `server:buyUpgrade`. |
| `sellMaterials` | `{}` | `'ok'` | Forwards to `server:sellMaterials`. |
| `smeltAll` | `{}` | `'ok'` | Forwards to `server:smeltAll`. |
| `setWorkOutfit` | `{}` | `{ ok = true }` | Applies `Config.WorkOutfit` for the detected gender. |
| `setCivilianOutfit` | `{}` | `{ ok = true }` | `skinchanger:loadSkin` with the saved civilian skin, or an error notification. |
| `closeMenu` | `{}` | `'ok'` | Releases NUI focus. |
| `refreshData` | `{}` | `'ok'` | Re-runs `UpdateMenuData()`. |

### NUI Messages (Lua → JavaScript)

| Message | Fields | When |
|---|---|---|
| `{ type = 'ui', status = boolean }` | — | Show/hide the panel. On show, the NUI programmatically clicks the Home nav link to reset the view. |
| `{ type = 'updateData', data = … }` | `minerData`, `leaderboards`, `inventory`, `isWorking`, `config` (`Picks`, `Furnaces`, `Recipes`, `Market`, `SellablePrices`, `Zones`, `RandomEvents`) | On open and on every refresh. The whole relevant config is pushed to the NUI each time, so config edits take effect on `restart nexus_minerjob` with no cache to clear. |
| `{ type = 'showProgress', label, duration }` | `label` *(string)*, `duration` *(ms)* | At the start of every mining swing. |

### Editable Functions (`shared/functions.lua`)

`shared/functions.lua` is in `escrow_ignore` and is a **shared** script, so the same `Functions` table exists on client and server. It is loaded **after** `config.lua` and the locales, so it may reference both.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `Functions.Notify` | `Functions.Notify(title, msg, type, time)` | **Client only.** `client/client.lua`: uniform on/off, civilian outfit missing, uniform required, job required, shift start/stop, pickaxe too weak (both zone and event), sync error, data load error, speed bonus, random event start/stop. | Show a notification to the local player. `type` is one of `'success'`, `'error'`, `'info'` (and `'inform'`, see below). `time` is milliseconds and defaults to `5000` in every branch. Must return nothing. |
| `Functions.NotifyServer` | `Functions.NotifyServer(player, title, msg, type, time)` | **Server only.** `server/server.lua`: level up, reward received, found nothing, level too low, already owned, item bought, not enough money, materials sold, nothing to sell, need furnace, no recipes, smelted items, not enough to smelt, no pickaxe. | Show a notification to a specific player from the server. `player` is the server ID. Must return nothing. |
| `Functions.ShowText` | `Functions.ShowText(msg)` | **Client only.** The NPC proximity loop and both mining loops. | Display a persistent TextUI prompt. Called once on entering range, not every frame. Must return nothing. |
| `Functions.HideText` | `Functions.HideText()` | **Client only.** On leaving NPC range, leaving a zone, and immediately before a swing starts. | Hide the TextUI prompt. Must return nothing. |
| `Functions.Lang` | `Functions.Lang(key, ...)` | Everywhere a player-facing string is produced, on both sides. | Resolve a locale key. Returns a `string`. Marked *"do not touch below here"* — the fallback chain (`Config.Locale` → `es` → raw key) and the `string.format` pass-through are relied on by callers that pass placeholders. |

Notes for anyone writing a `custom` branch:

- `type` values actually passed by the script are `'success'`, `'error'`, `'info'` and — at two call sites — **`'inform'`**. Handle both spellings, or map unknown types to a default, or those two notifications will render wrong on a strict notification resource.
- `Functions.Notify` and `Functions.NotifyServer` are both defined in the shared file but each is only ever called from one side. Guard any natives you add with `IsDuplicityVersion()` if you need side-specific behaviour.
- `Functions.ShowText` is called once per range entry, so if your TextUI needs a per-frame redraw you must add your own loop.

### Database Schema

One table, created by `nexus_minerjob.sql`:

```sql
CREATE TABLE IF NOT EXISTS `nexus_miner_data` (
  `identifier`   VARCHAR(60) NOT NULL COMMENT 'Identificador del jugador (ej: license:xxxx)',
  `level`        INT(11)     NOT NULL DEFAULT 1,
  `rocks_mined`  INT(11)     NOT NULL DEFAULT 0  COMMENT 'Estadística para el leaderboard',
  `money_earned` BIGINT(20)  NOT NULL DEFAULT 0  COMMENT 'Estadística para el leaderboard',
  `p_owned`      INT(11)     NOT NULL DEFAULT 0  COMMENT 'Contador de cuántos picos ha comprado en total',
  `f_owned`      INT(11)     NOT NULL DEFAULT 0  COMMENT 'Contador de cuántos hornos ha comprado en total',
  `p_highest`    INT(11)     NOT NULL DEFAULT 0  COMMENT 'El nivel del pico más alto que posee (0=ninguno)',
  `f_highest`    INT(11)     NOT NULL DEFAULT 0  COMMENT 'El nivel del horno más alto que posee (0=ninguno)',
  `xp`           INT(11)     NOT NULL DEFAULT 0  COMMENT 'Xp del minero',
  PRIMARY KEY (`identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| Column | Type | Default | Written by |
|---|---|---|---|
| `identifier` | `VARCHAR(60)` PK | — | Insert on `esx:playerLoaded` or on first `getData`. The ESX identifier (`license:…`). |
| `level` | `INT` | `1` | `requestMineRewards` on level-up. |
| `rocks_mined` | `INT` | `0` | `requestMineRewards`, `+1` per swing (including failed swings). |
| `money_earned` | `BIGINT` | `0` | `sellMaterials`, `+= total`. Lifetime stat, never decremented. |
| `p_owned` | `INT` | `0` | **Never written.** Declared and commented as a purchase counter but no query touches it. Always `0`. |
| `f_owned` | `INT` | `0` | **Never written.** Same. |
| `p_highest` | `INT` | `0` | `buyUpgrade` with `type = 'pick'`. Set to the purchased tier. |
| `f_highest` | `INT` | `0` | `buyUpgrade` with `type = 'furnace'`. Set to the purchased tier. |
| `xp` | `INT` | `0` | `requestMineRewards`. Set to `newXp`, or to the surplus after a level-up. |

**Indexes:** primary key on `identifier` only.

The leaderboard query joins the ESX `users` table:

```sql
SELECT u.firstname, u.lastname, nmd.rocks_mined, nmd.money_earned
FROM nexus_miner_data nmd
JOIN users u ON nmd.identifier = u.identifier
ORDER BY (nmd.money_earned + nmd.rocks_mined) DESC
LIMIT 10;
```

Two things to know about it: the ordering adds a money figure to a rock count, so money dominates the ranking entirely; and because it is an inner `JOIN`, a miner whose `users` row is gone simply disappears from the board.

Useful admin queries:

```sql
-- Grant a player the top pickaxe and furnace
UPDATE nexus_miner_data
SET p_highest = 4, f_highest = 3
WHERE identifier = 'license:xxxxxxxx';

-- Reset one player's progression, keeping their stats
UPDATE nexus_miner_data
SET level = 1, xp = 0, p_highest = 0, f_highest = 0
WHERE identifier = 'license:xxxxxxxx';

-- Wipe the leaderboard without touching progression
UPDATE nexus_miner_data SET rocks_mined = 0, money_earned = 0;
```

### Integration Example

Two realistic integrations.

**1. Drive mining from `ox_target` instead of the built-in proximity loop.** This is what the unused `nexus_minerjob:client:tryMine` event is for — it does the whole swing (pick check, animation, progress bar, reward request) when triggered.

```lua
-- my_mining_props/client.lua
-- Attach a mineable interaction to real rock props instead of invisible radius zones.

local ROCK_MODELS = { `prop_rock_4_c`, `prop_rock_4_big2` }

exports.ox_target:addModel(ROCK_MODELS, {
    {
        name  = 'nexus_mine_rock',
        icon  = 'fa-solid fa-gem',
        label = 'Mine this rock',
        distance = 2.0,
        onSelect = function()
            -- zoneId indexes Config.Zones; isEvent = false for a normal quarry.
            TriggerEvent('nexus_minerjob:client:tryMine', { isEvent = false, zoneId = 1 })
        end,
    },
})
```

Because `tryMine` reads `Config.Zones[data.zoneId]` and performs its own `requiredPick` check, the pickaxe gating and the reward table of zone 1 still apply. Note it does **not** check `isWorking`, so add that yourself if you want the shift to be mandatory:

```lua
-- Only allow it on shift, by asking the server for the player's state first.
onSelect = function()
    ESX.TriggerServerCallback('nexus_minerjob:getData', function(d)
        if d and d.minerData and (d.minerData.p_highest or 0) >= 1 then
            TriggerEvent('nexus_minerjob:client:tryMine', { isEvent = false, zoneId = 1 })
        end
    end)
end,
```

**2. Log progression and announce level-ups and events to Discord.** Nothing in `nexus_minerjob` needs changing; attach extra handlers to its events.

```lua
-- nexus_minerjob_logger/server.lua

local WEBHOOK = 'https://discord.com/api/webhooks/XXXX/YYYY'

local function send(title, desc, colour)
    PerformHttpRequest(WEBHOOK, function() end, 'POST', json.encode({
        embeds = {{ title = title, description = desc, color = colour or 9647850 }}
    }), { ['Content-Type'] = 'application/json' })
end

-- Purchases: fires alongside the script's own handler.
AddEventHandler('nexus_minerjob:server:buyUpgrade', function(data)
    local src = source
    local xPlayer = ESX.GetPlayerFromId(src)
    if not xPlayer then return end

    local item = (data.type == 'pick' and Config.Picks[data.level])
              or (data.type == 'furnace' and Config.Furnaces[data.level])
    if not item then return end

    -- The script validates independently; this only fires on the attempt,
    -- so confirm against the DB afterwards if you need certainty.
    send('Miner upgrade attempt',
        ('**%s** (ID %d) tried to buy **%s** for $%d')
            :format(xPlayer.getName(), src, item.name, item.price))
end)

-- Sales: read the resulting lifetime total straight from our own table.
AddEventHandler('nexus_minerjob:server:sellMaterials', function()
    local src = source
    local xPlayer = ESX.GetPlayerFromId(src)
    if not xPlayer then return end

    SetTimeout(500, function()  -- let the script's UPDATE land first
        MySQL.Async.fetchAll(
            'SELECT level, xp, rocks_mined, money_earned FROM nexus_miner_data WHERE identifier = @id',
            { ['@id'] = xPlayer.identifier },
            function(rows)
                local r = rows[1]
                if not r then return end
                send('Miner sold materials',
                    ('**%s** — level %d, %d XP, %d rocks, $%d lifetime')
                        :format(xPlayer.getName(), r.level, r.xp, r.rocks_mined, r.money_earned))
            end)
    end)
end)
```

And a client-side listener for the event announcements, so you can drive your own HUD or sound:

```lua
-- nexus_minerjob_logger/client.lua

RegisterNetEvent('nexus_minerjob:client:startRandomEvent', function(ev)
    -- ev.type is 'ZONE' | 'MULTIPLIER' | 'SPEED'
    if ev.type == 'MULTIPLIER' then
        exports['my_hud']:showBanner(('Gold Fever: x%.1f ore'):format(ev.rewardMultiplier))
    elseif ev.type == 'SPEED' then
        exports['my_hud']:showBanner(('Speed Cola: x%.1f mining'):format(ev.speedMultiplier))
    elseif ev.type == 'ZONE' and ev.coords then
        exports['my_hud']:showBanner(('Enriched vein at %s'):format(ev.name))
    end
end)

RegisterNetEvent('nexus_minerjob:client:stopRandomEvent', function()
    exports['my_hud']:clearBanner()
end)
```

**3. Read a player's mining rank from another resource** — there is no export, so query the table:

```lua
-- Expose an export of your own that other resources can use.
exports('GetMinerRank', function(identifier, cb)
    MySQL.Async.fetchAll([[
        SELECT COUNT(*) + 1 AS rank FROM nexus_miner_data
        WHERE (money_earned + rocks_mined) >
              (SELECT money_earned + rocks_mined FROM nexus_miner_data WHERE identifier = @id)
    ]], { ['@id'] = identifier }, function(rows)
        cb(rows[1] and rows[1].rank or nil)
    end)
end)
```

## ❓ FAQ

**Q: I bought a pickaxe but mining gives me nothing / gives errors about items.**
A: The shipped `config.lua` is **demo data**. Its reward tables hand out `bread`, `clothe`, `fish`, `phone` and `wool`, and its recipes turn `diamond` into `fabric`. Those are placeholders left in from testing. Replace every item name with a real registered item in all five places: `Config.Zones[].rewards`, `Config.Recipes` (both `raw` and `refined`), `Config.Market.RawItems`, `Config.Market.RefinedItems` and `Config.SellablePrices`. An item that is not registered in your framework will fail to be added and `ESX.GetItemLabel` will return `nil`.

**Q: Most images in the shop and the guide are broken.**
A: Only `html/img/horno_forja.png` ships. Nine more filenames are referenced by the default config and you must supply them yourself: `pico_piedra.png`, `pico_hierro.png`, `pico_acero.png`, `pico_diamante.png`, `horno_primitivo.png`, `horno_alto.png`, `zone_quarry.png`, `zone_mine.png`, `zone_cave.png`. Drop them in `html/img/` or point the `image` fields at files you have.

**Q: Some item icons are broken in the Mercado and Horno pages.**
A: The icon folder is chosen by which list the item is in — items in `Config.Market.RawItems` load from `html/items_images/raw/`, items in `RefinedItems` from `html/items_images/refined/`. The default config crosses those over in seven places (raw items pointed at `diamond_polished.png` and `gold_ingot.png`, which only exist in `refined/`; and five refined items pointed at `coal.png`, which only exists in `raw/`). Make sure each entry's `image` file actually exists in the folder matching its list, or copy the PNG into both folders.

**Q: The furnace recipe modal and the smelting list are empty even though I defined recipes.**
A: Check that `Config.Recipes` starts at `[1]` with no gaps. It is serialised to JSON for the NUI and read there as a JavaScript array. A contiguous `{[1]=…, [2]=…}` becomes a JSON array and works; a sparse `{[2]=…, [3]=…}` becomes a JSON object and the NUI finds nothing (while server-side smelting still works, which makes this confusing to diagnose). Also remember **each tier must repeat the lower tiers' recipes** — the lookup is exact, not cumulative.

**Q: The Steel pickaxe (tier 3) does not unlock any new area.**
A: Correct in the stock config. The three default zones require pickaxe tiers 1, 2 and 4, so tier 3 is a dead rung — a player pays 12 000 for nothing but a prerequisite. Either add a zone with `requiredPick = 3` or change the Crystal Cave's `requiredPick` from `4` to `3`.

**Q: My uniform's chest/vest piece is not applied, and the uniform check passes without it.**
A: The default config uses the component key `vest`, which is not in the script's internal component map (`face, mask, hair, arms, pants, bags, shoes, neck, tshirt, bproof, decals, torso`). Unmapped keys are silently skipped in both the apply and the verify pass, so the entry does nothing. Rename it to `torso` (component 11) or `bproof` (component 9) depending on which slot your clothing pack uses.

**Q: "Poner Ropa de Civil" says my civilian outfit could not be found.**
A: The civilian skin is captured by `SaveCivilianOutfit()`, which runs **only when you interact with the supervisor NPC**, and only once per session. If you opened the panel some other way, or `esx_skin` was not running at that moment, nothing was saved. Walk up to the NPC, press E to open the panel, and the skin is captured from then on. It also needs `skinchanger` to restore.

**Q: A player has money in the bank but cannot buy anything.**
A: Purchases use `xPlayer.getMoney()` / `removeMoney()`, which is **cash only**. There is no banking integration and no config option for it. Players must withdraw first.

**Q: Market and furnace pages show everything at 0 even though I am carrying ore.**
A: The NUI builds those views from `xPlayer.inventory`, which the stock ESX inventory populates. If you run `ox_inventory` or another replacement, that field may be empty or shaped differently, so nothing matches. Selling and smelting still work (they read `getInventoryItem` server-side) but the UI will look wrong. Test on a character holding ore before launch.

**Q: Notifications show as literal text like `job_name` or `received_items`.**
A: You have `Config.Leveling.Enabled = false`. On that code path two notification calls pass raw locale keys instead of resolved strings, so the player sees the key names. Keep levelling enabled, or report it so the escrowed `server/server.lua` can be fixed.

**Q: TextUI prompts never appear, or appear and never go away.**
A: Check `Config.TextUISystem`. Only `'okok'`, `'esx'` and `'custom'` behave correctly. `'nexus'` routes to `okokTextUI` for showing but has no close branch, so prompts stick. Any unrecognised value (including a typo) sends prompts to the server console with `print()` and the player sees nothing.

**Q: Notifications never appear at all.**
A: Same class of problem — `Config.NotifySystem` must be exactly `'nexus'`, `'okok'`, `'esx'`, `'mythic'` or `'custom'`. Anything else falls through to a `print()`. Also make sure the resource you selected is started **before** `nexus_minerjob` in `server.cfg`.

**Q: Random events never fire.**
A: Three things to check. Each event type has `enabled` — set it `true`. Each has its own `timer.min`/`timer.max` in **seconds**, defaulting to 1800–3600, so expect a 30–60 minute wait after a 15-second startup delay. And **only one event can be active at a time**; while one runs, the other timers skip their turn entirely rather than queueing, so on a short `duration` with long timers you will see them roughly round-robin, and with long durations the first type to fire tends to dominate. Watch the server console for `[nexus_minerjob] Evento aleatorio iniciado:`.

**Q: Players far from an event say nothing happened for them.**
A: You have `Config.RandomEvents.Notification.Type = 'radius'`. In that mode the event data is only sent to players within `Radius`, and players outside never receive it — so a `MULTIPLIER` or `SPEED` bonus does not apply to them and they cannot mine an event zone even if they drive there. Use `'server'` if the mechanics should be global. Note that the *stop* event is always broadcast to everyone regardless.

**Q: Can I run this on QBCore?**
A: Not without code changes. There is no `Config.Framework` key; the script calls ESX APIs directly throughout and the leaderboard query hardcodes the ESX `users` table with `firstname`/`lastname`. Those calls are in escrowed files.

**Q: Where is XP shown to the player?**
A: Nowhere, in this version. `xp` is tracked in the database and returned by `getData`, but the panel displays only Level, Rocks Mined, Earnings, Best Pickaxe and Best Furnace — there is no XP figure or progress-to-next-level bar. Players will only learn they levelled from the level-up notification. Logged as a gap in the incident report.

**Q: The leaderboard ranking looks wrong.**
A: It orders by `money_earned + rocks_mined`, adding a currency total to a count. Since earnings reach six figures while rock counts stay in the hundreds, money effectively decides the whole ranking despite the table showing both columns.

**Q: `p_owned` and `f_owned` are always 0 in the database.**
A: They are declared and commented as lifetime purchase counters but no query ever writes them. Harmless, but do not build anything on them.

**Q: Can players rebind the interaction key?**
A: No. `Config.Keys.interaction` is checked with a raw `IsControlJustReleased`, not registered through `RegisterKeyMapping`, so the only way to change it is to edit the config value — and it applies to everyone.

**Q: My pickaxe prop stays in my hand after I stop the resource.**
A: There is no `onResourceStop` cleanup, so stopping or restarting the resource mid-shift leaves the attached `prop_tool_pickaxe`, the NPC ped and the zone blips behind until the player reconnects. Press "Finalizar Turno" before a restart, or have players rejoin.

**Q: Does a failed swing still give XP?**
A: Yes, that is deliberate. `ChanceToFail` only suppresses the item reward; `XPPerRock` is granted either way and `rocks_mined` still increments. The player gets the `found_nothing_xp` notification telling them so.

### Before opening a ticket

- Make sure the resource folder name is exactly **`nexus_minerjob`** — any other name hard-errors at startup.
- Make sure you are using the latest version of the resource (current: **v2.7.1**).
- Confirm **`oxmysql` and `es_extended` both start before `nexus_minerjob`** in `server.cfg`, and that `nexus_minerjob.sql` has been imported.
- Confirm **every item name in your config exists in your inventory system**. This is the single most common cause of "mining does nothing".
- Check your server console for lines prefixed `[nexus_minerjob]` and for `[Notify]` / `[TextUI]` fallback prints — a `[Notify]` line in console means `Config.NotifySystem` is not matching any supported branch.
- Re-read this FAQ page.

## 📋 Changelog

### v2.7.1 — current
Version as declared in `fxmanifest.lua` (`version '2.7.1'`). No changelog file ships with the resource, so the entries below describe the feature set present in this build; in-code comments mark the most recent additions.

- `Config.Leveling.ChanceToFail` — marked in the config as a newly added section: a configurable chance for a swing to yield no ore while still granting XP.
- `requiredLevel` added to each `Config.Picks` entry, gated by `Config.Leveling.RequireLevelForPicks`.
- Duplicate tier-3 furnace removed from `Config.Furnaces` (noted as a correction in the config comments).
- Random event system with three mechanics (`ZONE`, `MULTIPLIER`, `SPEED`), independent per-event timers, and `server` / `radius` announcement modes.
- Persistent XP levelling with surplus carry-over, stored in `nexus_miner_data.xp`.
- In-panel wardrobe with per-gender uniform and civilian-outfit restore through `esx_skin` / `skinchanger`.
- Six-page NUI with shop, furnace, market, wardrobe and a four-tab in-game guide.
- Seven bundled locales (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`) with full key parity.
- `shared/functions.lua` shipped outside the escrow for notification and TextUI integration.
- Resource name validation via `shared/_resource.lua`.
