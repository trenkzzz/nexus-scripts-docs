# Update 1.0.2

This update fixes how vehicle pictures are found and drawn, makes the menus open faster the first time, and completes the integration with [Nexus Banking](../nexus-banking/) and [Nexus Vehicleshop](../nexus-vehicleshop/).

{% hint style="info" %}
Upgrade by replacing the resource files and restarting. There are **no database changes** and no new Config keys you are forced to set. Your `config.lua` and `functions.lua` stay yours: only the optional changes listed below are new.
{% endhint %}

## Vehicle pictures

### What was wrong

* The picture URL was built from the game's internal _display name_, which is not the spawn name for many vehicles, so `docs.fivem.net` returned nothing and the silhouette stayed.
* ESX stores only the model **hash**, so the spawn name could not be recovered at all.
* One failed download marked the picture as broken until the menu was reloaded.
* Pictures were cropped to fill the box, and the box itself had a border and a flat background.

### How a picture is resolved now

For every vehicle shown in any menu (garage, impound, keys, admin player vehicles, job vehicles, the impound form), the resource tries these sources **in order** and uses the first one that loads:

| # | Source                                         | Notes                                                                                                               |
| - | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 1 | `Functions.VehicleImage(model)`                | Your own URL. Return `nil` to skip. Best place for **addon vehicles**.                                              |
| 2 | `nexus_vehicleshop` thumbnails                 | Used automatically when `nexus_vehicleshop` is started. File: `html/images/vehicles/<model>.webp` of that resource. |
| 3 | `https://docs.fivem.net/vehicles/<spawn>.webp` | Base-game vehicles.                                                                                                 |

The spawn name is found from, in this order: the model string, a built-in table of **921 base-game models** matched by hash, and the game's display name as a last resort. If a download fails it is retried once, then the next source is tried, and a vehicle whose sources all fail is tried again about 45 seconds later instead of staying blank for the whole session.

{% hint style="warning" %}
Addon vehicles that are in neither your `Functions.VehicleImage` nor the Vehicleshop thumbnails keep the silhouette. Generate thumbnails with Nexus Vehicleshop or return your own URLs from `Functions.VehicleImage`.
{% endhint %}

### How it is drawn

The picture now fits inside its area without cropping (`contain`) and the inner box is gone: the vehicle sits on the card with a soft purple glow under it. Row, card, detail, keys and admin views all use the same component, so the change applies everywhere.

## Performance

* Fonts (Bebas Neue, Poppins, JetBrains Mono) are now **bundled** in `html/assets/fonts/` instead of loaded from Google Fonts, and they are preloaded when the resource starts. The first menu opening no longer waits on a font download or flashes a different font.
* The 20 MB `html/assets/map.png` is no longer listed in `fxmanifest.lua`, so players stop downloading a file the menus never use. You can delete it from the folder.
* Vehicle pictures decode asynchronously and are cached after the first successful load.
* All visual effects (glows, animations, blur on modals, map zoom) are unchanged.

Internet access from the game client is still needed for base-game vehicle pictures, blip sprites, marker previews and NPC previews.

## Nexus Banking and Nexus Vehicleshop

### Impound fines

`Config.Systems.Banking` has a new value, `'nexus_banking'`, and `'auto'` now checks Nexus Banking **first**.

* **Society income:** the fine is deposited into the job's society account in Nexus Banking, with the title _Impound_.
* **Paying the fine:** when `Config.Systems.Banking` resolves to `nexus_banking`, the player can pay from their bank account through `Functions.ChargeImpound`, so the payment appears in their bank history. Cash is used first when `Config.Payment` is `'both'` and the player has enough cash. If the bank charge is declined, the normal framework payment is tried before showing _not enough money_.

### Export `SendVehicleToImpound`

Nexus Banking (seizure mode `impound`) and Nexus Vehicleshop (repossession) send vehicles to an impound. The export now accepts both call forms:

```lua
-- player or admin action
exports.nexus_garage:SendVehicleToImpound(src, plate, impoundId, reason, fee, bypassJob)

-- system call (no player)
exports.nexus_garage:SendVehicleToImpound(plate, reason, fee, impoundId)
```

If `impoundId` is missing, `0` or unknown in a system call, the first enabled impound of the vehicle's class is used.

### Already in place

* `Functions.CanTransferOwnership` blocks ownership changes of vehicles that are financed by Nexus Vehicleshop or pledged in Nexus Banking (`HasLien`).
* Vehicles bought in Nexus Vehicleshop are added through `AddVehicle` and delivered with `SetVehicleOut`.

## Files added or changed

| File                                 | Change                                                 |
| ------------------------------------ | ------------------------------------------------------ |
| `client/vehiclenames.lua`            | New. Base-game model names.                            |
| `client/utils.lua`, `client/nui.lua` | Spawn name resolution and picture list.                |
| `server/api.lua`, `server/main.lua`  | Export forms and impound payment.                      |
| `functions.lua`                      | `BankingSystem`, `AddSocietyMoney`, `ChargeImpound`.   |
| `config.lua`, `fxmanifest.lua`       | New `nexus_banking` option, files list, version 1.0.2. |
| `html/*`, `web/src/*`                | Rebuilt interface and bundled fonts.                   |

## Troubleshooting

**A vehicle still shows the silhouette.** Check it is a base-game model or that it has a thumbnail in Nexus Vehicleshop or a URL from `Functions.VehicleImage`. Open the game's NUI devtools (`nui_devtools`) and look at the Network tab for the failing URL.

**Impound fines do not reach the society account.** Set `Config.Systems.Banking = 'nexus_banking'` and make sure the job exists as a society account in Nexus Banking.
