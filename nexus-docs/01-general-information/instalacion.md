---
description: >-
  On this page, you'll learn how to install most of our resources, take into
  account that some of our systems may have special needs so revise each
  script's docs carefully!
icon: gears
---

# Installation & Compatibility

## 📥 Universal Installation

1. Download the resource from your **own** cfx.re portal.
2. Unzip the folder and use the **unzipped** folder. **DO NOT** rename the folder's name, if so the script **won't work**.
3. Add the unzipped folder to your `resources` folder.
4. Add `ensure nexus_xxxxx` to your `server.cfg`, respecting the dependency order if indicated by the script.
5. Open the script's `config.lua` and adapt it to your server's needs (framework, language, notify...).
6. Restart the resource or server. We recommend you to restart the whole server since it ensures that the script settles correctly.

{% hint style="danger" %}
All of our scripts include a resource name check on startup. If the folder is not named exactly like the original resource, the script will stop executing and display an error in the console.
{% endhint %}

## 🧩 Modular Compatibility System

Our resources do not enforce using an specific dependecy : every function is customizable to guarantee a personalized setup, with the least effort.

| System    | Current pre-configured systems                  |
| --------- | ----------------------------------------------- |
| Framework | `esx`, `qbcore`                                 |
| Notifys   | `nexus`, `okok`, `esx`, `mythic`                |
| TextUI    | `nexus`, `okok`, `esx`, `custom`                |
| Garage    | `nexus`, `cd_garage`, `custom`, `default`       |
| Fuel      | `legacyfuel`, `custom`, `ninguno`               |
| Target    | `ox_target`, `qb-target`, `draw3dtext(nexus's`) |

This is controlled by the  `config.lua`, for exmaple:

```lua
Config.NotifySystem = 'nexus' -- 'nexus' | 'okok' | 'esx' | 'mythic'
Config.TextUISystem = 'nexus' -- 'nexus' | 'okok' | 'esx' | 'custom'
```

## 🧠 functions.lua & locales.lua

Almost every script comes with a `functions.lua` **file unencrypted**, with the principal functions pre-configured for a seemless start (with room for customization), apart from a `locales.lua` file where lay the script's traductions (we currently support english, spanish, french, german, italian and chinease. You can add your own or edit an existing one).
