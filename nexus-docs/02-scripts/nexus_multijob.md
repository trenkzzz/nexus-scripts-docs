---
icon: buildings
---

# Nexus Multijob

> Let players keep several jobs at once and switch between them from a clean, themeable NUI panel — for ESX and QBCore.

**Version:** 1.0.1 · **Frameworks:** ESX, QBCore · **Database:** oxmysql

***

## 📝 Description

Nexus Multijob gives every player a personal, persistent job roster. Instead of losing their police grade the moment they take a taxi shift, a player accumulates jobs in their own list and swaps the active one whenever they want from a single-key side panel. The roster is capped per server (and optionally per player), the currently active job is protected from accidental deletion, and going off duty is a one-click action.

The key design decision is that **the roster fills itself**. The resource does not replace your job centres, your police MDT or your `/setjob` admin tooling: it listens to the job-change events your framework already fires — `esx:setJob` and `esx:playerLoaded` on ESX, `QBCore:Server:SetJob` and `QBCore:Server:PlayerLoaded` on QBCore — and every time a player is given a job by _any_ resource, that job plus its grade and labels is upserted into the `nexus_multijob` table. There is no migration step and no integration work for existing job scripts; a player who was hired at the police station simply finds `Police` in their list the next time they open the panel. For first-time users whose current job predates the resource, the panel's own open handler calls `EnsureCurrentJobSaved`, which captures whatever job the player is wearing right now before building the list.

The grade a player switches into is always the one saved on the server — **never** the one sent by the client — so the panel cannot be used to self-promote. Server owners control how many jobs each player can hold, can grant specific players a higher limit, rename jobs and grades for display, choose the panel's colour theme and pick a notification backend.

Everything player-facing goes through a custom NUI panel — a slide-in side card with a header counter (`3/6`), one card per job showing the job label and grade label, an `ACTIVE` badge on the current job, a select button, a delete button, a confirmation drawer for deletions, a themed empty state for unemployed players, and a **Go Off-Duty** footer button that appears only when the player is actually on duty. The panel is **optimistic**: clicking _Select_ instantly repaints the cards and the header before the server has answered, then reconciles against the server's authoritative refresh when it arrives, so switching feels instant even with latency. Deletions animate out with a height collapse, and newly added jobs animate in — the refresh path does a real DOM diff rather than re-rendering the list. The panel only does any work while it is open: there is no per-frame polling thread, so it costs nothing while closed.

On the data side the resource is deliberately small: one table keyed by `(identifier, job)` with a unique index, so a job can never be duplicated for a player, and all queries go through `oxmysql`. Job and grade **labels** are resolved at write time from the framework (`ESX.GetJob` / `QBCore.Shared.Jobs`) and stored alongside the job name, which means the panel renders correct, pretty labels even for jobs whose definitions have since changed, and without a framework round-trip per card. Two config tables let a server owner override any job or grade label without touching the framework.

***

## ✨ Features

* **Persistent per-player job roster** — jobs are stored in a dedicated `nexus_multijob` table keyed by the framework identifier (ESX `identifier`, QBCore `citizenid`), surviving reconnects and restarts. Up to `Config.MaxJobs` jobs per player (default `6`).
* **Automatic job capture** — hooks `esx:setJob` / `esx:playerLoaded` (ESX) and `QBCore:Server:SetJob` / `QBCore:Server:PlayerLoaded` (QBCore), so any job handed out by any other resource is added to the roster with no integration work.
* **First-use capture** — opening the panel runs `EnsureCurrentJobSaved`, which adds the player's current job to the roster if it is missing, so existing players are not asked to re-apply for jobs they already have.
* **Upsert, never duplicate** — a `UNIQUE KEY (identifier, job)` index plus an existence check means re-hiring or a promotion **updates** the stored grade and labels instead of adding a second row.
* **Server-wide job cap** — `Config.MaxJobs` (default `6`) limits roster size; the cap is enforced server-side at insert time, so it cannot be bypassed from the client.
* **Per-player cap overrides** — `Config.MaxJobsOverrides` raises or lowers the cap for specific players by identifier (VIPs, staff…), with prefix-tolerant matching: the exact identifier is tried first, then the bare hash with any `license:` / `char1:` / `char2:` / `steam:` prefix stripped, so the same entry works across identifier formats.
* **Server-authoritative grade** — switching jobs always applies the grade stored in the database; a grade value sent by the client is ignored. A player cannot promote themselves through this panel.
* **Custom NUI panel** — a slide-in side card with: a header showing a briefcase glyph, `MY JOBS` and a live `used/max` counter; one card per job with job label, grade label and an animated `ACTIVE` badge; a select button; a delete button; and a close button.
* **Optimistic UI** — clicking _Select_ repaints every card, swaps the buttons, disables the newly-active one and updates the header immediately, without waiting for the server; the server's `refresh` then reconciles the real state.
* **DOM-diffing refresh** — the refresh path compares the previous and new job keys, animates removed cards out (fade, slide, then height collapse) and new cards in (fade + translate), and updates surviving cards in place rather than rebuilding the list.
* **Delete confirmation drawer** — a themed in-panel overlay naming the job, warning that the action cannot be reversed, with Cancel / Delete buttons; clicking the backdrop cancels.
* **Active-job protection** — the delete button is disabled on the active job in the UI _and_ the server rejects the deletion and pushes a UI resync, so the player can never end up with no job through this panel.
* **One-click off duty** — a footer **GO OFF-DUTY** button that sets the player to `Config.UnemployedJob` / `Config.UnemployedGrade`. It is hidden entirely when `Config.AllowOffDuty = false`, and also hidden while the player is already off duty.
* **Themed empty state** — unemployed players get a dedicated panel with a ringed briefcase icon, a `YOU ARE UNEMPLOYED` headline, a randomly generated reference code, guidance to visit a job office, and an `AVAILABLE JOBS` tag.
* **Six colour themes** — `Config.Theme` picks `purple` (default), `orange`, `green`, `red`, `blue` or `gold`; the theme is pushed to the NUI as a `data-theme` attribute and swaps a full set of CSS custom properties (three accent shades, glow, dim, three border alphas, the active-card background and a secondary accent).
* **Configurable, rebindable open key** — `Config.OpenKey` (default `F5`) is registered through `RegisterKeyMapping`, so it also appears in the player's own FiveM keybind settings and can be rebound per player. Changing the default in `config.lua` only affects players who have never bound the key themselves.
* **Command access** — `/nexus_jobs` toggles the panel, so it works for players who unbind the key, and can be called from your own code.
* **ESC-to-close** — captured inside the NUI (game controls are blocked while NUI has focus) and relayed back to Lua to release focus cleanly.
* **Admin grant command** — `/njgive [playerid] [job] [grade]` adds a job to a player's roster, gated on ESX `admin`/`superadmin` group or QBCore `admin` permission, and usable from the server console.
* **Label overrides** — `Config.JobLabels` and `Config.GradeLabels` override the framework's labels per job and per grade, applied both at capture time and at display time.
* **Seven-language notification locales** — `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`, selected with `Config.Locale` (English by default), with an automatic English fallback for any missing key.
* **Pluggable notifications** — `Config.NotifyStyle` selects `native` (GTA), `esx`, `qb` or `ox` (`ox_lib`) without touching code.
* **Optimised** — no per-frame polling threads; the resource does no idle work while the panel is closed.
* **Resource-name validation** — the resource aborts at start with a clear console error if the folder has been renamed.

