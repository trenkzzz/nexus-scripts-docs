# 🤖 nexus_helpcenter

## 📝 Descripción

`nexus_helpcenter` es un asistente de ayuda con inteligencia artificial (basado en Gemini) que actúa como centro de información de tu servidor. Interfaz hecha a mano y **standalone**, sin depender de ningún framework, para que cualquier jugador pueda resolver dudas sobre el servidor al instante.

## ✨ Características

- Asistente con IA que responde preguntas sobre tu servidor.
- Hub centralizado de información e instrucciones del servidor.
- Interfaz propia diseñada a medida.
- Standalone: no requiere ESX ni QBCore para funcionar.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| Framework (ESX/QBCore) | No — standalone |
| Sistema de notificaciones | Opcional |
| API key de IA (Gemini) | Sí |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md) y añade tu clave de API en `config.lua`.

## 🔧 Configuración

```lua
Config.AIApiKey = 'TU_API_KEY_AQUI'
Config.NotifySystem = 'nexus'
Config.ServerInfo = {
    -- Información del servidor que usará el asistente
}
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | Standalone (no requiere ESX ni QBCore) |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |

## ❓ Preguntas frecuentes

**P: ¿Necesito una API key propia?**
R: Sí, `nexus_helpcenter` funciona con tu propia clave de API de Gemini.

**P: ¿Funciona sin framework?**
R: Sí, es completamente standalone.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
