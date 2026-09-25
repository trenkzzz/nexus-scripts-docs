# ⛏️ nexus_minerjob

## 📝 Descripción

`nexus_minerjob` añade un trabajo de minero completo: extracción de minerales en puntos configurables, venta de lo recolectado y progresión pensada para servidores roleplay.

## ✨ Características

- Puntos de extracción de minerales configurables.
- Venta de minerales en un punto de compra dedicado.
- Progresión y variedad de minerales según la zona.
- Interacción mediante `target` o tecla, según tu configuración.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | Sí |
| Sistema económico del framework | Sí |
| Sistema de notificaciones | Sí (elige el tuyo) |
| Sistema de TextUI | Opcional |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md).

## 🔧 Configuración

```lua
Config.Framework = 'esx' -- 'esx' | 'qbcore'
Config.NotifySystem = 'nexus'
Config.TargetSystem = 'ox_target' -- 'ox_target' | 'qb-target' | 'draw3dtext'
Config.PayPerMineral = 15
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| TextUI | nexus_TextUI, okokTextUI, ESX, custom |
| Interacción / target | ox_target, qb-target, draw3dtext (propio) |

## ❓ Preguntas frecuentes

**P: ¿Puedo cambiar el pago por mineral extraído?**
R: Sí, desde `Config.PayPerMineral` en `config.lua`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
