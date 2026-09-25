# 🔔 nexus_notify

## 📝 Descripción

`nexus_notify` es el sistema de notificaciones propio de Nexus Scripts: ligero, con un diseño cuidado acorde a la identidad visual de Nexus, y pensado para que cualquier script (propio o de terceros) pueda integrarlo fácilmente.

## ✨ Características

- Varios tipos de notificación: éxito, error, información, aviso.
- Diseño propio con la identidad visual de Nexus Scripts.
- Fácil integración en otros scripts mediante exports.
- Duración y posición configurables.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | No — standalone |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md).

## 🔧 Configuración

```lua
Config.Position = 'top-right'
Config.Duration = 5000 -- milisegundos
```

## 🧩 Cómo integrarlo en tus propios scripts

```lua
exports['nexus_notify']:SendNotify('success', 'Título', 'Mensaje de la notificación')
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | Standalone (compatible con ESX y QBCore) |
| Uso | Exportable a cualquier script propio o de terceros |

## ❓ Preguntas frecuentes

**P: ¿Puedo usar `nexus_notify` en scripts que no sean de Nexus Scripts?**
R: Sí, está pensado para integrarse en cualquier script mediante exports.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
