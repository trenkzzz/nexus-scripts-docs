---
icon: clipboard-list
---

# General Requirements

Before getting any Nexus Scripts resource, make sure you meet these minimum requirements.

## Basic requirements

| Requirement    | Detail                                                                                         |
| -------------- | ---------------------------------------------------------------------------------------------- |
| Framework      | ESX Legacy or QBCore, in the newest version possible. Some scripts are ESX only or standalone. |
| Server version | A recent FXServer build.                                                                       |
| Portal account | A valid cfx.re account.                                                                        |
| License        | A genuine script license bought from our Tebex store.                                          |

## Common dependencies

Most of our scripts have dependencies, apart from the standalone ones:

* Framework: **ESX** or **QBCore**.
* Notifications: `nexus_notify` is recommended. `okokNotify`, ESX and `mythic_notify` are also supported by most scripts.
* Text prompts: `nexus_TextUI` is recommended. `okokTextUI` and ESX are also supported by most scripts.
* Target: `ox_target`, `qb-target` or `qtarget`, depending on the script.
* `oxmysql` on every script that uses a database.

{% hint style="warning" %}
Always check the dependencies section of each script's page. Some resources are mandatory and others are optional thanks to the modular compatibility system.
{% endhint %}

## Compatibility philosophy

All Nexus Scripts are designed to be modular. You can choose the system used for notifications, text prompts, keys and more by changing an option in the configuration file, often without touching any code. See [Installation & Compatibility](instalacion.md) for more details.
