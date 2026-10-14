---
description: >-
  How to install most of our resources. Some scripts have special needs, so read
  each script's page carefully.
icon: gears
---

# Installation & Compatibility

## Universal installation

1. Download the resource from your own cfx.re portal.
2. Unzip the folder and use the unzipped folder. **Do not rename it**, or the script will not work.
3. Add the folder to your `resources` directory.
4. Add `ensure nexus_xxxxx` to your `server.cfg`, respecting the dependency order indicated by the script.
5. Open the script's configuration file and adapt it to your server (framework, language, notifications and so on).
6. Restart the resource or the server. We recommend restarting the whole server so the script loads correctly.

{% hint style="danger" %}
All Nexus Scripts include a resource name check on startup. If the folder is not named exactly like the original resource, the script stops and prints an error in the console.
{% endhint %}

## Modular Compatibility System

Our resources do not force a specific dependency. Every bridge is customizable so you can adapt the script to your setup with the least effort.

| System        | Pre-configured options                                    |
| ------------- | --------------------------------------------------------- |
| Framework     | `esx`, `qbcore`                                           |
| Notifications | `nexus` (Nexus Notify), `okok`, `esx`, `mythic`, `custom` |
| Text prompts  | `nexus` (Nexus TextUI), `okok`, `esx`, `custom`           |
| Vehicle keys  | `cd_garage`, `custom`, `default` (no keys)                |
| Fuel          | `legacyfuel`, `custom`, none                              |
| Target        | `ox_target`, `qb-target`, `qtarget`                       |

The available options differ from script to script. The valid values are listed in the configuration section of each script's page.

The systems are selected in the configuration file, for example:

```lua
Config.NotifySystem = 'nexus'
Config.TextUISystem = 'okok'
```

## functions.lua and locales

Almost every script includes an unencrypted `functions.lua` with the main bridge functions preconfigured, ready to be extended with your own systems. Translations are stored in `locales/` files. Most scripts include English, Spanish, French, German, Italian, Portuguese and Chinese. You can edit them or add your own language.
