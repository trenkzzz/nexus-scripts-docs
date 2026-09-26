# 🎯 Nexus Bounty

Sistema de recompensas y caza de jugadores para tu servidor FiveM, con una app tipo "tablet" totalmente personalizada (NUI propia) y contratos tanto contra jugadores (PVP) como contra NPCs (PVE).

## 📝 Descripción

**nexus\_bounty** añade a tu servidor un sistema de recompensas completo: cualquier jugador puede poner precio a la cabeza de otro (o de la policía, en contratos exclusivos para agentes) mediante una tablet in-game con una interfaz moderna y personalizada.

El script gestiona automáticamente la búsqueda del objetivo (esté conectado o no), el cobro y descuento del dinero, el marcado del objetivo en el mapa cuando un cazador acepta el contrato, el pago al cazarrecompensas cuando el objetivo es eliminado y una clasificación global de los mejores cazadores del servidor.

Además, incluye un sistema paralelo de contratos contra NPCs (PVE), configurable por plantillas, para que siempre haya recompensas disponibles aunque no haya suficientes jugadores conectados.

## ✨ Características

* **Contratos PVP**: cualquier jugador puede poner una recompensa sobre otro jugador, esté online o no.
* **Contratos policiales**: un tipo de contrato exclusivo que solo pueden crear (y aceptar) miembros del cuerpo de policía.
* **Contratos PVE (NPC)**: recompensas contra objetivos NPC generados a partir de plantillas configurables (modelo, ubicación, probabilidad de policía/agresividad, monto).
* **Tablet NUI propia**: interfaz completa hecha a medida (no un menú genérico) con pantalla de inicio, creación de contrato, contratos activos, panel personal y clasificación.
* **Tracking del objetivo**: al aceptar un contrato, el cazador recibe un blip del objetivo cuando se encuentra dentro de la distancia configurada, con duración y cooldown ajustables.
* **Panel personal ("Mi Panel")**: gestiona tus propios contratos creados y los que has aceptado, con opción de editar la información/imagen o cancelarlos (con devolución del dinero).
* **Clasificación global**: leaderboard con los cazarrecompensas con más contratos completados y más dinero ganado.
* **Expiración automática**: los contratos caducan solos pasado el tiempo configurado y se limpian de la base de datos sin intervención manual.
* **Estadísticas en vivo en el homescreen**: número de jugadores online y contratos activos en el propio menú principal de la tablet.
* **Sistema de pruebas integrado**: comando para generar un NPC de prueba y validar el flujo completo de contrato sin tener que esperar a un contrato real (solo con `Config.DebugMode` activo).
* **Compatible con ESX y QBCore**, seleccionable desde `Config.Framework`.
* **Notificaciones personalizables de forma nativa** vía `functions.lua` (ver la página _Personalización de Notificaciones_).

## 📋 Requisitos

* **Framework**: ESX o QBCore (se elige en `Config.Framework`).
* **Base de datos MySQL** con el wrapper `MySQL.Async` disponible (proporcionado por `mysql-async` u `oxmysql` con su capa de compatibilidad). El script depende directamente de `@mysql-async/lib/MySQL.lua`.
* Un job de policía correctamente configurado en tu framework si vas a usar los contratos de tipo "Policial".

## ⚙️ Instalación

{% stepper %}
{% step %}
### Descarga y descomprime el recurso

Descarga y descomprime la carpeta `nexus_bounty` en tu carpeta `resources`.
{% endstep %}

{% step %}
### Importa la base de datos

Importa el archivo `nexus_bounty.sql` en tu base de datos. Esto crea las tablas `bounties_active` y `bounty_leaderboard`.
{% endstep %}

{% step %}
### Configura el orden de inicio

Asegúrate de que tu recurso de MySQL (`mysql-async` u `oxmysql`) se inicia **antes** que `nexus_bounty`.
{% endstep %}

{% step %}
### Añade el recurso al servidor

Añade a tu `server.cfg`:

```cfg
ensure nexus_bounty
```
{% endstep %}

{% step %}
### Configura el framework

Abre `shared/config.lua` y ajusta al menos `Config.Framework` según tu servidor (ver la página _Configuración_).
{% endstep %}

{% step %}
### Reinicia el recurso o el servidor

Reinicia el recurso o el servidor.
{% endstep %}

{% step %}
### Abre la tablet

Usa el comando configurado en `Config.OpenCommand` (por defecto `/recompensas`) para abrir la tablet.
{% endstep %}
{% endstepper %}

## 🔧 Configuración

Toda la configuración vive en `shared/config.lua`:

