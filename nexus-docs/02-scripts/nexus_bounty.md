# 🎯 nexus_bounty

## 📝 Descripción

`nexus_bounty` añade un sistema de recompensas: cualquier jugador puede poner precio a la cabeza de otro, y quien complete la recompensa recibe el pago correspondiente. Ideal para servidores roleplay con dinámicas de criminalidad y caza de recompensas.

## ✨ Características

- Creación de recompensas con importe personalizado.
- Marcador en el mapa para el objetivo activo.
- Pago automático al completar la recompensa.
- Historial de recompensas activas y completadas.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | Sí |
| Sistema económico del framework | Sí |
| Sistema de notificaciones | Sí (elige el tuyo) |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md).

## 🔧 Configuración

```lua
Config.Framework = 'esx' -- 'esx' | 'qbcore'
Config.NotifySystem = 'nexus'
Config.MinBounty = 500
Config.MaxBounty = 50000
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| Economía | Sistema bancario nativo de ESX / QBCore |

## ❓ Preguntas frecuentes

**P: ¿Se puede limitar el importe mínimo y máximo de una recompensa?**
R: Sí, desde `Config.MinBounty` y `Config.MaxBounty`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
