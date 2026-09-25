# 🔊 nexus_boombox

## 📝 Descripción

`nexus_boombox` añade una radio portátil que tus jugadores pueden llevar consigo, colocar en el mundo y compartir música con quien esté cerca. Interfaz cuidada y pensada para integrarse con tu servidor sin complicaciones.

## ✨ Características

- Radio física que se puede colocar y recoger del mundo.
- Reproducción por emisoras o enlaces de audio.
- Volumen ajustable y alcance de sonido configurable.
- Otros jugadores cercanos escuchan la música en tiempo real.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | Sí |
| Sistema de inventario (`ox_inventory`, `esx_inventory`...) | Sí |
| Sistema de notificaciones | Sí (elige el tuyo) |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md). Recuerda añadir el ítem de la boombox a tu sistema de inventario con el nombre indicado en `config.lua`.

## 🔧 Configuración

```lua
Config.Framework = 'esx' -- 'esx' | 'qbcore'
Config.NotifySystem = 'nexus'
Config.MaxRange = 15.0 -- Distancia máxima de escucha
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| Inventario | ox_inventory, esx_inventory, qb-inventory |

## ❓ Preguntas frecuentes

**P: ¿Puedo cambiar el alcance del sonido?**
R: Sí, desde `Config.MaxRange` en `config.lua`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
