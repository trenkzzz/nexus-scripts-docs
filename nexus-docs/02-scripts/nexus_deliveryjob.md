# 🚚 nexus_deliveryjob

## 📝 Descripción

`nexus_deliveryjob` añade un trabajo de repartidor completo: rutas de entrega generadas dinámicamente, vehículo asignado y pago por cada entrega completada con éxito.

## ✨ Características

- Rutas de entrega generadas de forma dinámica.
- Vehículo de trabajo asignado automáticamente.
- Pago configurable por entrega completada.
- Marcadores en el mapa para cada punto de entrega.

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
Config.KeySystem = 'nexus' -- 'nexus' | 'cd_garage' | 'custom' | 'default'
Config.PayPerDelivery = 75
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| TextUI | nexus_TextUI, okokTextUI, ESX, custom |
| Llaves de vehículo | nexus, cd_garage, custom, default |

## ❓ Preguntas frecuentes

**P: ¿Puedo cambiar el pago por entrega?**
R: Sí, desde `Config.PayPerDelivery` en `config.lua`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
