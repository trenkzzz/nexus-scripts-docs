# ⚙️ Instalación y Compatibilidad

## 📥 Instalación básica

1. Descarga el script desde tu panel de **Tebex** (Keymaster / mis compras).
2. Descomprime la carpeta y renómbrala **exactamente** como se indica en la documentación del script (por ejemplo, `nexus_boombox`). El nombre del recurso debe coincidir o el script no arrancará.
3. Coloca la carpeta dentro de tu carpeta `resources`.
4. Añade `ensure nexus_xxxxx` a tu `server.cfg`, respetando el orden de dependencias si el script lo indica.
5. Abre `config.lua` y ajusta las opciones a tu gusto (framework, idioma, sistemas de compatibilidad...).
6. Reinicia el recurso o el servidor.

{% hint style="danger" %}
Todos nuestros scripts incluyen una comprobación del nombre del recurso al arrancar. Si la carpeta no se llama exactamente como el recurso original, el script detendrá su ejecución y mostrará un error en consola.
{% endhint %}

## 🧩 Sistema de compatibilidad modular

Nuestros scripts no dependen de un único sistema externo: cada función crítica tiene su propia rama de configuración para que elijas la que ya usas en tu servidor, sin tocar código.

| Sistema | Opciones disponibles |
|---|---|
| Framework | `esx`, `qbcore` |
| Notificaciones | `nexus`, `okok`, `esx`, `mythic` |
| TextUI | `nexus`, `okok`, `esx`, `custom` |
| Llaves de vehículo | `nexus`, `cd_garage`, `custom`, `default` |
| Combustible | `legacyfuel`, `custom`, `ninguno` |
| Interacción / target | `ox_target`, `qb-target`, `draw3dtext` (propio) |

Esto se controla siempre desde `config.lua`, por ejemplo:

```lua
Config.NotifySystem = 'nexus' -- 'nexus' | 'okok' | 'esx' | 'mythic'
Config.TextUISystem = 'nexus' -- 'nexus' | 'okok' | 'esx' | 'custom'
```

## 🧠 functions.lua y locales.lua

Cada script incluye un archivo `functions.lua` **fuera del escrow**, con las funciones principales ya preparadas para que las adaptes o conectes con otros recursos, además de `locales.lua` para traducir todos los textos del script sin tocar el resto del código.

## 🚗 Compatibilidad con nexus_garage

Si usas **nexus_garage**, todos nuestros scripts se integran automáticamente con él en cuanto lo detectan — no necesitas configurar nada adicional.
