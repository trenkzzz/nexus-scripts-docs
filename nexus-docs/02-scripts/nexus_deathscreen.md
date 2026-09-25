# 💀 nexus_deathscreen

## 📝 Descripción

`nexus_deathscreen` añade una pantalla de muerte completa: cuenta atrás para el respawn, estado del jugador (sangrado, vida, gravedad de las heridas) y la opción de pedir ayuda para que otros jugadores puedan localizarte.

## ✨ Características

- Pantalla de muerte con cuenta atrás para respawn configurable.
- Opción de pedir ayuda y marcar tu ubicación en el mapa.
- Información visual del estado del jugador.
- Integrable con trabajos de emergencias (EMS/médicos).

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | Sí |
| Sistema de notificaciones | Sí (elige el tuyo) |
| Job de EMS/médicos | Opcional |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md).

## 🔧 Configuración

```lua
Config.Framework = 'esx' -- 'esx' | 'qbcore'
Config.NotifySystem = 'nexus'
Config.RespawnTime = 300 -- segundos
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| EMS/Médicos | Integrable con el job de tu servidor |

## ❓ Preguntas frecuentes

**P: ¿Puedo cambiar el tiempo de respawn?**
R: Sí, desde `Config.RespawnTime` en `config.lua`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
