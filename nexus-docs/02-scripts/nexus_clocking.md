# Nexus Clocking

A multi-job time-clock (shift tracking) system built around a physical clock-in terminal NUI, with per-job NPC terminals created live from an in-game admin panel, a boss panel with per-employee analytics, and a full session history stored in MySQL.

## 📝 Description

Nexus Clocking replaces the usual "/duty" command with a physical object in the world. Every job on the server gets its own clock-in point: an NPC (with a looping clipboard animation and an attached prop) that spawns and despawns dynamically as players approach, plus an optional map blip. Interacting with the NPC opens a custom NUI that renders a hardware-style clocking terminal — a bezel screen, a 17-key keypad, a fingerprint scanner and a status LED — not a generic list menu.

The flow is deliberately diegetic. The terminal starts locked; the player scans their fingerprint on the scanner to unlock it, after which the screen exposes four views navigated with the `MENU` key: the clocking screen (clock in/out, live shift timer, week/total/efficiency footer), the personal info screen (avatar, job, grade, location, first shift, average per shift, weekly efficiency), the history screen (paginated shift cards with in/out times and durations) and the statistics screen (total hours, efficiency, days worked, overtime, plus a per-day / per-week SVG line chart with a selectable period of today / 7 days / 30 days / 6 months). An optional re-scan interval re-locks the terminal if the player reopens it after a configured number of seconds.

Server-side, every clock-in inserts a row into `nexus_clocking_sessions` with the identifier, player name, job, job label, grade and the clock-in unix timestamp; the clock-out stamps `clock_out` and the computed `duration` in seconds. Active sessions are also held in memory so the admin and boss panels can show live elapsed time. Sessions survive crashes and restarts: `playerDropped` force-closes the open session, and on resource start the script scans for rows with a `NULL` `clock_out` and closes them with the current time, so orphaned sessions never pollute statistics.

Two privileged panels sit on top of that. The **admin panel** ("Clocking Creator") is the content editor: it creates, edits and deletes clocking points (internal name, display name, NPC model, coordinates filled in from your current position, interaction and spawn distances, expected hours per week and per day, blip toggle/sprite/colour/scale, enabled flag), lists live sessions with force clock-in / force clock-out, shows global statistics grouped by job, and assigns or removes bosses. The **boss panel** is scoped to the jobs the player has been assigned as boss of: employees on duty right now, a 14-day record table, and an employee list where each row opens a modal with donut and line/bar charts for that employee's daily hours and weekly session counts.

## ✨ Features

- **Physical terminal NUI** — a rendered clock-in machine: bezel screen, status bar with a live system clock, contextual key-hint bar, 4×4 numeric keypad plus `MENU` / `OK` / `▲` / `▼` / `ESC`, a fingerprint scanner with a scan-line animation, a PWR LED and vent detailing.
- **Fingerprint unlock gate** — the terminal boots locked; `handleFingerprint()` runs a 2-second scan animation, flips the status LED to active and transitions into the clocking view.
- **Auto re-lock** — `Config.RescanInterval` seconds after the menu was last closed, the next open starts locked again and demands a new scan (0 disables it).
- **Four keypad-navigated screens** — Clocking, Info, History and Statistics, cycled with `MENU`, with per-screen key hints rendered into the status bar and animated horizontal slide transitions between them.
- **Clocking screen** — large live clock, `● ON DUTY` / `○ OFF DUTY` badge, a running `HH:MM:SS` shift timer with the start time once clocked in, a single clock-in/clock-out button (also bound to `OK`), and a footer with week hours, lifetime total and weekly efficiency percentage.
- **Info screen** — initials avatar, name, job label and grade, plus current street, date of first recorded shift, average duration per shift and weekly efficiency.
- **History screen** — the last 30 sessions for that identifier and job as cards (weekday, date, clock-in time, clock-out time or "en curso", duration badge), paginated 3 per page with `▲`/`▼` and a page indicator, with a summary header of total shifts, total time and daily average.
- **Statistics screen** — four stat boxes (total hours, efficiency with a progress track, days worked, overtime) and an SVG line chart; `▲`/`▼` (or `*`/`#`) switch between today / 7 days / 30 days / 6 months, and the chart switches between per-day and per-week aggregation depending on the period. The period switch updates in place rather than re-rendering the whole screen.
- **Period-aware efficiency** — expected hours are scaled per period (day → `expected_hours_day`, week → `expected_hours_week`, month → week × 30/7, 6 months → week × 180/7).
- **Overtime calculation** — computed in SQL as `SUM(GREATEST(duration - expected_hours_day, 0))` over the selected period, so it is per-session overtime, not a period total.
- **Dynamic NPC terminals** — one NPC per enabled job row, spawned in its own thread when the player comes within `spawn_distance` and deleted when they leave, invincible, frozen, non-reactive, playing `missheistdockssetup1clipboard@base` with a `prop_notepad_01` attached to the right hand; an animation watchdog thread re-applies the loop every 5 seconds if it drops.
- **Job-scoped visibility** — when a framework is configured, the client filters the job list down to the player's own job, so each player only ever sees (and only ever gets a blip for) their own clocking point.
- **Live job-change handling** — `esx:setJob` / `QBCore:Client:OnJobUpdate` re-apply the filter immediately and then re-request the list from the server; if the player's new job has no clocking point and they were clocked in, they are force-clocked-out automatically.
- **Three interaction modes** — `key` (a 3D floating prompt plus `E`), `ox_target` or `qb_target`, selected by `Config.InteractionMode`; the target registration happens at NPC spawn time.
- **Admin panel (Clocking Creator)** — four tabs: Jobs (CRUD table), Live (active sessions with elapsed time, auto-refreshed every 5 seconds while the tab is open, force clock-in and force clock-out), Statistics (sessions / total hours / average per session grouped by job) and Bosses (assign and remove).
- **"Use current position"** — the job modal pulls your coordinates and heading from the client and snaps Z to the ground with `GetGroundZFor_3dCoord`, so the NPC never floats.
- **Per-job expected hours** — `expected_hours_week` and `expected_hours_day` are stored per job row and override the global config defaults in all efficiency and overtime maths.
- **Cascade delete** — deleting a job force-closes every active session for it, notifies those players, then deletes its sessions, its boss assignments and the job row, behind a typed confirmation modal.
- **Boss panel** — scoped to the jobs the caller is a boss of: live cards for employees on duty, a 14-day / 200-row history table, and an employee list with total sessions, total hours, week hours, first and last clock-in and an online flag.
- **Employee modal** — per-employee donut charts and an SVG line chart of the last 7 days plus a bar chart of session counts over the last 4 weeks, with hover tooltips, and a "Delete Employee" action that wipes that identifier's history for the boss's jobs only.
- **ACE-based admin permission** — `IsPlayerAceAllowed(source, Config.AdminPermission)`; boss access is checked against the `nexus_clocking_bosses` table instead.
- **Lua-side ESC handling** — a dedicated thread watches control 200, because the DOM `Escape` key does not reliably reach the NUI in FiveM.
- **Crash-safe sessions** — `playerDropped` closes the open session with `UPDATE … WHERE id = ? AND clock_out IS NULL` (so a double event is a no-op), and a start-up sweep closes any session left open by a crash or restart.
- **Self-healing schema** — the server creates all three tables on `onResourceStart` and runs `ALTER TABLE … ADD COLUMN IF NOT EXISTS` migrations for `blip_enabled`, `expected_hours_week` and `expected_hours_day`, so you can upgrade from an older install without importing anything.
- **Resource-name guard** — `shared/_resource.lua` aborts the resource with a clear console error if the folder is not named `nexus_clocking`.
- **Framework-agnostic hooks** — `functions.lua` ships outside the escrow with six override points for notifications, permission-denied feedback, clock-in/clock-out side effects, a clock-in veto and the 3D prompt renderer.

