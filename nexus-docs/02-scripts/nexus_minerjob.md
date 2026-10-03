# Nexus Miner Job

> A complete mining career for ESX: persistent XP levels, upgradeable pickaxes and furnaces, gated quarry zones, a smelting recipe tree, a sell market, a server-wide leaderboard, server-authoritative anti-cheat validation and three automatic random events — all driven from one custom NUI panel.

**Version:** 2.8.0 · **Framework:** ESX · **Database:** oxmysql

---

## 📝 Description

Nexus Miner Job turns mining into a progression loop instead of a single repeatable action. A player talks to a supervisor NPC, puts on the work uniform from the in-panel wardrobe, buys a pickaxe, and starts a shift. While on shift, blips and cylindrical ground markers appear over every configured quarry. Standing inside one and pressing the interaction key plays the pickaxe animation, runs a progress bar, and asks the server for a reward roll. Everything that matters — level, XP, rocks mined, money earned, the highest pickaxe owned and the highest furnace owned — lives in a single MySQL row keyed by the player's ESX identifier, so progress survives restarts and reconnects.

The progression has two independent gates. **Pickaxe level** gates which quarries you can mine: each zone declares a `requiredPick`, and the server-stored `p_highest` must meet or exceed it. **Miner level** gates which pickaxes you can buy: each pickaxe declares a `requiredLevel`, checked against the player's XP level when `Config.Leveling.RequireLevelForPicks` is on. The result is that you cannot skip ahead by buying your way in — you have to actually mine to unlock the ability to buy the better tool, which then unlocks the better quarry. **Furnace level** is a third, parallel track that gates the smelting recipe tier, and it is bought with money alone.

Raw ore is near-worthless on its own; the money is in refining. Every furnace tier exposes a recipe list (`Config.Recipes[tier]`) mapping a raw item to a refined item at a configurable consumption rate. Pressing "Smelt All" runs **every** recipe the player's furnace tier knows against their whole inventory in one pass, consuming raw stock and granting refined stock — and, as of 2.8.0, only when the player has room to carry the result, so a full inventory never silently eats raw ore for nothing. The Market tab then sells everything — raw and refined — in a single "Sell All" action at `Config.SellablePrices`, adding the total to the player's money and to their lifetime `money_earned` stat.

