# 🖥️ nexus_TextUI

## 📝 Descripción

`nexus_TextUI` es el sistema de TextUI propio de Nexus Scripts: textos de interacción en pantalla, ligeros y con un diseño acorde a la identidad visual de Nexus, listos para integrarse en cualquier script.

## ✨ Características

- Interfaz de texto ligera para acciones e interacciones.
- Diseño propio con la identidad visual de Nexus Scripts.
- Fácil integración en otros scripts mediante exports.
- Posición y estilo configurables.

## 📋 Requisitos

| Recurso | Obligatorio |
|---|---|
| ESX o QBCore | No — standalone |

## ⚙️ Instalación

Sigue los pasos generales de [⚙️ Instalación y Compatibilidad](../01-general-information/instalacion.md).

## 🔧 Configuración

```lua
Config.Position = 'bottom-center'
```

## 🧩 Cómo integrarlo en tus propios scripts

```lua
exports['nexus_TextUI']:ShowTextUI('Pulsa [E] para interactuar')
exports['nexus_TextUI']:HideTextUI()
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | Standalone (compatible con ESX y QBCore) |
| Uso | Exportable a cualquier script propio o de terceros |

## ❓ Preguntas frecuentes

**P: ¿Puedo usar `nexus_TextUI` en scripts que no sean de Nexus Scripts?**
R: Sí, está pensado para integrarse en cualquier script mediante exports.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