***

## 📋 Dependencies

| Dependency                         | Required                       | Notes                                                                                                                                                                                                                                                                                          |
| ---------------------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Framework: ESX or QBCore**       | ✅ Yes                          | Selected with `Config.Framework` (`'esx'` or `'qbcore'`). The resource reads the player object, job and grade through the framework and cannot run standalone.                                                                                                                                 |
| **`oxmysql`**                      | ✅ Yes                          | Hard dependency. `fxmanifest.lua` loads `@oxmysql/lib/MySQL.lua`, and the server uses `MySQL.query`, `MySQL.scalar`, `MySQL.insert` and `MySQL.update`. **`mysql-async` is not supported** — unlike some other resources in the catalogue, this one does not go through a compatibility layer. |
| **MySQL / MariaDB database**       | ✅ Yes                          | One table, created by the included `nexus_multijob.sql`.                                                                                                                                                                                                                                       |
| **Jobs defined in your framework** | ✅ Yes                          | Labels are resolved from `ESX.GetJob(name)` / `QBCore.Shared.Jobs[name]`. A job that does not exist in the framework still works, but falls back to showing the raw job name and the numeric grade.                                                                                            |
| **An `unemployed` job**            | ✅ Yes (if off duty is enabled) | `Config.UnemployedJob` must be a job your framework accepts. Both ESX and QBCore ship `unemployed` by default.                                                                                                                                                                                 |
| **`ox_lib`**                       | ⚠️ Optional                    | Only required if you set `Config.NotifyStyle = 'ox'`.                                                                                                                                                                                                                                          |
| **`nexus_notify`**                 | ❌ No                           | Not wired into this resource's notification branch — see Compatibility.                                                                                                                                                                                                                        |
| Internet access on the client      | ⚠️ Soft                        | The NUI loads Bebas Neue and Inter from Google Fonts; without internet it falls back to a generic sans-serif.                                                                                                                                                                                  |

The folder **must** be named exactly `nexus_multijob`. `shared/_resource.lua` raises a hard `error()` and aborts the resource otherwise.

***

## ⚙️ Installation

1. **Download and extract the resource.** Download it from the cfx.re (Keymaster) portal and place the folder in your `resources` directory. Keep the name `nexus_multijob`.
2. **Import the database.** Import `nexus_multijob.sql` into your database. It creates a single `nexus_multijob` table with a `UNIQUE KEY` on `(identifier, job)`. The script uses `CREATE TABLE IF NOT EXISTS`, so re-running it is safe.
3. **Check your MySQL resource.** `oxmysql` is required and must be started **before** `nexus_multijob`.
4.  **Correct `server.cfg` ordering.** The framework and `oxmysql` must both be initialised first:

    ```cfg
    ensure oxmysql
    ensure es_extended        -- or qb-core
    -- then
    ensure nexus_multijob
    ```
5. **Configure the essentials in `config.lua`:**
   * `Config.Framework` — `'esx'` or `'qbcore'`. **This is the one setting that must be right or nothing works.**
   * `Config.Locale` — one of `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`.
   * `Config.MaxJobs` — the roster cap (recommended maximum `6`).
   * `Config.OpenKey` — default `F5`.
   * `Config.NotifyStyle` — `'native'`, `'esx'`, `'qb'` or `'ox'`.
   * `Config.Theme` — `'purple'`, `'orange'`, `'green'`, `'red'`, `'blue'` or `'gold'`.
   * `Config.UnemployedJob` — must match the off-duty job name your framework uses.
6. **Restart.** A full server restart is recommended so the framework hooks register cleanly.
7. **Start using it.** In game, press **F5** (or run **`/nexus_jobs`**). Your current job is captured automatically on that first open. Take a second job from any job centre and it will appear in the list.
8. **Optional — rebind the key.** Players can rebind it themselves in _Settings → Key Bindings → FiveM_, where it appears under the label from the active locale's `keymap` string.
9. **Optional — grant a job manually.** From the server console or as an admin in game: `/njgive 3 police 2`.

### Updating from 1.0.0

Replace the resource files, keeping your own `config.lua` and `locales/` folder if you customised them. Add the two new locale keys, `njgive_usage` and `njgive_success`, to any custom locale file (see Locales below) — they are not present in 1.0.0 locale files. Note that **the default language changed from Spanish to English** in 1.0.1: set `Config.Locale = 'es'` explicitly (or any other supported language) if your server was relying on the old Spanish default.

***

## 🔧 Configuration

All settings live in `config.lua`, which is listed in `escrow_ignore` and ships unencrypted.