Running on top of that loop is an automatic event system. Each event type gets its own server-side thread that sleeps a random interval between `timer.min` and `timer.max`, then fires if no other event is active, runs for `duration` seconds, and stops. Three event kinds are implemented and each has distinct mechanics: **`ZONE`** spawns a temporary high-value quarry with its own blip and route, **`MULTIPLIER`** doubles (or whatever you set) the ore yield of every *normal* quarry server-wide, and **`SPEED`** divides the mining progress-bar time (and the server's hit cooldown follows it), making every swing faster. Events announce themselves either to the whole server or only to players within a radius, configurable per deployment — and as of 2.8.0, a player who joins mid-event is sent the current event state too, instead of only seeing it from the next one.

As of version 2.8.0, every mining reward is validated server-side rather than trusted from the client: the server tracks whether a player is really on shift, re-checks their job, reads their real position to confirm they are near the zone, reads their pickaxe tier from the database, and enforces a minimum cooldown between hits. Earlier builds (2.7.1 and before) trusted the client for all of this, which made the reward loop farmable; see the Changelog for the full list of what changed.

---

## ✨ Features

### Mining
- **Supervisor NPC with blip** — a frozen, invincible, event-blocking ped spawned at `Config.NPC.coords` with a configurable model and map blip. Walking within range shows a TextUI prompt; the interaction key opens the panel.
- **Optional job requirement** — with `Config.JobRequired = true` the panel only opens, and mining only works, for players whose ESX job matches `Config.JobName`; everyone else gets a "wrong job" notification. As of 2.8.0 this is enforced on the server as well as the client, both when the shift starts and on every hit — losing the job mid-shift now ends server-side mining instead of merely hiding the client UI. With `JobRequired` off, the NPC serves anybody.
- **Zones by tier** — every zone in `Config.Zones` has its own location, radius, required pickaxe level and weighted loot table. Blips appear when the shift starts; ground markers are drawn when nearby.
- **Shift system** — mining only works during a shift. The shift is started and (as of 2.8.0) validated and tracked on the server, not just assumed from a client flag.
- **Configurable failure chance** — `Config.Leveling.ChanceToFail` lets a swing yield no ore while still granting full XP, so grinding still progresses even on a bad roll.
- **Weighted reward tables** — each zone's `rewards` use cumulative `chance` weights; a single roll of 1–100 picks at most one entry, and the amount is a random integer between `min` and `max`.

### Progression
- **XP levelling with a geometric curve** — `XPPerRock` per swing; level-up at `floor((level ^ Multiplier) * BaseXP)` XP, with surplus XP carried into the next level rather than discarded.
- **Level-gated pickaxes** — optionally require a miner level to buy each pickaxe (`Config.Leveling.RequireLevelForPicks`).
- **Server-wide leaderboard** — a top-10 table joined against the ESX `users` table showing first name, last name, rocks mined and money earned, ordered by `money_earned + rocks_mined` descending. Leaderboard names are escaped in the NUI as of 2.8.0.

### Economy
- **Six-page custom NUI** — a 1300×800 glassmorphic panel with a sidebar: **Home** (stats + leaderboard), **Shop** (pickaxes/furnaces tabs), **Furnace** (smelting), **Market** (sell, raw/refined tabs), **Wardrobe** and **Guide** (a four-tab in-game guide). Start/Finish shift buttons live permanently in the sidebar footer.
- **Home dashboard** — five stat cards (Miner Level, Rocks Mined, lifetime Earnings, Best Pickaxe by name, Best Furnace by name) resolved from config so they read e.g. `"Steel Pickaxe"` rather than `3`.
- **Upgrade shop with dual gating** — pickaxe cards show price, description and image, and go to a disabled "Owned" state once owned. Purchases are locked per player and validated server-side (type, integer level, money, ownership, and for pickaxes, miner level) so a tampered NUI or a spammed buy button cannot grant an upgrade or double-charge.
- **Furnace recipe preview** — a "Materials" button on every furnace card opens a modal listing that tier's full recipe tree (`rate`× raw → 1× refined) *before* you spend the money, so players can see what an expensive furnace actually unlocks.
- **Smelting page with live feasibility** — reads your actual inventory, lists every raw ore you hold, marks each as smeltable or not for your current furnace tier, shows the required rate, and keeps the "Smelt All" button disabled until at least one recipe is satisfiable.
- **Market with live valuation** — raw and refined tabs, each card showing icon, label, quantity held, unit price and total value; a footer running total across both tabs; "Sell All" disabled at zero value.
- **Inventory-safe rewards and smelting** — mined items and smelted products are only given when the player has room to carry them (ESX weight/limit). A full inventory now shows a dedicated message instead of silently discarding the reward or charging raw ore for nothing (2.8.0).

### Immersion
- **In-panel wardrobe** — "Put On Uniform" applies `Config.WorkOutfit` per gender (components and props, with `drawable = -1` supported to *remove* a prop); "Put On Civilian Clothes" restores the outfit captured through `esx_skin` the first time the player interacted with the NPC.
- **Uniform enforcement** — when `Config.WorkOutfitRequired` is on, the client verifies every configured component drawable *and* texture on the ped before letting the shift start, and refuses otherwise.
- **Attached pickaxe prop and animation** — a pickaxe prop is created and bone-attached to the right hand for the duration of the shift, with a mining animation played per swing. The hit now runs in its own thread so movement/attack controls are genuinely disabled for its duration, and mining is cancelled client-side if the shift ends or the player dies mid-swing (2.8.0).
- **NUI progress bar** — a bottom-of-screen labelled bar, duration-matched to the actual mining time including any speed bonus.

### Random events
- **Three random event types** — `ZONE` (temporary quarry with blip + GPS route and its own reward table, now correctly carrying `requiredPick` to the client so `anyPickaxe = false` actually restricts it — 2.8.0 fix), `MULTIPLIER` (global yield multiplier, now correctly scoped to *regular* zones only — 2.8.0 fix), `SPEED` (global mining-speed multiplier, which also shortens the server-side hit cooldown). Each has its own independent timer, duration and enable flag.
- **Event announcement modes** — `Notification.Type = 'server'` broadcasts to everyone, `'radius'` only notifies players within `Notification.Radius` metres of the event coordinates. A player who joins the server while an event is already running is now sent the current event state (2.8.0); the *stop* event is always broadcast to everyone regardless of the announcement mode.

### Anti-cheat (new in 2.8.0)
Every mining reward is now validated on the server before anything is given:
- **Shift check** — the player must have started a shift through the server (`startWork`); the client flag alone is not trusted.
- **Job check** — when `Config.JobRequired = true`, the player's job is read from ESX on the server both when the shift starts and on every hit. Losing the job mid-shift ends server-side mining.
- **Distance check** — the server reads the player's real position (requires OneSync) and rejects hits farther than the zone radius + 3 m from the zone (or event vein).
- **Pickaxe check** — the player's pickaxe tier is read from the database and must be `>= zone.requiredPick` (event veins with `anyPickaxe = true` accept any owned pickaxe).
- **Hit cooldown** — a minimum time between hits equal to 90% of `Config.ProgressBarTime` (divided by the speed multiplier while a SPEED event is active). Spammed events are dropped silently.
- **Shift end** — ending the shift (or losing the required job) clears the server-side shift state immediately via the new `nexus_minerjob:server:stopWork` event.
- **Input validation** — zone ids, upgrade types and levels sent by the client are type-checked on the server.
- Rejected mining attempts are logged in the server console with the player id and the rejection reason.

### Localization & integrations
- **Seven bundled languages** — `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`, selected with `Config.Locale`, all with identical key coverage (37 keys as of 2.8.0 — see Locales below). English is now the fallback and the shipped default throughout (NUI, config comments, console messages, SQL comments) as of 2.8.0; earlier builds defaulted to Spanish.
- **The NUI itself is now editable** — `html/` (markup, script, styles and images) is no longer escrowed as of 2.8.0, so the panel can be translated or restyled directly, not just the backend strings.
- **Pluggable notifications and TextUI** — `shared/functions.lua` ships outside the escrow with branches for `nexus_notify`, `okokNotify`, ESX natives, `mythic_notify`, `okokTextUI`, native `nexus_TextUI` world-anchored prompts (new in 2.8.0) and a `custom` slot.
- **Resource name validation** — the script hard-errors at startup if the folder is not named exactly `nexus_minerjob`.

---

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **ESX (`es_extended`)** | **Required** | Hard requirement, ESX Legacy. Listed as a manifest dependency as of 2.8.0. The script calls `esx:getSharedObject` (now preferentially via `exports['es_extended']:getSharedObject()` with the legacy event kept as a fallback, 2.8.0), `ESX.RegisterServerCallback`, `ESX.TriggerServerCallback`, `ESX.GetPlayerFromId`, `ESX.GetPlayers`, `ESX.GetItemLabel`, `xPlayer.getMoney/removeMoney/addMoney`, `xPlayer.getInventoryItem/addInventoryItem/removeInventoryItem`, `xPlayer.identifier` and `xPlayer.getName()`. **There is no QBCore branch.** |
| **oxmysql** | **Required** | Declared in `dependencies {}`. As of 2.8.0 the server script uses the modern `MySQL.query` / `MySQL.single` / `MySQL.insert` / `MySQL.update` API (previously `MySQL.Async.fetchAll` / `MySQL.Async.execute`); `@oxmysql/lib/MySQL.lua` is now loaded before the server script to avoid a startup race. |
| **A MySQL database** | **Required** | One table, `nexus_miner_data`, created by the bundled `nexus_minerjob.sql`. The leaderboard also reads the ESX `users` table. |
| **OneSync** | **Required** | New requirement in 2.8.0: the server-side distance check needs to read the player's real position, which requires OneSync (`set onesync on`). Enabled by default on modern servers; without it every mining attempt is rejected as "too far". |
| **`esx_skin` + `skinchanger`** | **Required if** `Config.WorkOutfitRequired = true` *or* players will use the wardrobe | The civilian-outfit save/restore uses the `esx_skin:getPlayerSkin` server callback and the `skinchanger:loadSkin` client event. `illenium-appearance` provides both and is the tested alternative setup. Without them the uniform can still be applied, but "Put On Civilian Clothes" will fail with the "civilian outfit not found" message. |
| **The items in your config must exist in your inventory system** | **Required** | Every `item =` in `Config.Zones[].rewards`, every `raw`/`refined` in `Config.Recipes`, and every `name` in `Config.Market` must be a real registered item. See the FAQ — the shipped config ships with demo/placeholder item names. |
| **A notification resource** | Recommended | `nexus_notify` (default), `okokNotify`, `mythic_notify`, or ESX natives. Set `Config.NotifySystem`. With `custom`/unknown the script falls back to `print()` to console and the player sees nothing. |
| **A TextUI resource** | Recommended | `okokTextUI` (default) or ESX `ShowHelpNotification`, or native `nexus_TextUI` for world-anchored 3D prompts (new in 2.8.0). Set `Config.TextUISystem`. Without a working one, the "[E] Talk to the supervisor" prompts go to console only. |
| **`lua54`** | **Required** | The manifest declares `lua54 'yes'`. |
| **Internet access from the client** | Optional | The NUI loads Font Awesome and a web font from a CDN. Without it, icons and the font fall back; layout still works. |

---

## ⚙️ Installation

1. **Get the resource.** Download it from your Tebex/Cfx.re purchase and unzip it into your `resources` folder. The folder **must** be named exactly `nexus_minerjob` — the script hard-errors at startup otherwise.

2. **Import the SQL.** Run `nexus_minerjob.sql` against your database. It creates one table with `CREATE TABLE IF NOT EXISTS`, so re-running it on an existing install (e.g. when updating) is safe and makes no schema changes.

3. **Enable OneSync.** `set onesync on` in `server.cfg`. Required as of 2.8.0 for the server-side distance check on mining hits.

4. **Check load order in `server.cfg`.** `oxmysql` and ESX must both be started before this resource, and your appearance/notification/TextUI resources before that if you use them:
   ```cfg
   set onesync on
   ensure oxmysql
   ensure es_extended
   ensure esx_skin          # or illenium-appearance
   ensure skinchanger       # if your appearance resource needs it
   ensure nexus_notify      # or your chosen notify resource
   ensure okokTextUI        # or nexus_TextUI, or your chosen TextUI resource

   # THEN:
   ensure nexus_minerjob
   ```

5. **Register your items.** This is the step people skip. The shipped config uses placeholder item names. Replace every one of them with real items from **your** `items` table / inventory system, in all five places: `Config.Zones[].rewards`, `Config.Recipes` (both `raw` and `refined`), `Config.Market.RawItems`, `Config.Market.RefinedItems` and `Config.SellablePrices`.

6. **Add the missing images.** Only one placeholder PNG ships. You must supply the rest of the pickaxe, furnace and zone images referenced by the config into `html/img/` — or change the `image` fields in `Config.Picks`, `Config.Furnaces` and `Config.Zones` to filenames you do have. Item icons live in `html/items_images/raw/` and `html/items_images/refined/`; the folder an icon is loaded from is decided by which market list the item is in, so a raw item must have its icon in `raw/` and a refined item in `refined/`.

7. **Set your coordinates.** Move `Config.NPC.coords` (a `vector4`, the `w` component is the heading) and each `Config.Zones[].coords` (`vector3`) to locations on your map. Take your coordinates standing on the ground — the NPC and zone markers are drawn slightly below the given `z`.

8. **Pick your integrations.** Set `Config.Locale`, `Config.NotifySystem` and `Config.TextUISystem` (and `Config.TextUIKey` if you use `nexus_TextUI`). If you use something not on the list, set it to `'custom'` and fill the marked block in `shared/functions.lua` — it is outside the escrow.

9. **Restart.** A full server restart is recommended on first install.

10. **Start using it.** Walk to the supervisor NPC, press **E** (`Config.Keys.interaction = 38`), buy a pickaxe in the Shop, put the uniform on in the Wardrobe, press **Start Shift**, then walk to a quarry blip and press **E** on the marker.

### Updating from 2.7.x to 2.8.0

Replace the resource folder, keeping your customised `shared/config.lua`, `shared/functions.lua` and `locales/` if you changed them. No SQL changes are required — the existing table works as-is (2.8.0 only changes how `p_owned`/`f_owned` are written, not the schema). Make sure **OneSync is enabled** (see Dependencies) or every mining attempt will be rejected as "too far". If you maintain custom locale files, add the three new keys: `inventory_full`, `mining` and `mining_vein` (see Locales below). The default `shared/config.lua` texts and the NUI are now in English; replace them only if you specifically want that change kept.

---

## 🔧 Configuration

All of this lives in `shared/config.lua`, which is outside the escrow (`escrow_ignore`).

### General

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Locale` | `string` | `'en'` | Which `Locales[...]` table to use. Must match a file in `locales/`. Bundled: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`. If a key is missing in the chosen language, `Functions.Lang` falls back to English (`en` — the fallback language was `es` before 2.8.0), then to the raw key name. |
| `Config.NotifySystem` | `string` | `'nexus'` | Notification backend. Recognised: `'nexus'`, `'okok'`, `'esx'`, `'mythic'`, `'custom'`. Anything else (including a typo) silently falls through to a `print()` and the player sees nothing. |
| `Config.TextUISystem` | `string` | `'okok'` | TextUI backend for the "[E] …" prompts. Recognised: `'nexus'` (native `nexus_TextUI` world-anchored prompts, see below), `'okok'`, `'esx'`, `'custom'`. Anything else prints to console. |
| `Config.TextUIKey` | `string` | `'E'` | Only used when `Config.TextUISystem = 'nexus'`: the key name shown/used by `nexus_TextUI`. Keep it equal to whatever control `Config.Keys.interaction` maps to. |
| `Config.Keys.interaction` | `number` | `38` (E) | GTA control index checked with `IsControlJustReleased(0, …)`. Used for both the NPC and the mining prompt when not using `nexus_TextUI`. This is a raw control check, **not** a `RegisterKeyMapping`, so players cannot rebind it from `nexus_minerjob` alone — the only way to change it is to edit the config value, which then applies to everyone. |
| `Config.JobRequired` | `boolean` | `false` | `true` restricts the panel to players whose ESX job name equals `Config.JobName`, and — as of 2.8.0 — is also re-checked server-side both when the shift starts and on every hit. |
| `Config.JobName` | `string` | `'miner'` | The ESX job **name** (not label) checked when `JobRequired` is on. Must match the `name` column in your `jobs` table. |
| `Config.ProgressBarTime` | `number` (ms) | `5000` | Base duration of one mining swing. Divided by `speedMultiplier` while a `SPEED` event is active. As of 2.8.0, this value also sets the base of the server-side hit cooldown (90% of this duration, similarly divided during a SPEED event). |

> ℹ️ **Pre-2.8.0 builds (2.7.1 and earlier) had `Config.TextUISystem = 'nexus'` miswired**: the `'nexus'` branch of `Functions.ShowText` called `okokTextUI` instead, and had no matching close branch, so prompts never closed. As of 2.8.0 `'nexus'` is a real, working option backed by native `nexus_TextUI` world prompts (see Compatibility and the FAQ). If you are still running a pre-2.8.0 build, use `'okok'` or `'custom'` instead until you update.

### NPC

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.NPC.coords` | `vector4` | `vector4(2946.52, 2747.07, 43.40, 283.46)` | Where the supervisor stands. `x, y, z` position (the ped is spawned slightly below ground level) and `w` as heading in degrees. |
| `Config.NPC.model` | `string` | `"cs_floyd"` | Ped model name. Must be a valid, streamed model; the script `RequestModel`s it and waits. |
| `Config.NPC.blip.enabled` | `boolean` | `true` | Whether to draw a map blip at the NPC. |
| `Config.NPC.blip.sprite` | `number` | `47` | Blip sprite ID. |
| `Config.NPC.blip.display` | `number` | `4` | Blip display mode (`4` = visible on both main map and minimap). |
| `Config.NPC.blip.scale` | `number` | `0.8` | Blip size. |
| `Config.NPC.blip.color` | `number` | `2` | Blip colour ID (`2` = green). |
| `Config.NPC.blip.text` | `string` | `'Miner Job'` | Blip label on the map. **Not** routed through the locale system — it is a raw config string, so translate it by hand if needed. |

### Uniform

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.WorkOutfitRequired` | `boolean` | `true` | When `true`, the client verifies every configured component's drawable **and** texture before a shift may start. When `false`, the check passes unconditionally and the wardrobe becomes purely cosmetic. |
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
            ['vest']   = { drawable = 250, texture = 0 },  -- see warning below
            ['decals'] = { drawable = 0,   texture = 0 },
            ['bproof'] = { drawable = 0,   texture = 0 },
        },
        props = {
            -- hats=0, glasses=1, ears=2, watches=6, bracelets=7
            -- drawable = -1 REMOVES the prop instead of setting it
            -- ['hats'] = { drawable = -1, texture = 0 },
        },
    },
    female = { components = { ... }, props = {} },
}
```

> ⚠️ **The shipped config uses the component key `vest`, which is not in the internal component map** (`face, mask, hair, arms, pants, bags, shoes, neck, tshirt, bproof, decals, torso`). Unmapped keys are silently skipped — both when applying the uniform and when verifying it — so the `vest` entry currently does nothing, and the uniform check passes without it being applied. The torso slot is called **`torso`** (component 11) in this script, and the slot usually labelled "vest"/"body armour" in appearance menus is **`bproof`** (component 9). Rename `vest` → `torso` (or `bproof`) to make it take effect. This has not changed as of 2.8.0.

### Levelling

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Leveling.Enabled` | `boolean` | `true` | Master switch. When `false`, XP and levels are not touched; only `rocks_mined` is incremented. |
| `Config.Leveling.RequireLevelForPicks` | `boolean` | `true` | When `true`, buying a pickaxe also requires `minerData.level >= pick.requiredLevel`. When `false`, only money and ownership are checked. |
| `Config.Leveling.XPPerRock` | `number` | `15` | XP granted per swing — granted **whether or not** ore was found. |
| `Config.Leveling.BaseXP` | `number` | `100` | Base of the level curve. |
| `Config.Leveling.Multiplier` | `number` | `1.1` | **Exponent**, not a factor. The XP needed to leave level `L` is `floor((L ^ Multiplier) * BaseXP)`. |
| `Config.Leveling.ChanceToFail.Enabled` | `boolean` | `true` | Whether a swing can come up empty. |
| `Config.Leveling.ChanceToFail.Probability` | `number` (%) | `20` | Percentage chance of finding nothing. Rolled **before** the reward table, so it is a flat 20% whiff rate independent of zone weights. A failed swing still grants `XPPerRock` and increments `rocks_mined`. |

