# Nexus Garage

> A complete garage, vehicle-key, impound and vehicle-transfer system for ESX and QBCore, built around a fully custom React NUI and an in-game admin panel that lets you create, place and edit every garage and impound live — without ever touching a config file.

**Version:** 1.0.0 · **Framework(s):** ESX, QBCore · **Database:** MySQL / MariaDB (oxmysql)

---

## 📝 Description

Nexus Garage replaces the usual "hardcoded list of garages in config.lua" approach with a **database-driven, admin-editable garage network**. Every garage and every impound lives in its own MySQL table, is created from an in-game panel (`/garageadmin`), and is placed in the world by walking to the spot and pressing a key — the script captures the coordinates, the heading, the spawn points and the blip for you. Changes apply instantly to every connected player; no resource restart is needed.

The script supports five garage types — **public**, **private**, **shared**, **job** and **gang** — plus a separate **impound** entity type. Public garages are open to everyone and can charge parking by the day. Private garages have a single owner and can be put up for sale so a player buys them in-game. Shared garages add a member list the owner manages from the garage UI itself. Job and gang garages don't store player-owned vehicles at all: they hand out *service vehicles* from a configurable fleet, with per-grade restrictions, optional fixed plates, liveries and a cooldown. Each garage is locked to one vehicle class — **land**, **air** or **sea** — so a boat garage will not accept a helicopter.

Internally, the resource keeps the framework's own vehicle table as the single source of truth (`owned_vehicles` on ESX, `player_vehicles` on QBCore) and adds its own `garage` and `mileage` columns to it at boot, creating them with `ALTER TABLE` if they're missing. Everything that is *not* native — garages, impound records, keys, per-vehicle metadata (nickname, favourite, parking timestamp), live settings and the audit log — goes into eight dedicated `nexus_*` tables. Vehicles that are physically out in the world are tracked in memory per plate (`Nexus.Vehicles.Out`) with their network id, which is what lets the script tell the difference between *stored*, *out*, *lost* (the entity no longer exists) and *destroyed* (the entity exists but the engine or body is gone) — and charge a recovery fee for the last two instead of leaving the player stranded.

The key system is its own subsystem. Permanent keys are rows in `nexus_keys` and survive restarts; temporary keys live only in a server-side cache and either expire after a configurable number of hours or die with the server. A player taking a car out of a *shared* garage automatically receives a temporary key restricted to that garage, so they can drive it but can only park it back where it came from. With `Config.Systems.Keys = 'nexus'` the script also enforces keys on the engine, handles lock/unlock with a key-fob animation and light flash, and exposes a `/llaves` keyring UI with a live map of your cars on the street. If you'd rather keep your existing key resource, one config line bridges to qb-vehiclekeys, qs-vehiclekeys, cd_garage, okokGarage, t1ger_keys, Renewed-Vehiclekeys or wasabi_carlock instead.

---

## ✨ Features