| Config key                | Type            | Default                      | Description                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------- | --------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Locale`           | string          | `'en'`                       | Language for the notification strings. One of `en`, `es`, `de`, `fr`, `it`, `pt`, `zh` — each is a file in `locales/`. An unknown value falls back to `en`. **Does not translate the NUI panel** — see Locales.                                                                                                                                                                         |
| `Config.Framework`        | string          | `'esx'`                      | `'esx'` or `'qbcore'`. Selects every framework branch in both `client/client.lua` and `server/server.lua`: how the player object is fetched, how the identifier is read (`identifier` vs `citizenid`), how jobs are set (`xPlayer.setJob` vs `Player.Functions.SetJob`), how labels are resolved, and which job-change hooks are registered. Any other value leaves the resource inert. |
| `Config.Theme`            | string          | `'purple'`                   | NUI colour theme: `'purple'`, `'orange'`, `'green'`, `'red'`, `'blue'`, `'gold'`. Pushed to the NUI on open and applied as `<html data-theme="…">`. `'purple'` is the base `:root` palette (there is no separate `data-theme="purple"` block — any unrecognised value therefore also renders as purple).                                                                                |
| `Config.OpenKey`          | string          | `'F5'`                       | Key that opens the panel, passed to `RegisterKeyMapping` as the default binding. Changing it and restarting the resource applies the new default only to players who have not rebound it themselves — FiveM key mappings are per player.                                                                                                                                                |
| `Config.MaxJobs`          | number          | `6`                          | Maximum jobs a player can hold. Enforced server-side before every insert; also sent to the NUI for the header counter. The config comment advises `6` as the practical maximum.                                                                                                                                                                                                         |
| `Config.MaxJobsOverrides` | table           | `{}` (example commented out) | Per-player cap overrides, keyed by identifier (VIPs, staff…). Matching is prefix-tolerant: the full identifier is tried first, then the bare hash after stripping any `prefix:`. Use the identifier format your framework uses (ESX: `license:…`; QBCore: the citizenid, e.g. `AB12345`). Enable `Config.Debug` to print the identifier the resource actually sees.                     |
| `Config.AllowOffDuty`     | boolean         | `true`                       | `true` shows the **GO OFF-DUTY** footer button and lets the server honour an off-duty request. `false` hides the button and makes the server reject the request with the `offduty_unavailable` notification.                                                                                                                                                                            |
| `Config.UnemployedJob`    | string          | `'unemployed'`               | Job name set when going off duty. Also used as a sentinel: a job with this name is **never** written to the roster.                                                                                                                                                                                                                                                                     |
| `Config.UnemployedGrade`  | number          | `0`                          | Grade set alongside `UnemployedJob`.                                                                                                                                                                                                                                                                                                                                                    |
| `Config.JobLabels`        | table           | `{}` (example commented out) | Overrides the framework's job label, keyed by job name: `['police'] = 'Police Department'`. Applied both when a job is captured into the database and when labels are resolved. Leave empty to use framework labels.                                                                                                                                                                    |
| `Config.GradeLabels`      | table of tables | `{}` (example commented out) | Overrides grade labels, keyed by job name then by **numeric** grade: `['police'] = { [0]='Recruit', [1]='Officer' }`. Leave empty to use framework grade labels.                                                                                                                                                                                                                        |
| `Config.NotifyStyle`      | string          | `'native'`                   | Notification backend: `'native'` (GTA `SetNotificationTextEntry` / `DrawNotification`, with `~r~`/`~g~`/`~b~` colour prefixes applied by type), `'esx'` (`ESX.ShowNotification`), `'qb'` (`QBCore.Functions.Notify`), `'ox'` (`exports.ox_lib:notify`, title `Jobs`). Any unrecognised value falls through to `native`.                                                                 |
| `Config.Debug`            | boolean         | `false`                      | Prints server-side diagnostics: the identifier seen when no cap override matched, and every roster register/update/cap-hit. Useful for filling in `MaxJobsOverrides`. Turn it off in production.                                                                                                                                                                                        |

### Example

```lua
Config.Locale    = 'en'
Config.Framework = 'qbcore'
Config.Theme     = 'blue'
Config.MaxJobs   = 4

Config.MaxJobsOverrides = {
    ['ABC12345'] = 8,   -- QBCore citizenid
}

Config.JobLabels   = { ['police'] = 'LSPD' }
Config.GradeLabels = { ['police'] = { [0] = 'Cadet', [1] = 'Officer', [2] = 'Sergeant' } }
```

### Nested table structures

```lua
-- Per-player job cap. The key is the identifier; the value is that player's cap.
-- Matching order: exact key, then the bare hash with the prefix stripped.
Config.MaxJobsOverrides = {
    ['license:1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b'] = 10,  -- ESX, full identifier
    ['1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b']         = 10,  -- same player, bare hash
    ['AB12345']                                          = 8,   -- QBCore citizenid
}

-- Job label overrides. Key = job name as your framework knows it.
Config.JobLabels = {
    ['police']    = 'Los Santos Police Department',
    ['ambulance'] = 'EMS',
}

