---
icon: clipboard-list
---

# General Requirements

Before obtaining any nexus resource, ensure you meet these minimal requirements .

## ✅ Basical Requirements

| Requirement      | Detail                                               |
| ---------------- | ---------------------------------------------------- |
| Framework        | ESX Legacy o QBCore (the newest version posible)     |
| Server's Version | Recent FXserver build.                               |
| Portal Account   | Having a valid cfx.re account.                       |
| License          | Hold a genuine script license bought from our tebex. |

## 🔌 Frequent Dependecies

Most of our scripts have dependecies, apart from the standalone ones:

* Framework: **ESX** o **QBCore**
* Notify (recommended `nexus_notify`, `okokNotify`, `ESX`, `mythic_notify`...)
* TextUI (recommended `nexus_TextUI`, `okokTextUI`, `ESX`...)
* Target (`ox_target`, `qb-target`...) — may vary from script to script.
* `oxmysql` on every script that manage databases.

{% hint style="warning" %}
Always check the **Requirements** section on each script's page — some resources are mandatory, while others are completely optional thanks to our modular compatibility system.
{% endhint %}

## 🧩 Compatibility Philosophy

All Nexus Scripts are designed with maximum modularity: you can choose which system you want to use for notifications, TextUI, garage, etc., simply by changing an option in `config.lua`, often without having to touch any code functions. See [⚙️ Installation & Compatibility](instalacion.md) for more details.
