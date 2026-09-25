# 🧰 nexus_multijob

## 📝 Descripción

`nexus_multijob` permite que tus jugadores tengan varios trabajos activos a la vez y cambien entre ellos sin perder su progreso, en lugar de estar limitados a un único empleo como marca el framework por defecto.

## ✨ Características

- Varios trabajos activos simultáneamente por jugador.
- Cambio rápido entre trabajos desde una interfaz sencilla.
- Compatible con los trabajos propios de Nexus Scripts y con trabajos de terceros.
- Progreso independiente por cada trabajo.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | Sí |
| Sistema de notificaciones | Sí (elige el tuyo) |
| Sistema de TextUI | Opcional |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md).

## 🔧 Configuración

```lua
Config.Framework = 'esx' -- 'esx' | 'qbcore'
Config.NotifySystem = 'nexus'
Config.MaxActiveJobs = 3
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| TextUI | nexus_TextUI, okokTextUI, ESX, custom |
| Otros scripts de empleo | nexus_jobcenter, nexus_deliveryjob, nexus_minerjob |

## ❓ Preguntas frecuentes

**P: ¿Cuántos trabajos puede tener activos un jugador a la vez?**
R: Lo que definas en `Config.MaxActiveJobs`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
