---
icon: buildings
---

# Nexus Multijob

> Let players keep several jobs at once and switch between them from a themeable panel, for ESX and QBCore.

**Version:** 1.0.1 · **Frameworks:** ESX, QBCore · **Database:** oxmysql

***

## Description

Nexus Multijob gives every player a personal, persistent job roster. Instead of losing a police grade when taking a taxi shift, the player keeps every job in a list and swaps the active one from a side panel opened with a single key. The roster size is limited per server and can be raised for specific players. The active job cannot be deleted, and going off duty is a single click.

The roster fills itself. The resource listens to the job-change events of your framework, so every time a player receives a job from any resource (job centers, MDT, admin commands), that job with its grade and labels is saved in the `nexus_multijob` table. No integration work is needed in your existing job scripts. A player whose current job predates the installation gets it saved automatically the first time the panel is opened.

The grade applied when switching is always the grade stored on the server, never the value sent by the client, so the panel cannot be used to self-promote.

***

## Features

* Personal job roster stored in the database, with job and grade labels.
* Automatic capture of jobs on login and on every job change, for ESX and QBCore.
* Switch the active job from the panel, with the grade stored for that job.
* Delete jobs from the roster with a confirmation step. The active job cannot be deleted.
* Optional off-duty button that sets the unemployed job.
* Maximum number of jobs per player, with per-player overrides.
* Job and grade label overrides for display.
* Six color themes: `purple`, `orange`, `green`, `red`, `blue`, `gold`.
* Open key registered as a rebindable FiveM key mapping.
* Admin command to add a job to a player's roster.
* Seven languages for notifications: English, Spanish, German, French, Italian, Portuguese and Chinese.
* Notification output selectable between native, ESX, QBCore and ox\_lib.
* Optional integration hooks in `shared/functions.lua`.

***

## Dependencies

| Dependency                 | Required | Notes                                       |
| -------------------------- | -------- | ------------------------------------------- |
| `oxmysql`                  | Yes      | Stores the roster.                          |
| `es_extended` or `qb-core` | Yes      | Select it with `Config.Framework`.          |
| `ox_lib`                   | No       | Only needed if `Config.NotifyStyle = 'ox'`. |

The resource folder must be named `nexus_multijob`. Any other name stops the resource on start.

***

## Installation

1. Place the `nexus_multijob` folder in your `resources` directory.
2. Do not rename the folder.
3. Import `nexus_multijob.sql` into your database. It creates the `nexus_multijob` table.
4. Add it to `server.cfg` after your framework and `oxmysql`:

```cfg
ensure oxmysql
ensure es_extended
ensure nexus_multijob
```

5. Open `config.lua` and set `Config.Framework`, `Config.Locale`, `Config.Theme`, `Config.OpenKey` and `Config.MaxJobs`.
6. Restart the server.
7. In game, press the open key (F5 by default) to open the panel.

***

## Configuration

Everything is in `config.lua`, which is not encrypted.

| Key                       | Type    | Default        | Description                                                                                                                              |
| ------------------------- | ------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Locale`           | string  | `'en'`         | Language of notifications: `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`.                                                                     |
| `Config.Framework`        | string  | `'esx'`        | `'esx'` or `'qbcore'`.                                                                                                                   |
| `Config.Theme`            | string  | `'purple'`     | Panel color theme: `purple`, `orange`, `green`, `red`, `blue`, `gold`.                                                                   |
| `Config.OpenKey`          | string  | `'F5'`         | Default key. Players can rebind it in the FiveM key bindings settings.                                                                   |
| `Config.MaxJobs`          | number  | `6`            | Default maximum number of jobs per player. Values above 6 are not recommended because the panel layout is designed for up to six cards.  |
| `Config.MaxJobsOverrides` | table   | `{}`           | Different maximum for specific players. The key is the player identifier.                                                                |
| `Config.AllowOffDuty`     | boolean | `true`         | Shows the off-duty button and allows switching to the unemployed job.                                                                    |
| `Config.UnemployedJob`    | string  | `'unemployed'` | Job name used when going off duty. This job is never stored in the roster. The panel also uses this name to detect an unemployed player. |
| `Config.UnemployedGrade`  | number  | `0`            | Grade used when going off duty.                                                                                                          |
| `Config.JobLabels`        | table   | `{}`           | Overrides the display name of a job.                                                                                                     |
| `Config.GradeLabels`      | table   | `{}`           | Overrides the display name of a grade, per job.                                                                                          |
| `Config.NotifyStyle`      | string  | `'native'`     | Notification output: `'native'`, `'esx'`, `'qb'`, `'ox'`.                                                                                |
| `Config.Debug`            | boolean | `false`        | Prints roster activity to the server console.                                                                                            |

### Per-player limits

The identifier is `license:xxxxxxxx...` on ESX (the `identifier` of the player) and the citizen id on QBCore. A key without the prefix is also accepted.

```lua
Config.MaxJobsOverrides = {
    ['license:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'] = 10,
    ['AB12345'] = 8,
}
```

### Labels

```lua
Config.JobLabels = {
    ['police'] = 'Police Department',
}

