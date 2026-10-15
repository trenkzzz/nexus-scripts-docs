# Integration API (1.0.1)

> Server exports, events and hooks that let dealerships, reward systems, property scripts and admin tools register, transfer and remove vehicles and garages in nexus\_garage without touching its database directly.

**Version:** 1.0.1 · **Framework(s):** ESX, QBCore · **Database:** MySQL / MariaDB (oxmysql)

***

## Overview

Version 1.0.1 adds a stable integration layer in `server/integrations.lua`. Every export validates its input, writes through the same internal functions the garage itself uses, keeps vehicle keys in sync and writes an audit entry whose actor is `resource:<calling resource>`, so every external change is visible in the admin panel's audit log.

The default `Config.Locale` is now `'en'`, and missing locale keys fall back to English.

{% hint style="info" %}
Call these exports from server scripts only, and only after nexus\_garage has finished starting. Until then every export returns `false, 'not_ready'`.
{% endhint %}

***

## Vehicle exports

| Export            | Parameters                                                                                     | Returns                  | Description                                                                                                                                              |
| ----------------- | ---------------------------------------------------------------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IsPlateTaken`    | `plate: string`                                                                                | `boolean`                | `true` when the plate is registered or currently tracked as out. Empty plates also return `true`.                                                        |
| `AddVehicle`      | `identifier`, `model: string`, `plate: string`, `garageKey?`, `vehicleClass?`, `props?: table` | `true` or `false, error` | Registers a new owned vehicle and verifies the write.                                                                                                    |
| `SetVehicleOwner` | `plate: string`, `identifier`, `opts?: { force = boolean }`                                    | `true` or `false, error` | Transfers a vehicle to another character, respecting `Functions.CanTransferOwnership`.                                                                   |
| `DeleteVehicle`   | `plate: string`                                                                                | `true` or `false, error` | Removes a vehicle from the database, deleting its entity first when it is out.                                                                           |
| `SetVehicleOut`   | `plate: string`, `netId?: number`, `garageKey?`                                                | `true` or `false, error` | Marks a vehicle as out and tracks the networked entity, for scripts that spawn owned vehicles themselves (door delivery from a dealership, for example). |

### Parameter details

* **`identifier`** accepts a server ID (number, or a numeric string of an online player) or a character identifier (ESX identifier or QBCore citizenid). Server IDs are resolved to the character identifier.
* **`model`** is lowercased and spaces are removed.
* **`plate`** is normalised with the same rules as the garage (uppercase, trimmed). It must be between 1 and 8 characters.
* **`garageKey`** is a garage ID or key. When omitted, `AddVehicle` uses the first enabled public garage of the vehicle class.
* **`vehicleClass`** is `'car'`, `'air'` or `'sea'`. Anything else is treated as `'car'`.
* **`props`** is a vehicle properties table (colours, mods, extras). It is merged into the stored properties, so a model and plate are always present.

### Error codes

| Code                 | Returned by                                         | Meaning                                                                                                       |
| -------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `not_ready`          | all                                                 | nexus\_garage is still starting.                                                                              |
| `invalid_identifier` | `AddVehicle`, `SetVehicleOwner`                     | The identifier could not be resolved.                                                                         |
| `invalid_model`      | `AddVehicle`                                        | Empty or non-string model.                                                                                    |
| `invalid_plate`      | `AddVehicle`                                        | Empty plate or longer than 8 characters.                                                                      |
| `plate_taken`        | `AddVehicle`                                        | The plate already exists.                                                                                     |
| `garage_not_found`   | `AddVehicle`                                        | The given garage does not exist.                                                                              |
| `insert_failed`      | `AddVehicle`                                        | The database write failed or could not be read back. The reason is printed in the server console.             |
| `vehicle_not_found`  | `SetVehicleOwner`, `DeleteVehicle`, `SetVehicleOut` | No vehicle with that plate.                                                                                   |
| `vehicle_financed`   | `SetVehicleOwner`                                   | The transfer was vetoed by `Functions.CanTransferOwnership` (by default, an open loan in nexus\_vehicleshop). |

### Example

```lua
local plate = 'NEXUS01'
if exports.nexus_garage:IsPlateTaken(plate) then return end

local ok, err = exports.nexus_garage:AddVehicle(source, 'sultan', plate, nil, 'car', {
    color1 = 145,
    color2 = 0,
})
if not ok then
    print('could not register vehicle: ' .. err)
