# 📻 nexus_carradio

## 📝 Descripción

`nexus_carradio` añade una radio integrada en los vehículos de tu servidor, con emisoras seleccionables y sonido sincronizado por proximidad para el resto de ocupantes.

## ✨ Características

- Radio integrada en cualquier vehículo.
- Selección de emisoras mediante teclas configurables.
- Sonido sincronizado para todos los ocupantes del vehículo.
- Volumen ajustable desde la propia interfaz.

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
Config.OpenRadioKey = 'F6'
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| TextUI | nexus_TextUI, okokTextUI, ESX, custom |

## ❓ Preguntas frecuentes

**P: ¿Puedo cambiar la tecla para abrir la radio?**
R: Sí, desde `Config.OpenRadioKey` en `config.lua`.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