## 📋 Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| **oxmysql** | **Required** | Hard-loaded as `@oxmysql/lib/MySQL.lua` in `server_scripts`. The script uses the `MySQL.query` / `MySQL.insert` / `MySQL.update` API including the `.await` variants. |
| **MySQL / MariaDB** | **Required** | Three tables. They are created automatically on first start; `nexus_clocking.sql` is provided if you prefer importing manually. MariaDB 10.5+ (or MySQL 8+) is recommended because the migrations use `ADD COLUMN IF NOT EXISTS`. |
| **es_extended (ESX)** | Optional | Only when `Config.Framework = 'esx'`. Resolved through `exports['es_extended']:getSharedObject()` inside a `pcall`. |
| **qb-core (QBCore)** | Optional | Only when `Config.Framework = 'qbcore'`. Resolved through `exports['qb-core']:GetCoreObject()` inside a `pcall`. |
| **ox_target** | Optional | Only when `Config.InteractionMode = 'ox_target'`. |
| **qb-target** | Optional | Only when `Config.InteractionMode = 'qb_target'`. |
| **A job set up in your framework** | Conditional | The internal job name you type into the admin panel must match a real job name in your framework, otherwise no player will ever pass the job filter. Not needed in standalone mode (`Config.Framework = nil`). |

No notification resource, TextUI resource, key system, fuel system or banking resource is required. Notifications are routed through `functions.lua`, which by default only draws native GTA notifications for the permission-denied case and does nothing for clock-in/clock-out (the terminal shows its own inline toasts).

## ⚙️ Installation

1. **Extract the resource.** Download the resource from the cfx.re (Keymaster) portal and place the folder in your `resources` directory. The folder **must** be named exactly `nexus_clocking` — `shared/_resource.lua` stops the resource otherwise.
2. **Import the database (optional).** The script creates its tables itself on start. If you prefer to do it up front, or your database user cannot run DDL at runtime, import `nexus_clocking.sql`. It creates `nexus_clocking_sessions`, `nexus_clocking_jobs` and `nexus_clocking_bosses`.
3. **Order your `server.cfg` correctly.** oxmysql and your framework must be started before this resource:
   ```cfg
   ensure oxmysql
   ensure es_extended      # or qb-core
   # ensure ox_target      # only if Config.InteractionMode = 'ox_target'
   ensure nexus_clocking
   ```
4. **Grant the admin permission.** Add the ACE to your `server.cfg` (or your permissions file):
   ```cfg
   add_ace group.admin nexus_clocking.admin allow
   ```
   Change the ACE name in `Config.AdminPermission` if you use another convention.
5. **Configure `shared/config.lua`.** At minimum set `Config.Framework` (`'esx'`, `'qbcore'` or `nil` for standalone) and `Config.InteractionMode`. Review the commands, the re-scan interval and the default NPC model, animation and prop.
6. **Restart the server.** A full server restart is recommended over `refresh; restart nexus_clocking`, because the database initialisation and the orphan-session sweep both run on `onResourceStart`.
7. **Create your first clocking point.** In game, run `/adminclocking` (the value of `Config.AdminCommand`). Go to the **Jobs** tab → **New Job**, fill in the internal job name (exactly as in your framework, e.g. `police`) and the display name, stand where you want the NPC and press **📍 Use current position**, then **Save Job**. The NPC spawns for every player of that job as soon as they come within the spawn distance.
8. **Assign bosses (optional).** In the same panel, **Bosses** tab → **Assign Boss**, pick the job and an online player. That player can then open the boss panel with `/bossclocking`.
9. **Clock in.** Walk up to the NPC, press `E` (or use your target), scan the fingerprint on the scanner and press `OK` (or click the button) to register your entry.

## 🔧 Configuration

Everything lives in `shared/config.lua`, which ships outside the escrow.

