# 📋 Requisitos Generales

Antes de instalar cualquier script de Nexus Scripts, asegúrate de cumplir estos requisitos mínimos.

## ✅ Requisitos base

| Requisito | Detalle |
|---|---|
| Framework | ESX Legacy o QBCore (última versión estable) |
| Servidor FiveM | Build reciente de FXServer (recomendada por CFX) |
| Escrow | Cuenta de servidor autorizada con el sistema de escrow de cfx.re |
| Licencia | Compra válida a través de nuestra tienda en Tebex |

## 🔌 Dependencias habituales

La mayoría de nuestros scripts pueden depender de uno o varios de estos recursos, según el caso:

- Un framework: **ESX** o **QBCore**
- Un sistema de notificaciones (propio `nexus_notify`, `okokNotify`, `ESX`, `mythic_notify`...)
- Un sistema de TextUI (propio `nexus_TextUI`, `okokTextUI`, `ESX`...)
- Un sistema de interacción/target (`ox_target`, `qb-target`...) — opcional según el script
- `oxmysql` en los scripts que requieran guardar datos en base de datos

{% hint style="warning" %}
Revisa siempre la sección **Requisitos** dentro de la página de cada script — algunos recursos son obligatorios y otros son totalmente opcionales gracias a nuestro sistema de compatibilidad modular.
{% endhint %}

## 🧩 Filosofía de compatibilidad

Todos los scripts de Nexus Scripts están diseñados con **máxima modularidad**: puedes elegir qué sistema quieres usar para notificaciones, TextUI, llaves de vehículos, etc. simplemente cambiando una opción en el `config.lua`, sin tener que tocar ninguna función. Consulta [⚙️ Instalación y Compatibilidad](instalacion.md) para más detalle.