end
```

***

## Garage exports

| Export             | Parameters                                | Returns                                | Description                                                                    |
| ------------------ | ----------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------ |
| `SetGarageOwner`   | `garageKey`, `identifier?`                | `true, moved, movedTo` or `nil, error` | Sets or clears the owner of a private or shared garage.                        |
| `DeleteGarage`     | `garageKey`, `opts?: { force = boolean }` | `true, moved, movedTo` or `nil, error` | Deletes a garage and relocates the vehicles stored in it.                      |
| `GetGarageByOwner` | `identifier`                              | `table[]`                              | Garages owned by that character, including dynamic garages (`dynamic = true`). |

* **`SetGarageOwner`** only works on `private` and `shared` garages that are not dynamic. Passing `nil` or `''` as the identifier clears the owner. On every real owner change the member list is cleared, vehicles that belong to other characters are moved to the nearest public garage of the same class, `moved` is the number of relocated vehicles, `movedTo` the name of the garage they went to, and `Functions.OnGarageOwnerChanged` fires.
* **`DeleteGarage`** refuses `public`, `job` and `gang` garages unless `opts.force` is `true`. Stored vehicles are moved to the nearest public garage of the same class; dynamic garages are unregistered the same way and fire `Functions.OnGarageDeleted`.
* Garage error codes: `not_ready`, `garage_not_found`, `garage_is_dynamic`, `garage_not_ownable`, `garage_protected`, `player_not_found`, `invalid_identifier`.

The same three operations are available as local server events for resources that prefer events over exports. The last argument is an optional callback that receives the same values the export returns.

| Event                                  | Arguments                        |
| -------------------------------------- | -------------------------------- |
| `nexus_garage:server:setGarageOwner`   | `garageKey`, `identifier`, `cb?` |
| `nexus_garage:server:deleteGarage`     | `garageKey`, `opts`, `cb?`       |
| `nexus_garage:server:getGarageByOwner` | `identifier`, `cb`               |

```lua
-- Property script: hand the garage of a sold house to its new owner
local ok, moved, movedTo = exports.nexus_garage:SetGarageOwner('house_12', newCitizenId)
if ok and moved > 0 then
    print(('%d vehicles of the previous owner were moved to %s'):format(moved, movedTo))
end
```

***

## Events — Emitted

| Event                                 | Side           | Payload                            | When                                                                                                                               |
| ------------------------------------- | -------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_garage:server:vehicleReleased` | Server (local) | `src`, `plate`, `impoundId`, `fee` | A player paid an impound fine and the vehicle was released. Used by nexus\_vehicleshop to close the loan of a repossessed vehicle. |

```lua
AddEventHandler('nexus_garage:server:vehicleReleased', function(src, plate, impoundId, fee)
    print(('%s released %s from impound %d for %d'):format(GetPlayerName(src), plate, impoundId, fee))
end)
```

***

## New editable functions (`functions.lua`)

| Function                         | Signature                                                     | Called                                                                 | Purpose                                                                            |
| -------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `Functions.OnVehicleAdded`       | `(plate, identifier, model, garageKey, resource)`             | After `AddVehicle` succeeds                                            | `resource` is the name of the calling resource. Logs, welcome messages, insurance. |
| `Functions.CanTransferOwnership` | `(plate, fromIdentifier, toIdentifier) -> allowed, reasonKey` | Before `SetVehicleOwner` and before an owner change in the admin panel | Return `false, 'locale_key'` to block the transfer.                                |
| `Functions.OnGarageOwnerChanged` | `(garageId, oldOwner, newOwner, movedVehicles, movedTo)`      | After every real owner change of a garage                              | Notify both characters, update property records.                                   |

By default `Functions.CanTransferOwnership` blocks the transfer when `nexus_vehicleshop` is started and reports the plate as financed:

```lua
function Functions.CanTransferOwnership(plate, fromIdentifier, toIdentifier)
    if GetResourceState('nexus_vehicleshop') == 'started' then
        local ok, financed = pcall(function()
            return exports.nexus_vehicleshop:IsFinanced(plate)
        end)
        if ok and financed then return false, 'vehicle_financed' end
    end
    return true
end
```

The message shown to admins uses the new locale key `vehicle_financed`, available in all seven languages.

***

## Audit entries

| Action              | Written by                           | Details                                     |
| ------------------- | ------------------------------------ | ------------------------------------------- |
| `vehicle_added`     | `AddVehicle`                         | `owner`, `model`, `garage`                  |
| `vehicle_owner_set` | `SetVehicleOwner`                    | `from`, `to`                                |
| `vehicle_deleted`   | `DeleteVehicle`                      | `owner`                                     |
| `garage_owner_set`  | `SetGarageOwner` and the admin panel | `name`, `from`, `owner`, `moved`, `movedTo` |
| `garage_deleted`    | `DeleteGarage`                       | `name`, `moved`, `to`                       |

***

## FAQ

**`AddVehicle` returns `plate_taken` for a plate that does not appear in any garage.** The plate may belong to a vehicle that is currently out. `IsPlateTaken` checks both the database and the vehicles tracked as out.

**The admin panel refuses to change the owner of a vehicle.** The vehicle has an open loan in nexus\_vehicleshop. Settle or cancel the loan first, or call `SetVehicleOwner` with `{ force = true }` from a trusted script.

**Can `DeleteGarage` remove a public garage?** Only with `{ force = true }`. Vehicles stored inside are moved to the nearest public garage of the same class before the garage is removed.

**Which resource made a change?** Open the audit log in the admin panel: external changes are recorded with the actor `resource:<name>`.