With the defaults, levelling up costs: L1→2 = 100 XP (7 rocks), L2→3 = 214 (15), L3→4 = 334 (23), L5→6 = 585 (39), L10→11 = 1258 (84), L20→21 = 2702 (181). `BaseXP` scales the whole curve linearly while `Multiplier` changes its steepness — raise `Multiplier` to 1.5 and L20 costs 8944 XP instead of 2702.

### `Config.Picks`

A **positional array** — `Config.Picks[n]` is the pickaxe bought when the NUI sends `level = n`. Keep `level` equal to the array index.

```lua
Config.Picks = {
    {
        name          = 'Stone Pickaxe',
        description   = 'The basic pick to get started...',
        price         = 1500,
        level         = 1,   -- must equal the array index
        image         = 'pick_stone.png',  -- resolved as html/img/<image>
        requiredLevel = 1,   -- miner level needed to buy it
    },
}
```

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Shown on the shop card, in the "Best Pickaxe" stat, and in the purchase notification. |
| `description` | `string` | Shop card body text. |
| `price` | `number` | Cost in cash. Checked against `xPlayer.getMoney()` — **cash only**, bank is not consulted. |
| `level` | `number` | The pickaxe tier. Stored in `p_highest`. Compared against each zone's `requiredPick`. **Must be unique and sequential starting at 1.** |
| `image` | `string` | Filename inside `html/img/`. |
| `requiredLevel` | `number` | Miner level gate, enforced only when `Config.Leveling.RequireLevelForPicks` is `true`. |

Defaults: Stone (1,500 / tier 1 / lvl 1), Iron (5,000 / tier 2 / lvl 5), Steel (12,000 / tier 3 / lvl 10), Diamond (25,000 / tier 4 / lvl 20).

### `Config.Furnaces`

Same positional-array rule. There is **no** `requiredLevel` on furnaces — money alone.

```lua
Config.Furnaces = {
    {
        name        = 'Primitive Furnace',
        description = 'Built from clay and stone...',
        price       = 10000,
        level       = 1,   -- must equal the array index; keys Config.Recipes
        image       = 'furnace_primitive.png',
    },
}
```

The furnace `level` is what indexes `Config.Recipes`, so a tier-2 furnace uses `Config.Recipes[2]`.

Defaults: Primitive (10,000 / tier 1), Forge (30,000 / tier 2), Blast Furnace (75,000 / tier 3).

### `Config.Zones`

```lua
Config.Zones = {
    {
        name         = 'Main Quarry',
        description  = 'The starting zone...',
        image        = 'zone_quarry.png',  -- html/img/<image>, used in the guide tab
        coords       = vector3(2941.29, 2797.94, 41.0),
        radius       = 2.0,
        requiredPick = 1,
        rewards = {
            { item = 'stone',    min = 3, max = 8, chance = 60 },
            { item = 'iron_ore', min = 1, max = 3, chance = 40 },
        },
    },
}
```

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Blip label (created while on shift) and guide-tab heading. |
| `description` | `string` | Guide-tab body text. |
| `image` | `string` | Filename in `html/img/`, used as the guide card's background image. |
| `coords` | `vector3` | Centre of the zone. |
| `radius` | `number` | Interaction radius in units; the ground marker is drawn at roughly `radius * 2.0` wide. As of 2.8.0, this same `radius` is also used by the **server-side distance check** (with a +3 m tolerance), so shrinking it too far relative to your terrain/MLO can make legitimate hits get rejected as "too far" — see the FAQ. |
| `requiredPick` | `number` | Minimum `p_highest` to mine here. Checked client-side before the animation, and — as of 2.8.0 — re-checked server-side against the database value before any reward is granted. |
| `rewards` | `table[]` | Weighted table, below. |