| Opción                            | Descripción                                                                                                                                                                 |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Framework`                | `"ESX"` o `"QBCore"`.                                                                                                                                                       |
| `Config.DebugMode`                | Activa logs por consola y el comando `/test_bounty_npc` para probar el flujo completo sin esperar a un contrato real. Recomendado **desactivarlo** (`false`) en producción. |
| `Config.OpenCommand`              | Comando para abrir la tablet (por defecto `recompensas`).                                                                                                                   |
| `Config.PoliceJobName`            | Nombre del job de policía en tu base de datos/framework, usado para los contratos de tipo "Policial".                                                                       |
| `Config.UseApproximateNameSearch` | Reservado para una futura búsqueda exacta/aproximada del objetivo por nombre.                                                                                               |
| `Config.PVP.Enabled`              | Activa o desactiva los contratos contra jugadores.                                                                                                                          |
| `Config.PVP.MinBountyAmount`      | Monto mínimo permitido al crear un contrato.                                                                                                                                |
| `Config.PVP.Durations`            | Lista de duraciones disponibles en el desplegable de la tablet.                                                                                                             |
| `Config.PVP.EnableTracking`       | Activa el blip de seguimiento del objetivo tras aceptar un contrato.                                                                                                        |
| `Config.PVP.BlipRevealDistance`   | Distancia (en unidades del juego) a la que aparece el blip del objetivo.                                                                                                    |
| `Config.PVP.BlipDuration`         | Milisegundos que el blip permanece visible cada vez que se revela.                                                                                                          |
| `Config.PVE.Enabled`              | Activa o desactiva los contratos contra NPCs.                                                                                                                               |
| `Config.PVE.MaxActiveNPCBounties` | Límite de contratos PVE activos a la vez.                                                                                                                                   |
| `Config.PVE.NPCTemplates`         | Plantillas de NPCs disponibles para contratos PVE (modelo, monto, ubicaciones, probabilidades).                                                                             |
| `Config.Locales`                  | Todos los textos y notificaciones del script, listos para traducir.                                                                                                         |

{% hint style="info" %}
Recuerda añadir también `Config.NotifySystem` (ver la siguiente página) para elegir tu sistema de notificaciones.
{% endhint %}

## 🔗 Compatibilidad

* **Frameworks**: ESX y QBCore, sin necesidad de tocar el código principal — solo cambia `Config.Framework`.
* **Base de datos**: cualquier wrapper compatible con la sintaxis `MySQL.Async` (mysql-async / oxmysql).
* **Notificaciones**: nexus\_notify, okokNotify, mythic\_notify, o las notificaciones nativas de ESX/QBCore, seleccionable en `functions.lua` sin tocar el código protegido (ver la página siguiente).
* El script **no depende** de ningún sistema de TextUI, target, llaves de vehículo ni combustible: la tablet es una NUI propia y las recompensas se resuelven con eventos de framework estándar (dinero y muerte del jugador).

## 💻 Personalización de notificaciones (functions.lua)

`functions.lua` viaja fuera del escrow para que puedas conectar el script con el sistema de notificaciones que ya uses en tu servidor, sin tocar el código protegido.

Solo tienes que añadir en `shared/config.lua`:

```lua
Config.NotifySystem = 'esx' -- 'nexus', 'okok', 'esx', 'qb', 'mythic', 'custom'
```

Sistemas soportados de forma nativa: `nexus_notify`, `okokNotify`, `mythic_notify`, las notificaciones propias de ESX y de QBCore, o `'custom'` para conectar cualquier otro sistema editando directamente el bloque correspondiente en `functions.lua`.

Los textos de `Config.Locales` incluyen prefijos de color nativos de GTA (`~r~`, `~g~`, `~o~`...). `functions.lua` los detecta automáticamente y los traduce a un tipo (`error`, `success`, `warning`, `info`) para que se muestren correctamente incluso en sistemas de notificación que no interpretan esos códigos.

## 🆚 Contratos PVP vs. PVE

nexus\_bounty combina dos tipos de contrato en el mismo sistema:

* **PVP (contra jugadores)**: se crean indicando un nombre (o parte de él); el script busca primero entre los jugadores conectados y, si no encuentra coincidencia, en la base de datos. Pueden ser civiles o, si el creador es policía, de tipo "Policial" (solo aceptables por agentes).
* **PVE (contra NPCs)**: se generan a partir de las plantillas de `Config.PVE.NPCTemplates`, con su propio modelo, ubicación y probabilidades. Sirven para que siempre haya contratos disponibles, aunque no haya suficientes jugadores conectados para generar contratos PVP.

Ambos tipos comparten el mismo flujo de aceptación, tracking, cobro y clasificación.

## ❓ Preguntas frecuentes

{% hint style="info" %}
**Antes de abrir un ticket de soporte:**

* Revisa que el nombre de la carpeta del recurso sea exactamente `nexus_bounty`.
* Comprueba que estás usando la última versión del script.
* Repasa esta página de preguntas frecuentes.
{% endhint %}

<details>

<summary>¿Qué pasa si el objetivo se desconecta mientras estoy siguiéndolo?</summary>

El tracking se detiene automáticamente y el blip desaparece hasta que el objetivo vuelva a conectarse.

</details>

<details>

<summary>¿Puedo poner una recompensa sobre un jugador que está offline?</summary>

Sí. Si no se encuentra entre los jugadores conectados, el script busca la coincidencia en la base de datos.

</details>

<details>

<summary>¿Qué pasa si un contrato expira sin que nadie lo complete?</summary>

Se elimina automáticamente de la base de datos y de la lista de contratos activos; no hay devolución del dinero al creador.

</details>

<details>

<summary>¿Puedo cancelar un contrato que he creado?</summary>

Sí, desde "Mi Panel" puedes eliminar tus propios contratos activos; el dinero se te devuelve automáticamente.

</details>

<details>

<summary>¿Qué información debo incluir si abro un ticket?</summary>

Nombre exacto del script y versión, framework usado (ESX/QBCore), el error tal cual aparece en consola y los pasos para reproducirlo.

</details>

## 📋 Registro de cambios

### v1.0.0

* Lanzamiento inicial de nexus\_bounty.
* Contratos PVP (civiles y policiales) y PVE (por plantillas de NPC).
* Tablet NUI propia con homescreen, creación de contrato, contratos activos, panel personal y clasificación.
* Tracking del objetivo mediante blip configurable.
* Expiración automática de contratos.
* Sistema de pruebas con NPC bajo `Config.DebugMode`.
* Compatibilidad nativa con ESX y QBCore.
* Notificaciones personalizables vía `functions.lua`.
