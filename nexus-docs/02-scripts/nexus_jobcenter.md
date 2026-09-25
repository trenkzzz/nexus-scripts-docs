# 💼 nexus_jobcenter

## 📝 Descripción

`nexus_jobcenter` es el centro de empleos de tu servidor: un punto centralizado donde los jugadores pueden consultar todos los trabajos disponibles, sus requisitos y acceder rápidamente a ellos.

## ✨ Características

- Listado centralizado de todos los trabajos del servidor.
- Acceso rápido a cada trabajo desde una única interfaz.
- Totalmente integrable con el resto de scripts de empleo de Nexus.
- Personalizable para incluir trabajos de terceros.

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
Config.Jobs = {
    -- Lista de trabajos disponibles en el centro de empleos
}
```

## 🔗 Compatibilidad

| Sistema | Opciones |
|---|---|
| Framework | ESX, QBCore |
| Notificaciones | nexus_notify, okokNotify, ESX, mythic_notify |
| TextUI | nexus_TextUI, okokTextUI, ESX, custom |
| Otros scripts de empleo | nexus_deliveryjob, nexus_minerjob, nexus_multijob |

## ❓ Preguntas frecuentes

**P: ¿Puedo añadir trabajos que no sean de Nexus Scripts?**
R: Sí, `Config.Jobs` admite añadir cualquier trabajo de tu servidor.

## 📝 Registro de cambios

- **v1.0.0** — Lanzamiento inicial.