- **Five garage types, one system** — `public` (open to all), `private` (one owner, optionally for sale), `shared` (owner + member list), `job` and `gang` (service fleets). Type is chosen in the admin panel, not in code.
- **In-game garage creation and editing** — a five-to-six step editor (Information → Capacity & costs → Map blip → Interaction point → Spawn points → Service vehicles) with per-section completion checks and a save button that stays disabled until every blocking issue is resolved.
- **In-world point capture** — pressing "Place in world" hides the UI and drops you into a capture mode with on-screen instructional buttons: a translucent ghost vehicle (or ghost ped for NPC mode) follows you, `E` adds a point, `←`/`→` rotate it, `Backspace` undoes, `Enter` finishes and `Esc`/`Backspace` cancels. Up to 20 spawn points per location; if you're driving, your vehicle's own position and heading are used.
- **Three interaction modes per point** — floating marker with TextUI, a spawned NPC with a scenario animation, or a target zone (ox_target / qb-target). A point set to `target` silently falls back to marker mode when no target system is configured.
- **Fully configurable blips** — searchable sprite picker with ~43 curated game blips plus raw ID entry (0–1000), a 50-colour swatch grid plus raw colour ID (0–85), scale slider and short-range toggle, previewed live on an in-UI Los Santos map.
- **Marker designer** — 29 marker types with image previews, RGBA colour (preset swatches + custom colour picker), size, opacity, and rotate/bob/face-camera toggles, with an animated approximation of the result.
- **Vehicle list with list and grid views** — toggle persisted per player in `localStorage`; grid cards show fuel, engine and body condition at a glance, list rows expand into a detail panel.
- **Tabs and search** — All, Favourites, *Parked here* (public garages only), Out, Impound and Members (shared-garage owners only), each with a live count. Free-text search matches nickname, model label, plate, make and the garage label.
- **Favourites and nicknames** — star any vehicle you own to pin it to the top of every list; rename it (30 chars, `<`/`>` stripped) and the nickname replaces the model name everywhere in the UI.
- **Mileage tracking** — distance is accumulated client-side every `Config.MileageIntervalMs` while you're the driver and you hold a key for that plate, flushed to the server every `Config.MileageFlushMs`, on exiting the vehicle, on storing it, and on resource stop. Displayed in km or miles.
- **Vehicle transfers between garages** — from a stored vehicle's detail panel, pick a destination from a searchable list of every compatible garage you have access to, sorted cheapest first. The fee is a flat amount plus an optional per-kilometre charge computed from the real distance between the two interaction points.
- **"Bring it here"** — the mirror image of a transfer: a vehicle stored elsewhere shows a one-click button to pull it into the garage you're standing in, at the same calculated fee.
- **Lost and destroyed vehicle recovery** — a vehicle whose entity no longer exists is marked *lost*; one whose engine health is at the floor or whose body is at zero is marked *destroyed*. Either can be recovered at its own garage or at any public garage for a separate configurable fee; a destroyed vehicle is fully repaired, refuelled to at least 50 and cleaned in the process.
- **Daily parking in public garages** — a public garage with a daily cost charges on *withdrawal*, pro-rated per started 24 h, with a configurable free grace period.
- **Garage purchase in-game** — a private or shared garage with a price and no owner shows a dedicated purchase screen (price, slots, maintenance, vehicle class) instead of the vehicle list; buying it transfers ownership immediately and reopens the garage as yours.
- **Daily maintenance / rent** — owned garages can carry a daily cost. Once rent is overdue the garage UI shows an amber banner, blocks taking, storing and transferring, and offers a single "Pay X" button that settles exactly the days owed.
- **Shared garage membership** — the owner gets a Members tab listing everyone with access (name + identifier), can revoke access with a confirmation, add a player by server ID, or pick from an auto-refreshing list of players within 10 m. Members can store and retrieve vehicles there; non-owner vehicles stored in a shared garage are visible to everyone with access.
- **Impounds as first-class locations** — their own table, blip, interaction point, spawn points, vehicle class, assigned job list, receiving society and maximum fine, all editable in the same editor as garages.
- **Impound form for officers** — `/impound` (or the `OpenImpoundMenu` export) opens a form over the nearest vehicle: pick the impound, pick a reason from six quick chips or type up to 200 characters, and set the fine with a slider plus numeric input capped at that impound's maximum. Non-networked NPC traffic is simply deleted instead of being recorded.
- **Impound release** — the owner walks to the impound, sees the reason, the officer's name, the date and the fine, and pays to get the car back. The money goes to the impound's society account through your banking resource. Vehicles of theirs sitting in *other* impounds are listed separately so they know where to go.
- **Own key system** — permanent keys (database-backed, capped per vehicle) and temporary keys (memory cache, optional expiry in hours). A three-step give-key wizard: pick the vehicle, pick the player (nearby list with distances, or raw server ID), pick the key type, with the permanent-key counter and limit shown.
- **Keyring UI (`/llaves`)** — three tabs: *Received* (keys others gave you, with remote lock/unlock, GPS marking and a "return key" action), *Given* (keys you handed out, revocable), and *Cars on the street* (your vehicles currently out, with their live coordinates plotted on an interactive, pannable, zoomable map with a distance scale).
- **Key-fob lock/unlock** — a key mapping (default `L`) plays the GTA key-fob animation with a key prop, flashes the indicators, plays the remote-control sound and toggles the doors on every client within 40 m. Range and server-side distance validation are configurable.
- **Engine immobiliser** — with `keysRequiredForEngine` on, sitting in the driver's seat of a tracked vehicle you have no key for keeps the engine off and blocks the ignition control, with a one-time notification.
- **Service vehicle fleets** — job and gang garages hand out vehicles that are *not* owned: each entry has a model (validated live against the client's loaded models), a display label, an optional fixed plate, a minimum grade, a livery index and an optional colour. Vehicles above your grade are shown locked. Taking one starts a cooldown and gives you a temporary key; the vehicle must be returned to the same garage and is deleted on return, not stored.
- **Generated service plates** — fleet vehicles without a fixed plate get a unique plate built from a configurable prefix, checked against both the database and vehicles currently out.
- **Admin player manager** — search connected players, offline characters (by name or identifier) and *by plate*, with each result showing online status, server ID and vehicle count. Open a player to get a paginated, searchable table of all their vehicles with model, plate, garage, status, fuel/engine/body condition and permanent-key count.
- **Admin vehicle actions** — assign a brand-new vehicle (live model validation, live plate-availability check, random plate generator, destination garage filtered by the model's actual class), move a vehicle to any garage (deleting it from the world first), release it from an impound without charging, send it to an impound bypassing job and maximum-fine checks, change its owner (which wipes all its keys), or delete it outright (removing its metadata, keys and impound record too).
- **Live settings panel** — 23 server settings across six pages (General, Garages, Transfers, Impounds & recovery, Keys, Interface), stored as JSON in `nexus_settings`, mirrored into `GlobalState` and applied hot to every client with no restart. One-click reset back to the `Config.Defaults` values.
- **Audit log** — 30 distinct event types recorded with actor identifier, actor name, plate and a JSON payload, browsable in the admin panel with event-type filtering, free-text search across plate/actor/payload, 50-per-page pagination and a human-readable one-line summary per row.
- **Interactive map, everywhere** — the same map component powers the garage list, the impound list, the blip preview, the editor preview and the keyring street view: a real Los Santos image, world-to-pixel projection from the configured bounds, wheel zoom at the cursor, drag to pan, "fit island" reset, metric scale bar and clickable pins that render the actual blip sprite in the actual blip colour.
- **Automatic framework detection** — `Config.Framework = 'auto'` detects `es_extended` or `qb-core` at startup and hard-errors with a clear message if neither is present.
- **Automatic schema setup and legacy migration** — `sql/install.sql` is executed from the resource at every boot (all statements are `CREATE TABLE IF NOT EXISTS`), the native vehicle table is patched with the columns the script needs, and a pre-existing older `nexus_garages` schema is detected and renamed aside instead of breaking.
- **Seven languages** — `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`, with 86 keys each and an automatic fallback to Spanish for any missing key.
- **Resource-name validation** — the resource refuses to start under any folder name other than `nexus_garage`, twice (in `shared/_resource.lua` and again in `shared/framework.lua`).

---

## 📋 Dependencies

### Required

| Dependency | Notes |
|---|---|
| **ESX** (`es_extended`) **or QBCore** (`qb-core`) | Auto-detected, or forced with `Config.Framework`. The resource errors out at startup if neither is found. |
| **oxmysql** | Declared in `fxmanifest.lua` as both a `dependency` and `@oxmysql/lib/MySQL.lua`. The script uses the `MySQL.query/single/scalar/insert/update` + `.await` API throughout. mysql-async is **not** supported. |
| **MySQL / MariaDB** | Eight `nexus_*` tables are created automatically; two columns are added to the framework's vehicle table automatically. |

### Optional

| Dependency | What it's for | Selected with |
|---|---|---|
| `nexus_notify` | Notifications (default) | `Config.Systems.Notify = 'nexus'` |
| `nexus_TextUI` | Interaction prompts (default) | `Config.Systems.TextUI = 'nexus'` |
| `okokNotify`, `ox_lib`, `mythic_notify`, `ps-ui` | Alternative notification systems | `Config.Systems.Notify` |
| `okokTextUI`, `ox_lib`, `ps-ui`, native ESX/QB TextUI | Alternative prompt systems | `Config.Systems.TextUI` |
| `ox_target` / `qb-target` | `target` interaction mode on garages and impounds, and clickable NPCs | `Config.Systems.Target` |
| `LegacyFuel`, `ps-fuel`, `ox_fuel`, `cdn-fuel`, `okokGasStation`, `nd_fuel` | Reading and writing fuel on store/spawn | `Config.Systems.Fuel` |
| `esx_addonaccount`, `qb-management`, `qb-banking`, `okokBanking`, `Renewed-Banking` | Paying impound fines into the job's society account | `Config.Systems.Banking` (`'auto'` probes them in order) |
| `qb-vehiclekeys`, `qs-vehiclekeys`, `cd_garage`, `okokGarage`, `t1ger_keys`, `Renewed-Vehiclekeys`, `wasabi_carlock` | Replacing the built-in key system | `Config.Systems.Keys` |
| `rcore_gangs` | Gang resolution for gang garages on ESX or as an alternative to QB gangs | `Config.Systems.Gangs = 'rcore_gangs'` |

### Server requirements

- **A configured job for every impound.** An impound is useless until at least one job name is attached to it, and the society that receives the fines is a job name too (defaults to the first job in the list).
- **Jobs and gangs must exist in the framework** for job/gang garages. On ESX the admin panel reads jobs and grades from the `jobs` and `job_grades` tables; on QBCore it reads `QBCore.Shared.Jobs` and `QBCore.Shared.Gangs`.
- **An ACE permission** (`nexus.garage.admin` by default) for anyone who should reach the admin panel.
- **Internet access from the game client** for vehicle thumbnails, blip sprites, marker previews and NPC previews, which are fetched from `docs.fivem.net` / `docs-backend.fivem.net`. Everything degrades to a drawn silhouette or a coloured badge when offline — see the FAQ for how to self-host them.

---

## ⚙️ Installation

1. **Download and extract the resource.**
   Download it from the cfx.re portal (Keymaster) and drop the folder into your `resources` directory.

2. **Keep the folder name exactly `nexus_garage`.**
   The resource validates its own name at startup and refuses to load otherwise. Do not rename it, do not append a version suffix.

3. **Import the SQL (optional but recommended).**
   `sql/install.sql` creates the eight tables the script needs:
   `nexus_garages`, `nexus_garage_members`, `nexus_keys`, `nexus_impounds`, `nexus_impound_vehicles`, `nexus_vehicle_meta`, `nexus_settings`, `nexus_audit_log`.
   You can import it manually, or simply start the resource — it executes the same file itself on every boot, and every statement is `CREATE TABLE IF NOT EXISTS`, so running it twice is safe.

4. **Let it patch the native vehicle table.**
   On first start the script adds, if they're missing:
   - `garage VARCHAR(60) NULL` and `mileage FLOAT NOT NULL DEFAULT 0` on `owned_vehicles` (ESX) or `player_vehicles` (QBCore);
   - `stored TINYINT(1) NOT NULL DEFAULT 1` and `type VARCHAR(20) NOT NULL DEFAULT 'car'` on `owned_vehicles` (ESX only).

   This is why **the framework must start before `nexus_garage`** — if the vehicle table doesn't exist yet the script prints a warning and skips the patch.

5. **Get the order in `server.cfg` right.**

   ```cfg
   ensure oxmysql
   ensure es_extended        -- or: ensure qb-core
   -- optional: notify / textui / target / fuel / banking resources
   ensure nexus_notify
   ensure nexus_TextUI
   -- THEN nexus_garage
   ensure nexus_garage
   ```

6. **Grant the admin ACE.**

   ```cfg
   add_ace group.admin nexus.garage.admin allow
   # or per-identifier:
   add_principal identifier.license:xxxxxxxx group.admin
   ```

7. **Adapt `config.lua` to your server.**
   At minimum review `Config.Locale`, `Config.Systems` (notify, TextUI, target, keys, fuel, banking, gangs) and `Config.Commands`. Everything else — fees, limits, parking, key rules — is editable in-game from the admin panel and is stored in the database, so you rarely need to touch `Config.Defaults`.

8. **Restart the server.**
   A full server restart is recommended for the first install so the SQL patches and the framework's shared objects are all in place. Look for `[nexus_garage] listo (esx|qbcore)` in the console.

9. **Create your first garage.**
   In-game, run `/garageadmin` → **Garages** → **Create garage**:
   - give it a name and pick the type and vehicle class;
   - optionally set slots, parking/maintenance cost and price;
   - pick the blip sprite and colour;
   - **Interaction point** → *Place in world* → walk to the garage entrance and press `E`;
   - **Spawn points** → *Place in world* → stand (or drive) where vehicles should appear, press `E` for each point, `Enter` to finish;
   - **Save**. The garage appears for every connected player immediately.

10. **Start using it.**
    - Walk into a garage marker on foot and press `E` (or use the TextUI/target prompt) to open it.
    - Drive up to a garage and press `E` to store the vehicle you're driving.
    - `/llaves` opens the keyring, `/impound` opens the impound form (requires a job attached to an impound), `/garageadmin` opens the admin panel.
    - Default key `L` locks/unlocks the nearest vehicle you hold a key for.

---

## 🔧 Configuration

All build-time configuration lives in `config.lua`, which ships **outside the escrow**. The 23 keys under `Config.Defaults` are only *defaults*: on first boot they are copied into the `nexus_settings` table and from then on the live values are whatever the admin panel last saved. "Reset settings" in the panel restores them from `Config.Defaults`.

### Top level

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Framework` | string | `'auto'` | `'auto'`, `'esx'` or `'qbcore'`. In `auto` mode the script looks for `es_extended` then `qb-core` and errors out if neither is present. |
| `Config.Locale` | string | `'es'` | Active language. Must match a file in `locales/`: `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`. Falls back to Spanish per missing key. |
| `Config.Debug` | boolean | `false` | Prints `Nexus.Debug(...)` traces to the console and turns on the debug drawing of `ox_target` / `qb-target` zones. Turn it off in production. |

### `Config.Systems` — external resource bridges

Every one of these is a plain string; the matching branch in `client/utils.lua`, `client/vehicles.lua` or `functions.lua` is picked at runtime, and each branch is wrapped in `pcall` so a missing resource degrades instead of erroring.

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Systems.Notify` | string | `'nexus'` | Notification backend: `nexus`, `okok`, `esx`, `qb`, `ox_lib`, `mythic`, `ps-ui`, `custom`. `nexus` calls `exports['nexus_notify']:Alert(title, text, 5000, kind, true)`. If the chosen system isn't running (or is `custom` and `Functions.Notify` does nothing), the script falls back to the native GTA feed ticker so a notification is never silently lost. |
| `Config.Systems.TextUI` | string | `'nexus'` | Interaction prompt backend: `nexus`, `okok`, `esx`, `qb`, `ox_lib`, `ps-ui`, `native`, `custom`. `nexus` uses `nexus_TextUI`'s `Create(config)` / `Delete(id)` with its own interaction key, so the prompt handles `E` itself; everything else uses the generic `[E] …` help text loop. |
| `Config.Systems.Target` | string | `'none'` | `none`, `ox_target`, `qb-target`, `custom`. When `none`, any location set to `target` interaction mode silently behaves as a marker, and garage NPCs are interacted with via the proximity prompt instead of being clickable. |
| `Config.Systems.Keys` | string | `'nexus'` | `nexus`, `qb-vehiclekeys`, `qs-vehiclekeys`, `cd_garage`, `okokGarage`, `t1ger_keys`, `Renewed`, `wasabi_carlock`, `custom`. **Only `'nexus'` enables the built-in keyring, the `L` key mapping and the engine immobiliser** — with any other value `/llaves` and the lock key do nothing, and keys are forwarded to that resource instead when a vehicle spawns. |
| `Config.Systems.Fuel` | string | `'LegacyFuel'` | `LegacyFuel`, `ps-fuel`, `ox_fuel`, `cdn-fuel`, `okokGasStation`, `nd_fuel`, `native`, `custom`. Used both to read fuel when storing and to write it when spawning. `ox_fuel` reads/writes the `fuel` entity statebag. Any failure falls back to `GetVehicleFuelLevel` / `SetVehicleFuelLevel`. |
| `Config.Systems.Banking` | string | `'auto'` | Where impound fines land: `auto`, `esx_addonaccount`, `qb-management`, `qb-banking`, `okokBanking`, `Renewed-Banking`, `custom`. `auto` probes, in order, `Renewed-Banking` → `okokBanking` → `qb-banking` → `qb-management` → `esx_addonaccount` and uses the first one that's started. |
| `Config.Systems.Gangs` | string | `'qb'` | `qb` (read `PlayerData.gang`, QBCore only), `rcore_gangs` (call `exports['rcore_gangs']:GetPlayerGang(src)`, works on both frameworks), or `none` to disable gang garages entirely. On ESX with `'qb'`, gang resolution always returns nil, so gang garages can never be entered. |

### `Config.Resources` — resource name overrides

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Resources.nexus_notify` | string | `'nexus_notify'` | Folder name of your Nexus notification resource. Resolved case-insensitively at runtime, so `Nexus_Notify` also works. |
| `Config.Resources.nexus_textui` | string | `'nexus_TextUI'` | Folder name of your Nexus TextUI resource. Same case-insensitive resolution. |

### `Config.Commands`, keys and permissions

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Commands.Admin` | string | `'garageadmin'` | Chat command that requests the admin panel. The client only asks the server; the server checks the ACE and replies, so a non-admin gets a "no permission" notification and nothing opens. |
| `Config.Commands.Keys` | string | `'llaves'` | Chat command that opens the keyring. Does nothing unless `Config.Systems.Keys == 'nexus'`. |
| `Config.Commands.Impound` | string | `'impound'` | Chat command that opens the impound form over the nearest vehicle (within 8 m, or the one you're sitting in). |
| `Config.AdminAce` | string | `'nexus.garage.admin'` | ACE permission checked with `IsPlayerAceAllowed` for every `admin:*` server callback and for opening the panel. |
| `Config.LockKey` | string | `'L'` | Default key for the `nexus_garage_lock` key mapping (lock/unlock nearest keyed vehicle). Players can rebind it in the FiveM keybind settings. |
| `Config.InteractKey` | number | `38` | Control index used by the proximity prompt (38 = `E`). Also the confirm/add key during in-world point capture. |
| `Config.Payment` | string | `'both'` | Which account pays fees: `'cash'`, `'bank'` or `'both'` (try cash, then bank). Impound fines additionally honour the `allowImpoundBankPayment` live setting, which can force cash only. |

### `Config.Defaults` — live settings (editable in-game)

These are the **initial** values for the 23 settings stored in `nexus_settings` and published to clients through `GlobalState.nexusGarageSettings`. Changing them in `config.lua` after first boot has no effect until you press "Reset settings" in the admin panel. The merge is type-safe: a stored value is only accepted if a default with the same key and the same Lua type exists.

| Config key | Type | Default | Description |
|---|---|---|---|
| `saveDamage` | boolean | `true` | Persist body/engine/tank damage, broken windows, broken doors and burst tyres when a vehicle is stored. When `false`, the vehicle is restored to pristine condition on store. Each garage can override this individually (`saveDamage` per garage: inherit / force save / force repair). |
| `spawnInsideVehicle` | boolean | `true` | Warp the player into the driver's seat when a vehicle is taken out. |
| `lockOnSpawn` | boolean | `false` | Spawn vehicles with the doors locked, so the player has to unlock with their key first. |
| `transfersEnabled` | boolean | `true` | Master switch for garage-to-garage transfers. When off, no destinations are offered and `garage:transfer` refuses with `transfers_disabled`. |
| `transferFee` | number | `250` | Flat fee charged for every transfer (and for "bring it here"). |
| `transferFeePerKm` | number | `0` | Extra charge per kilometre of straight-line distance between the two garages' interaction points. `0` means flat fee only. |
| `destroyedRecoveryFee` | number | `1500` | Fee to recover a vehicle whose entity is destroyed. The vehicle is repaired, refuelled to at least 50 and cleaned. |
| `lostRecoveryFee` | number | `500` | Fee to recover a vehicle whose entity no longer exists in the world (server restart, cleanup, despawn). |
| `parkingGraceHours` | number | `2` | In public garages with a daily cost, how many hours are free before parking starts being billed. |
| `allowImpoundBankPayment` | boolean | `true` | Allow impound fines to be paid from the bank account. When `false`, only cash is accepted regardless of `Config.Payment`. |
| `notifyOwnerOnImpound` | boolean | `true` | Notify the vehicle's owner (if online) with the plate, the impound name and the fine when their vehicle is impounded. |
| `maxPermanentKeys` | number | `5` | Maximum number of *permanent* key holders per plate. Temporary keys are not counted. Giving one past the limit fails with `key_limit`. |
| `tempKeyHours` | number | `0` | Lifetime of temporary keys in hours. `0` means they last until the next server restart. A background thread sweeps expired keys once a minute and notifies the holder. |
| `keysRequiredForEngine` | boolean | `true` | With the built-in key system, keep the engine off and block the ignition control for a driver with no key for that plate. Only applies to vehicles the script is tracking. |
| `lockDistance` | number | `15.0` | Maximum distance (m) at which the key mapping finds a keyed vehicle. The server validates `lockDistance + 5` for the key press and `lockDistance × 4` for a remote lock from the keyring UI. |
| `jobPlatePrefix` | string | `'NX'` | Prefix (max 4 characters) for the plates generated for service vehicles that have no fixed plate. |
| `defaultInteractionMode` | string | `'marker'` | Interaction mode pre-selected when creating a new garage or impound: `marker`, `npc` or `target`. |
| `defaultNpcModel` | string | `'s_m_y_valet_01'` | Ped model used by NPC-mode points that don't specify one. |
| `markerDrawDistance` | number | `25.0` | Distance (m) from which garage/impound markers start being drawn. |
| `interfaceSounds` | boolean | `true` | Enable the UI sound effects (open, close, click, success, error — synthesised in the browser) and the in-game garage-door sounds on open/store. |
| `showMileage` | boolean | `true` | Show the odometer column in vehicle lists. |
| `mileageUnit` | string | `'km'` | `'km'` or `'mi'`. Mileage is always stored in kilometres and converted for display. |
| `currency` | string | `'$'` | Currency symbol prefixed to every amount in the UI (max 3 characters). |

### Mileage, capture and map

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.MileageIntervalMs` | number | `2000` | How often (ms) the client samples the driven vehicle's position to accumulate distance. Samples under 0.5 m or over 400 m are discarded as noise/teleports. |
| `Config.MileageFlushMs` | number | `60000` | How often (ms) accumulated distance is sent to the server. Also flushed on leaving the vehicle, on storing it and on resource stop. The server rejects any single report that is ≤ 0 or > 50 km. |
| `Config.CaptureModels.car` | string | `'sultan'` | Ghost model shown while placing **land** spawn points. |
| `Config.CaptureModels.air` | string | `'frogger'` | Ghost model shown while placing **air** spawn points. |
| `Config.CaptureModels.sea` | string | `'seashark'` | Ghost model shown while placing **sea** spawn points. For sea points the capture uses the water height instead of the ground height. |
| `Config.Map.image` | string | `'assets/map.png'` | Path (relative to `html/`) of the Los Santos map image used by every in-UI map. The shipped file is `html/assets/map.png`. |
| `Config.Map.minX` | number | `-5661.2` | West edge of the map image in world coordinates. |
| `Config.Map.maxX` | number | `6694.0` | East edge of the map image in world coordinates. |
| `Config.Map.minY` | number | `-4058.5` | South edge of the map image in world coordinates. |
| `Config.Map.maxY` | number | `8429.3` | North edge of the map image in world coordinates. Together with the other three bounds these define the world→pixel projection; change them only if you replace the image. |

### Nested structures stored in the database, not in config

Garages and impounds are **not** defined in `config.lua`. They are rows whose `blip`, `interaction` and `spawns` columns hold JSON with the following shapes. You only need these if you're writing them yourself (through `exports:RegisterGarage` or directly in SQL) — the admin panel produces them for you, and the server sanitises and clamps every field on read and on write.

```lua
-- blip (nexus_garages.blip / nexus_impounds.blip)
{
    enabled    = true,   -- boolean; anything other than exactly false means true
    sprite     = 357,    -- integer, clamped 0..1000
    color      = 27,     -- integer, clamped 0..85
    scale      = 0.8,    -- number, clamped 0.3..2.0
    shortRange = true,   -- boolean; true = only visible on the minimap when close
}

-- interaction (nexus_garages.interaction / nexus_impounds.interaction)
{
    mode   = 'marker',                        -- 'marker' | 'npc' | 'target'; invalid -> settings.defaultInteractionMode
    coords = { x = 215.0, y = -810.0, z = 30.7, w = 157.0 },  -- REQUIRED; w = heading. x/y/z rounded to 3 decimals, w to 2
    radius = 2.0,                             -- number, clamped 0.5..15.0
    marker = {
        type       = 36,                      -- integer, clamped 0..43
        size       = 1.0,                     -- number, clamped 0.2..5.0
        color      = { 147, 51, 234, 180 },   -- R, G, B, A each clamped 0..255
        bob        = false,                   -- boolean; must be exactly true to enable
        rotate     = false,
        faceCamera = false,
    },
    npc = {
        model    = 's_m_y_valet_01',          -- any ped model; empty/invalid -> settings.defaultNpcModel
        scenario = 'WORLD_HUMAN_CLIPBOARD',   -- '' for no animation
    },
}

-- spawns (nexus_garages.spawns / nexus_impounds.spawns) — max 20 entries, extras dropped
{
    { x = 230.1, y = -795.2, z = 30.5, w = 160.0 },
    { x = 233.4, y = -795.1, z = 30.5, w = 160.0 },
}

-- job_vehicles (nexus_garages.job_vehicles) — only used by 'job' and 'gang' garages
{
    {
        model    = 'police',          -- REQUIRED non-empty string; lowercased, spaces stripped
        label    = 'Patrol',          -- display name, truncated to 40 chars; defaults to model
        plate    = 'LSPD01',          -- fixed plate, uppercased and truncated to 8 chars; '' = generate one
        minGrade = 0,                 -- integer, clamped 0..100
        livery   = -1,                -- integer, clamped -1..50; -1 = leave default
        color    = nil,               -- optional table, passed through untouched
    },
}

-- jobs (nexus_impounds.jobs) — REQUIRED, at least one non-empty string
{ 'police', 'mechanic' }
```

---

## 🌐 Locales & Editable Strings

### Included languages

Seven complete translations, each with **86 keys** and byte-for-byte identical key sets:

| File | Language |
|---|---|
| `locales/es.lua` | Spanish (fallback language) |
| `locales/en.lua` | English |
| `locales/de.lua` | German |
| `locales/fr.lua` | French |
| `locales/it.lua` | Italian |
| `locales/pt.lua` | Portuguese |
| `locales/zh.lua` | Chinese |

### Where the strings live

- `locales/_init.lua` just creates the global `Locales` table.
- `locales/<lang>.lua` each assign `Locales['<lang>'] = { ... }`. **All of these files, and `locales.lua`, ship outside the escrow** (`escrow_ignore` in `fxmanifest.lua` covers `config.lua`, `functions.lua`, `locales.lua`, `locales/*.lua` and `sql/*.sql`), so you can edit every string and add your own language.
- `locales.lua` defines the `_T(key, ...)` helper. It resolves against `Locales[Config.Locale]`, then against `Locales['es']`, then returns the key itself. Arguments are applied with `string.format` inside a `pcall`, so a translation with a wrong number of `%s` placeholders degrades to the unformatted string instead of erroring.

### Adding a language

1. Copy `locales/en.lua` to `locales/<code>.lua`.
2. Change the first line to `Locales['<code>'] = {`.
3. Translate the values, keeping every key and every `%s` placeholder.
4. Set `Config.Locale = '<code>'`. No manifest change is needed — `locales/*.lua` is already globbed.

### Server-side error codes are locale keys

Every server callback failure returns a short code (`garage_full`, `rent_due`, `not_enough_money`, `wrong_class`, …) rather than a sentence. The client translates it at the last moment: `client/nui.lua` wraps every NUI response through `localized()`, which fills `r.message` with `_T(r.error, r.extra)`. All 86 keys are reachable this way, which means every error the player can see is translatable.

### Known gaps

Two things are **not** covered by `locales/`:

1. **The NUI is Spanish-only.** Every label, heading, button, tab, empty state, confirmation text and tooltip in the React interface is a hardcoded Spanish literal inside `web/src/**` and therefore inside the compiled `html/bundle.js`. Only *notification and error text* reaches the UI translated. `html/bundle.js` is not escrow-protected, but it is minified, so translating the interface realistically means rebuilding from the `web/` source (`npm install && npm run build`). This is a limitation to be aware of if you run a non-Spanish server.
2. **A handful of Spanish strings sit in escrow-protected Lua.** The `L` key-mapping description, the `" (copia)"` suffix added to a duplicated garage's name, the `"Depósito #<id>"` fallback label for a deleted impound, the `"Sistema"` actor name used when a vehicle is impounded by a script instead of a player, and the startup/console error messages are all hardcoded in protected files rather than coming from `locales/`. None of them is player-facing in normal play except the key-mapping description and the duplicate suffix.

---

## 🔗 Compatibility

Everything below is selected with one line in `Config.Systems`. No protected code has to be edited; `functions.lua` is the escape hatch for anything not listed.

### Frameworks

| Framework | Support |
|---|---|
| **ESX** (`es_extended`) | Full. Uses `getSharedObject()`, `GetPlayerFromId`, `GetPlayerFromIdentifier`, `p.identifier`, `p.job`, `p.getMoney()/getAccount('bank')`, `p.removeMoney/removeAccountMoney/addAccountMoney`, `Game.GetVehicleProperties/SetVehicleProperties`, the `users` table for offline names and `jobs`/`job_grades` for the admin job list. Vehicle table `owned_vehicles`, owner column `owner`, stored column `stored`. |
| **QBCore** (`qb-core`) | Full. Uses `GetCoreObject()`, `Functions.GetPlayer`, `GetPlayerByCitizenId`, `PlayerData.citizenid`, `PlayerData.job`/`.gang`, `PlayerData.money`, `Functions.AddMoney/RemoveMoney`, `Functions.GetVehicleProperties/SetVehicleProperties`, the `players` table for offline names and `QBCore.Shared.Jobs`/`.Gangs` for the admin lists. Vehicle table `player_vehicles`, owner column `citizenid`, stored column `state`. |

Selection: `Config.Framework = 'auto' | 'esx' | 'qbcore'`.

### Notifications — `Config.Systems.Notify`

`nexus` (`nexus_notify`, signature `Alert(title, text, duration, type, playSound)`), `okok` (`okokNotify:Alert`), `esx` (`ShowNotification`), `qb` (`Functions.Notify`, mapping `info` → `primary`), `ox_lib` (`notify`, mapping `info` → `inform`), `mythic` (`mythic_notify:DoHudText`), `ps-ui` (`ps-ui:Notify`), `custom` (`Functions.Notify(message, kind)` in `functions.lua`). Any failure or missing resource falls back to the native GTA feed ticker.

### TextUI / prompts — `Config.Systems.TextUI`

`nexus` (`nexus_TextUI`'s `Create(config)` / `Delete(id)` — note the Nexus API is `Create`/`Delete`, not `Open`/`Close`; the config passed includes `id`, `coords`, `viewDistance`, `interactionDistance`, `text`, `key`, `onInteract` and `canInteract`), `okok` (`okokTextUI:Open/Close`), `esx` (`TextUI`/`HideUI`), `qb` (`qb-core:DrawText/HideText`), `ox_lib` (`showTextUI/hideTextUI`), `ps-ui` (`DisplayText/HideText`), `native` / `custom` / anything unavailable → the native `DisplayHelp` loop.

With `nexus_TextUI` the prompt owns the keypress and is suppressed while the NUI is open or while you're in a vehicle or capturing points. With every other system the script draws its own `[E] …` prompt and reads `Config.InteractKey` itself.

### Target — `Config.Systems.Target`

`ox_target` (`addSphereZone` / `removeZone`, `addLocalEntity` / `removeLocalEntity`), `qb-target` (`AddCircleZone` / `RemoveZone`, `AddTargetEntity` / `RemoveTargetEntity`), `custom` (`Functions.AddTarget(id, coords, radius, options)` / `Functions.RemoveTarget(id)`), or `none`. Target is used for locations whose interaction mode is `target`, and to make NPC-mode peds clickable. The admin editor warns you inline when you pick `target` mode with no target system configured.

### Vehicle keys — `Config.Systems.Keys`

| Value | Behaviour on spawn | Behaviour on store |
|---|---|---|
| `nexus` | Built-in system: local key cache, `L` key mapping, keyring UI, engine immobiliser, server-validated remote lock | — |
| `qb-vehiclekeys` | `TriggerEvent('vehiclekeys:client:SetOwner', plate)` | — |
| `qs-vehiclekeys` | `exports['qs-vehiclekeys']:GiveKeys(plate, model, true)` | `RemoveKeys(plate, model)` |
| `cd_garage` | `TriggerEvent('cd_garage:AddKeys', plate)` | — |
| `okokGarage` | `TriggerServerEvent('okokGarage:GiveKeys', plate)` | — |
| `t1ger_keys` | `exports['t1ger_keys']:GiveTemporaryKeys(plate, model, 'nexus_garage')` | — |
| `Renewed` | `exports['Renewed-Vehiclekeys']:addKey(plate)` | `removeKey(plate)` |
| `wasabi_carlock` | `exports.wasabi_carlock:GiveKey(plate)` | `RemoveKey(plate)` |
| `custom` | `Functions.GiveKeys(vehicle, plate)` | `Functions.RemoveKeys(vehicle, plate)` |

With anything other than `nexus`, the keyring command, the key mapping and the immobiliser are disabled — the external resource owns keys entirely. Note that the `nexus_keys` table and the `keys:*` callbacks still exist and the `GiveVehicleKey` / `RemoveVehicleKey` / `DoesPlayerHaveKeys` server exports still work; they just stop affecting gameplay.

### Fuel — `Config.Systems.Fuel`

`LegacyFuel`, `ps-fuel`, `cdn-fuel`, `okokGasStation` (all `GetFuel`/`SetFuel`), `nd_fuel` (`getFuel`/`setFuel`), `ox_fuel` (the `fuel` entity statebag), `native` (`GetVehicleFuelLevel`/`SetVehicleFuelLevel`), `custom` (`Functions.GetFuel` / `Functions.SetFuel`). Everything is `pcall`-wrapped with a native fallback, so an uninstalled fuel resource never blocks storing a vehicle.

### Banking / societies — `Config.Systems.Banking`

Used for one thing only: crediting the impound's society when a fine is paid. `esx_addonaccount` (`getSharedAccount` on `society_<name>`), `qb-banking` (`AddMoney(society, amount, 'nexus_garage')`), `qb-management` (`AddMoney(society, amount)`), `okokBanking` (`AddMoney`), `Renewed-Banking` (`addAccountMoney`), `custom` (your own code in `Functions.AddSocietyMoney`), or `auto` to probe them in order.

### Gangs — `Config.Systems.Gangs`

`qb` (QBCore `PlayerData.gang`; returns nothing on ESX), `rcore_gangs` (`exports['rcore_gangs']:GetPlayerGang(src)`, works on both frameworks), `none` (gang garages never grant access). The admin panel hides the "Gang" garage type on ESX unless at least one gang is reported.

### Not used

The script does **not** integrate with any housing, inventory, phone or job-creator resource out of the box. If you want a housing script to create per-house garages, use the `CreatePrivateGarage` export or the `nexus_garage:server:createPrivateGarage` event (see **Integration Example**). `cd_garage` and `okokGarage` appear only as *key* bridges, not as garage providers.

---

## 💻 Developer API

Everything in this section was read directly from the source. Unless noted otherwise, server exports live in `server/api.lua`, client exports in `client/exports.lua` and `client/keys.lua`.

### Client Exports

| Export | Parameters | Returns | Description |
|---|---|---|---|
| `OpenGarage` | `key: string`, `spawn?: table` | — | Opens the garage UI for the garage with that key (`ng_<id>`). `spawn` is an optional extra `{x,y,z,w}` used as a fallback spawn point when the garage has none free or none at all — useful for housing garages. Runs `Functions.CanOpenGarage(key)` first and aborts silently if it returns false. |
| `StoreVehicle` | `key: string` | — | Stores the vehicle the player is currently driving into that garage. Validates that the player is in a vehicle and is the driver, flushes pending mileage, collects properties and calls the server. |
| `OpenKeysMenu` | — | — | Opens the keyring UI. No-op unless `Config.Systems.Keys == 'nexus'`. |
| `OpenImpoundMenu` | — | — | Opens the impound form over the nearest vehicle (within 8 m, or the one the player is in). Runs `Functions.CanImpound()` first. Notifies and aborts if there's no vehicle or the player's job has no impound. |
| `GetPlate` | `vehicle: number` | `string` | The vehicle's plate, uppercased and trimmed the same way the script stores it. |
| `GetCorrectPlateFormat` | `plate: string` | `string` | Normalises an arbitrary plate string to the script's canonical form (uppercase, outer whitespace stripped). Use this before comparing plates with the script's data. |
| `GetGarageType` | `key: string` | `string \| nil` | The type (`public`/`private`/`shared`/`job`/`gang`, or `impound` for impound keys) of a location in the client's synced world data, or `nil` if the client doesn't know that key. |
| `GetVehicleProperties` | `vehicle: number` | `table` | The framework's vehicle properties, plus the script's normalised `plate`, `bodyHealth`, `engineHealth`, `tankHealth`, `fuelLevel` (read through the configured fuel system) and `dirtLevel`. |
| `SetVehicleProperties` | `vehicle: number`, `props: table` | — | Applies properties through the framework, then force-applies plate, engine/body/tank health and fuel (through the configured fuel system). |
| `HasKey` | `plate: string` | `boolean` | Whether the local player currently holds a key for that plate, according to the key cache the server pushes. Defined in `client/keys.lua`. |

```lua
-- Open a housing garage at a custom spawn, from another resource
exports['nexus_garage']:OpenGarage('ng_14', { x = 265.4, y = -1008.2, z = 29.1, w = 180.0 })

-- Gate your own tuning menu on the player actually having the keys
local plate = exports['nexus_garage']:GetPlate(vehicle)
if not exports['nexus_garage']:HasKey(plate) then
    return -- no key, no tuning
end
```

### Server Exports

| Export | Parameters | Returns | Description |
|---|---|---|---|
| `RegisterGarage` | `garageId: string\|number`, `options: table` | `true` | Registers an **in-memory, dynamic** garage under that key. `options`: `label`/`name`, `type` (default `'private'`), `vehicleClass` (`car`/`air`/`sea`), `capacity`, `owner`, `access`, `spawns` (array) or `spawn` (single), `coords` (the interaction point). Dynamic garages have `capacityMode = 'player'`, no price, no daily cost, no blip, and are **not** persisted — they vanish on restart, so re-register them from your resource's start handler. |
| `UpdateGarage` | `garageId: string\|number`, `options: table` | `boolean` | Merges `options` into an existing **dynamic** garage and re-registers it. Returns `false` for an unknown key or for a database-backed garage. |
| `UnregisterGarage` | `garageId: string\|number` | — | Removes a dynamic garage from memory. |
| `RefreshGarageAccess` | `garageId: string\|number`, `src?: number` | `boolean` | With `src`, evaluates the full access check for that player against that garage (ownership, membership, job, grade, enabled state). Without `src`, just reports that the garage exists. |
| `GetAllGarages` | — | `table[]` | Every database garage as a full public record (id, key, name, type, vehicleClass, job, minGrade, owner, ownerName, price, capacity, capacityMode, dailyCost, enabled, blip, interaction, spawns, jobVehicles, jobCooldown, saveDamage, occupied, members, rentPaidUntil, dynamic), followed by every dynamic garage as a short record (`key`, `name`, `type`, `vehicleClass`, `dynamic = true`). |
| `GetGarageLabel` | `key: string` | `string \| nil` | Human-readable name for a garage key *or* an impound key (`impound_<id>`). Falls back to the key itself. |
| `GetFirstGarageIdForVehicleType` | `vclass?: string` | `string \| nil` | The key of the first enabled **public** garage matching that vehicle class (`'car'` when omitted). Handy as a default destination. |
| `GetPlayerVehicles` | `identifier: string \| number` | `table[]` | Every vehicle owned by that identifier, fully serialised (plate, model, hash, nickname, favorite, class, fuel 0-100, engine 0-100, body 0-100, mileage, garage, garageLabel, status, parkedAt, owner, and an `impound` sub-table when impounded). Accepts a player server id, which is resolved to an identifier. |
| `GetVehicleGarage` | `plate: string` | `string \| nil` | The `garage` column of that plate's row — a garage key, an impound key, or `nil`. |
| `IsVehicleInImpound` | `plate: string` | `boolean` | Whether an `nexus_impound_vehicles` row exists for that plate. |
| `GetVehicleImpoundData` | `plate: string` | `table \| nil` | `{ impoundId, impound, reason, fee, by, at }` or `nil`. |
| `GetGarageLimit` | `key: string` | `number` | The garage's configured capacity (`0` = unlimited). |
| `GetGarageCount` | `key: string` | `number` | How many vehicles are currently stored in that garage. |
| `GetKeysData` | `identifier: string \| number` | `table` | `{ received = {...}, given = {...} }` — every key that identifier holds and every key they've handed out, each entry with `plate`, `kind`, owner/holder identifier and name, the serialised `vehicle`, an optional `restricted` garage label and an optional `expires` timestamp. |
| `DoesPlayerHaveKeys` | `src: number`, `plate: string` | `boolean` | Full "can this player use this vehicle" check: owner, permanent key, live temporary key, or holder of a service vehicle. |
| `GetConfig` | — | `table` | `{ config = Config, settings = <live settings> }`. Read-only snapshot; mutating it does not change the running config. |
| `SendVehicleToImpound` | `src: number`, `plate: string`, `impoundId: number`, `reason: string`, `fee: number`, `bypassJob?: boolean` | `boolean, string?` | Impounds a plate. With `bypassJob = true` the job check and the maximum-fine cap are skipped (admin/script mode). Deletes the entity if it's out, notifies the owner, writes the audit row and fires `Functions.OnVehicleImpounded`. Returns `false` plus an error code on failure. |
| `GiveVehicleKey` | `ownerIdentifier: string`, `holder: string \| number`, `plate: string`, `kind?: string` | `boolean` | Gives a key. `kind = 'permanent'` writes to `nexus_keys` and respects `maxPermanentKeys` (returning `false, 'key_limit'`); anything else gives a temporary key. `holder` may be a server id. |
| `RemoveVehicleKey` | `holder: string \| number`, `plate: string`, `kind?: string` | `true` | Removes a key. `kind = 'permanent'` or `'temporary'` restricts which; omit it to remove both. |
| `AddGarageMember` | `key: string`, `identifier: string`, `addedBy?: string` | `boolean` | Adds a member to a database-backed garage (`INSERT IGNORE`) and re-broadcasts the world to every client. `false` for an unknown or dynamic garage. |
| `RemoveGarageMember` | `key: string`, `identifier: string` | `boolean` | Removes a member and re-broadcasts. |
| `TransferVehicle` | `plate: string`, `toKey: string` | `boolean` | Moves a stored vehicle to another garage **without charging a fee or checking capacity or class**. Script-level move, not the player-facing transfer. |
| `SetVehicleGarageState` | `plate: string`, `garageKey: string\|nil`, `stored: boolean` | — | Low-level setter for the `garage` and stored columns. Also updates the `parked_at` / `last_garage` metadata. |
| `CreatePrivateGarage` | `ownerIdentifier: string`, `name: string`, `interactionCoords: table`, `spawns: table[]`, `vclass?: string` | `string\|nil, string?` | Creates a **persistent** private garage in the database (marker interaction, no blip), owned by that identifier, and broadcasts it to every client. Returns the new garage key (`ng_<id>`) or `nil` plus an error code. This is the intended hook for housing scripts. |

```lua
-- Register a temporary garage for a rented warehouse, from your own resource
exports['nexus_garage']:RegisterGarage('warehouse_42', {
    label        = 'Warehouse 42',
    type         = 'private',
    vehicleClass = 'car',
    capacity     = 8,
    owner        = 'char1:3f9a1c2b7d',
    coords       = { x = 1037.1, y = -2398.4, z = 29.3, w = 90.0 },
    spawns       = {
        { x = 1041.2, y = -2402.0, z = 29.3, w = 180.0 },
        { x = 1045.8, y = -2402.0, z = 29.3, w = 180.0 },
    },
})

-- Access list instead of a single owner
exports['nexus_garage']:RegisterGarage('crew_lockup', {
    label  = 'Crew Lockup',
    coords = { x = 480.0, y = -1310.0, z = 29.2, w = 0.0 },
    spawns = { { x = 486.0, y = -1316.0, z = 29.2, w = 270.0 } },
    access = { 'char1:aaa', 'char1:bbb', 'char1:ccc' },
})

-- Impound a vehicle from your own speed-camera script, bypassing job checks
exports['nexus_garage']:SendVehicleToImpound(nil, 'APEXRS', 1, 'Unpaid fines', 2500, true)
```

### Events — Emitted

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_garage:server:ready` | Server → server (local) | — | Once, after the SQL install file has run, the native columns are patched and settings, garages, impounds and keys are loaded. **Wait for this before calling any server export at startup.** |
| `nexus_garage:server:privateGarageCreated` | Server → server (local) | `ownerIdentifier: string`, `key: string\|nil` | After a garage is created through the `nexus_garage:server:createPrivateGarage` event. |
| `nexus_garage:client:sync` | Server → client | `world: table`, `identifier: string` | Full world refresh. `world = { garages = {...}, impounds = {...} }` with every enabled location that has an interaction point (garage records have `jobVehicles` stripped out). Sent 1.5 s after the player loads, on explicit request, and broadcast to everyone whenever a garage, impound or membership changes. |
| `nexus_garage:client:keys` | Server → client | `plates: string[]` | The complete list of plate keys the player can currently use (owned + permanent + live temporary + held service vehicles). Replaces the client cache wholesale. Sent on load, after any key change, and after a vehicle spawns. |
| `nexus_garage:client:notify` | Server → client | `message: string`, `kind: string` | A server-side notification, already translated. `kind` is `info`, `success`, `warning` or `error`. |
| `nexus_garage:client:lockFx` | Server → all clients | `netId: number`, `locked: boolean` | Broadcast so every nearby client plays the remote-control sound, flashes the lights and applies the door-lock state. Ignored beyond 40 m. |
| `nexus_garage:client:openAdmin` | Server → client | — | Sent after the server has verified the ACE, telling the client to open the admin panel. |
| `nexus_garage:client:callback` | Server → client | `requestId: number`, `result: table` | Internal transport for the promise-based callback system. Not part of the public API. |

### Events — Listened

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `nexus_garage:server:createPrivateGarage` | Server | `ownerIdentifier: string`, `name: string`, `interactionCoords: table`, `spawns: table[]`, `vclass?: string`, `cb?: function` | Event wrapper around the `CreatePrivateGarage` export, for resources that prefer events to exports. Calls `cb(key)` if given, then emits `nexus_garage:server:privateGarageCreated`. |
| `nexus_garage:server:transferVehicle` | Server | `plate: string`, `toKey: string` | Moves a stored vehicle to another garage, free of charge. |
| `nexus_garage:server:setGarageState` | Server | `plate: string`, `garageKey: string\|nil`, `stored: boolean` | Low-level stored-state setter. |
| `nexus_garage:server:impoundVehicleDirect` | Server (net) | `netId: number`, `impoundId: number`, `reason: string`, `fee: number` | Net event a client can fire to impound a vehicle by network id, going through the normal job and maximum-fine checks. Replies to the caller with a success or error notification. |
| `nexus_garage:server:toggleLock` | Server (net) | `netId: number` | Toggles the door lock of a vehicle, validating the caller's distance (`lockDistance + 5`) and key. |
| `nexus_garage:server:mileage` | Server (net) | `plate: string`, `km: number` | Accepts an odometer increment. Rejects anything ≤ 0 or > 50, and anything from a player with no key for that plate. |
| `nexus_garage:server:requestSync` | Server (net) | — | Asks for a full world + key resync for the calling player. |
| `nexus_garage:server:openAdmin` | Server (net) | — | Requests the admin panel; the ACE is checked here. |
| `nexus_garage:client:openGarage` | Client (net) | `key: string`, `spawn?: table` | Event equivalent of the `OpenGarage` export. |
| `nexus_garage:client:storeVehicle` | Client (net) | `key: string` | Event equivalent of `StoreVehicle`. |
| `nexus_garage:client:showKeys` | Client (net) | — | Event equivalent of `OpenKeysMenu`. |
| `nexus_garage:client:impoundVehicle` | Client (net) | — | Event equivalent of `OpenImpoundMenu`. |
| `nexus_garage:client:impoundVehicleDirect` | Client (net) | `vehicle?: number`, `impoundId: number`, `reason: string`, `fee: number` | Impounds a specific vehicle entity. If `vehicle` is nil or invalid, falls back to the vehicle the player is in, then to the nearest within 8 m. |
| `nexus_garage:client:toggleVehicleLock` | Client (net) | `plate: string` | Toggles the lock of the vehicle with that plate (or the one the player is in). |
| `nexus_garage:client:setVehicleLocked` | Client (net) | `plate: string` | Force-locks that vehicle locally. |
| `nexus_garage:client:setVehicleUnlocked` | Client (net) | `plate: string` | Force-unlocks that vehicle locally. |

Framework events the script subscribes to, for awareness: `esx:playerLoaded`, `esx:setJob` (client and server), `QBCore:Server:PlayerLoaded`, `QBCore:Client:OnPlayerLoaded`, `QBCore:Client:OnJobUpdate`, `QBCore:Client:OnGangUpdate`, plus the natives `playerDropped`, `entityRemoved` and `onResourceStop`.

### Editable Functions (`functions.lua`)

`functions.lua` ships **outside the escrow**. It is a single file split by `IsDuplicityVersion()`, so the server half and the client half are separate function sets. Every hook is already defined as an empty function, so you can fill in only what you need. Nothing here is required for the script to work.

#### Server side

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `Functions.AddSocietyMoney` | `(society: string, amount: number)` | `impound:pay` callback, after the fine is taken from the player | Credits the impound's society. Already implemented for five banking resources plus an `auto` probe; the `custom` branch is where you add your own. |
| `Functions.RemoveMoney` | `(source: number, amount: number, reason: string) -> handled: boolean, result: boolean` | `Nexus.RemoveMoney`, before any built-in payment logic | Override **every** charge the script makes (parking, transfers, recovery, garage purchase, rent, impound fines). Return `true, true` to say "I took the money", `true, false` to say "I refused the payment", or `false` (the shipped default) to let the script handle it with `Config.Payment`. Reasons passed: `garage-parking`, `garage-transfer`, `garage-recover`, `garage-purchase`, `garage-rent`, `impound-release`. |
| `Functions.OnVehicleStored` | `(source, plate, garageKey)` | End of the `garage:store` callback, after the audit row | Post-store hook. |
| `Functions.OnVehicleTaken` | `(source, plate, garageKey)` | End of the `garage:take` callback | Post-withdrawal hook. |
| `Functions.OnVehicleTransferred` | `(source, plate, fromKey, toKey, fee)` | End of the `garage:transfer` callback | Transfer hook, with the fee actually charged. |
| `Functions.OnVehicleRecovered` | `(source, plate, garageKey, fee, status)` | End of the `garage:recover` callback | `status` is `'lost'` or `'destroyed'`. |
| `Functions.OnVehicleImpounded` | `(source, plate, impoundId, reason, fee)` | End of `Nexus.Impounds.Send`, after the audit row | Fired for player impounds, admin impounds and the `SendVehicleToImpound` export alike. `source` may be `nil` for script-driven impounds. |
| `Functions.OnVehicleReleased` | `(source, plate, impoundId, fee)` | End of the `impound:pay` callback | The owner paid and got the vehicle back. |
| `Functions.OnKeyGiven` | `(source, target, plate, kind)` | End of the `keys:give` callback | `target` is a server id, `kind` is `'permanent'` or `'temporary'`. |
| `Functions.OnKeyRemoved` | `(source, holderIdentifier, plate)` | End of the `keys:revoke` callback | A key was revoked by the owner. |
| `Functions.OnGarageCreated` | `(garageId: number, data: table)` | End of `Nexus.Garages.Create` | `data` is the sanitised garage record that was inserted. |
| `Functions.OnGarageDeleted` | `(garageId, movedVehicles: number, movedTo: string\|nil)` | End of `Nexus.Garages.Delete` | How many vehicles were relocated and to which garage name (`nil` if they were left in the "lost" state). |
| `Functions.OnGaragePurchased` | `(source, garageId, price)` | End of the `garage:buy` callback | A player bought a garage. |
| `Functions.OnImpoundCreated` | `(impoundId: number, data: table)` | End of `Nexus.Impounds.Create` | `data` is the sanitised impound record. |

#### Client side

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `Functions.Notify` | `(message: string, kind: string)` | `Nexus.Notify` when `Config.Systems.Notify == 'custom'` | Route notifications to your own resource. Return anything truthy — if the call errors, the script falls back to the native GTA ticker. |
| `Functions.ShowTextUI` | `(message: string)` | `Nexus.ShowText` when `Config.Systems.TextUI == 'custom'` | Show your own prompt. |
| `Functions.HideTextUI` | `()` | `Nexus.HideText` when TextUI is `custom` | Hide it again. |
| `Functions.AddTarget` | `(id: string, coords: table, radius: number, options: table[]) -> handle` | `Nexus.AddTargetZone` when `Config.Systems.Target == 'custom'` | Create a target zone. Each option has `icon`, `label`, `action` and `canInteract`. Return a handle; if you return nothing, `id` is used as the handle. |
| `Functions.RemoveTarget` | `(handle)` | `Nexus.RemoveTargetZone` when Target is `custom` | Destroy the zone. |
| `Functions.GiveKeys` | `(vehicle: number, plate: string)` | `Nexus.GiveExternalKeys` when `Config.Systems.Keys == 'custom'` | Hand keys to your own key resource when a vehicle spawns. |
| `Functions.RemoveKeys` | `(vehicle: number, plate: string)` | `Nexus.RemoveExternalKeys` when Keys is `custom` | Take them back when it's stored. |
| `Functions.GetFuel` | `(vehicle: number) -> number` | `Nexus.GetFuel` when `Config.Systems.Fuel == 'custom'` | Read fuel. Ships returning `GetVehicleFuelLevel(vehicle)`. |
| `Functions.SetFuel` | `(vehicle: number, fuel: number)` | `Nexus.SetFuel` when Fuel is `custom` | Write fuel. Ships calling `SetVehicleFuelLevel`. |
| `Functions.VehicleImage` | `(model: string\|number) -> string\|nil` | `Nexus.ModelInfo`, for every vehicle sent to the UI | **Return a URL to use your own vehicle thumbnails.** Receives the spawn name when known, otherwise the model hash. Returning `nil` (the default) makes the UI fall back to `https://docs.fivem.net/vehicles/<spawn>.webp`, and then to a drawn silhouette. This is the hook to point at a local image pack and stop depending on the internet. |
| `Functions.OnVehicleSpawned` | `(vehicle: number, plate: string, garageKey: string)` | End of `Nexus.SpawnVehicle`, after keys are given and the player is seated | Apply your own extras (tuning, decals, statebags, blip). |
| `Functions.OnVehicleStored` | `(plate: string, garageKey: string)` | `Nexus.StoreCurrentVehicle`, after the server confirms | Client-side post-store hook. Note the client signature has no `source`, unlike the server hook of the same name. |
| `Functions.CanOpenGarage` | `(garageKey: string) -> boolean` | Start of `Nexus.OpenGarage` | Return `false` to block opening a garage (handcuffed, dead, wanted, inside a minigame…). Ships returning `true`. The script shows no message, so notify the player yourself. |
| `Functions.CanImpound` | `() -> boolean` | Start of `Nexus.OpenImpoundForm` | Return `false` to block the impound form. Ships returning `true`. |

### Database Schema

`sql/install.sql` is executed from inside the resource at every boot and ships outside the escrow. All eight statements are `CREATE TABLE IF NOT EXISTS`, so importing it manually first is optional and running it twice is harmless.

```sql
-- Garages. One row per garage; `key` in the API is always 'ng_' .. id.
CREATE TABLE IF NOT EXISTS `nexus_garages` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(60) NOT NULL,
    `type` VARCHAR(16) NOT NULL DEFAULT 'public',       -- public|private|shared|job|gang
    `vehicle_class` VARCHAR(8) NOT NULL DEFAULT 'car',  -- car|air|sea
    `job` VARCHAR(50) DEFAULT NULL,                     -- job or gang name (job/gang types)
    `min_grade` INT NOT NULL DEFAULT 0,
    `owner_identifier` VARCHAR(64) DEFAULT NULL,        -- ESX identifier or QB citizenid
    `price` INT UNSIGNED NOT NULL DEFAULT 0,            -- 0 = not for sale
    `capacity` INT UNSIGNED NOT NULL DEFAULT 0,         -- 0 = unlimited
    `capacity_mode` VARCHAR(8) NOT NULL DEFAULT 'total',-- total|player
    `daily_cost` INT UNSIGNED NOT NULL DEFAULT 0,       -- parking (public) or maintenance (owned)
    `rent_paid_until` BIGINT DEFAULT NULL,              -- unix seconds
    `enabled` TINYINT(1) NOT NULL DEFAULT 1,
    `blip` LONGTEXT NOT NULL,                           -- JSON, see Configuration
    `interaction` LONGTEXT NOT NULL,                    -- JSON, see Configuration
    `spawns` LONGTEXT NOT NULL,                         -- JSON array, max 20
    `job_vehicles` LONGTEXT DEFAULT NULL,               -- JSON array (job/gang types)
    `job_cooldown` INT UNSIGNED NOT NULL DEFAULT 0,     -- seconds, 0..86400
    `save_damage` TINYINT(1) DEFAULT NULL,              -- NULL = inherit global setting
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Members of shared garages.
CREATE TABLE IF NOT EXISTS `nexus_garage_members` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `garage_id` INT UNSIGNED NOT NULL,
    `identifier` VARCHAR(64) NOT NULL,
    `added_by` VARCHAR(64) DEFAULT NULL,
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `garage_member` (`garage_id`, `identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Permanent vehicle keys. Temporary keys are memory-only and never land here.
CREATE TABLE IF NOT EXISTS `nexus_keys` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `plate` VARCHAR(12) NOT NULL,
    `owner_identifier` VARCHAR(64) NOT NULL,
    `holder_identifier` VARCHAR(64) NOT NULL,
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `plate_holder` (`plate`, `holder_identifier`),
    KEY `idx_keys_holder` (`holder_identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Impound locations. `key` in the API is always 'impound_' .. id.
CREATE TABLE IF NOT EXISTS `nexus_impounds` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(60) NOT NULL,
    `jobs` LONGTEXT NOT NULL,                           -- JSON array of job names, min 1
    `society` VARCHAR(50) NOT NULL,                     -- receives the fines
    `max_fee` INT UNSIGNED NOT NULL DEFAULT 0,
    `vehicle_class` VARCHAR(8) NOT NULL DEFAULT 'car',
    `enabled` TINYINT(1) NOT NULL DEFAULT 1,
    `blip` LONGTEXT NOT NULL,
    `interaction` LONGTEXT NOT NULL,
    `spawns` LONGTEXT NOT NULL,
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- One row per impounded vehicle. The plate is unique, so a vehicle is only
-- ever in one impound; re-impounding overwrites reason, fee, officer and date.
CREATE TABLE IF NOT EXISTS `nexus_impound_vehicles` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `plate` VARCHAR(12) NOT NULL,
    `impound_id` INT UNSIGNED NOT NULL,
    `reason` VARCHAR(255) NOT NULL,
    `fee` INT UNSIGNED NOT NULL DEFAULT 0,
    `impounded_by` VARCHAR(64) DEFAULT NULL,
    `impounded_by_name` VARCHAR(100) DEFAULT NULL,
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `impound_plate` (`plate`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Per-vehicle extras that don't belong in the framework's own table.
CREATE TABLE IF NOT EXISTS `nexus_vehicle_meta` (
    `plate` VARCHAR(12) NOT NULL,
    `nickname` VARCHAR(40) DEFAULT NULL,
    `favorite` TINYINT(1) NOT NULL DEFAULT 0,
    `vehicle_class` VARCHAR(8) DEFAULT NULL,            -- car|air|sea, detected on store
    `parked_at` BIGINT DEFAULT NULL,                    -- unix seconds, drives parking fees
    `last_garage` VARCHAR(40) DEFAULT NULL,
    PRIMARY KEY (`plate`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Single-row table (id is always 1) holding the live settings as JSON.
CREATE TABLE IF NOT EXISTS `nexus_settings` (
    `id` TINYINT UNSIGNED NOT NULL DEFAULT 1,
    `data` LONGTEXT NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Audit log. See the event list below.
CREATE TABLE IF NOT EXISTS `nexus_audit_log` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `event` VARCHAR(40) NOT NULL,
    `actor_identifier` VARCHAR(64) DEFAULT NULL,
    `actor_name` VARCHAR(100) DEFAULT NULL,
    `plate` VARCHAR(12) DEFAULT NULL,
    `data` LONGTEXT DEFAULT NULL,                       -- JSON payload, varies per event
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `idx_audit_event` (`event`),
    KEY `idx_audit_plate` (`plate`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### Columns added to the framework's own table

These are **not** in `install.sql`. They are applied at boot by `server/db.lua`, which checks `information_schema` first and only runs the `ALTER` if the column is missing. The equivalent manual SQL is:

```sql
-- ESX
ALTER TABLE `owned_vehicles` ADD COLUMN `garage`  VARCHAR(60) DEFAULT NULL;
ALTER TABLE `owned_vehicles` ADD COLUMN `mileage` FLOAT NOT NULL DEFAULT 0;
ALTER TABLE `owned_vehicles` ADD COLUMN `stored`  TINYINT(1) NOT NULL DEFAULT 1;
ALTER TABLE `owned_vehicles` ADD COLUMN `type`    VARCHAR(20) NOT NULL DEFAULT 'car';

-- QBCore (only the first two; `state` and `garage` already exist in qb-core)
ALTER TABLE `player_vehicles` ADD COLUMN `garage`  VARCHAR(60) DEFAULT NULL;
ALTER TABLE `player_vehicles` ADD COLUMN `mileage` FLOAT NOT NULL DEFAULT 0;
```

#### How the stored state is encoded

| Framework | Table | Owner column | Stored column | Stored | Out | Impounded |
|---|---|---|---|---|---|---|
| ESX | `owned_vehicles` | `owner` | `stored` | `1` | `0` | `0` + a row in `nexus_impound_vehicles` |
| QBCore | `player_vehicles` | `citizenid` | `state` | `1` | `0` | `2` + a row in `nexus_impound_vehicles` |

The `garage` column holds the garage key (`ng_<id>`) while stored, the impound key (`impound_<id>`) while impounded, and the last known garage while out. Plates are matched with `plate IN (?, ?)` against both the trimmed and the 8-character space-padded form, so the script is tolerant of whichever convention your existing data uses.

#### Audit log events

`vehicle_taken`, `vehicle_stored`, `vehicle_transferred`, `vehicle_recovered`, `impound_sent`, `impound_released`, `impound_npc_removed`, `key_given`, `key_revoked`, `key_returned`, `garage_created`, `garage_updated`, `garage_deleted`, `garage_duplicated`, `garage_toggled`, `garage_purchased`, `garage_rent_paid`, `garage_owner_set`, `member_added`, `member_removed`, `impound_created`, `impound_updated`, `impound_deleted`, `settings_saved`, `settings_reset`, `admin_vehicle_assigned`, `admin_vehicle_deleted`, `admin_vehicle_moved`, `admin_vehicle_owner`, `job_vehicle_taken`, `job_vehicle_returned`.

### Integration Example

A complete, working housing resource integration: when a player buys a house, create a persistent private garage for them; when they sell it, delete the garage; mirror every garage event into a Discord webhook; and gate garage access on the player not being handcuffed.

**`my_housing/server.lua`**

```lua
local ready = false
AddEventHandler('nexus_garage:server:ready', function() ready = true end)

-- Never call nexus_garage exports before it says it's ready
CreateThread(function()
    while not ready do Wait(250) end
    print('[my_housing] nexus_garage is up')
end)

RegisterNetEvent('my_housing:houseBought', function(houseId)
    local src = source
    local identifier = <your identifier lookup>(src)
    local house = Houses[houseId]
    if not house.garage then return end

    local key, err = exports['nexus_garage']:CreatePrivateGarage(
        identifier,
        house.label .. ' Garage',
        house.garage.entrance,            -- { x, y, z, w }
        house.garage.spawns,              -- { { x, y, z, w }, ... }
        'car'
    )
    if not key then
        return print(('[my_housing] garage creation failed: %s'):format(err or 'unknown'))
    end

    -- Persist the key so you can find the garage again later
    MySQL.update('UPDATE houses SET garage_key = ? WHERE id = ?', { key, houseId })
end)

RegisterNetEvent('my_housing:houseSold', function(houseId)
    local key = MySQL.scalar.await('SELECT garage_key FROM houses WHERE id = ?', { houseId })
    if not key then return end
    -- Hand every vehicle inside back to the nearest public garage first
    for _, v in ipairs(exports['nexus_garage']:GetAllGarages()) do
        if v.key == key then
            local fallback = exports['nexus_garage']:GetFirstGarageIdForVehicleType(v.vehicleClass)
            -- (optional) move the vehicles yourself before dropping the garage
        end
    end
    MySQL.query('DELETE FROM nexus_garages WHERE id = ?', { tonumber(key:match('%d+')) })
end)

-- Read-only: show a house's garage occupancy on your property UI
exports('HouseGarageUsage', function(houseId)
    local key = MySQL.scalar.await('SELECT garage_key FROM houses WHERE id = ?', { houseId })
    if not key then return 0, 0 end
    return exports['nexus_garage']:GetGarageCount(key), exports['nexus_garage']:GetGarageLimit(key)
end)
```

**`my_housing/webhook.lua`** — hook the server-side `functions.lua` events instead of patching protected code. Add this to `nexus_garage/functions.lua` (it ships unencrypted):

```lua
local WEBHOOK = 'https://discord.com/api/webhooks/...'

local function log(title, description)
    PerformHttpRequest(WEBHOOK, function() end, 'POST', json.encode({
        embeds = { { title = title, description = description, color = 9639146 } },
    }), { ['Content-Type'] = 'application/json' })
end

function Functions.OnVehicleImpounded(source, plate, impoundId, reason, fee)
    log('Vehicle impounded', ('**%s** → impound #%d\nReason: %s\nFine: $%d'):format(plate, impoundId, reason, fee))
end

function Functions.OnGaragePurchased(source, garageId, price)
    log('Garage purchased', ('%s bought garage #%d for $%d'):format(GetPlayerName(source), garageId, price))
end

function Functions.OnVehicleTransferred(source, plate, fromKey, toKey, fee)
    log('Vehicle transferred', ('**%s**: %s → %s ($%d)'):format(
        plate,
        exports['nexus_garage']:GetGarageLabel(fromKey) or '—',
        exports['nexus_garage']:GetGarageLabel(toKey) or '—',
        fee))
end
```

**`nexus_garage/functions.lua`** (client half) — block garage use while handcuffed and point vehicle images at your own image pack:

```lua
function Functions.CanOpenGarage(garageKey)
    if LocalPlayer.state.isCuffed then
        exports['nexus_notify']:Alert('Garage', 'Not with those cuffs on.', 5000, 'error', true)
        return false
    end
    return true
end

function Functions.VehicleImage(model)
    -- model is the spawn name when known, otherwise the model hash
    if type(model) ~= 'string' then return nil end
    return ('nui://my_vehicle_images/images/%s.png'):format(model)
end
```

**Reading key data from your own locksmith job**

```lua
-- Server side: list every key a player holds and every one they've handed out
local data = exports['nexus_garage']:GetKeysData(source)
for _, k in ipairs(data.received) do
    print(('%s key for %s, from %s'):format(k.kind, k.plate, k.ownerName))
end

-- Give a permanent key as part of a vehicle sale
local ok = exports['nexus_garage']:GiveVehicleKey(sellerIdentifier, buyerSource, plate, 'permanent')
if not ok then
    -- maxPermanentKeys reached for this plate
end
```

---

## ❓ FAQ

**The resource doesn't start and the console says the folder must be called `nexus_garage`.**
It is validated twice, in `shared/_resource.lua` and in `shared/framework.lua`. Rename the folder to exactly `nexus_garage` — no version suffix, no capitals, no `-main`. This is also why you can't run two copies side by side.

**The console says ESX/QBCore wasn't detected.**
`Config.Framework = 'auto'` looks for the resources `es_extended` and `qb-core` by those exact names. If your framework folder is renamed, set `Config.Framework` to `'esx'` or `'qbcore'` explicitly. Also make sure the framework is `ensure`d *before* `nexus_garage` in `server.cfg`.

**The console says the `owned_vehicles` / `player_vehicles` table doesn't exist.**
`nexus_garage` started before the framework had created its own tables. Fix the `server.cfg` order (`oxmysql` → framework → `nexus_garage`) and restart. Nothing is lost — the column patch simply runs on the next boot.

**I get MySQL errors on startup, or nothing loads.**
This resource requires **oxmysql**, not mysql-async. It loads `@oxmysql/lib/MySQL.lua` and uses the `MySQL.*.await` API. The mysql-async compatibility layer that other scripts rely on is not enough here.

**`/garageadmin` does nothing.**
The command only asks the server; the server checks `IsPlayerAceAllowed(src, Config.AdminAce)` and replies with a "no permission" notification if you fail. Add the ACE: `add_ace group.admin nexus.garage.admin allow`, and make sure your identifier is actually in that group. ACE changes need a server restart or a `refresh`/`restart` of the resource that owns them.

**`/llaves` does nothing and the `L` key doesn't work.**
Both are gated on `Config.Systems.Keys == 'nexus'`. If you've pointed the script at qb-vehiclekeys, qs-vehiclekeys, cd_garage or any other key resource, the built-in keyring, the key mapping and the engine immobiliser are all disabled on purpose, because that resource owns keys now.

**`/impound` says my job can't use any impound.**
An impound is only usable by the jobs listed on it. Open `/garageadmin` → **Impounds** → edit the impound → *Jobs that can impound vehicles* and add yours. Remember the player's **current active job** is what's checked, not a job they merely have.

**The fine I set is rejected with "the fine exceeds the allowed maximum".**
Every impound has its own `maxFee`. The officer's slider is capped at it, but the server re-validates, so a tampered request still fails. Administrators using *Send to impound* from the player manager bypass both the job check and the maximum.

**My new garage doesn't appear in the world.**
Three things must all be true: the garage is **enabled**, it has an **interaction point**, and you must be allowed to *see* it. Private and shared garages are only visible to their owner, their members, or — if they have no owner and a price above zero — to everyone as a for-sale garage. Job and gang garages are only visible to players with that exact job or gang. Blips additionally need `blip.enabled`.

**I created the garage but vehicles won't come out — "this garage has no spawn points".**
The editor won't let you save without at least one spawn point, but a *duplicated* garage is deliberately created with no spawn points and disabled, so you have to place them before it works. Also check "all spawn points are blocked": a spawn is considered occupied if any vehicle is within 2.8 m or any non-player ped is within 1.2 m — place several points.

**Vehicles in a boat or helicopter garage are missing from the list.**
Each garage is locked to one vehicle class, and the list filters by it. The class comes from `nexus_vehicle_meta.vehicle_class`, which is written when the vehicle is stored (or assigned by an admin). A vehicle that has never passed through the script yet has no class recorded; on ESX it falls back to the `type` column (`boat` → sea, `airplane`/`helicopter` → air). Store it once from the right garage and it will be classified correctly from then on.

**A player's car shows as "lost" after a server restart.**
That's by design, and it's the safety net. The script tracks out-in-the-world vehicles in memory; after a restart that memory is gone, so a vehicle that was out is no longer findable and is reported as *lost*. The owner recovers it at its own garage or any public garage for `lostRecoveryFee`. If you'd rather it were free, set that setting to `0` in the admin panel.

**"Destroyed" vehicles come back repaired — can I stop that?**
A destroyed recovery intentionally resets body, engine and tank health to 1000, tops fuel up to at least 50 and clears dirt, broken windows, broken doors and burst tyres. That's what the `destroyedRecoveryFee` buys. If you want wrecks to stay wrecked, handle it yourself in `Functions.OnVehicleRecovered` or set the fee high enough that nobody uses it.

**Nothing happens when I change `Config.Defaults` and restart.**
Those 23 values are only the *seed*. After the first boot the live values come from the `nexus_settings` table and are edited in the admin panel. To force your `config.lua` values back, use **Settings → Reset** in the panel (or delete the single row in `nexus_settings`).

**The admin panel's settings say "applies instantly" — is that true for everything?**
Yes for all 23 settings: they're saved to the database, mirrored into `GlobalState.nexusGarageSettings` and read live by both the client and the server on every use. `Config.Systems`, `Config.Commands`, the ACE name, the key mapping and the map bounds are **not** settings and do need a restart.

**Vehicle images, blip icons and marker previews are blank or show placeholders.**
They are fetched from `docs.fivem.net` and `docs-backend.fivem.net`, so the game client needs internet access, and addon vehicles will never have an image there. The UI degrades gracefully (a drawn silhouette for vehicles, a coloured badge with the sprite ID for blips). To use your own images, implement `Functions.VehicleImage(model)` in `functions.lua` and return a `nui://` URL to your own image pack.

**The map in the UI says the map image is missing.**
Check that `html/assets/map.png` exists and that `Config.Map.image` matches its path relative to `html/`. The file ships with the resource and is large; make sure your file transfer didn't skip or truncate it. If you replace the image, update `Config.Map.minX/maxX/minY/maxY` to its real world bounds or every pin will be misplaced.

**Mileage doesn't go up.**
Distance is only accumulated when you are the **driver** and `HasKey(plate)` is true for that vehicle, and the per-report cap is 50 km. Service vehicles and vehicles you hold no key for don't accumulate. With a non-`nexus` key system the local key cache is still filled on spawn, so your own vehicles do count.

**Gang garages don't work on my ESX server.**
With `Config.Systems.Gangs = 'qb'` (the default) gang resolution reads QBCore's `PlayerData.gang`, which doesn't exist on ESX, so nobody ever matches. Either use `rcore_gangs` — which works on both frameworks — or model your gangs as jobs and use job garages.

**I deleted a garage that had cars in it. Where did they go?**
They were moved to the nearest enabled **public** garage of the same vehicle class, and the panel tells you how many and where. If no such garage exists, they are set to the "lost" state with no garage, and their owners can recover them at any public garage for `lostRecoveryFee`. Deleting an impound behaves similarly: its vehicles move to another impound keeping their fines, or — if it was the last impound — back to the nearest public garage with the fine dropped.

**Two players clicking at once, or someone spamming a button, duplicated something.**
Every money-moving callback (take, store, transfer, recover, buy, pay rent, pay fine) takes a per-player lock and returns `busy` while another is in flight, and the server re-validates ownership, access, capacity, class and stored state on every call. Nothing in the UI is trusted. If you see a "wait a moment, an action is in progress" notification, that's the lock working.

**A vehicle got stuck as "out" even though it isn't in the world.**
Open `/garageadmin` → **Players**, find the owner, and use **Move to a garage** on that vehicle. It deletes any lingering entity, clears the impound record and stores it where you choose.

**Can a player store someone else's car?**
Only in a shared garage they have access to, and only if they received the restricted temporary key the garage handed them when they took it out — in which case they can *only* return it to that same garage. Everywhere else, storing a vehicle you don't own is refused.

### Before opening a ticket

- Make sure the resource folder is named **exactly** `nexus_garage`.
- Make sure you are on the **latest version** of the resource.
- Make sure **oxmysql** is running and is `ensure`d before your framework and before `nexus_garage`.
- Check your `server.cfg` order: `oxmysql` → `es_extended` / `qb-core` → optional notify/TextUI/target/fuel/banking resources → `nexus_garage`.
- Set `Config.Debug = true`, reproduce the problem, and include the **full** server and client (F8) console output.
- Re-read this FAQ page — most reports are an ACE permission, a resource order, or a `Config.Systems` value pointing at a resource that isn't running.

---

## 📋 Changelog

### 1.0.0 — Initial release

Version as declared in `fxmanifest.lua` (`version '1.0.0'`).

- Database-driven garages with five types (public, private, shared, job, gang) and three vehicle classes (land, air, sea).
- In-game admin panel: garage and impound editors, in-world point capture, blip and marker designers, player/vehicle manager, 23 live settings and a searchable audit log.
- Impound system with per-impound job list, maximum fine, receiving society and officer impound form.
- Built-in key system with permanent and temporary keys, keyring UI, key-fob lock/unlock and engine immobiliser, plus bridges to seven third-party key resources.
- Garage-to-garage transfers with flat and per-kilometre fees, lost/destroyed vehicle recovery, daily parking and garage maintenance rent.
- Garage purchase in-game, shared-garage membership management, service-vehicle fleets with grade restrictions and cooldowns.
- Mileage tracking, favourites, nicknames, search, and list/grid views.
- Custom React NUI with an interactive Los Santos map.
- ESX and QBCore support with automatic framework detection, automatic schema creation and automatic patching of the native vehicle table.
- Seven languages (es, en, de, fr, it, pt, zh) and a public developer API of 24 server exports, 10 client exports, 8 emitted events, 16 listened events and 28 editable hook functions (14 server, 14 client).