-- Grade label overrides. Key = job name, then NUMERIC grade level.
-- Grades not listed fall back to the framework's own label.
Config.GradeLabels = {
    ['police'] = {
        [0] = 'Cadet',
        [1] = 'Officer',
        [2] = 'Sergeant',
        [3] = 'Lieutenant',
        [4] = 'Chief',
    },
}
```

### Label resolution order

Understanding this avoids most "wrong label" tickets. For both the job label and the grade label:

1. `Config.JobLabels[job]` / `Config.GradeLabels[job][grade]` — your override always wins.
2. The framework's label — `ESX.GetJob(job).label` and `.grades[grade].label` or `.name`; on QBCore, `QBCore.Shared.Jobs[job].label` and `.grades[tostring(grade)].name`.
3. The raw job name, and `tostring(grade)` for the grade.

Labels are resolved **at capture time** and stored in the `job_label` / `grade_label` columns, so the panel shows what was true when the job was granted. Renaming a job in your framework does not retroactively relabel existing rows — a promotion or re-hire refreshes them.

***

## 🌐 Locales & Editable Strings

**Languages included: 7** — `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`. Each is one file in `locales/`, all are listed in `escrow_ignore` (`locales/*.lua`), and all seven define the same ten keys with no gaps. **English is the default language** as of 1.0.1 (it was Spanish in 1.0.0). If the selected language is missing a key, the English text is used.

| Where                                                 | What                                                                                   | Editable without escrow?                                                                                                |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `locales/<lang>.lua`                                  | All ten notification strings, per language.                                            | ✅ Yes — `escrow_ignore` covers `locales/*.lua`.                                                                         |
| `config.lua`                                          | Every setting, plus `JobLabels` / `GradeLabels`, i.e. all job and grade display names. | ✅ Yes — in `escrow_ignore`.                                                                                             |
| `locales.lua`                                         | The `_T()` lookup helper (contains no strings itself).                                 | ✅ Yes as of 1.0.1 — added to `escrow_ignore`.                                                                           |
| `shared/functions.lua`                                | Editable integration hooks (see Developer API).                                        | ✅ Yes as of 1.0.1 — added to `escrow_ignore`, and now actually loaded by the manifest.                                  |
| `html/index.html`, `html/script.js`, `html/style.css` | All NUI panel text and the whole visual design, in English by default.                 | ✅ Yes — NUI assets ship as plain files and are never encrypted. ⚠️ Still **not** part of the locale system — see below. |
| `server/server.lua`, `client/client.lua`              | Internal strings.                                                                      | ❌ No — inside the escrow.                                                                                               |

The ten locale keys:

| Key                   | Used for                                                        | Format args                     |
| --------------------- | --------------------------------------------------------------- | ------------------------------- |
| `cant_delete_active`  | Rejecting deletion of the active job.                           | —                               |
| `job_added`           | Telling a player a job was granted via `/njgive`.               | `%s` = job label                |
| `job_deleted`         | Confirming a deletion.                                          | —                               |
| `job_not_owned`       | Rejecting a switch to a job not in the roster.                  | —                               |
| `keymap`              | The label shown in FiveM's own key-bindings menu.               | —                               |
| `njgive_success`      | **(new in 1.0.1)** Confirms to the admin that `/njgive` worked. | `%s`, `%s` = job, target player |
| `njgive_usage`        | **(new in 1.0.1)** Usage line for `/njgive`.                    | —                               |
| `no_permission`       | `/njgive` used by a non-admin.                                  | —                               |
| `offduty_unavailable` | Off duty requested while `AllowOffDuty = false`.                | —                               |
| `player_not_found`    | `/njgive` against an invalid player id.                         | —                               |

Before 1.0.1, the `/njgive` usage and confirmation text were two strings hardcoded in Spanish **inside the escrow** in `server/server.lua`, which a customer could not change at all. As of 1.0.1 they are ordinary locale keys (`njgive_usage` / `njgive_success`) translated into all seven languages — if you maintain a custom locale file from a 1.0.0 install, add these two keys when updating.

Adding a language: copy `locales/en.lua` to `locales/<code>.lua`, change the table key on the first line (`Locales['<code>'] = {`), translate the ten values, and set `Config.Locale = '<code>'`. The `locales/*.lua` glob in both `shared_scripts` and `escrow_ignore` picks the new file up with no manifest edit.

**The NUI panel itself is still not localised.** Every string the player reads inside the panel is hardcoded in `html/index.html` and `html/script.js` and is not covered by `locales/`, so `Config.Locale` has no effect on it. In 1.0.0 this was worse than a simple limitation: the hardcoded set actually **mixed Spanish and English** — `MY JOBS`, `GO OFF-DUTY`, `ACTIVE`, `DELETE JOB`, `YOU ARE UNEMPLOYED` etc. were English, while `SELECCIONAR`, `En servicio` and the `Grado <n>` grade fallback were Spanish, so even an `en`-configured server showed two Spanish buttons. **As of 1.0.1, the NUI's default language changed from Spanish to English** (`SELECCIONAR` → `SELECT`, `En servicio` → `ON DUTY`, `Grado <n>` → `Grade <n>`), so the panel is now internally consistent in English — but it remains a single hardcoded language, not driven by `Config.Locale`/`locales/`. A server running in another language still needs to hand-edit `html/script.js` to translate the panel text, since NUI files are never encrypted.

***

## 🔗 Compatibility

| System                                                            | How it integrates                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frameworks**                                                    | ESX Legacy and QBCore, selected with `Config.Framework` — no code editing. Every framework-specific operation is branched: identifier (`xPlayer.identifier` / `PlayerData.citizenid`), current job (`xPlayer.job` / `PlayerData.job`), setting a job (`xPlayer.setJob(job, grade)` / `Player.Functions.SetJob(job, grade)`), label lookup (`ESX.GetJob` / `QBCore.Shared.Jobs`), admin check (`xPlayer.getGroup()` / `QBCore.Functions.HasPermission`), and which job-change hooks are registered. |
| **Database**                                                      | **`oxmysql` only.** The manifest loads `@oxmysql/lib/MySQL.lua` directly. `mysql-async` will not work without replacing that line and the relevant `MySQL.*` call sites.                                                                                                                                                                                                                                                                                                                           |
| **Notifications**                                                 | Four backends via `Config.NotifyStyle`: `native` (GTA natives with automatic `~r~`/`~g~`/`~b~` prefixing by notification type), `esx` (`ESX.ShowNotification`), `qb` (`QBCore.Functions.Notify`), `ox` (`ox_lib`). All notifications are generated server-side and relayed to the client through `nexus_multijob:notify`, so the backend is resolved on the client. **`nexus_notify` is not one of the options**, unlike the rest of the catalogue.                                                |
| **Job centres / hiring scripts**                                  | Fully compatible with no integration work, as long as they set jobs through the framework (which fires `esx:setJob` / `QBCore:Server:SetJob`). A script that writes the `users`/`players` table directly without firing the event will not be captured — the player's job is still picked up the next time they open the panel, via `EnsureCurrentJobSaved`.                                                                                                                                       |
| **Other multijob resources**                                      | Do **not** run two. Both would hook the same job-change events and fight over the active job.                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Duty / on-off-duty systems (`qb-policejob`, ESX duty scripts)** | Compatible, but be aware they are a different concept: those toggle a duty flag on one job, while this switches _which_ job is active. If your duty script defines separate off-duty jobs (`police` / `offpolice`), both will be captured as separate roster entries.                                                                                                                                                                                                                              |
| **TextUI / target / keys / fuel / banking / inventory**           | Not used. The panel is a standalone NUI opened by a keybind or a command; there is no world interaction, no blip and no NPC.                                                                                                                                                                                                                                                                                                                                                                       |
| **`ox_lib`**                                                      | Optional, only for `Config.NotifyStyle = 'ox'`.                                                                                                                                                                                                                                                                                                                                                                                                                                                    |

**Auto-save sources by framework**

| Source                        | ESX                                           | QBCore                                                                       |
| ----------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------- |
| Player loads in               | `esx:playerLoaded`                            | `QBCore:Server:PlayerLoaded`                                                 |
| Job changed by another script | `esx:setJob`                                  | `QBCore:Server:SetJob` (server-only handler as of 1.0.1 — see Developer API) |
| First panel open              | Current job saved via `EnsureCurrentJobSaved` | Current job saved via `EnsureCurrentJobSaved`                                |
| Admin                         | `/njgive`                                     | `/njgive`                                                                    |

***

## 💻 Developer API

> Read directly out of `client/client.lua`, `server/server.lua`, `shared/functions.lua`, `locales.lua` and `html/script.js`.

### Client Exports

**None.** `client/client.lua` declares no `exports(...)` and the manifest has no `exports` block. To open or close the panel from another resource, use the command or the events below.

```lua
-- Open/close the panel from your own client code
ExecuteCommand('nexus_jobs')

-- Or drive it directly: ask the server for the roster, which triggers the open
TriggerServerEvent('nexus_multijob:getJobs')
```

### Server Exports

**None.** `server/server.lua` declares no exports.

To read a player's roster from your own server code, query the table directly:

```lua
MySQL.query('SELECT job, job_grade, job_label, grade_label FROM nexus_multijob WHERE identifier = ?',
    { identifier }, function(rows)
        for _, row in ipairs(rows or {}) do
            print(row.job_label, row.grade_label)
        end
    end)

-- or, with the oxmysql await style:
local function HasMultiJob(identifier, job)
    return MySQL.scalar.await('SELECT 1 FROM nexus_multijob WHERE identifier = ? AND job = ?', { identifier, job }) ~= nil
end
```

### Events — Emitted

| Event                        | Side            | Payload                                                           | When                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------- | --------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_multijob:getJobs`     | client → server | _(none)_                                                          | `OpenMenu()` — fired when the player presses the open key or runs `/nexus_jobs` while the panel is closed. Saves the current job if needed, then the server answers with `receiveJobs`.                                                                                                                                                                    |
| `nexus_multijob:setJob`      | client → server | `job` (string), `grade` (number)                                  | The _Select_ button in the NUI (`setJob` callback) and the _Go Off-Duty_ button (`offDuty` callback, which sends `Config.UnemployedJob` / `Config.UnemployedGrade`). **The `grade` argument is advisory only** — the server re-reads the grade for `job` from the database and ignores whatever the client sent, so a tampered client cannot self-promote. |
| `nexus_multijob:removeJob`   | client → server | `job` (string)                                                    | Confirming a deletion in the confirmation drawer (`removeJob` callback). Refused for the player's active job.                                                                                                                                                                                                                                              |
| `nexus_multijob:receiveJobs` | server → client | `jobs` (array of rows), `currentJob` (string), `maxJobs` (number) | Answer to `getJobs`. Opens the panel. Ignored by the client if the panel is already open.                                                                                                                                                                                                                                                                  |
| `nexus_multijob:refreshJobs` | server → client | `jobs`, `currentJob`, `maxJobs`                                   | Pushed after a successful job switch or a successful deletion. Ignored by the client if the panel is closed.                                                                                                                                                                                                                                               |
| `nexus_multijob:notify`      | server → client | `msg` (string), `ntype` (string)                                  | Every server-side notification. `ntype` is `'info'`, `'success'` or `'error'`.                                                                                                                                                                                                                                                                             |

`jobs` rows are raw `nexus_multijob` table rows: `{ id, identifier, job, job_grade, job_label, grade_label, added_at }`. `currentJob` is the **job name only**.

### Events — Listened

| Event                        | Side            | Payload                        | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ---------------------------- | --------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `nexus_multijob:getJobs`     | server          | _(none)_                       | Runs `EnsureCurrentJobSaved` to capture the player's current job if missing, then replies with `receiveJobs`.                                                                                                                                                                                                                                                                                                                                                                                          |
| `nexus_multijob:setJob`      | server          | `job`, `grade` (grade ignored) | Rejects off duty when `AllowOffDuty = false`; for any other job, verifies the player actually owns it and replies `job_not_owned` if not; otherwise applies the **grade stored in the database**, calls the framework's set-job function and schedules a refresh.                                                                                                                                                                                                                                      |
| `nexus_multijob:removeJob`   | server          | `job`                          | Refuses if `job` is the player's active job (notifies `cant_delete_active` and resyncs the UI); otherwise deletes the row and, if a row was affected, notifies `job_deleted` and pushes a refresh.                                                                                                                                                                                                                                                                                                     |
| `nexus_multijob:refreshJobs` | server          | _(none)_                       | A client-callable resync that just re-sends the player's own roster.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `nexus_multijob:receiveJobs` | client          | `jobs, currentJob, maxJobs`    | Sets `isMenuOpen`, grabs NUI focus and sends `open` to the NUI with `jobs`, `currentJob`, `allowOffDuty`, `maxJobs` and `theme`.                                                                                                                                                                                                                                                                                                                                                                       |
| `nexus_multijob:refreshJobs` | client          | `jobs, currentJob, maxJobs`    | Sends `refresh` to the NUI for a DOM-diff update. Dropped if the panel is closed.                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `nexus_multijob:notify`      | client          | `msg, ntype`                   | Routes to the backend chosen by `Config.NotifyStyle`.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `esx:getSharedObject`        | client + server | callback                       | Framework bootstrap when `Config.Framework = 'esx'`.                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `esx:setJob`                 | server          | `source, job, lastJob`         | **Automatic capture.** Upserts the new job into the roster with resolved labels.                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `esx:playerLoaded`           | server          | `source, xPlayer, isNew`       | Captures the job the player logs in with, unless it is `Config.UnemployedJob`.                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `QBCore:Server:SetJob`       | server          | `source, jobName, grade`       | **Automatic capture** on QBCore. As of 1.0.1 this is registered with `AddEventHandler`, **not** `RegisterNetEvent` — it is a server-only handler, so a client can no longer trigger it to assign jobs to arbitrary players. You can still fire it safely from your own server-side resources with `TriggerEvent('QBCore:Server:SetJob', source, job, grade)`. The exact event name your QBCore build fires for a job change can still vary by fork/version — verify it matches your `qb-core` install. |
| `QBCore:Server:PlayerLoaded` | server          | `Player`                       | Captures the job the player logs in with on QBCore.                                                                                                                                                                                                                                                                                                                                                                                                                                                    |

### Commands & Key Mapping

| Command                            | Side   | Restricted                      | Purpose                                                                                                                                                                                                                                                                                                                               |
| ---------------------------------- | ------ | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/nexus_jobs`                      | client | No                              | Toggles the panel. Also the command bound by `RegisterKeyMapping`.                                                                                                                                                                                                                                                                    |
| `/njgive [playerid] [job] [grade]` | server | No (permission-checked in code) | Adds or updates a job in a player's roster. Allowed from the server console (`source == 0`), for ESX `admin`/`superadmin`, or for QBCore `admin`. Non-admins get the `no_permission` notification. `grade` defaults to `0`. Does **not** set the job as active — it only adds it to the roster, and it is still subject to `MaxJobs`. |

`RegisterKeyMapping('nexus_jobs', _T('keymap'), 'keyboard', Config.OpenKey)` — the panel key appears in FiveM's own key-bindings menu under the active locale's `keymap` label, so each player can rebind it. As of 1.0.1 this is the **only** way the key is detected — the 1.0.0 fallback polling thread (which used an incorrect control-id table and only checked every 500 ms) has been removed, so there is no stray input matching and no idle CPU cost while the panel is closed.

### NUI Callbacks

| Callback    | Payload          | Effect in `client/client.lua`                                                                                                                                                              |
| ----------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `close`     | `{}`             | Clears `isMenuOpen` and releases NUI focus. Fired by the X button and by the ESC handler in `index.html`.                                                                                  |
| `setJob`    | `{ job, grade }` | `TriggerServerEvent('nexus_multijob:setJob', job, grade)`. The panel stays open; the server's refresh moves the `ACTIVE` badge (using the server-side grade, regardless of what was sent). |
| `removeJob` | `{ job }`        | `TriggerServerEvent('nexus_multijob:removeJob', job)`.                                                                                                                                     |
| `offDuty`   | `{}`             | `TriggerServerEvent('nexus_multijob:setJob', Config.UnemployedJob, Config.UnemployedGrade)`.                                                                                               |

### NUI Messages (Lua → JavaScript)

| Action    | `data` payload                                       | Effect                                                                                                          |
| --------- | ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `open`    | `{ jobs, currentJob, allowOffDuty, maxJobs, theme }` | Applies `data-theme`, merges into `state`, full render, unhides the panel.                                      |
| `refresh` | `{ jobs, currentJob, allowOffDuty, maxJobs }`        | DOM-diff render: animates out removed cards, animates in new ones, syncs the rest. **Does not resend `theme`.** |
| `close`   | `{}`                                                 | Plays the `panelOut` animation and hides the panel without posting back to Lua.                                 |

### Editable Functions

**`shared/functions.lua` is loaded as of 1.0.1** — it is now included in the manifest's script lists (fixing the 1.0.0 issue where the file existed on disk but was wired into no `shared_scripts` / `client_scripts` / `server_scripts` entry, making every function below dead code). It is loaded on both client and server, is listed in `escrow_ignore`, and is provided as an **optional integration surface**: `nexus_multijob` itself does not call any of these functions internally, so using them is entirely up to your own resources.

| Function        | Signature                         | Purpose                                                                                                                                                                                                                                       |
| --------------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `T`             | `T(key, ...)`                     | Alias of `_T` from `locales.lua`, kept only for backward compatibility. The separate 1.0.0 translation implementation (which duplicated `_T` by loading `locales/<lang>.lua` itself with `LoadResourceFile` + `load()`) was removed in 1.0.1. |
| `GetActiveJobs` | `GetActiveJobs()` → table         | Returns the internal `playerId → jobData` table.                                                                                                                                                                                              |
| `SetActiveJob`  | `SetActiveJob(playerId, jobData)` | Stores job data for a player in that in-memory table.                                                                                                                                                                                         |
| `CompleteJob`   | `CompleteJob(playerId, reward)`   | Fires `multijob:jobCompleted(playerId, reward)` and clears the player's entry.                                                                                                                                                                |
| `CancelJob`     | `CancelJob(playerId)`             | Fires `multijob:jobCancelled(playerId)` and clears the player's entry.                                                                                                                                                                        |

```lua
-- Listen for the optional integration events from your own resource
AddEventHandler('multijob:jobCompleted', function(playerId, reward)
    print(('Player %s completed a job, reward %s'):format(playerId, reward))
end)
```

The `_T` function itself lives in `locales.lua` (loaded as a shared script, so available on client and server):

| Function | Signature               | Called from                                                             | Purpose                                                                                                                                                                                                                                                                 |
| -------- | ----------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `_T`     | `_T(key, ...)` → string | `client/client.lua` (`keymap`), `server/server.lua` (all notifications) | Resolves `key` against `Locales[Config.Locale]`, then `Locales['en']`, then returns the key itself. Extra arguments are passed to `string.format` inside a `pcall` (`nil` args become `''`), so a malformed format string returns the raw template instead of erroring. |

`locales.lua` itself was added to `escrow_ignore` in 1.0.1, so `_T`'s source is now editable like the rest of the editable files (it was still inside the escrow in 1.0.0, though the strings it reads from `locales/*.lua` were always unencrypted).

### Database Schema

One table, created by `nexus_multijob.sql`:

```sql
CREATE TABLE IF NOT EXISTS `nexus_multijob` (
    `id`          INT(11)      NOT NULL AUTO_INCREMENT,
    `identifier`  VARCHAR(60)  NOT NULL,
    `job`         VARCHAR(50)  NOT NULL DEFAULT 'unemployed',
    `job_grade`   INT(11)      NOT NULL DEFAULT 0,
    `job_label`   VARCHAR(100) NOT NULL DEFAULT 'Unemployed',
    `grade_label` VARCHAR(100) NOT NULL DEFAULT 'None',
    `added_at`    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_job` (`identifier`, `job`),
    KEY `identifier` (`identifier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| Column        | Type            | Notes                                                                                                                                           |
| ------------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`          | `INT(11)` AI PK | Surrogate key; not used by the resource's logic.                                                                                                |
| `identifier`  | `VARCHAR(60)`   | ESX `xPlayer.identifier` (e.g. `license:…`, `char1:…`) or QBCore `citizenid`. 60 chars comfortably fits both.                                   |
| `job`         | `VARCHAR(50)`   | Framework job name. Never `Config.UnemployedJob` — it is never written to the roster.                                                           |
| `job_grade`   | `INT(11)`       | Numeric grade level, as stored by the framework. This is the value a switch always applies, regardless of what the client requests.             |
| `job_label`   | `VARCHAR(100)`  | Display label resolved at capture time (override → framework → raw name).                                                                       |
| `grade_label` | `VARCHAR(100)`  | Grade display label resolved at capture time.                                                                                                   |
| `added_at`    | `TIMESTAMP`     | Capture time. **Also the sort order of the panel** — `ORDER BY added_at ASC`, so cards are listed oldest job first. Not updated on a promotion. |

| Index                                     | Purpose                                                                           |
| ----------------------------------------- | --------------------------------------------------------------------------------- |
| `UNIQUE KEY unique_job (identifier, job)` | Guarantees one row per player per job, so a re-hire or promotion can only update. |
| `KEY identifier`                          | Supports the roster lookup and the count query.                                   |

One row per job per player. Queries used (all through `oxmysql`): `SELECT *` ordered by `added_at` (roster), `SELECT`/`SELECT COUNT(*)` (ownership and cap checks), `INSERT` (new job), `UPDATE` (promotion), `DELETE` (removal, issued through `MySQL.update` to read the affected-row count).

### Integration Example

A dispatch resource that gates its features on the roster and reacts to job switches, using only the real surface — the framework's own job events plus a direct roster query.

```lua
-- ───────────────────────── server.lua of your resource
-- Does this player have police in their Nexus Multijob roster,
-- whether or not it is the job they are currently wearing?
local function hasRosterJob(identifier, job, cb)
    MySQL.scalar('SELECT job_grade FROM nexus_multijob WHERE identifier = ? AND job = ?',
        { identifier, job }, function(grade) cb(grade ~= nil, grade) end)
end

RegisterNetEvent('mydispatch:requestPanel', function()
    local src = source
    local xPlayer = ESX.GetPlayerFromId(src)
    if not xPlayer then return end

    hasRosterJob(xPlayer.identifier, 'police', function(owns, grade)
        if not owns then
            exports['nexus_notify']:Alert(src, 'Dispatch', 'You are not on the force.', 4000, 'error')
            return
        end
        -- They own police even if currently working as a taxi driver
        TriggerClientEvent('mydispatch:openPanel', src, grade)
    end)
end)

-- React to a switch made through the Multijob panel.
-- The panel calls the framework's own setter, so the framework event fires
-- normally — hook THAT, not a Multijob event (there is no public one).
AddEventHandler('esx:setJob', function(source, job, lastJob)
    if job.name == 'police' then
        TriggerClientEvent('mydispatch:enable', source, job.grade)
    elseif lastJob and lastJob.name == 'police' then
        TriggerClientEvent('mydispatch:disable', source)
    end
end)

-- Grant a job and let Multijob capture it automatically:
-- just use the framework setter, no Multijob call needed.
RegisterNetEvent('mydispatch:hire', function(targetId)
    local xTarget = ESX.GetPlayerFromId(targetId)
    if xTarget then
        xTarget.setJob('police', 0)   -- fires esx:setJob → captured into the roster
    end
end)

-- QBCore: register a job in the multijob list from another server script
-- (safe as of 1.0.1 — this handler is server-only, clients cannot trigger it)
TriggerEvent('QBCore:Server:SetJob', source, 'mechanic', 2)

-- ───────────────────────── client.lua of your resource
-- Open the Multijob panel from your own UI
RegisterNetEvent('mydispatch:openJobSwitcher', function()
    ExecuteCommand('nexus_jobs')
end)
```

***

## ❓ FAQ

**I set `Config.Framework` and nothing happens — no panel, no notifications.** The value must be exactly lowercase `'esx'` or `'qbcore'`. `'ESX'`, `'QBCore'` and `'qb-core'` all fall through every branch in both the client and the server, leaving `Framework` nil and the resource inert with no error. This is the single most common cause of "the script does nothing".

**I get MySQL errors about `MySQL.query` being nil, or the resource fails to start.** `oxmysql` is a hard requirement — `fxmanifest.lua` loads `@oxmysql/lib/MySQL.lua`. **`mysql-async` is not supported here**, unlike some other Nexus resources. Install `oxmysql` and make sure it is `ensure`d before `nexus_multijob`.

**The panel is empty even though I have a job.** The roster fills from job-change events, so a job granted before you installed the resource was never captured. Opening the panel is supposed to fix that — `EnsureCurrentJobSaved` captures your current job on open. If it stays empty: check `Config.Debug = true` output for the identifier, confirm the `nexus_multijob` table exists and the SQL was imported into the _right_ database, and confirm your current job is not `Config.UnemployedJob` (that job is deliberately never stored).

**A job granted by my job centre never appears.** It only gets captured if the hiring script sets the job through the framework (`xPlayer.setJob` / `Player.Functions.SetJob`), which is what fires `esx:setJob` / the QBCore job event. A script that writes the `users` or `players` table directly bypasses the hook; the job will still be picked up the next time the player opens the panel, or use `/njgive`.

**On QBCore, jobs granted mid-session by other scripts aren't always captured immediately.** The resource hooks `QBCore:Server:SetJob` (now a server-only handler as of 1.0.1), but QBCore forks/versions can differ on the exact event a given job-grant path fires — some use an `OnJobUpdate`-style event instead. If a grant doesn't show up immediately, the player still gets the job through `EnsureCurrentJobSaved` the next time they open the panel or on their next login, so nothing is lost permanently; verify which event your specific `qb-core` build fires if you need instant capture.

**`Config.MaxJobsOverrides` is not working for a player.** Turn on `Config.Debug` and have the player open the panel; the server prints the exact identifier it sees. Paste that identifier verbatim as the key. The matcher also tries the bare hash with the `prefix:` stripped, so either form works — but a typo or a different character set will not.

**The "GO OFF-DUTY" button stays visible while I am already unemployed.** It hides only when the active job name is literally `unemployed`. The NUI compares against the hardcoded string `'unemployed'`, not against `Config.UnemployedJob`, so if you changed that setting to something else (`'civil'`, `'offduty'`) the button no longer hides. This has not changed as of 1.0.1 — either keep `Config.UnemployedJob = 'unemployed'` or edit the comparison in `html/script.js`.

**I cannot delete a job — the bin button is greyed out.** That is the active-job protection. Select a different job first, then delete the old one. The server enforces this too, so a client-side bypass fails with the `cant_delete_active` notification.

**Two of the buttons used to be in Spanish even though I had `Config.Locale = 'en'` — is that fixed?** Yes, as of 1.0.1. In 1.0.0 the NUI's hardcoded text mixed Spanish and English (`SELECCIONAR`, `En servicio`, `Grado <n>` alongside English strings) regardless of `Config.Locale`, because the panel text is not driven by the locale system at all. In 1.0.1 those three strings were changed to English (`SELECT`, `ON DUTY`, `Grade <n>`), so the panel is now internally consistent — but it is still one hardcoded language baked into `html/script.js`, not translated per `Config.Locale`. Edit the NUI files directly if you need another language in the panel itself.

**Grades show as a bare number, or as "Grade 2".** The grade label could not be resolved from the framework, so it fell back. Check the job exists in `ESX.GetJob` / `QBCore.Shared.Jobs` with that grade defined, or set the label explicitly in `Config.GradeLabels['<job>'][<grade>]`. Note the key must be the **numeric** grade (`[2]`), not a string (`['2']`).

**I promoted a player but the panel still shows the old grade.** The roster row is updated on the framework's job event, and the panel only refreshes while it is open. Close and reopen the panel. If the promotion was done by writing the database directly, no event fired and the roster was not updated.

**Pressing F5 sometimes did nothing, or the menu opened from an unrelated input — is that fixed?** Yes, as of 1.0.1. In 1.0.0 the resource both registered `F5` through `RegisterKeyMapping` _and_ ran a fallback polling loop with an incorrect control-id table that only checked every 500 ms, so it could both miss real presses and respond to the wrong input. That polling thread was removed in 1.0.1 — the key now works purely through FiveM's key mapping, with no idle CPU usage while the panel is closed. Press the key, or rebind it in _Settings → Key Bindings → FiveM_, or use `/nexus_jobs`.

**Can a player set themselves to a higher grade than they earned?** No, as of 1.0.1. The server has always verified that the player **owns** the job before allowing a switch, but in 1.0.0 it then applied whatever grade the client sent, so a player who owned `police` at grade 0 could request `police` at a higher grade — this was the highest-priority security issue in the 1.0.0 incident report. As of 1.0.1 the server reads the grade from the `nexus_multijob` table itself and ignores the client's value entirely, so the panel can no longer be used to self-promote.

**`/njgive` says I have no permission even though I am an admin.** ESX checks `xPlayer.getGroup()` for exactly `admin` or `superadmin` — a custom group name (`moderator`, `owner`, `dev`) is rejected. QBCore checks `QBCore.Functions.HasPermission(source, 'admin')`. From the server console the check is bypassed entirely, so `/njgive 3 police 2` always works there.

**`/njgive` added the job but did not make it active.** By design — it only adds to the roster. The player opens the panel and selects it.

**Can I run this alongside another multijob resource?** No. Both would hook the same job-change events and compete over the active job.

**Does the panel work while dead, in a vehicle, or in another menu?** There is no state gate at all — the panel opens whenever the key is pressed. Add your own check if you need to block it during certain activities.

**What order are the job cards in?** Oldest captured job first (`ORDER BY added_at ASC`). `added_at` is set on first capture and never updated, so the order is stable across promotions.

**I updated from 1.0.0 — what do I need to change?** Replace the resource files, keep your `config.lua` and customised `locales/` files, add the `njgive_usage` / `njgive_success` keys to any custom locale file, and explicitly set `Config.Locale` if you were relying on the old Spanish default (it is now English).

### Before opening a ticket

* Make sure the resource's name is exactly **`nexus_multijob`**.
* Make sure you are using the latest version of the resource (**1.0.1**).
* Confirm `Config.Framework` is exactly `'esx'` or `'qbcore'` — lowercase.
* Confirm `oxmysql` is installed and `ensure`d **before** `nexus_multijob`, and that `nexus_multijob.sql` was imported into the database your server actually uses.
* Confirm your framework (`es_extended` / `qb-core`) starts before this resource.
* Set `Config.Debug = true`, reproduce the problem, and include the server console output — it prints the identifier, every roster register/update and every cap hit.
* Check F8 (client) and the server console for errors mentioning `nexus_multijob`.
* Review this FAQ page.

***

## 📋 Changelog

**Current version: 1.0.1** (`fxmanifest.lua`)

### 1.0.1

* **Security:** switching job now applies the grade stored in the database instead of the grade sent by the client, closing a self-promotion exploit.
* **Security:** `QBCore:Server:SetJob` is now registered as a server-only handler (`AddEventHandler`, not `RegisterNetEvent`); clients can no longer trigger it directly to assign jobs to other players.
* **Fix:** `shared/functions.lua` is now included in the manifest's script lists, so `GetActiveJobs`, `SetActiveJob`, `CompleteJob` and `CancelJob` exist and run at runtime instead of being dead code.
* **Fix:** removed the duplicate translation system in `shared/functions.lua`; `T()` is now a plain alias of `_T()` from `locales.lua`.
* **Fix:** removed the fallback key-polling thread that used an incorrect control-id table and only checked every 500 ms; the panel now opens purely through FiveM's `RegisterKeyMapping`, which also removes its idle CPU cost while closed.
* **Language:** English is now the default language, for the config fallback and for the NUI panel's previously mixed-language strings (`SELECCIONAR` → `SELECT`, `En servicio` → `ON DUTY`, `Grado <n>` → `Grade <n>`). Servers relying on the old Spanish default should set `Config.Locale = 'es'` explicitly.
* **Locales:** the `/njgive` usage and success messages, previously hardcoded in Spanish inside the escrowed `server/server.lua`, are now the editable locale keys `njgive_usage` / `njgive_success`, translated into all 7 languages.
* **Escrow:** `locales.lua`, `shared/functions.lua` and the NUI files are now listed in `escrow_ignore`, making them fully editable without an escrow exemption.

### 1.0.0

* Initial release. Persistent per-player job roster on `oxmysql`, automatic capture from ESX and QBCore job events, server-enforced job cap with per-player overrides, custom NUI panel with optimistic updates and DOM-diff refresh, delete confirmation drawer, active-job protection, one-click off duty, six colour themes, configurable open key with `RegisterKeyMapping`, `/njgive` admin command, four notification backends, and seven languages (Spanish default).