**How the reward roll works.** One `math.random(1, 100)` is drawn. The script walks `rewards` in order accumulating `chance`; the first entry whose cumulative total is `>= roll` wins, and the loop breaks. So:

- `chance` values are **weights out of 100**, not independent probabilities. Keep each zone's weights summing to 100.
- Only **one** entry can win per swing.
- If your weights sum to **less than 100**, the remainder is a silent miss on top of `ChanceToFail`.
- If they sum to **more than 100**, the later entries become unreachable.
- Order matters: put the rare, low-weight entries first if you want them reachable at all — with `{A, chance=90}, {B, chance=10}`, B only wins on rolls 91–100, which is correct; but with `{A, chance=100}, {B, chance=10}`, B can never win.
- `min`/`max` are inclusive bounds on the amount, then multiplied by any active `MULTIPLIER` event and floored.

Defaults: Main Quarry (pick 1), Abandoned Mine (pick 2), Crystal Cave (pick 4). **Note there is no zone requiring pick 3** in the stock config, so the Steel pickaxe unlocks nothing new by default — add a zone with `requiredPick = 3` or lower the Crystal Cave's requirement to `3` if you want every tier to matter.

### `Config.Recipes`

Keyed by furnace tier. **A higher tier must repeat the lower tiers' recipes** — the lookup is `Config.Recipes[f_highest]` exactly, not a merge.

```lua
Config.Recipes = {
    [1] = {
        { raw = 'iron_ore', refined = 'iron_ingot', rate = 2 },  -- 2 raw -> 1 refined
    },
    [2] = {
        { raw = 'iron_ore', refined = 'iron_ingot', rate = 2 },  -- repeated
        { raw = 'gold_ore', refined = 'gold_ingot', rate = 2 },  -- new
    },
}
```

| Field | Type | Description |
|---|---|---|
| `raw` | `string` | Item consumed. Must also appear in `Config.Market.RawItems` to show up on the smelting page. |
| `refined` | `string` | Item produced. |
| `rate` | `number` | How many `raw` are consumed per **1** `refined`. `rate = 3` means 3 ore → 1 ingot. Smelting computes `floor(count / rate)` ingots and consumes `ingots * rate` ore, leaving the remainder. As of 2.8.0 the smelt also checks the player has room to receive the refined items first, and never consumes ore it cannot replace with the refined product. |

> ⚠️ **`Config.Recipes` must be a contiguous array starting at `[1]`.** It is serialised to JSON and sent to the NUI, which reads it as a JavaScript array. A contiguous `{[1]=…, [2]=…, [3]=…}` serialises as a JSON array and works. A sparse table such as `{[2]=…, [3]=…}` serialises as a JSON **object**, and the NUI's recipe list and preview modal will come up empty — even though the server-side smelt still works, which makes this confusing to diagnose. Always define tier 1 first and leave no gaps. This has not changed as of 2.8.0.

### `Config.RandomEvents`

```lua
Config.RandomEvents = {
    Notification = {
        Type   = 'server',  -- 'server' | 'radius'
        Radius = 500.0,
    },
    EventTypes = { ... },
}
```

| Config key | Type | Default | Description |
|---|---|---|---|
| `Notification.Type` | `string` | `'server'` | `'server'` sends the event to all players (`-1`). `'radius'` sends it only to players within `Radius` of the event coordinates. **`'server'` is also forced for any event with no `coords`** (i.e. `MULTIPLIER` and `SPEED`), since a radius is meaningless for them. |
| `Notification.Radius` | `number` | `500.0` | Metres, used only when `Type = 'radius'`. |

> ℹ️ With `Type = 'radius'`, the event payload itself is only delivered to nearby players. A player who was 600 m away when the event fired never receives the event data, so even if they walk into the temporary quarry they cannot mine it and get no speed/announcement benefit for the whole duration. If you want the mechanics to apply server-wide, keep `Type = 'server'`. The *stop* event is always broadcast to everyone regardless of this setting, and as of 2.8.0 a player who joins the server mid-event is sent the current state as well.

Each entry in `EventTypes` shares these fields:

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Shown in the announcement notification and in the guide tab. |
| `type` | `string` | **Do not change.** One of `'ZONE'`, `'MULTIPLIER'`, `'SPEED'`. Selects the mechanic. |
| `description` | `string` | Guide-tab text only. |
| `enabled` | `boolean` | When `false`, no thread is created for this event at all. |
| `duration` | `number` (seconds) | How long the event stays active. |
| `timer.min` / `timer.max` | `number` (seconds) | Random wait between this event's occurrences. Each event type has its own independent timer thread. |
| `blip` | `table` or `nil` | `{ sprite, color, scale, text }`. Only meaningful for `ZONE`; set `nil` for global events. A `ZONE` blip also gets a GPS route. |

`ZONE`-only fields:

| Field | Type | Description |
|---|---|---|
| `coords` | `vector3` | Centre of the temporary quarry. |
| `radius` | `number` | Interaction radius; also used by the server-side distance check as of 2.8.0 (same +3 m tolerance as normal zones). |
| `anyPickaxe` | `boolean` | `true` lets any owned pickaxe tier mine it. **When `true` it overrides `requiredPick` entirely.** As of 2.8.0 this field is correctly sent to clients (previously it could be silently ignored because `requiredPick` wasn't carried in the event payload). |
| `requiredPick` | `number` | Minimum `p_highest`, only consulted when `anyPickaxe` is falsy. |
| `rewards` | `table[]` | Same weighted format as `Config.Zones[].rewards`. |

`MULTIPLIER`-only: `rewardMultiplier` (`number`, `2.0` = double ore). As of 2.8.0 this multiplies only the regular zones' rewards, matching its description — in earlier builds it could also affect things it should not have.
`SPEED`-only: `speedMultiplier` (`number`, `2.0` = half the progress-bar time, applied client-side; the server-side hit cooldown follows the same multiplier as of 2.8.0).

Shipped defaults: *Enriched Vein* (`ZONE`, 600 s, every 30–60 min, `anyPickaxe = true`), *Gold Fever* (`MULTIPLIER` ×2, 300 s, every 30–60 min), *Speed Boost* (`SPEED` ×2, 480 s, every 30–60 min). All three threads start after a short boot delay, and only one event can be active at a time — whichever timer fires first wins and the others skip their turn rather than queueing.

### `Config.Market` and `Config.SellablePrices`

```lua
Config.Market = {
    RawItems = {
        { name = 'iron_ore', image = 'iron_ore.png' },  -- html/items_images/raw/<image>
    },
    RefinedItems = {
        { name = 'iron_ingot', image = 'iron_ingot.png' },  -- html/items_images/refined/<image>
    },
}

Config.SellablePrices = {
    ['iron_ore']    = 15,
    ['iron_ingot']  = 120,
}
```

| Config key | Type | Description |
|---|---|---|
| `Config.Market.RawItems` | `{name, image}[]` | Items shown on the market's "Raw Minerals" tab **and** on the smelting page. Membership in this list is also what makes an item count as "raw" for icon-path resolution — a raw item's icon is loaded from `html/items_images/raw/`. |
| `Config.Market.RefinedItems` | `{name, image}[]` | Items on the "Refined Minerals" tab. Icons load from `html/items_images/refined/`. |
| `Config.SellablePrices` | `table<string, number>` | Price per unit. **An item missing from this table shows `$0/u` in the NUI and is skipped entirely by the sell handler** — so adding an item to `Config.Market` without a price makes it unsellable. |

"Sell All" iterates `RawItems` then `RefinedItems`, sells the player's entire stock of every priced item at once, credits `xPlayer.addMoney(total)` and adds the total to lifetime `money_earned`. There is no partial-sell UI and no per-item sell button.

### `Config.Keys`

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Keys.interaction` | `number` | `38` | GTA control index. `38` is `E`. Others you might use: `47` = `G`, `74` = `H`, `246` = `Y`, `182` = `L`. This is a raw control check, **not** a `RegisterKeyMapping`, used when `Config.TextUISystem` is not `'nexus'`. |

---

## 🌐 Locales & Editable Strings

**Everything a server owner may want to change is outside the escrow as of 2.8.0** — this is a significant change from earlier builds (2.7.1 and before), where the entire NUI (`html/index.html`, `html/script.js`, `html/style.css`, images) was inside the escrow and permanently in Spanish regardless of `Config.Locale`. As of 2.8.0, `html/` and its images were moved to `escrow_ignore`, the NUI ships in English by default, and it can be translated directly by editing those files.

| File | Contents |
|---|---|
| `shared/config.lua` | All options, names, descriptions, items and prices. |
| `shared/functions.lua` | Notify / TextUI integration branches + `Functions.Lang`. |
| `locales/*.lua` | In-game texts (notifications, TextUI prompts) in 7 languages: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`. |
| `html/index.html`, `html/script.js`, `html/style.css`, `html/img/`, `html/items_images/` | The NUI and its images (English by default as of 2.8.0; edit directly to translate or restyle). |
| `nexus_minerjob.sql` | Database table. |

**Seven languages ship, all with identical key coverage.** `locales/en.lua`, `es.lua`, `de.lua`, `fr.lua`, `it.lua`, `pt.lua`, `zh.lua` each define the same set of keys — 34 in builds before 2.8.0, **37 as of 2.8.0** (three keys were added, see below). `locales/_init.lua` is a one-liner (`Locales = Locales or {}`) loaded first so the per-language files can append.

Selected with `Config.Locale`. `Functions.Lang(key, ...)` resolves in this order: `Locales[Config.Locale][key]` → `Locales['en'][key]` (the fallback language as of 2.8.0; it was `es` before) → the literal key string. When extra arguments are passed they go through `string.format`, so the `%s` placeholders must be preserved in every translation.

### Locale key reference

| Key | Placeholders | Used by |
|---|---|---|
| `job_name` | — | The title of every notification in the script. |
| `start_work` / `stop_work` | — | Shift start/end notifications. |
| `uniform_on` / `uniform_off` | — | Wardrobe confirmations. |
| `civilian_outfit_not_found` | — | When no civilian skin was captured. |
| `uniform_required` | — | Shift refused because the uniform is not worn. |
| `job_required` | — | NPC/shift refused because of `Config.JobRequired`. |
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
| `inventory_full` *(new in 2.8.0)* | — | "You cannot carry any more items." Shown when a reward or smelted item cannot be given because the inventory is full. |
| `mining` *(new in 2.8.0)* | — | Progress-bar label, "Mining...". |
| `mining_vein` *(new in 2.8.0)* | — | Progress-bar label in an event zone, "Extracting vein...". |

### Strings not covered by the locale system

- **`Config.NPC.blip.text`** is in config and therefore editable, but note it is *not* wired to the locale system, so it must be translated by hand, separately from the rest.
- Developer diagnostics printed to the server/client console (resource name errors, random-event start/stop logs, rejected-hit logs) are plain strings in the Lua files and are not localized.

### Theming

`html/style.css` exposes a `:root` block of CSS variables, now freely editable (it was inside the escrow before 2.8.0):

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

---

## 🔗 Compatibility

| System | How it is handled |
|---|---|
| **Framework** | **ESX Legacy only** (`es_extended`). There is no `Config.Framework` key and no QBCore branch anywhere in the Lua files. The leaderboard query additionally hardcodes `JOIN users u ON nmd.identifier = u.identifier` with `u.firstname` / `u.lastname`, which is the ESX schema. Running this on QBCore requires code changes, not config. |
| **Database** | `oxmysql`, via the modern `MySQL.query` / `MySQL.single` / `MySQL.insert` / `MySQL.update` API (as of 2.8.0; earlier builds used the `MySQL.Async` compatibility wrapper, which still works with `mysql-async` too). |
| **Server sync** | **OneSync required** as of 2.8.0 for the server-side mining distance check. |
| **Inventory** | Uses the ESX `xPlayer` inventory API (`getInventoryItem`, `addInventoryItem`, `removeInventoryItem`) and reads `xPlayer.inventory` to build the market and smelting views. This works with the stock ESX inventory. If you run **`ox_inventory`**, `xPlayer.inventory` is not populated the same way, so the market and furnace pages may show every item at `0` even when the player is holding ore — selling and smelting still work server-side regardless, but verify the UI on a test character before going live. |
| **Money** | Cash only: `xPlayer.getMoney()` / `removeMoney()` / `addMoney()`. Bank balance is never checked and no banking resource is integrated, so a player with money only in the bank cannot buy a pickaxe. |
| **Item labels** | `ESX.GetItemLabel(item)` for the names shown in reward and smelt notifications. An unregistered item will produce `nil` here. |
| **Notifications** | Selected by `Config.NotifySystem` in `shared/functions.lua` (outside the escrow). Branches: `nexus` → `nexus_notify`; `okok` → `okokNotify`; `esx` → ESX's own notification; `mythic` → `mythic_notify`; `custom` → an empty block for you to fill. |
| **TextUI** | Selected by `Config.TextUISystem`. `nexus` → native `nexus_TextUI` world-anchored prompts (new in 2.8.0 — see below); `okok` → `okokTextUI`; `esx` → `ESX.ShowHelpNotification`; `custom` → empty block. |
| **Clothing / appearance** | `esx_skin` (`esx_skin:getPlayerSkin` server callback) and `skinchanger` (`skinchanger:loadSkin` client event). `illenium-appearance` provides both. The uniform itself is applied with raw natives, so it does not depend on an appearance resource — only the *civilian restore* does. |
| **Target systems** | Not used by the built-in proximity loop (interaction is proximity + raw key check, not `ox_target`/`qb-target`). The `nexus_minerjob:client:tryMine` event exists specifically so an external target/interaction resource can drive a mining swing instead — see the Developer API. |
| **Keys / fuel / vehicles** | Not used. No vehicle is spawned and no keys or fuel resource is touched. |
| **Progress bars** | Self-contained. The script draws its own NUI progress bar; it does not call `ox_lib`, `progressBars` or `mythic_progbar`. |
| **Other mining scripts** | No conflict beyond overlapping coordinates and item names. Change `Config.NPC.coords` and `Config.Zones[].coords` if you run another mining job nearby. |

### Using native `nexus_TextUI` (new in 2.8.0)

Start `nexus_TextUI` before `nexus_minerjob`, set `Config.TextUISystem = 'nexus'` and keep `Config.TextUIKey` equal to whatever key `Config.Keys.interaction` maps to. The supervisor NPC gets a world-anchored prompt, and while a shift is active every zone (and the enriched-vein event) gets its own prompt too. Prompts are only created while `nexus_TextUI` is actually running — `Functions.PromptsAvailable()` checks for this — so make sure it is started first; with any other `TextUISystem` value the classic key-press flow is used instead, with identical mining/menu logic either way.

---

## 💻 Developer API

### Client Exports

**None.** The client script declares no `exports(...)` and no `exports['nexus_minerjob']` handlers.

### Server Exports

**None.** The server script declares no exports either. There is no built-in export to read a player's stats from another resource — query the `nexus_miner_data` table directly (see the Integration Example).

The public surface of this resource is: **one ESX server callback**, a handful of **network events**, and the **`Functions` table** in `shared/functions.lua` (which, being a shared script outside the escrow, you can read and extend).

### ESX Server Callback

| Callback | Parameters | Returns | Description |
|---|---|---|---|
| `nexus_minerjob:getData` | *(none)* | `nil`, or a table — see below | The single read path for all player state. Called from the client when the menu opens and whenever a refresh is requested. Returns `nil` if `ESX.GetPlayerFromId(source)` fails. **Creates the player's DB row on demand** if it does not exist yet (using `INSERT IGNORE` as of 2.8.0, which removes a duplicate-row race that could occur on the very first call). |

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
        rocks = { { firstname, lastname, rocks_mined, money_earned }, ... }  -- top 10
    },
    inventory = { ... },  -- xPlayer.inventory
}
```

> Note that on the **first** call for a brand-new player the function inserts the row and returns a *synthetic* table that may omit `xp`, `p_owned` and `f_owned`. Guard with `or 0` if you read those fields.

### Events — Emitted

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_minerjob:server:startWork` | Client → Server | *(none)* | Player pressed "Start Shift" **and** passed the client-side uniform check. |
| `nexus_minerjob:server:stopWork` | Client → Server | *(none)* | New in 2.8.0. Player pressed "Finish Shift" (or lost the required job mid-shift). Clears the server-side shift state immediately; before 2.8.0, ending a shift was purely client-side. |
| `nexus_minerjob:server:requestMineRewards` | Client → Server | `zoneId` *(number\|nil)*, `isEventZone` *(boolean)* | A mining swing finished. `zoneId` is the 1-based index into `Config.Zones`; for an event zone it is `nil` and `isEventZone` is `true`. |
| `nexus_minerjob:server:buyUpgrade` | Client → Server | `data` = `{ type = 'pick'|'furnace', level = number }` | Player clicked a "Buy" button. |
| `nexus_minerjob:server:sellMaterials` | Client → Server | *(none)* | Player clicked "Sell All". |
| `nexus_minerjob:server:smeltAll` | Client → Server | *(none)* | Player clicked "Smelt All". |
| `nexus_minerjob:client:startWorkResponse` | Server → Client | `{ success = boolean, reason = string? }` | Answer to `startWork`. `success = false` carries a localised `reason` (e.g. `no_pickaxe_start`, `job_required`). |
| `nexus_minerjob:client:refreshUI` | Server → Client | *(none)* | After a successful purchase, sale or smelt, to make the open panel re-fetch its data. |
| `nexus_minerjob:client:startRandomEvent` | Server → Client (`-1` or targeted) | `CurrentEvent` table — `{ active, type, name, rewardMultiplier, speedMultiplier, coords, radius, anyPickaxe, requiredPick, rewards, blip }` | An event started. Broadcast to `-1` when `Notification.Type = 'server'` or the event has no `coords`; otherwise sent only to players within `Notification.Radius`. Also sent to any player who joins the server while the event is still running (2.8.0). |
| `nexus_minerjob:client:stopRandomEvent` | Server → Client (`-1`) | *(none)* | An event ended. Always broadcast to everyone, even if the start was radius-limited. |
| `esx:getSharedObject` | Both | callback | Standard ESX bootstrap. As of 2.8.0, obtained preferentially through `exports['es_extended']:getSharedObject()`, with the legacy event kept as a fallback. |

### Events — Listened

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `esx:playerLoaded` | Server | `playerId`, `xPlayer` | Ensures a `nexus_miner_data` row exists for the identifier, inserting one if not. |
| `nexus_minerjob:server:startWork` | Server | — | As of 2.8.0: checks `Config.JobRequired`/`Config.JobName` (if enabled) and `p_highest >= 1` on the server, marks the player as working in server-side state, and replies with `startWorkResponse`. |
| `nexus_minerjob:server:stopWork` | Server | — | New in 2.8.0. Clears the server-side "is working" flag for that player. |
| `nexus_minerjob:server:requestMineRewards` | Server | `zoneId`, `isEventZone` | As of 2.8.0: validates the server-side shift flag, the job (if required), the player's real distance to the zone/vein (requires OneSync), the player's pickaxe tier from the database, and a minimum hit cooldown — then rolls failure chance, rolls the weighted reward table, grants the item (only if there is inventory room), applies any active multiplier, grants XP, handles level-up, increments `rocks_mined`, and notifies. Invalid/failed-validation calls give nothing and are logged server-side with the rejection reason. |
| `nexus_minerjob:server:buyUpgrade` | Server | `{type, level}` | Validates miner level (pickaxes only), ownership and cash; type- and range-checks `type`/`level`; charges and updates `p_highest`/`f_highest` (and, as of 2.8.0, the corresponding `p_owned`/`f_owned` purchase counter). |
| `nexus_minerjob:server:sellMaterials` | Server | — | Sells all priced items from both market lists. |
| `nexus_minerjob:server:smeltAll` | Server | — | Runs every recipe of the player's furnace tier against their inventory; as of 2.8.0, checks inventory space before granting refined items and never destroys ore it can't replace. |
| `nexus_minerjob:client:startWorkResponse` | Client | `{success, reason}` | On success, re-fetches `getData` to prime `PlayerData.minerData`, then starts the client-side work state. |
| `nexus_minerjob:client:refreshUI` | Client | — | Re-fetches and redraws the open panel's data. |
| `nexus_minerjob:client:startRandomEvent` | Client | event table | Stores the current event, notifies, creates the routed blip. |
| `nexus_minerjob:client:stopRandomEvent` | Client | — | Clears the current event and removes the blip. |
| `nexus_minerjob:client:tryMine` | Client | `{ isEvent = boolean, zoneId = number }` | **Not triggered by the built-in proximity loop.** A complete alternative mining entry point: it runs the pick check, animation, progress bar and reward request. It exists so an external resource (an `ox_target` zone, a custom prop interaction) can drive a mining swing instead of the built-in proximity loop — rewards still go through the same server-side validation either way. See the Integration Example. |

### Security notes on these events

As of version 2.8.0, the mining-reward path is server-validated end to end (see the Anti-cheat feature section above): shift state, job, real distance (OneSync), pickaxe tier from the database, and a hit cooldown are all checked before anything is granted, and rejected attempts are logged with their reason.

- `buyUpgrade`, `sellMaterials` and `smeltAll` were already properly validated server-side in earlier builds (level, ownership, funds, real inventory counts) and remain so.
- The client-side uniform check (`WorkOutfitRequired`) is not independently re-verified server-side; it gates whether the client allows a shift to start, not the reward path itself.
- If you are still running a pre-2.8.0 build (2.7.1 or earlier), be aware that `requestMineRewards` performed **no** server-side validation at all in that version — shift status, distance and pickaxe tier were client-side-only checks, and a player able to trigger the event directly could farm rewards freely. Update to 2.8.0 or later to close this.

### NUI Callbacks

Internal to the resource (NUI callbacks are scoped to the owning frame) but documented for completeness:

| Callback | Body | Response | Purpose |
|---|---|---|---|
| `startWork` | `{}` | `'ok'` | Runs the client-side uniform check, then triggers `server:startWork` and closes the panel. |
| `stopWork` | `{}` | `'ok'` | Stops the client-side work state (deletes the prop, removes zone blips), triggers `server:stopWork` (new in 2.8.0 — earlier builds sent no server event here at all), and closes the panel. |
| `buyUpgrade` | `{ type, level }` | `'ok'` | Forwards to `server:buyUpgrade`. |
| `sellMaterials` | `{}` | `'ok'` | Forwards to `server:sellMaterials`. |
| `smeltAll` | `{}` | `'ok'` | Forwards to `server:smeltAll`. |
| `setWorkOutfit` | `{}` | `{ ok = true }` | Applies `Config.WorkOutfit` for the detected gender. |
| `setCivilianOutfit` | `{}` | `{ ok = true }` | `skinchanger:loadSkin` with the saved civilian skin, or an error notification. |
| `closeMenu` | `{}` | `'ok'` | Releases NUI focus. |
| `refreshData` | `{}` | `'ok'` | Re-runs the menu data refresh. |

### NUI Messages (Lua → JavaScript)

| Message | Fields | When |
|---|---|---|
| `{ type = 'ui', status = boolean }` | — | Show/hide the panel. On show, the NUI resets to the Home view. |
| `{ type = 'updateData', data = ... }` | `minerData`, `leaderboards`, `inventory`, `isWorking`, `config` (`Picks`, `Furnaces`, `Recipes`, `Market`, `SellablePrices`, `Zones`, `RandomEvents`) | On open and on every refresh. The whole relevant config is pushed to the NUI each time, so config edits take effect on `restart nexus_minerjob` with no cache to clear. |
| `{ type = 'showProgress', label, duration }` | `label` *(string)*, `duration` *(ms)* | At the start of every mining swing. |

### Editable Functions (`shared/functions.lua`)

`shared/functions.lua` is outside the escrow and is a **shared** script, so the same `Functions` table exists on client and server. It is loaded **after** `config.lua` and the locales, so it may reference both.

| Function | Signature | Side | Purpose |
|---|---|---|---|
| `Functions.Notify` | `Functions.Notify(title, msg, type, time)` | Client | Show a notification to the local player. `type` is one of `'success'`, `'error'`, `'info'`, and at two call sites, `'inform'` — handle both spellings in a custom branch, or those notifications will render wrong. `time` is milliseconds and defaults to `5000`. |
| `Functions.NotifyServer` | `Functions.NotifyServer(player, title, msg, type, time)` | Server | Show a notification to a specific player (by server id) from the server. |
| `Functions.ShowText` | `Functions.ShowText(msg)` | Client | Display a persistent TextUI prompt (classic key-press mode). Called once on entering range, not every frame. |
| `Functions.HideText` | `Functions.HideText()` | Client | Hide the TextUI prompt (classic key-press mode). |
| `Functions.PromptsAvailable` | `Functions.PromptsAvailable()` | Client | New in 2.8.0. Returns `true` when `Config.TextUISystem = 'nexus'` and `nexus_TextUI` is actually started. |
| `Functions.CreatePrompt` / `Functions.DeletePrompt` | `Functions.CreatePrompt(id, coords, text, viewDistance, interactionDistance, onInteract, canInteract)` / `Functions.DeletePrompt(id)` | Client | New in 2.8.0. `nexus_TextUI` bridge used for the NPC and zone world-anchored prompts. |
| `Functions.Lang` | `Functions.Lang(key, ...)` | Shared | Resolve a locale key. The fallback chain (`Config.Locale` → `en` → raw key) and the `string.format` pass-through are relied on by every caller that passes placeholders. |

Notes for anyone writing a `custom` notify/TextUI branch:

- `Functions.Notify` and `Functions.NotifyServer` are both defined in the shared file but each is only ever called from one side. Guard any natives you add with `IsDuplicityVersion()` if you need side-specific behaviour.
- `Functions.ShowText` is called once per range entry in classic mode, so if your TextUI needs a per-frame redraw you must add your own loop.

### Database Schema

One table, created by `nexus_minerjob.sql`:

```sql
CREATE TABLE IF NOT EXISTS `nexus_miner_data` (
  `identifier`   VARCHAR(60) NOT NULL,
  `level`        INT(11)     NOT NULL DEFAULT 1,
  `rocks_mined`  INT(11)     NOT NULL DEFAULT 0,
  `money_earned` BIGINT(20)  NOT NULL DEFAULT 0,
  `p_owned`      INT(11)     NOT NULL DEFAULT 0,
  `f_owned`      INT(11)     NOT NULL DEFAULT 0,
  `p_highest`    INT(11)     NOT NULL DEFAULT 0,  -- highest pickaxe level owned (0 = none)
  `f_highest`    INT(11)     NOT NULL DEFAULT 0,  -- highest furnace level owned (0 = none)
  `xp`           INT(11)     NOT NULL DEFAULT 0,
  PRIMARY KEY (`identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| Column | Type | Default | Written by |
|---|---|---|---|
| `identifier` | `VARCHAR(60)` PK | — | Insert on `esx:playerLoaded` or on first `getData` (with `INSERT IGNORE` as of 2.8.0 to avoid a duplicate-row race). The ESX identifier (`license:...`). |
| `level` | `INT` | `1` | `requestMineRewards` on level-up. |
| `rocks_mined` | `INT` | `0` | `requestMineRewards`, `+1` per swing (including failed swings). |
| `money_earned` | `BIGINT` | `0` | `sellMaterials`, `+= total`. Lifetime stat, never decremented. |
| `p_owned` | `INT` | `0` | Lifetime pickaxe-purchase counter. **As of 2.8.0 this is actually updated** by `buyUpgrade`; in builds before 2.8.0 it was declared but never written and always stayed `0`. |
| `f_owned` | `INT` | `0` | Lifetime furnace-purchase counter. Same fix as `p_owned` — written as of 2.8.0, previously always `0`. |
| `p_highest` | `INT` | `0` | `buyUpgrade` with `type = 'pick'`. Set to the purchased tier. Read by the server-side pickaxe check on every mining hit as of 2.8.0. |
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

**1. Read a player's mining stats or rank from another resource.** There is no export, so query the table directly:

```lua
local function GetMinerStats(identifier)
    return MySQL.single.await(
        'SELECT level, xp, rocks_mined, money_earned, p_highest, f_highest FROM nexus_miner_data WHERE identifier = ?',
        { identifier }
    )
end

RegisterCommand('minerstats', function(source)
    local xPlayer = ESX.GetPlayerFromId(source)
    if not xPlayer then return end
    local s = GetMinerStats(xPlayer.identifier)
    if not s then return end
    xPlayer.showNotification(('Level %s | Rocks: %s | Earned: $%s'):format(s.level, s.rocks_mined, s.money_earned))
end)

-- Rank among all miners:
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

**2. Drive mining from `ox_target` instead of the built-in proximity loop.** This is what `nexus_minerjob:client:tryMine` is for — it does the whole swing (pick check, animation, progress bar, reward request) when triggered; rewards still go through the full server-side validation described above.

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

Because `tryMine` reads `Config.Zones[data.zoneId]` and performs its own `requiredPick` check, the pickaxe gating and the reward table of zone 1 still apply. It does **not** itself check `isWorking`, so add that yourself if you want the shift to be mandatory:

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

**3. Log progression and announce level-ups and events to Discord.** Nothing in `nexus_minerjob` needs changing; attach extra handlers to its events.

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
        exports['my_hud']:showBanner(('Speed Boost: x%.1f mining'):format(ev.speedMultiplier))
    elseif ev.type == 'ZONE' and ev.coords then
        exports['my_hud']:showBanner(('Enriched vein at %s'):format(ev.name))
    end
end)

RegisterNetEvent('nexus_minerjob:client:stopRandomEvent', function()
    exports['my_hud']:clearBanner()
end)
```

---

## ❓ FAQ

**Q: I bought a pickaxe but mining gives me nothing / gives errors about items.**
A: The shipped `config.lua` ships with **demo/placeholder item names**. Replace every item name with a real registered item in all five places: `Config.Zones[].rewards`, `Config.Recipes` (both `raw` and `refined`), `Config.Market.RawItems`, `Config.Market.RefinedItems` and `Config.SellablePrices`. An item that is not registered in your framework will fail to be added and `ESX.GetItemLabel` will return `nil`.

**Q: I mine but never get anything, and the console shows "mine rejected".**
A: As of 2.8.0 every mining hit is validated server-side and the console message says exactly why: `not working` (start the shift from the menu again, e.g. after a resource restart), `too far` (zone `radius` too small for your terrain, or OneSync not reading positions — see below), `job mismatch` (player lacks `Config.JobName`), or `pickaxe ... < required` (player doesn't own the needed pickaxe tier).

**Q: "too far" for every player on my server.**
A: Enable OneSync (`set onesync on`). Without it the server can't read player positions, so the 2.8.0 distance check rejects every hit.

**Q: "You cannot carry any more items."**
A: The player's inventory is full (ESX weight/limit). As of 2.8.0 nothing is removed or lost when this happens — the reward (or smelt) simply isn't given; free some space and try again.

**Q: Most images in the shop and the guide are broken.**
A: Only a handful of placeholder PNGs ship. The default config references several more filenames you must supply yourself in `html/img/` — or point the `image` fields in `Config.Picks`, `Config.Furnaces` and `Config.Zones` at filenames you actually have.

**Q: Some item icons are broken in the Market and Furnace pages.**
A: The icon folder is chosen by which list the item is in — items in `Config.Market.RawItems` load from `html/items_images/raw/`, items in `RefinedItems` from `html/items_images/refined/`. Make sure each entry's `image` file actually exists in the folder matching its list, or copy the PNG into both folders.

**Q: The furnace recipe modal and the smelting list are empty even though I defined recipes.**
A: Check that `Config.Recipes` starts at `[1]` with no gaps. It is serialised to JSON for the NUI and read there as a JavaScript array. A contiguous `{[1]=…, [2]=…}` becomes a JSON array and works; a sparse `{[2]=…, [3]=…}` becomes a JSON object and the NUI finds nothing (while server-side smelting still works, which makes this confusing to diagnose). Also remember **each tier must repeat the lower tiers' recipes** — the lookup is exact, not cumulative.

**Q: The Steel pickaxe (tier 3) does not unlock any new area.**
A: Correct in the stock config. The three default zones require pickaxe tiers 1, 2 and 4, so tier 3 is a dead rung — a player pays for a prerequisite with no new area attached. Either add a zone with `requiredPick = 3` or change the Crystal Cave's `requiredPick` from `4` to `3`.

**Q: My uniform's chest/vest piece is not applied, and the uniform check passes without it.**
A: The default config uses the component key `vest`, which is not in the script's internal component map (`face, mask, hair, arms, pants, bags, shoes, neck, tshirt, bproof, decals, torso`). Unmapped keys are silently skipped in both the apply and the verify pass, so the entry does nothing. Rename it to `torso` (component 11) or `bproof` (component 9) depending on which slot your clothing pack uses. This is unchanged as of 2.8.0.

**Q: "Put On Civilian Clothes" says my civilian outfit could not be found.**
A: The civilian skin is captured only when you interact with the supervisor NPC, and only once per session. If you opened the panel some other way, or `esx_skin` was not running at that moment, nothing was saved. Walk up to the NPC, press E to open the panel, and the skin is captured from then on. It also needs `skinchanger` to restore.

**Q: A player has money in the bank but cannot buy anything.**
A: Purchases use `xPlayer.getMoney()` / `removeMoney()`, which is **cash only**. There is no banking integration and no config option for it. Players must withdraw first.

**Q: Market and furnace pages show everything at 0 even though I am carrying ore.**
A: The NUI builds those views from `xPlayer.inventory`, which the stock ESX inventory populates. If you run `ox_inventory` or another replacement, that field may be empty or shaped differently, so nothing matches. Selling and smelting still work (they read real inventory counts server-side) but the UI will look wrong. Test on a character holding ore before launch.

**Q: TextUI prompts never appear, or appear and never go away.**
A: Check `Config.TextUISystem`. `'nexus'`, `'okok'`, `'esx'` and `'custom'` are the recognised values. For `'nexus'`, make sure `nexus_TextUI` is started before `nexus_minerjob` and that `Config.TextUIKey` matches your interaction key — `Functions.PromptsAvailable()` only returns true when it detects `nexus_TextUI` running. Any unrecognised `TextUISystem` value (including a typo) sends prompts to the server console with `print()` and the player sees nothing. (Builds before 2.8.0 also had a bug where `'nexus'` silently routed to `okokTextUI` with no close branch, so prompts got stuck open — fixed in 2.8.0 by adding real native `nexus_TextUI` support.)

**Q: Notifications never appear at all.**
A: `Config.NotifySystem` must be exactly `'nexus'`, `'okok'`, `'esx'`, `'mythic'` or `'custom'`. Anything else falls through to a `print()`. Also make sure the resource you selected is started **before** `nexus_minerjob` in `server.cfg`.

**Q: Random events never fire.**
A: Three things to check. Each event type has `enabled` — set it `true`. Each has its own `timer.min`/`timer.max` in **seconds**, defaulting to 1800–3600, so expect a 30–60 minute wait after a short startup delay. And **only one event can be active at a time**; while one runs, the other timers skip their turn entirely rather than queueing. Watch the server console for the random-event start/stop log lines.

**Q: Players far from an event say nothing happened for them.**
A: You have `Config.RandomEvents.Notification.Type = 'radius'`. In that mode the event data is only sent to players within `Radius`, and players outside never receive it — so a `MULTIPLIER` or `SPEED` bonus does not apply to them and they cannot mine an event zone even if they drive there. Use `'server'` if the mechanics should be global. The *stop* event is always broadcast to everyone regardless, and as of 2.8.0, anyone who joins mid-event is sent the current state too.

**Q: Can I run this on QBCore?**
A: Not without code changes. There is no `Config.Framework` key; the script calls ESX APIs directly throughout and the leaderboard query hardcodes the ESX `users` table with `firstname`/`lastname`.

**Q: Where is XP shown to the player?**
A: Nowhere, as of this version. `xp` is tracked in the database and returned by `getData`, but the panel displays only Level, Rocks Mined, Earnings, Best Pickaxe and Best Furnace — there is no XP figure or progress-to-next-level bar. Players will only learn they levelled from the level-up notification.

**Q: The leaderboard ranking looks wrong.**
A: It orders by `money_earned + rocks_mined`, adding a currency total to a count. Since earnings reach much higher numbers than rock counts, money effectively decides the whole ranking despite the table showing both columns.

**Q: `p_owned` and `f_owned` show 0 in the database.**
A: If you are on 2.8.0 or later, these are now updated on every purchase — check you're running the current version. In builds before 2.8.0, these columns were declared and commented as lifetime purchase counters but no query ever wrote to them, so they always stayed `0`; this was fixed in 2.8.0.

**Q: Can players rebind the interaction key?**
A: Not from `nexus_minerjob` itself when `Config.TextUISystem` is not `'nexus'` — `Config.Keys.interaction` is checked with a raw `IsControlJustReleased`, not registered through `RegisterKeyMapping`, so the only way to change it is to edit the config value, and it applies to everyone.

**Q: My pickaxe prop stays in my hand, or the NPC/blips stay, after a resource restart.**
A: As of 2.8.0, stopping the resource cleans up the NPC, blips and pickaxe prop, and duplicated marker/init threads were removed so a prop can no longer be duplicated. If you are still seeing this, make sure you're on 2.8.0 or later; on earlier builds there was no `onResourceStop` cleanup at all, and the fix was to press "Finish Shift" before any restart.

**Q: Notifications show as literal text like `job_name` or `received_items`.**
A: This happened in builds before 2.8.0 when `Config.Leveling.Enabled = false` — two notification calls on that code path passed raw locale keys instead of resolved strings. It was fixed in 2.8.0 along with the invalid `'inform'` notification type; update if you're still seeing it.

**Q: Does a failed swing still give XP?**
A: Yes, that is deliberate. `ChanceToFail` only suppresses the item reward; `XPPerRock` is granted either way and `rocks_mined` still increments. The player gets the "found nothing" notification (with XP shown, if levelling is enabled) telling them so.

### Before opening a ticket

- Make sure the resource folder name is exactly **`nexus_minerjob`** — any other name hard-errors at startup.
- Make sure you are using the latest version of the resource (current: **v2.8.0**) — several bugs described above (TextUI, unvalidated rewards, `p_owned`/`f_owned`, prop/NPC cleanup) are fixed only from 2.8.0 onward.
- Confirm **OneSync is enabled** (`set onesync on`) — required since 2.8.0 for mining to work at all.
- Confirm **`oxmysql` and `es_extended` both start before `nexus_minerjob`** in `server.cfg`, and that `nexus_minerjob.sql` has been imported.
- Confirm **every item name in your config exists in your inventory system**. This is the single most common cause of "mining does nothing".
- Check your server console for lines prefixed `[nexus_minerjob]` and for notify/TextUI fallback prints — a raw `print()` fallback means `Config.NotifySystem` or `Config.TextUISystem` is not matching any supported branch.
- Re-read this FAQ page.

---

## 📋 Changelog

### 2.8.0
- **Security:** `requestMineRewards` is now fully validated on the server: active shift (server-side state), job, real player distance to the zone/vein, pickaxe tier from the database (`p_highest >= zone.requiredPick`) and a minimum cooldown between hits that follows SPEED events.
- **Security:** `startWork` re-checks `Config.JobRequired` / `Config.JobName` on the server instead of trusting the client menu check; a player who loses the job mid-shift can no longer collect rewards.
- **Security:** new `stopWork` server event ends the server-side shift. Purchases are locked per player and validated (type, integer level) to prevent double charges.
- **Fix:** event veins now carry `requiredPick` to the clients, so `anyPickaxe = false` works.
- **Fix:** "Gold Fever" now multiplies only the regular zones, as its description says.
- **Fix:** mining controls are actually disabled during the hit (the hit runs in its own thread); mining is cancelled client-side if the shift ends or the player dies mid-hit.
- **Fix:** no more items lost or free XP when the inventory is full (new `inventory_full` message); smelting checks space first.
- **Fix:** notification arguments in the "levels disabled" branch and the invalid `'inform'` type.
- **Fix:** `@oxmysql/lib/MySQL.lua` is now loaded before the server script; `INSERT IGNORE` removes the duplicate-row race when a player's row is created; `p_owned` / `f_owned` counters are now updated.
- **Fix:** duplicated marker and init threads removed; resource stop cleans up the NPC, blips and pickaxe prop; a pickaxe prop can no longer be duplicated.
- **Fix:** players who join during a random event now receive it; ESX is obtained through `exports['es_extended']:getSharedObject()` with the legacy event as fallback; shift ends automatically if the required job is lost.
- **New:** native `nexus_TextUI` support (`Config.TextUISystem = 'nexus'`): world-anchored prompts for the supervisor, every mining zone and the enriched vein event. The key is handled by `nexus_TextUI`; mining and the menu use the exact same logic as the classic key press.
- **Fix:** the old `'nexus'` TextUI option pointed to `okokTextUI` by mistake.
- **Database:** all queries use the oxmysql API (`MySQL.query`/`single`/`insert`/`update`) instead of `MySQL.Async`.
- **Language:** English is now the default everywhere: NUI texts, `shared/config.lua` names/descriptions/comments, console messages, SQL comments, locale fallback (`en`). New locale keys (`inventory_full`, `mining`, `mining_vein`) in all 7 languages.
- **Manifest:** `html/` files and images added to `escrow_ignore`; `es_extended` added to dependencies; version bumped.
- **Improved:** rejected mining attempts are logged in the server console with the reason; leaderboard names are escaped in the NUI.

### 2.7.1
- `Config.Leveling.ChanceToFail` — a configurable chance for a swing to yield no ore while still granting XP.
- `requiredLevel` added to each `Config.Picks` entry, gated by `Config.Leveling.RequireLevelForPicks`.
- Duplicate tier-3 furnace entry removed from `Config.Furnaces`.
- Random event system with three mechanics (`ZONE`, `MULTIPLIER`, `SPEED`), independent per-event timers, and `server` / `radius` announcement modes.
- Persistent XP levelling with surplus carry-over, stored in `nexus_miner_data.xp`.
- In-panel wardrobe with per-gender uniform and civilian-outfit restore through `esx_skin` / `skinchanger`.
- Six-page NUI with shop, furnace, market, wardrobe and a four-tab in-game guide.
- Seven bundled locales (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`) with full key parity (34 keys).
- `shared/functions.lua` shipped outside the escrow for notification and TextUI integration.
- Resource name validation at startup.
- Known issues in this release (all addressed in 2.8.0 — see above): mining rewards were not validated server-side; the `'nexus'` TextUI option was miswired to `okokTextUI` with no close branch; the `vest` uniform component key did not map to any real slot; `p_owned`/`f_owned` were never written; no resource-stop cleanup for the NPC/blips/pickaxe prop; the entire NUI was escrowed and Spanish-only regardless of `Config.Locale`; no XP figure shown in the panel.