Config.GradeLabels = {
    ['police'] = { [0] = 'Recruit', [1] = 'Officer', [2] = 'Sergeant' },
}
```

Label resolution order: the override in `Config.JobLabels` / `Config.GradeLabels`, then the label provided by the framework, then the raw job name or grade number.

***

## Using the panel

| Element     | Behavior                                                                             |
| ----------- | ------------------------------------------------------------------------------------ |
| Job cards   | One card per job with job label, grade label and an ACTIVE badge on the current job. |
| Select      | Switches to that job using the grade stored in the roster.                           |
| Delete      | Opens a confirmation. Not available for the active job.                              |
| Go off duty | Sets `Config.UnemployedJob`. Shown only when `Config.AllowOffDuty` is `true`.        |
| Close       | X button or the Escape key.                                                          |

The panel texts are in `html/index.html` and `html/script.js` and are in English. They are not tied to `Config.Locale` and can be edited directly in those files.

***

## Commands

| Command                            | Side   | Description                                                                                                                                                                                                                                                                     |
| ---------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/nexus_jobs`                      | Client | Opens or closes the panel. This is the command bound to the key mapping.                                                                                                                                                                                                        |
| `/njgive [playerid] [job] [grade]` | Server | Adds a job to the roster of a player, or updates its grade. Allowed from the server console, for ESX `admin` and `superadmin` groups, and for QBCore `admin` permission. The grade defaults to `0`. The command does not make the job active, and it respects the roster limit. |

```cfg
njgive 12 police 2
```

***

## Locales

Notifications use `locales/<lang>.lua` with the language set in `Config.Locale`. Missing keys fall back to English.

| Key                   | Used for                                       | Arguments      |
| --------------------- | ---------------------------------------------- | -------------- |
| `cant_delete_active`  | Deleting the active job is refused.            | none           |
| `job_added`           | Tells a player a job was added with `/njgive`. | job label      |
| `job_deleted`         | Confirms a deletion.                           | none           |
| `job_not_owned`       | Switching to a job that is not in the roster.  | none           |
| `keymap`              | Label in the FiveM key bindings menu.          | none           |
| `njgive_success`      | Confirms `/njgive` to the admin.               | job, player id |
| `njgive_usage`        | Usage line of `/njgive`.                       | none           |
| `no_permission`       | `/njgive` used without permission.             | none           |
| `offduty_unavailable` | Off duty requested while it is disabled.       | none           |
| `player_not_found`    | `/njgive` with an invalid player id.           | none           |

To add a language, copy `locales/en.lua` to `locales/<code>.lua`, change the key in the first line to `Locales['<code>']`, translate the values and set `Config.Locale = '<code>'`. No manifest change is needed.

***

## Compatibility

| System                        | Behavior                                                                                                                                                       |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ESX                           | Uses `esx:setJob` and `esx:playerLoaded` to fill the roster, `xPlayer.setJob` to switch and the `admin` / `superadmin` groups for `/njgive`.                   |
| QBCore                        | Uses `QBCore:Server:PlayerLoaded` and `QBCore:Server:SetJob` to fill the roster, `Player.Functions.SetJob` to switch and the `admin` permission for `/njgive`. |
| Job centers, MDT, admin menus | Any resource that assigns jobs through the framework is captured automatically.                                                                                |
| Notifications                 | Selected with `Config.NotifyStyle`.                                                                                                                            |

***

## Developer API

### Exports

The resource does not declare client or server exports. To open or close the panel from another resource use the command:

```lua
ExecuteCommand('nexus_jobs')
```

To read a player's roster from your server code, query the table:

```lua
MySQL.query('SELECT job, job_grade, job_label, grade_label FROM nexus_multijob WHERE identifier = ?',
    { identifier }, function(rows)
        for _, row in ipairs(rows or {}) do
            print(row.job_label, row.grade_label)
        end
    end)

local function HasJobInRoster(identifier, job)
    return MySQL.scalar.await('SELECT 1 FROM nexus_multijob WHERE identifier = ? AND job = ?', { identifier, job }) ~= nil
end
```

To add a job to a roster from another server resource:

```lua
ExecuteCommand(('njgive %d %s %d'):format(targetId, 'police', 2))
```

Commands executed from the server through `ExecuteCommand` run with console permissions.