### Top-level keys

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.Command` | string | `'fichar'` | Chat command that opens the terminal. It only works when the player is already inside the interaction distance of a clocking NPC; otherwise it silently does nothing. Also used as the command name for `RegisterKeyMapping`. |
| `Config.OpenKey` | string or nil | `nil` | When set (e.g. `'e'`, `'f'`, `'g'`), registers a key mapping bound to `Config.Command` so players can rebind it in the GTA settings. `nil` disables the mapping. Independent of `Config.InteractionMode`. |
| `Config.Framework` | string or nil | `'esx'` | `'esx'`, `'qbcore'`, or `nil` for standalone. Drives player data lookup (name, identifier, job, grade, salary), the per-player job filter on the clocking points, and the job-change event handlers. In standalone mode no filtering happens and every player sees every clocking point. |
| `Config.HorasDia` | number | `8` | Fallback for expected hours per day, used only when the job row has no `expected_hours_day` value. Drives the overtime calculation on the statistics screen. |
| `Config.HorasEsperadasSemana` | number | `40` | Intended as the global fallback for expected hours per week. **Currently not read by any file** — the per-job `expected_hours_week` column and a hardcoded `40` in the NUI are used instead. Set the value in the admin panel per job. |
| `Config.NotificacionChat` | boolean | `true` | When `true`, a successful clock-in/clock-out also calls `Nexus.Notify(response)` in `functions.lua` in addition to the terminal's own inline toast. The shipped `Nexus.Notify` body is commented out, so nothing extra appears until you enable one of the examples. |
| `Config.RescanInterval` | number (seconds) | `120` | How long the terminal stays "unlocked" after the menu is closed. If the player reopens it after this many seconds, the fingerprint scan is required again. `0` disables re-locking for the session. |
| `Config.AdminCommand` | string | `'adminclocking'` | Chat command that requests the admin panel. The server validates the ACE before the panel is sent. |
| `Config.BossCommand` | string | `'bossclocking'` | Chat command that requests the boss panel. The server validates the caller against `nexus_clocking_bosses` before the panel is sent. |
| `Config.AdminPermission` | string | `'nexus_clocking.admin'` | ACE object checked with `IsPlayerAceAllowed`. Grant it with `add_ace group.admin nexus_clocking.admin allow`. |
| `Config.InteractionMode` | string | `'key'` | `'key'` → a 3D floating prompt over the NPC plus control 38 (`E`). `'ox_target'` → registers an `ox_target` local entity option. `'qb_target'` → registers a `qb-target` entity option. In target modes no floating text is drawn and `E` is not polled. |

### `Config.DefaultNPC`

These are the defaults applied to NPCs. Per-job coordinates, model, interaction distance and spawn distance come from the database row created in the admin panel and override the values here; the animation, the prop and the prompt text are global and only configurable here.

| Config key | Type | Default | Description |
|---|---|---|---|
| `Config.DefaultNPC.InteractDistance` | number (m) | `2.5` | Fallback interaction distance when a job row has no `interact_distance`. Also used as the `distance` for `qb-target`. |
| `Config.DefaultNPC.SpawnDistance` | number (m) | `80.0` | Fallback distance at which the NPC is created; beyond it the NPC (and its prop) is deleted. |
| `Config.DefaultNPC.PromptText` | string | `'[E] Fichar'` | The floating 3D text drawn above the NPC in `'key'` mode. This is the only place this string exists — it is not read from the locale files. |
| `Config.DefaultNPC.Model` | string | `'cs_andreas'` | Fallback ped model when a job row has no `npc_model`. An invalid model logs a warning and the NPC is simply not created (and is retried on the next loop). |
| `Config.DefaultNPC.Anim.Dict` | string | `'missheistdockssetup1clipboard@base'` | Animation dictionary played in a loop on every clocking NPC. |
| `Config.DefaultNPC.Anim.Name` | string | `'base'` | Animation clip name. |
| `Config.DefaultNPC.Anim.Flag` | number | `49` | `TaskPlayAnim` flag (49 = looped + upper body + allow player control). |
| `Config.DefaultNPC.Prop.Enabled` | boolean | `true` | Whether to create and attach the hand prop. |
| `Config.DefaultNPC.Prop.Model` | string | `'prop_notepad_01'` | Prop model. |
| `Config.DefaultNPC.Prop.Bone` | number | `60309` | Bone index the prop is attached to (60309 = right hand). |
| `Config.DefaultNPC.Prop.Offset` | vector3 | `vector3(0.12, 0.02, 0.0)` | Positional offset of the prop relative to the bone. |
| `Config.DefaultNPC.Prop.Rotation` | vector3 | `vector3(-160.0, 0.0, 0.0)` | Rotational offset of the prop. |

Structure reference:

```lua
Config.DefaultNPC = {
    InteractDistance = 2.5,
    SpawnDistance    = 80.0,
    PromptText       = '[E] Fichar',
    Model            = 'cs_andreas',

    Anim = {
        Dict = 'missheistdockssetup1clipboard@base',
        Name = 'base',
        Flag = 49,
    },

    Prop = {
        Enabled  = true,
        Model    = 'prop_notepad_01',
        Bone     = 60309,
        Offset   = vector3(0.12,  0.02, 0.0),
        Rotation = vector3(-160.0, 0.0, 0.0),
    },
}
```

### Per-job settings (admin panel, not config.lua)

These are **not** in `config.lua` by design — they are columns of `nexus_clocking_jobs`, edited in the admin panel's job modal:

| Field | Column | Default | Description |
|---|---|---|---|
| Internal Name | `job_name` | — | Must match the framework job name. Unique. Read-only once created. |
| Display Name | `job_label` | — | Shown in the UI, in the blip name and in session rows. |
| NPC Model | `npc_model` | `cs_officeworker` (SQL) / `cs_andreas` (runtime) | Ped model for this point. |
| NPC Coordinates | `npc_x`, `npc_y`, `npc_z`, `npc_heading` | `0` | Position and heading. "Use current position" fills these and ground-snaps Z. |
| Interaction Distance | `interact_distance` | `2.5` | Metres within which the prompt shows / `E` works. |
| Spawn Distance | `spawn_distance` | `80.0` | Metres within which the NPC exists. |
| Expected Hours / Week | `expected_hours_week` | `40` | Used for the weekly efficiency percentage. |
| Expected Hours / Day | `expected_hours_day` | `8` | Used for per-day efficiency and the overtime calculation. |
| Map Blip | `blip_enabled` | `1` | Toggles the blip. The blip is only ever created for players whose job matches, so it is naturally private to that job. |
| Blip Sprite | `blip_sprite` | `280` | GTA blip sprite id. |
| Blip Color | `blip_color` | `5` | GTA blip colour id. |
| Blip Scale | `blip_scale` | `0.75` | Blip scale. |
| Status | `enabled` | `1` | Disabled jobs are not sent to clients and cannot be clocked into. |

## 🌐 Locales & Editable Strings

**Current state — read this carefully.** `nexus_clocking` ships two locale files, `locales/en.lua` and `locales/es.lua`, both outside the escrow and both containing the same 96 keys (verified identical key sets). Each file defines a local `Translations` table, a global `T(key, ...)` helper and `return Translations`.

However, **the locale layer is not wired up in this version**:

- No file in the resource ever calls `T()`, and there is no `Config.Locale` key in `config.lua` (the comment in `fxmanifest.lua` referring to `Config.Locale` is stale).
- Both files are listed individually in `shared_scripts` (not as `locales/*.lua`), and because `es.lua` is loaded after `en.lua` it is the Spanish `T()` that ends up globally defined — but nothing calls it.
- The NUI does its own thing: `html/index.html` carries a handful of `data-i18n` attributes, but `html/script.js` contains no code that reads them.

So in practice the user-facing text lives in three places:

| Where | Escrow? | Editable? | Language as shipped |
|---|---|---|---|
| `locales/en.lua`, `locales/es.lua` | outside | yes | en / es — **but currently unused** |
| `html/index.html`, `html/script.js` | outside the escrow (shipped as `files`, never encrypted) | yes | Terminal screens in **Spanish**; admin and boss panels in **English** |
| `server/server.lua` | **inside the escrow** | **no** | Mostly **Spanish**, a few in English |
| `Config.DefaultNPC.PromptText` | outside | yes | Spanish (`'[E] Fichar'`) |
| `functions.lua` (permission-denied strings) | outside | yes | Spanish |

**Strings hardcoded inside the escrow.** `server/server.lua` builds its response messages inline, so these cannot be translated or edited by a customer:

- `'Ya tienes una sesión activa.'`
- `'No perteneces a este trabajo.'`
- `'Punto de fichaje no disponible.'`
- `'Entrada registrada correctamente.'`
- `'No tienes ninguna sesión activa.'`
- `'Salida registrada correctamente.'`
- `'Sesión cerrada: el trabajo fue eliminado.'`
- `'Entrada forzada por administrador.'` / `'Salida forzada por administrador.'`
- `'Faltan campos obligatorios.'`, `'Ya existe un trabajo con ese nombre interno.'`, `'ID no especificado.'`, `'Trabajo actualizado.'`, `'No se encontró el trabajo.'`, `'Jugador no encontrado.'`, `'El jugador ya tiene sesión activa.'`, `'Trabajo no encontrado.'`, `'El jugador no tiene sesión activa.'`, `'%s asignado como jefe de %s.'`, `'Jefe eliminado correctamente.'`, `'No encontrado.'`, `'No eres jefe de ningún trabajo.'`
- In English: `'This clocking point is not available.'`, `'Job not found.'`, `'Job "%s" and all its data deleted.'`, `'Not found.'`

**How to translate this version today.** Until the locale layer is connected, translating means editing `html/script.js` and `html/index.html` directly (both are plain files, not escrowed) for everything the player sees, and accepting that the server-side response messages stay as shipped. The 96 keys in `locales/*.lua` are a ready-made glossary for that work.

**Target languages.** The Nexus standard set is `es`, `en`, `de`, `fr`, `it`, `pt`, `zh`. This resource currently ships only `es` and `en`, and because `fxmanifest.lua` lists the locale files individually rather than with a glob, adding a language also requires adding a line to `fxmanifest.lua`.

## 🔗 Compatibility

| System | How it is selected | Notes |
|---|---|---|
| **Framework** | `Config.Framework = 'esx' \| 'qbcore' \| nil` | ESX via `exports['es_extended']:getSharedObject()`, QBCore via `exports['qb-core']:GetCoreObject()`, both wrapped in `pcall` on client and server. `nil` runs the script standalone with no job filtering and generic player data from `GetPlayerName` / `GetPlayerIdentifier`. |
| **Database** | hard dependency | oxmysql only. `@oxmysql/lib/MySQL.lua` is loaded directly in `server_scripts`, and the `MySQL.query.await` style calls are oxmysql-specific. mysql-async is **not** supported. |
| **Target system** | `Config.InteractionMode = 'ox_target' \| 'qb_target'` | Registered per NPC at spawn time. `ox_target` uses `addLocalEntity` with `onSelect`; `qb-target` uses `AddTargetEntity` with `options` + `action`. Both guard on the export existing. |
| **Key interaction** | `Config.InteractionMode = 'key'` | No dependency at all: a native 3D text prompt plus control 38. |
| **Key mapping** | `Config.OpenKey` | Standard `RegisterKeyMapping`, so the player can rebind it in GTA's own keybind settings. |
| **Notifications** | `functions.lua` → `Nexus.Notify` | **Not** modular via Config in this version. There is no `Config.NotifySystem` branch; instead `functions.lua` ships commented-out example bodies for native GTA notifications, ESX and ox_lib, and `Config.NotificacionChat` only decides whether the function is called at all. The permission-denied notification (`Nexus.NotifyPermissionDenied`) uses native GTA notifications out of the box. |
| **3D text / TextUI** | `functions.lua` → `Nexus.DrawPromptText` | Default implementation is native `DrawText` + `DrawRect`. Override the function to use `ox_lib`'s `DrawText3d`, nexus_TextUI or anything else. There is no TextUI resource dependency. |
| **Permissions** | FiveM ACE | `IsPlayerAceAllowed`, so it works with txAdmin groups, `add_ace`/`add_principal` or any ACE-based permission setup. Boss permission is data-driven from the `nexus_clocking_bosses` table, not ACE. |
| **Job changes** | automatic | `esx:setJob` (ESX) and `QBCore:Client:OnJobUpdate` (QBCore) are listened to, so changing a player's job updates their visible clocking points and force-clocks-them-out if needed without a reconnect. |
| **Banking / fuel / keys** | not used | This script never touches money, fuel or vehicle keys. Pay logic belongs in your own `Nexus.OnClockOut` implementation. |

## 💻 Developer API

### Client Exports

**None.** `nexus_clocking` does not register any client export. Integrate through the events below and through the `Nexus.*` functions in `functions.lua`.

### Server Exports

**None.** `nexus_clocking` does not register any server export. The two intended integration surfaces are `Nexus.OnClockIn` / `Nexus.OnClockOut` (for reacting to a shift) and `Nexus.CanClockIn` (for gating a shift), all in `functions.lua`.

### Events — Emitted

Server → client, all sent with `TriggerClientEvent`:

| Event | Side | Payload | When |
|---|---|---|---|
| `nexus_clocking:receiveJobs` | server → client | `jobs: table[]` — every enabled row of `nexus_clocking_jobs` | In reply to `nexus_clocking:requestJobs`. |
| `nexus_clocking:reloadJobs` | server → **all** clients (`-1`) | *none* | After an admin creates, updates or deletes a job, so every client re-requests the list. |
| `nexus_clocking:openAdminPanel` | server → client | *none* | The caller passed the ACE check. |
| `nexus_clocking:openBossPanel` | server → client | *none* | The caller was found in `nexus_clocking_bosses`. |
| `nexus_clocking:permissionDenied` | server → client | `panelType: 'admin' \| 'boss'` | The caller failed the admin or boss check. Handled by `Nexus.NotifyPermissionDenied`. |
| `nexus_clocking:receivePlayerInfo` | server → client | `{ player, jobName, jobConfig = { expectedHoursWeek, expectedHoursDay }, stats = { totalDias, totalHours, avgDaily, weekHours, weekSeconds, totalSeconds }, history[], isClockedIn, clockInTime }` | In reply to `nexus_clocking:getPlayerInfo`. |
| `nexus_clocking:receiveStatistics` | server → client | `{ dailyData[], extraSeconds, extraFormatted, totalDias, totalSeconds, totalHours, weekHours, weekSeconds }` | In reply to `nexus_clocking:getStatistics`. |
| `nexus_clocking:clockResponse` | server → client | `{ success, action = 'clockIn' \| 'clockOut', timestamp, duration?, formatted?, message }` | After a clock-in / clock-out attempt (including a forced one and a cascade close). On failure only `success = false` and `message` are set. |
| `nexus_clocking:admin:data` | server → client | `{ jobs[], bosses[], liveSessions[], globalStats[] }` | In reply to `nexus_clocking:admin:getAll`. |
| `nexus_clocking:admin:liveData` | server → client | `sessions: table[]` — `{ source, name, identifier, job, jobLabel, streetName, startTime, elapsed }` | In reply to `nexus_clocking:admin:getLive` (polled every 5 s while the Live tab is open). |
| `nexus_clocking:admin:onlinePlayers` | server → client | `players: table[]` — `{ source, name, identifier, hasSession, session? }` | In reply to `nexus_clocking:admin:getOnlinePlayers`. |
| `nexus_clocking:admin:response` | server → client | `{ success, message, action? = 'refresh' }` | Result of any admin write action. |
| `nexus_clocking:boss:data` | server → client | `{ success, bossJobs[], liveSessions[], history[], employees[] }` or `{ success = false, message }` | In reply to `nexus_clocking:boss:getData` and after `nexus_clocking:boss:deleteEmployee`. |

### Events — Listened

Client → server, all registered with `RegisterNetEvent` on the server:

| Event | Side | Payload | Purpose |
|---|---|---|---|
| `nexus_clocking:requestJobs` | client → server | *none* | Ask for the enabled clocking points. |
| `nexus_clocking:checkAdminAndOpen` | client → server | *none* | Validate the ACE and open the admin panel. |
| `nexus_clocking:checkBossAndOpen` | client → server | *none* | Validate boss status and open the boss panel. |
| `nexus_clocking:getPlayerInfo` | client → server | `targetJobName: string?` | Load player data, aggregate stats and the last 30 sessions for that job. |
| `nexus_clocking:clockIn` | client → server | `jobName: string`, `streetName: string?` | Open a session. Rejected if a session is already open, if the player's job does not match (unless they hold the admin ACE), if `Nexus.CanClockIn` returns false, or if the job row is missing/disabled. |
| `nexus_clocking:clockOut` | client → server | *none* | Close the open session and compute its duration. |
| `nexus_clocking:getStatistics` | client → server | `periodo: 'day' \| 'week' \| 'month' \| '6months' \| 'year'`, `jobName: string?` | Build the statistics payload. `'year'` is accepted by the server but is not offered by the NUI. |
| `nexus_clocking:admin:getAll` | client → server | *none* | Jobs + bosses + live sessions + global stats. **Admin only.** |
| `nexus_clocking:admin:createJob` | client → server | `d: table` (job fields) | Insert a job row. **Admin only.** |
| `nexus_clocking:admin:updateJob` | client → server | `d: table` (must include `id`) | Update a job row. **Admin only.** |
| `nexus_clocking:admin:deleteJob` | client → server | `jobId: number` | Cascade-delete a job. **Admin only.** |
| `nexus_clocking:admin:getLive` | client → server | *none* | Current live sessions. **Admin only.** |
| `nexus_clocking:admin:forceClockIn` | client → server | `targetSource: number`, `jobName: string` | Open a session for another player, bypassing the job match and `Nexus.CanClockIn`. **Admin only.** |
| `nexus_clocking:admin:forceClockOut` | client → server | `targetSource: number` | Close another player's session. **Admin only.** |
| `nexus_clocking:admin:assignBoss` | client → server | `jobName`, `identifier`, `playerName` | Upsert a boss assignment. **Admin only.** |
| `nexus_clocking:admin:removeBoss` | client → server | `bossId: number` | Delete a boss assignment. **Admin only.** |
| `nexus_clocking:admin:getOnlinePlayers` | client → server | *none* | List online players with their session state. **Admin only.** |
| `nexus_clocking:boss:getData` | client → server | *none* | Boss panel payload, scoped to the caller's boss jobs. |
| `nexus_clocking:boss:deleteEmployee` | client → server | `identifier: string` | Delete that identifier's session history, restricted to the caller's boss jobs. |

Third-party / framework events the resource listens to:

| Event | Side | Purpose |
|---|---|---|
| `esx:setJob` | client | Re-filter visible clocking points and force clock-out if needed (ESX only). |
| `QBCore:Client:OnJobUpdate` | client | Same, for QBCore. |
| `onClientResourceStart` | client | `functions.lua` pushes `{ action = 'setConfig', rescanInterval = Config.RescanInterval }` into the NUI. |
| `onResourceStart` | server | Creates/migrates the tables, then sweeps orphaned sessions after a 1 s delay. |
| `playerDropped` | server | Force-closes the dropping player's open session. |

### NUI Callbacks

Registered on the client with `RegisterNUICallback`. Useful if you replace or extend the interface.

| Callback | Payload | Effect |
|---|---|---|
| `close` / `closeAdmin` / `closeBoss` | — | Release NUI focus and hide the corresponding screen. |
| `clockIn` | `{ jobName }` | Resolves the current street name and triggers `nexus_clocking:clockIn`. |
| `clockOut` | — | Triggers `nexus_clocking:clockOut`. |
| `getStatistics` | `{ periodo, jobName }` | Triggers `nexus_clocking:getStatistics`. |
| `refreshData` | `{ jobName }` | Triggers `nexus_clocking:getPlayerInfo`. |
| `adminGetAll`, `adminGetLive`, `adminGetOnlinePlayers` | — | Pass-through to the matching server event. |
| `adminCreateJob`, `adminUpdateJob` | the job form object | Pass-through. |
| `adminDeleteJob` | `{ id }` | Pass-through. |
| `adminForceClockIn` | `{ source, jobName }` | Pass-through. |
| `adminForceClockOut` | `{ source }` | Pass-through. |
| `adminAssignBoss` | `{ jobName, identifier, playerName }` | Pass-through. |
| `adminRemoveBoss` | `{ id }` | Pass-through. |
| `adminGetCoords` | — | **Returns** `{ x, y, z, heading }` for the calling player, with Z ground-snapped. |
| `bossGetData` | — | Pass-through. |
| `bossDeleteEmployee` | `{ identifier }` | Pass-through. |

### Editable Functions (functions.lua)

`functions.lua` is listed in `escrow_ignore` and is loaded both as a `client_script` (before `client/client.lua`) and as a `server_script` (before `server/server.lua`), so the same file provides both the client and the server override points. All six are called defensively (`if Nexus and Nexus.X then`), so you can delete one if you do not need it.

| Function | Signature | Called from | Purpose |
|---|---|---|---|
| `Nexus.Notify` | `Nexus.Notify(response)` | **Client** — `nexus_clocking:clockResponse` handler, only when `Config.NotificacionChat` is `true` and `response.success` is truthy. | Fire an extra notification on top of the terminal's inline toast. `response` carries `action` (`'clockIn'`/`'clockOut'`), `timestamp`, and for clock-out also `duration` (seconds) and `formatted` (e.g. `"3h 22m"`). Return value ignored. Ships with every body commented out (examples for native GTA, ESX and ox_lib). |
| `Nexus.NotifyPermissionDenied` | `Nexus.NotifyPermissionDenied(panelType)` | **Client** — `nexus_clocking:permissionDenied` handler. | Tell the player they lack access. `panelType` is `'admin'` or `'boss'`. Ships with a native GTA notification. Return value ignored. **Note:** the shipped implementation does not guard against `Nexus` being nil, so do not delete this function. |
| `Nexus.OnClockIn` | `Nexus.OnClockIn(source, data)` | **Server** — immediately after the session row is inserted and the console line is printed. | Side effects on clock-in: Discord webhooks, logging, handing out items, starting a paycheck timer. `data` = `{ identifier, name, job, jobLabel, grade, salary, sessionId, startTime }`. Return value ignored. |
| `Nexus.OnClockOut` | `Nexus.OnClockOut(source, data)` | **Server** — after `clock_out` and `duration` are written, before the session is removed from memory and before the client is notified. | Side effects on clock-out — this is where you pay the player. `data` = `{ identifier, name, job, jobLabel, duration, formatted, clockOut }`. Note that `salary` is **not** included here (it is only in the clock-in payload), so cache it in `OnClockIn` if you need it. Return value ignored. |
| `Nexus.CanClockIn` | `Nexus.CanClockIn(source, jobName)` → `boolean, string?` | **Server** — after the "already clocked in" and job-match checks, before the job row is looked up. Skipped entirely by `admin:forceClockIn`. | Veto a clock-in. Return `true` to allow. Return `false, 'reason'` to block; the reason is sent back as `clockResponse.message` and shown as an error toast in the terminal. Falls back to `'No puedes fichar en este momento.'` if you return `false` with no message. Must return `true` explicitly — the shipped body does. |
| `Nexus.DrawPromptText` | `Nexus.DrawPromptText(x, y, z, text)` | **Client** — every frame while the player is within the interaction distance of an NPC and `Config.InteractionMode = 'key'`. | Render the floating interaction prompt. `x, y, z` are the NPC's coordinates plus 1.15 on Z; `text` is `Config.DefaultNPC.PromptText`. Ships with a native `DrawText` + `DrawRect` implementation. Return value ignored. Because this runs every frame, keep it cheap. |

Minimal overrides:

```lua
-- functions.lua

function Nexus.OnClockIn(source, data)
    PerformHttpRequest('https://discord.com/api/webhooks/XXX', function() end, 'POST',
        json.encode({ content = ('IN  %s (%s) @ %s'):format(data.name, data.identifier, data.jobLabel) }),
        { ['Content-Type'] = 'application/json' })
end

function Nexus.OnClockOut(source, data)
    local xPlayer = exports['es_extended']:getSharedObject().GetPlayerFromId(source)
    if xPlayer then
        xPlayer.addAccountMoney('bank', math.floor((data.duration / 3600) * 150))
    end
end

function Nexus.CanClockIn(source, jobName)
    if GetEntityHealth(GetPlayerPed(source)) <= 100 then
        return false, 'You cannot clock in while injured.'
    end
    return true
end
```

### Database Schema

Three tables, created automatically on `onResourceStart` and also provided in `nexus_clocking.sql`.

```sql
-- Shift history. One row per clock-in; clock_out and duration are filled on clock-out.
CREATE TABLE IF NOT EXISTS `nexus_clocking_sessions` (
    `id`          INT          NOT NULL AUTO_INCREMENT,
    `identifier`  VARCHAR(100) NOT NULL,
    `player_name` VARCHAR(100) NOT NULL,
    `job`         VARCHAR(100) NOT NULL DEFAULT 'unknown',
    `job_label`   VARCHAR(100) NOT NULL DEFAULT 'Desconocido',
    `grade`       INT          NOT NULL DEFAULT 0,
    `clock_in`    INT          NOT NULL,                        -- unix timestamp
    `clock_out`   INT                   DEFAULT NULL,           -- unix timestamp, NULL while on duty
    `duration`    INT                   DEFAULT NULL  COMMENT 'Duración en segundos',
    `created_at`  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_identifier` (`identifier`),
    INDEX `idx_clock_in`   (`clock_in`),
    INDEX `idx_job`        (`job`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Clocking points, managed from the admin panel.
CREATE TABLE IF NOT EXISTS `nexus_clocking_jobs` (
    `id`                  INT          NOT NULL AUTO_INCREMENT,
    `job_name`            VARCHAR(100) NOT NULL UNIQUE  COMMENT 'Nombre interno (police, mechanic…)',
    `job_label`           VARCHAR(100) NOT NULL          COMMENT 'Nombre visible',
    `npc_model`           VARCHAR(100) NOT NULL DEFAULT 'cs_officeworker',
    `npc_x`               FLOAT        NOT NULL DEFAULT 0,
    `npc_y`               FLOAT        NOT NULL DEFAULT 0,
    `npc_z`               FLOAT        NOT NULL DEFAULT 0,
    `npc_heading`         FLOAT        NOT NULL DEFAULT 0,
    `interact_distance`   FLOAT        NOT NULL DEFAULT 2.5,
    `spawn_distance`      FLOAT        NOT NULL DEFAULT 80.0,
    `blip_enabled`        TINYINT(1)   NOT NULL DEFAULT 1,
    `blip_sprite`         INT          NOT NULL DEFAULT 280,
    `blip_color`          INT          NOT NULL DEFAULT 5,
    `blip_scale`          FLOAT        NOT NULL DEFAULT 0.75,
    `expected_hours_week` FLOAT        NOT NULL DEFAULT 40,
    `expected_hours_day`  FLOAT        NOT NULL DEFAULT 8,
    `enabled`             TINYINT(1)   NOT NULL DEFAULT 1,
    `created_at`          TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Upgrade path from an earlier 2.x install.
ALTER TABLE `nexus_clocking_jobs` ADD COLUMN IF NOT EXISTS `blip_enabled`        TINYINT(1) NOT NULL DEFAULT 1;
ALTER TABLE `nexus_clocking_jobs` ADD COLUMN IF NOT EXISTS `expected_hours_week` FLOAT      NOT NULL DEFAULT 40;
ALTER TABLE `nexus_clocking_jobs` ADD COLUMN IF NOT EXISTS `expected_hours_day`  FLOAT      NOT NULL DEFAULT 8;

-- Boss assignments, managed from the admin panel.
CREATE TABLE IF NOT EXISTS `nexus_clocking_bosses` (
    `id`          INT          NOT NULL AUTO_INCREMENT,
    `job_name`    VARCHAR(100) NOT NULL,
    `identifier`  VARCHAR(100) NOT NULL,
    `player_name` VARCHAR(100) NOT NULL,
    `assigned_by` VARCHAR(100) NOT NULL DEFAULT 'admin',
    `assigned_at` TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uq_boss_job` (`job_name`, `identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

The `identifier` written to `nexus_clocking_sessions` depends on the framework: ESX writes `xPlayer.identifier`, QBCore writes `citizenid`, standalone writes `GetPlayerIdentifier(source, 0)`. Do not switch framework on a live database without migrating identifiers, or history will appear empty.

`nexus_clocking.sql` also contains a commented-out `INSERT` with three example jobs (`police`, `mechanic`, `ems`) if you prefer seeding from SQL instead of the panel.

### Integration Example

A complete paycheck + audit-log integration, written entirely in `functions.lua` with no escrowed code touched. It pays a bank salary proportional to the shift length, logs both ends of the shift to a Discord webhook and to your own table, and refuses to let a player clock in without their ID card.

```lua
-- functions.lua  —  nexus_clocking integration example
--
-- Everything below runs on the SERVER side of functions.lua
-- (the file is loaded as both a client_script and a server_script).

local WEBHOOK   = 'https://discord.com/api/webhooks/XXXXX/YYYYY'
local RATE_HOUR = { police = 220, mechanic = 160, ambulance = 200 }

local salaryCache = {}   -- source -> grade_salary captured at clock-in

local function log(msg)
    PerformHttpRequest(WEBHOOK, function() end, 'POST',
        json.encode({ username = 'Clocking', content = msg }),
        { ['Content-Type'] = 'application/json' })
end

-- Gate: must hold an id_card item (ESX)
function Nexus.CanClockIn(source, jobName)
    local ESX = exports['es_extended']:getSharedObject()
    local xPlayer = ESX.GetPlayerFromId(source)
    if not xPlayer then return false, 'Player not loaded.' end

    local card = xPlayer.getInventoryItem('id_card')
    if not card or card.count < 1 then
        return false, 'You need your ID card to clock in.'
    end
    return true
end

-- Shift opened
function Nexus.OnClockIn(source, data)
    salaryCache[source] = data.salary

    log(('IN  **%s** (`%s`) · %s · grade %s · session #%s')
        :format(data.name, data.identifier, data.jobLabel, data.grade, data.sessionId))

    MySQL.insert('INSERT INTO my_duty_audit (identifier, job, action, at) VALUES (?, ?, ?, ?)',
        { data.identifier, data.job, 'in', data.startTime })
end

-- Shift closed → pay
function Nexus.OnClockOut(source, data)
    local ESX = exports['es_extended']:getSharedObject()
    local xPlayer = ESX.GetPlayerFromId(source)

    local hourly = RATE_HOUR[data.job] or salaryCache[source] or 0
    local pay    = math.floor((data.duration / 3600) * hourly)

    if xPlayer and pay > 0 then
        xPlayer.addAccountMoney('bank', pay)
        TriggerClientEvent('esx:showNotification', source,
            ('Shift paid: $%s for %s'):format(pay, data.formatted))
    end

    log(('OUT **%s** (`%s`) · %s · worked %s · paid $%s')
        :format(data.name, data.identifier, data.jobLabel, data.formatted, pay))

    MySQL.insert('INSERT INTO my_duty_audit (identifier, job, action, at) VALUES (?, ?, ?, ?)',
        { data.identifier, data.job, 'out', data.clockOut })

    salaryCache[source] = nil
end
```

If you need to react from a **separate resource** instead of from `functions.lua`, listen for the client-side response event and relay it, since the script exposes no server-side broadcast of its own:

```lua
-- my_resource/client.lua
RegisterNetEvent('nexus_clocking:clockResponse', function(response)
    if response.success then
        TriggerServerEvent('my_resource:dutyChanged', response.action, response.duration)
    end
end)
```

Reading state directly from the database also works and is the only way to read it from another resource server-side:

```lua
-- Is this identifier on duty right now?
local open = MySQL.query.await(
    'SELECT job, clock_in FROM nexus_clocking_sessions WHERE identifier = ? AND clock_out IS NULL LIMIT 1',
    { identifier })
local onDuty = open and open[1] or nil
```

## ❓ FAQ

**The NPC does not appear at all.**
Four things to check, in order. (1) The job row must have `enabled = 1`. (2) If `Config.Framework` is set, your character's job name must match the row's `job_name` **exactly** — the client filters the list and an empty list means no NPC and no blip. The console prints `[nexus_clocking] N job(s) loaded for player (job: X)` on load, which tells you both numbers. (3) You must be within `spawn_distance` (80 m by default). (4) An invalid `npc_model` logs `ADVERTENCIA: modelo NPC invalido "…"` and skips the NPC — fix the model name in the admin panel.

**`/fichar` does nothing.**
By design: the command only opens the terminal when you are already within the interaction distance of a clocking NPC. If nothing happens, you are either too far away, or the NPC for your job does not exist (see above). The command is also a no-op while the admin or boss panel is open.

**`/adminclocking` says I have no permission.**
The ACE is not granted. Add `add_ace group.admin nexus_clocking.admin allow` to your `server.cfg` and make sure your identifier is actually in `group.admin` (`add_principal identifier.license:… group.admin`). Restart the server after changing ACEs — they are not hot-reloaded reliably.

**`/bossclocking` says I am not a boss.**
Boss access is not an ACE. An admin has to assign you in the admin panel's **Bosses** tab, which writes your identifier into `nexus_clocking_bosses`. The assignment is per identifier, not per job grade, so promoting someone in your framework does not make them a boss here.

**I see "⚠ DEBUG MODE" and fake employees named "⚠ DEBUG — Juan García" in the boss panel.**
`html/script.js` ships with `const DEBUG_MODE = true` near the bottom of the file. Set it to `false` and restart the resource. With it enabled, a debug badge is drawn over the NUI and ten fictional employees are injected into every boss panel load.

**Can I use mysql-async instead of oxmysql?**
No. `@oxmysql/lib/MySQL.lua` is loaded directly in `fxmanifest.lua` and the code uses oxmysql-specific `MySQL.query.await` / `MySQL.insert.await` / `MySQL.update.await` calls. oxmysql is a hard requirement.

**The tables were not created.**
Table creation runs on `onResourceStart`, so it needs oxmysql to already be up — make sure `ensure oxmysql` comes before `ensure nexus_clocking`. If your database user cannot run `CREATE`/`ALTER`, import `nexus_clocking.sql` manually with a privileged user. The `ADD COLUMN IF NOT EXISTS` migrations need MariaDB 10.5+ or MySQL 8+; on older servers add `blip_enabled`, `expected_hours_week` and `expected_hours_day` by hand.

**Everything in the terminal is in Spanish but the admin panel is in English. How do I translate it?**
In this version the locale files are not wired up — nothing calls `T()` and there is no `Config.Locale`. The player-facing text is hardcoded in `html/script.js` and `html/index.html` (terminal screens in Spanish, admin/boss panels in English), and both files ship unencrypted, so you can edit them directly. The server's own response messages (`'No perteneces a este trabajo.'` and similar) are inside the escrow and cannot be changed. See **Locales & Editable Strings**.

**My shift statistics are empty after I changed framework.**
The `identifier` column stores a different value per framework (ESX `identifier`, QBCore `citizenid`, standalone `license:…`). Switching `Config.Framework` on a live database makes all existing history invisible to the player. Migrate the `identifier` column before switching.

**A player disconnected while clocked in. Is the shift lost?**
No. `playerDropped` closes the session with the current time. If the server crashed outright, the start-up sweep closes any session still missing a `clock_out` and prints `Limpieza: N sesión(es) huérfana(s) cerrada(s) al arrancar.` Both writes are guarded with `AND clock_out IS NULL`, so a session is never closed twice.

**Why does the efficiency percentage cap at 100% on one screen and 999% on another?**
Deliberate: the clocking screen footer and the info screen clamp weekly efficiency to 100%, while the statistics screen allows up to 999% so overtime is visible. The statistics progress bar still caps its width at 100%.

**Does changing a player's job while they are clocked in break anything?**
No. The job-change handler re-filters the visible points immediately and, if the new job has no clocking point, triggers a clock-out and closes the terminal. The session is written to the database correctly with the old job.

**I deleted a job by mistake. Can I get the data back?**
No. Deleting a job cascades: every session row for that `job_name` and every boss assignment for it are deleted, and active sessions are force-closed. There is no soft delete. If you only want to hide a point, set its **Status** to *Inactive* instead — that keeps all history.

**Does this script pay salaries?**
No. It only records time. Pay logic belongs in your `Nexus.OnClockOut` implementation in `functions.lua` — see the **Integration Example**.

### Before opening a ticket

- Make sure the resource folder is named exactly **`nexus_clocking`**. Any other name aborts the resource with a console error.
- Make sure you are on the latest version of the resource (**v2.2.0**, per `fxmanifest.lua`).
- Confirm `ensure oxmysql` and your framework come **before** `ensure nexus_clocking` in `server.cfg`.
- Check the server console on start for `[nexus_clocking] Base de datos inicializada.` and the client console (F8) for `[nexus_clocking] N job(s) loaded for player (job: X)`.
- Re-read this FAQ page.

## 📋 Changelog

**v2.2.0 — current release** (per `fxmanifest.lua`: `version '2.2.0'`, `description 'Sistema de fichaje multitrabajo para FiveM — Nexus Clocking v2.2'`)

No version history file ships with the resource. What the code itself documents about the upgrade path:

- `nexus_clocking.sql` is labelled `v2` and the server runs `ADD COLUMN IF NOT EXISTS` migrations for `blip_enabled`, `expected_hours_week` and `expected_hours_day`, described in-code as *"add new columns if upgrading from v2 base"* — so per-job blip control and per-job expected hours (per week and per day) were added after the initial 2.x release.
- Orphaned-session cleanup on resource start and the `AND clock_out IS NULL` guards on every close are present, indicating the session-integrity work is part of this release.
