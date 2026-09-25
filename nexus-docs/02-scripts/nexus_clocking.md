# 🕒 nexus_clocking

## 📝 Descripción

`nexus_clocking` añade un sistema de fichaje de entrada y salida para los trabajos de tu servidor, permitiendo controlar el tiempo trabajado y pagar a los jugadores en función de las horas fichadas.

## ✨ Características

- Fichaje de entrada y salida desde una interfaz sencilla.
- Registro del tiempo trabajado por jugador.
- Pago configurable en función del tiempo fichado.
- Integrable con cualquier trabajo de tu servidor.

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
Config.PayPerHour = 50
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| Trabajos | Compatible con cualquier trabajo del servidor |

## ❓ Preguntas frecuentes

**P: ¿Puedo cambiar el pago por hora fichada?**
R: Sí, desde `Config.PayPerHour` en `config.lua`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