### Events

| Event                        | Direction        | Payload                         | Description                                                                                                                     |
| ---------------------------- | ---------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_multijob:getJobs`     | Client to server | none                            | Requests the roster. The current job is saved first if missing. The server answers with `receiveJobs`.                          |
| `nexus_multijob:setJob`      | Client to server | `job`, `grade`                  | Switches the active job. The `grade` argument is ignored, the stored grade is used. Refused if the player does not own the job. |
| `nexus_multijob:removeJob`   | Client to server | `job`                           | Deletes a job from the roster. Refused for the active job.                                                                      |
| `nexus_multijob:refreshJobs` | Client to server | none                            | Re-sends the player's own roster to an open panel.                                                                              |
| `nexus_multijob:receiveJobs` | Server to client | `jobs`, `currentJob`, `maxJobs` | Opens the panel. Ignored if it is already open.                                                                                 |
| `nexus_multijob:refreshJobs` | Server to client | `jobs`, `currentJob`, `maxJobs` | Updates the open panel after a switch or deletion. Ignored if the panel is closed.                                              |
| `nexus_multijob:notify`      | Server to client | `msg`, `ntype`                  | Shows a notification. `ntype` is `info`, `success` or `error`.                                                                  |

`jobs` contains rows of the `nexus_multijob` table: `id`, `identifier`, `job`, `job_grade`, `job_label`, `grade_label`, `added_at`. `currentJob` is the job name.

### Framework events listened

| Framework | Event                        | Purpose                                                                                  |
| --------- | ---------------------------- | ---------------------------------------------------------------------------------------- |
| ESX       | `esx:setJob`                 | Saves the new job in the roster.                                                         |
| ESX       | `esx:playerLoaded`           | Saves the job the player logs in with.                                                   |
| QBCore    | `QBCore:Server:SetJob`       | Saves the new job in the roster. Server-side handler, it cannot be triggered by clients. |
| QBCore    | `QBCore:Server:PlayerLoaded` | Saves the job the player logs in with.                                                   |

The unemployed job is never saved. When a job already exists in the roster, its grade and labels are updated. When the roster is full, a new job is not added.

### Hooks

`shared/functions.lua` contains optional helpers that your own resources can use. The resource itself does not call them.

| Function                          | Description                                                                           |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| `T(key, ...)`                     | Alias of the translation function `_T`.                                               |
| `GetActiveJobs()`                 | Returns the table of tracked jobs, indexed by player id.                              |
| `SetActiveJob(playerId, jobData)` | Stores job data for a player.                                                         |
| `CompleteJob(playerId, reward)`   | Triggers `multijob:jobCompleted` with `playerId` and `reward`, then clears the entry. |
| `CancelJob(playerId)`             | Triggers `multijob:jobCancelled` with `playerId`, then clears the entry.              |

```lua
AddEventHandler('multijob:jobCompleted', function(playerId, reward)
    print(('Player %s completed a job, reward %s'):format(playerId, reward))
end)
```

### Database

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

The identifier is the ESX `identifier` or the QBCore citizen id. There is one row per player and job.

***

## FAQ

**A job is missing from the panel.** Jobs are saved when the player receives them. If a player had the job before the resource was installed, it is saved when they open the panel with that job active. Other old jobs can be added with `/njgive`.

**A player received a job but it was not added.** The roster is full. Raise `Config.MaxJobs` or add a player entry in `Config.MaxJobsOverrides`.

**Can players promote themselves from the panel?** No. The grade is read from the database on the server and the value sent by the client is ignored.

**Why can I not delete a job?** The active job cannot be deleted. Switch to another job or go off duty first.

**The off-duty button is missing.** Set `Config.AllowOffDuty = true`.

**Does the unemployed job use a slot?** No. The unemployed job is never stored in the roster.

**The job names show in English or in the wrong language.** Job and grade labels come from the framework. Override them with `Config.JobLabels` and `Config.GradeLabels`.

**The key does not open the panel.** The key is a FiveM key mapping. If the player already rebound or cleared it, they can change it in the FiveM key bindings settings. Default keys are only applied the first time.

**How do I change the panel texts?** Edit `html/index.html` and `html/script.js`.

**I use QBCore and jobs assigned by another resource are not captured.** The resource listens to `QBCore:Server:SetJob` on the server. If your job-change resource does not trigger that event, trigger it after changing the job: `TriggerEvent('QBCore:Server:SetJob', source, jobName, grade)`.

### Before opening a ticket

* Confirm the folder is named `nexus_multijob` and the SQL was imported.
* Confirm `Config.Framework` matches your framework.
* Enable `Config.Debug` and check the server console while changing a job.
* Confirm `oxmysql` starts before this resource.
