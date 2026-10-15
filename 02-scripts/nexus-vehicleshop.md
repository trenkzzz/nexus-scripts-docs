# Nexus Vehicleshop

> A cinematic vehicle dealership for ESX, QBCore and Qbox: a 3D showroom with its own camera, hardened test drives, server-side prices and plates, real-time financing with offline payments and repossession, player-run dealerships with employees and a society account, sell-back with a pre-owned market, and verified delivery into nexus\_garage or the garage resource your server already uses.

**Version:** 1.1.0 · **Framework(s):** ESX, QBCore, Qbox · **Database:** MySQL / MariaDB (oxmysql)

***

## Description

Nexus Vehicleshop turns buying a vehicle into a moment. The player walks into a dealership, the camera cuts to the selected car parked under the showroom lights, and the interface frames the scene instead of covering it: a slim catalogue rail at the bottom, a detail panel on the right with animated stats, a GTA colour palette and a custom RGB picker that repaint the real vehicle in front of the camera, a three-way comparator, a spec sheet that flips like a printed catalogue card and a financing simulator that shows the exact instalment, interest and payment calendar while the down payment slider moves.

Every decision that involves money is taken by the server. The client only ever sends a catalogue reference and the colours the player picked; the price, the stock, the plate, the financing plan and the payment are resolved, validated and charged server-side. Plates are generated from a configurable pattern and reserved atomically before any money moves, so two players buying at the same instant can never receive the same plate. If writing the vehicle to the garage fails for any reason, the purchase is rolled back and the money is refunded automatically.

Delivery is verified, not assumed. The script writes the vehicle through a garage adapter and reads it back before the sale is confirmed. Adapters ship for nexus\_garage, the framework's own tables (ESX and QBCore), Qbox (`qbx_vehicles`), jg-advancedgarages, cd\_garage, qs-advancedgarages, okokGarage and a fully custom bridge. By default the new owner is placed behind the wheel outside the dealership with the keys in hand; delivery straight to the garage is one config line away.

Financing runs in real time, even while the player is offline. Each loan has a down payment, a number of instalments, an interest rate and an interval. A server job charges every instalment from the bank account, warns the player before a payment is due, opens a grace period when the balance is not enough and finally repossesses the vehicle. With nexus\_garage the vehicle is sent to a configurable impound with the outstanding debt as the fine; paying that fine closes the loan automatically. On garages without an impound the vehicle is held in custody and the owner can redeem it from the financing screen.

Everything is managed from an in-game admin panel: a statistics dashboard, dealerships, interaction points, delivery spawns, the showroom camera point, test drive area, display vehicles with floating price cards, per-dealership prices and stock, the catalogue itself, vehicle thumbnails, pre-owned vehicles, active loans and the sales history.

Any dealership can also be handed to players. A player dealership has an owner, employees with ranks and permissions, a society account that receives every sale, stock that is ordered at a wholesale price and a management menu with sales, stock, staff, finances, settings and charts. Employees can sell in person: the buyer receives the offer on screen and accepts it, and the seller earns a commission paid by the society.

***

## Features

* **Showroom with a real 3D preview** – the selected vehicle is spawned locally at the dealership's showroom point and framed by a dedicated scripted camera. Drag to rotate, scroll to zoom, open and close the doors.
* **Live GTA colour palette** – primary and secondary colours are chosen from a configurable palette of GTA colour IDs and applied to the preview instantly. The purchased vehicle stores the colour **ID**, so mechanics and tuning scripts can repaint it normally.
* **Custom RGB colours** – an optional colour picker for any RGB colour, with an optional secondary colour and pearlescent. The server validates every channel, adds the configured price and stores both the custom colour and the closest GTA colour ID, in the props format of each framework.
* **Animated stats** – top speed, acceleration, braking and traction read from the game's own vehicle data, plus the number of seats.
* **Comparator** – pin up to three vehicles and compare them side by side; the best value of each stat is highlighted.
* **Brand chip and camera glide** – a dealership chip with the logo and the dealership name, automatic camera framing based on the size of each model and a smooth camera transition between vehicles.
* **Spec sheet** – the detail panel flips into a printed-catalogue style card with the full specification, price and the lowest available instalment.
* **Categories, search and a pre-owned filter** – category chips (land, air and sea categories are separated automatically per dealership), free-text search by model or brand and a separate pre-owned section when used vehicles are in stock (`Config.Showroom.mixUsed` mixes them into the regular categories). The carousel can be driven with the keyboard.
* **Cars, motorcycles, boats, helicopters and planes** – each dealership is locked to one vehicle class (land, air or sea) and each catalogue entry can declare its spawn type, so planes and boats spawn correctly for test drives and delivery.
* **Three payment methods** – cash, bank or financing, each one individually enabled in the config.
* **Financing simulator** – choose a plan, drag the down payment and see the instalment, number of payments, interest, total cost and a bar chart of the payment calendar in real time. The numbers in the interface are calculated with exactly the same integer formula the server uses.
* **Real-time financing with offline charges** – instalments are charged by the server on schedule even when the player is offline, with an advance warning, a grace period, a configurable number of missed payments and repossession.
* **Repossession for every garage** – with nexus\_garage, QBCore and Qbox the vehicle is impounded with the outstanding debt as the fine, and paying the fine marks the loan as paid off. With ESX and third-party garages the vehicle is held in custody and can be redeemed for the debt plus a configurable fee. `delete` and `none` modes are also available.
* **My financing** – `/myfinances` lists every loan with an instalment timeline, the next payment countdown, the total owed and buttons to pay the next instalment, settle the whole loan or redeem a vehicle held in custody.
* **Plate patterns** – plates are generated from a pattern such as `11AAA111` (`A` letter, `1` digit, `.` letter or digit, `^X` fixed character, 8 characters maximum).
* **Custom plates** – an optional paid custom plate with live availability checking, length and character rules, a word blacklist and a reserved test-drive prefix.
* **Atomic plate reservation** – every plate is reserved in its own table before money is taken, then checked against the garage; collisions are retried automatically.
* **Registration ticket** – after paying, the plate is stamped onto a registration ticket with a short animation before the keys are handed over.
* **Delivery at the door or in the garage** – the vehicle is created by the server at a free delivery point and the buyer is seated inside it, with fuel and keys handled by the configured systems. If every point is blocked, the vehicle goes to the garage instead.
* **Hardened test drives** – the test vehicle is created by the server, timed by the server and fenced by a server-side radius with a warning and a grace period. It is always deleted and the player always returned to the dealership when time runs out, when they leave the area, disconnect or the resource stops. Optional private routing bucket, per-player cooldown, simultaneous test drive limit per dealership and a reserved test plate prefix that can never be registered.
* **Sell-back** – players drive to the dealership's sell-back point and receive an offer based on a percentage of the list price, reduced by the mileage recorded by the garage. Financed or impounded vehicles cannot be sold. A cooldown prevents buy-and-sell loops.
* **Pre-owned market** – vehicles sold back appear in the same dealership as pre-owned stock with their mileage and modifications, at a configurable markup.
* **Optional stock** – unlimited by default, or limited stock per vehicle and dealership with automatic restocking up to a maximum. Player dealerships always use stock and order it themselves.
* **Player-run dealerships** – owner, employees with ranks and permissions, assisted sales with a buyer acceptance screen and rank-limited discounts, commissions paid from the society, wholesale stock orders with delayed delivery, price adjustments within limits, deposits and withdrawals, and a management menu with charts. Every editable setting is whitelisted in `config/dealers.lua`.
* **Society accounts** – sales, instalments, commissions, stock orders and sell-backs go through `Functions.Get/Add/RemoveSocietyMoney` (Renewed-Banking, okokBanking, qb-banking, qb-management, esx\_addonaccount) or an internal ledger, and every movement is recorded.
* **Display vehicles** – park vehicles inside or outside the dealership with a floating price card; interacting with one opens the showroom on that model.
* **In-game admin panel** – `/vshopadmin` creates and edits dealerships, places every point in the world, sets per-dealership prices, stock and visibility, removes pre-owned vehicles, repossesses or cancels loans, browses the sales history and shows a dashboard with revenue, sales, payment methods, top models and financing. Every destructive action asks for confirmation.
* **Catalogue editor** – add vehicles, change prices and categories, hide models and reset them to their configuration values from the admin panel. Changes are stored in the database on top of `config/catalog.lua`.
* **Thumbnail generator** – photographs every catalogue model in a studio with `screenshot-basic` and saves the images inside the resource.
* **Verified garage integration** – adapters for nexus\_garage, ESX/QBCore native tables, Qbox, jg-advancedgarages, cd\_garage, qs-advancedgarages, okokGarage and a custom bridge, each one written and read back on every purchase.
* **Self-diagnostics** – `/vshop diag` performs a ghost purchase through the active garage adapter, reads it back, checks the columns, deletes it and prints a pass or fail line for every step, plus the database schema, the repossession mode, the society system, thumbnail coverage and performance timings.
* **Logs** – every sale, refund, loan payment, missed payment, repossession, sell-back and test drive is stored in the database and can be mirrored to a Discord webhook.
* **Custom currencies** – a dealership can sell in any currency defined in `Config.Currencies` through `Functions.CustomCurrency`.
* **Seven languages** – English (default), Spanish, German, French, Italian, Portuguese and Chinese, for both the game messages and the interface.
* **Accessibility and performance** – respects `prefers-reduced-motion` (or a forced setting), contains no permanent blur, ships its fonts and icons locally and keeps idle threads asleep.

***

## Dependencies

### Required

| Dependency                                                               | Notes                                                                                                    |
| ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| **ESX** (`es_extended`), **QBCore** (`qb-core`) **or Qbox** (`qbx_core`) | Detected automatically, or forced with `Config.Framework`. Qbox is checked first, then ESX, then QBCore. |
| **oxmysql**                                                              | Declared as a dependency and loaded through `@oxmysql/lib/MySQL.lua`. mysql-async is not supported.      |
| **MySQL / MariaDB**                                                      | Eleven `nexus_vs_*` tables are created and upgraded automatically on start.                              |

### Optional

| Dependency                                                                                                | What it is for                                                                                                                             | Selected with                                 |
| --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------- |
| **nexus\_garage 1.0.1 or newer**                                                                          | Recommended garage: verified writes, door delivery tracking, key sync, repossession into an impound and loan closing when the fine is paid | `Config.Systems.Garage = 'auto'` or `'nexus'` |
| `qbx_vehicles`                                                                                            | Qbox vehicle creation                                                                                                                      | `Config.Systems.Garage = 'qbox'`              |
| jg-advancedgarages, cd\_garage, qs-advancedgarages, okokGarage                                            | Third-party garage storage                                                                                                                 | `Config.Systems.Garage`                       |
| `nexus_notify`                                                                                            | Notifications (default)                                                                                                                    | `Config.Systems.Notify = 'nexus'`             |
| `nexus_TextUI`                                                                                            | Floating interaction prompts (default)                                                                                                     | `Config.Systems.TextUI = 'nexus'`             |
| okokNotify, ox\_lib, mythic\_notify, ps-ui                                                                | Alternative notifications                                                                                                                  | `Config.Systems.Notify`                       |
| okokTextUI, ox\_lib, ps-ui, ESX / QBCore text UI                                                          | Alternative prompts                                                                                                                        | `Config.Systems.TextUI`                       |
| ox\_target, qb-target                                                                                     | Target interaction on dealerships and NPCs                                                                                                 | `Config.Systems.Target`                       |
| qb-vehiclekeys, qs-vehiclekeys, cd\_garage, okokGarage, t1ger\_keys, Renewed-Vehiclekeys, wasabi\_carlock | Vehicle keys on delivery and test drives                                                                                                   | `Config.Systems.Keys`                         |
| LegacyFuel, ps-fuel, ox\_fuel, cdn-fuel, okokGasStation, nd\_fuel                                         | Fuel on delivery and test drives                                                                                                           | `Config.Systems.Fuel`                         |
| Renewed-Banking, okokBanking, qb-banking, qb-management, esx\_addonaccount                                | Dealership society accounts (an internal ledger is used when none is running)                                                              | `Config.Systems.Banking`                      |
| `screenshot-basic`                                                                                        | In-game thumbnail generator                                                                                                                | `Config.Thumbnails.resource`                  |

### Server requirements

* **An ACE permission** (`nexus.vehicleshop.admin` by default) for anyone who should open the admin panel or run `/vshop diag`.
* **A public garage or a default garage key** for every vehicle class you sell. With nexus\_garage the first enabled public garage of the class is used; with the SQL adapters the keys in `Config.Delivery.defaultGarages` are written.
* **An impound for repossessions** when using nexus\_garage: create one in `/garageadmin` (ideally assigned to the dealership job, so the fines reach its society) and put its ID in `Finance.Repossession.impoundId`.
* **Internet access from the game client** only for vehicle thumbnails that have not been generated, which are loaded from `docs.fivem.net`. Fonts and icons are bundled. Generated thumbnails are stored in `html/images/vehicles/`, and `Functions.VehicleImage` can point to any other image pack.

***

## Installation

1. **Download the resource** from the cfx.re portal (Keymaster) and extract it into your `resources` folder.
2. **Keep the folder name exactly `nexus_vehicleshop`.** The resource validates its own name and refuses to start under any other name.
3. **Database.** `sql/install.sql` is executed automatically on every start (all statements are `CREATE TABLE IF NOT EXISTS`), followed by an upgrade step that adds any missing column or index without touching existing rows. The console prints `database upgraded: …` when something was added. Importing the file manually beforehand is optional.
4. **Order in `server.cfg`.** The framework and the garage must start first:

```cfg
ensure oxmysql
ensure qb-core            # or es_extended / qbx_core
ensure nexus_notify       # optional
ensure nexus_TextUI       # optional
ensure nexus_garage       # or your garage resource
ensure nexus_vehicleshop
```

5. **Grant the admin ACE.**

```cfg
add_ace group.admin nexus.vehicleshop.admin allow
```

6. **Review the configuration.** At minimum check `Config.Systems` in `config/config.lua`, the catalogue in `config/catalog.lua`, the plans, the impound ID and the repossession mode in `config/finance.lua`, the plate pattern in `config/plates.lua` and, for player dealerships, `config/dealers.lua`.
7. **Start the server** and look for `[nexus_vehicleshop] ready (<framework>, garage adapter: <adapter>, <n> dealerships)` in the console.
8. **Run the self-test.** Join as an admin and type `/vshop diag` (it also works from the server console). Every line should read `[OK]`.
9. **Create a dealership.** Run `/vshopadmin` → **New dealership** and fill in:
   * name, vehicle class and categories;
   * **Interaction point** → _Place in world_ → walk to the counter and press `E`;
   * **Delivery spawns** → _Place in world_ → press `E` on each free parking spot, `Enter` to finish;
   * **Showroom camera** → the open spot where the preview vehicle will be shown (optional);
   * **Test drive** start point, duration and radius (optional);
   * **Sell-back point** where players drive in to sell (optional);
   * **Display vehicles** with their model and colour (optional);
   * **Type** → _Server_ or _Player-owned_; for a player dealership, the owner, job, society and **Management point**;
   * **Save.** The dealership appears for every player immediately.
10. **Optional: generate thumbnails.** `/vshopadmin` → **Catalog** → **Generate missing**, then restart the resource so the new images are served.

***

## Configuration

The configuration is split into five files inside `config/`, all of them outside the escrow. Dealerships themselves are not defined in a file: they are created in-game and stored in the database.

### `config/config.lua` — general

| Config key                                       | Type            | Default                     | Description                                                                                                                                                                                                                                       |
| ------------------------------------------------ | --------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Framework`                               | string          | `'auto'`                    | `'auto'`, `'esx'`, `'qbcore'` or `'qbox'`. In `auto` mode `qbx_core`, `es_extended` and `qb-core` are probed in that order.                                                                                                                       |
| `Config.Locale`                                  | string          | `'en'`                      | `en`, `es`, `de`, `fr`, `it`, `pt` or `zh`. Applies to game messages and to the interface. Missing keys fall back to English.                                                                                                                     |
| `Config.Debug`                                   | boolean         | `false`                     | Prints bridge errors and draws target zones.                                                                                                                                                                                                      |
| `Config.Systems.Garage`                          | string          | `'auto'`                    | `auto`, `nexus`, `default`, `qbox`, `jg`, `cd_garage`, `qs`, `okok`, `custom`. `auto` picks nexus\_garage, then `qbx_vehicles` on Qbox, then jg, cd\_garage, qs and okok if they are started, and finally `default`.                              |
| `Config.Systems.Keys`                            | string          | `'auto'`                    | `auto`, `nexus`, `qb-vehiclekeys`, `qs-vehiclekeys`, `cd_garage`, `okokGarage`, `t1ger_keys`, `Renewed`, `wasabi_carlock`, `custom`, `none`. With `nexus` the keys follow ownership in nexus\_garage.                                             |
| `Config.Systems.Fuel`                            | string          | `'LegacyFuel'`              | `LegacyFuel`, `ps-fuel`, `ox_fuel`, `cdn-fuel`, `okokGasStation`, `nd_fuel`, `native`, `custom`.                                                                                                                                                  |
| `Config.Systems.Banking`                         | string          | `'auto'`                    | Where society income goes: `auto`, `Renewed-Banking`, `okokBanking`, `qb-banking`, `qb-management`, `esx_addonaccount`, `custom`. `auto` uses the first one that is started. `nexus_banking` is reserved for the upcoming Nexus Banking resource. |
| `Config.Systems.Notify`                          | string          | `'nexus'`                   | `nexus`, `okok`, `esx`, `qb`, `ox_lib`, `mythic`, `ps-ui`, `custom`. Falls back to the native GTA feed if the chosen system is not running.                                                                                                       |
| `Config.Systems.TextUI`                          | string          | `'nexus'`                   | `nexus`, `okok`, `esx`, `qb`, `ox_lib`, `ps-ui`, `native`, `custom`.                                                                                                                                                                              |
| `Config.Systems.Target`                          | string          | `'none'`                    | `none`, `ox_target`, `qb-target`, `custom`. A dealership set to `target` mode behaves as a marker when no target system is available.                                                                                                             |
| `Config.Resources`                               | table           | —                           | Folder names of `nexus_notify`, `nexus_TextUI` and `nexus_garage`, resolved case-insensitively.                                                                                                                                                   |
| `Config.Commands.Admin`                          | string          | `'vshopadmin'`              | Opens the admin panel (ACE checked server-side).                                                                                                                                                                                                  |
| `Config.Commands.Main`                           | string          | `'vshop'`                   | `/vshop diag` runs the self-test, `/vshop reload` reloads the catalogue, stock, dealerships and loans.                                                                                                                                            |
| `Config.Commands.Finances`                       | string          | `'myfinances'`              | Opens the player's financing panel.                                                                                                                                                                                                               |
| `Config.AdminAce`                                | string          | `'nexus.vehicleshop.admin'` | ACE permission for admin callbacks and commands.                                                                                                                                                                                                  |
| `Config.InteractKey` / `Config.InteractKeyLabel` | number / string | `38` / `'E'`                | Interaction control and the label shown in prompts.                                                                                                                                                                                               |
| `Config.Currency`                                | string          | `'$'`                       | Currency symbol used in game messages and the interface.                                                                                                                                                                                          |
| `Config.NumberLocale`                            | string          | `'en-US'`                   | Number formatting in the interface (for example `de-DE` shows `1.250`).                                                                                                                                                                           |
| `Config.Payment.cash` / `.bank` / `.finance`     | boolean         | `true`                      | Enables each payment method.                                                                                                                                                                                                                      |

#### Delivery

| Config key                         | Type          | Default                                                        | Description                                                                                                                                            |
| ---------------------------------- | ------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Config.Delivery.mode`             | string        | `'door'`                                                       | `door`: the server creates the vehicle at a free delivery point and the buyer is seated inside. `garage`: the vehicle is stored in the default garage. |
| `Config.Delivery.garage`           | string or nil | `nil`                                                          | Force a garage key for every purchase. `nil` uses the adapter's default (first public garage of the class with nexus\_garage).                         |
| `Config.Delivery.fallbackToGarage` | boolean       | `true`                                                         | When every delivery point is blocked or the vehicle cannot be created, store it in the garage instead of cancelling.                                   |
| `Config.Delivery.fuel`             | number        | `100.0`                                                        | Fuel level of new vehicles.                                                                                                                            |
| `Config.Delivery.defaultGarages`   | table         | `car = 'pillboxgarage'`, `air = 'intairport'`, `sea = 'lsymc'` | Garage keys written by the SQL adapters (default, jg, cd\_garage, qs, okok). Change them to the names your garage resource uses.                       |

#### Purchase protection

| Config key                        | Type   | Default                  | Description                                                                                                                                                                  |
| --------------------------------- | ------ | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Purchase.maxShopDistance` | number | `60.0`                   | Maximum distance between the player and the dealership (interaction point, showroom point, sell-back point or a display slot) for a purchase or a test drive to be accepted. |
| `Config.Purchase.cooldownSeconds` | number | `3`                      | Minimum time between two purchase requests from the same player.                                                                                                             |
| `Config.Purchase.rateLimit`       | table  | `window = 10`, `max = 8` | Request limiter applied to every server callback.                                                                                                                            |
| `Config.Purchase.maxRetries`      | number | `6`                      | Attempts to find a free plate before giving up.                                                                                                                              |

#### Test drives

| Config key                                | Type    | Default | Description                                                                   |
| ----------------------------------------- | ------- | ------- | ----------------------------------------------------------------------------- |
| `Config.TestDrive.enabled`                | boolean | `true`  | Master switch.                                                                |
| `Config.TestDrive.seconds`                | number  | `60`    | Default duration (each dealership can override it).                           |
| `Config.TestDrive.radius`                 | number  | `600.0` | Default allowed radius in metres around the starting point.                   |
| `Config.TestDrive.leaveGraceSeconds`      | number  | `8`     | Time to return after leaving the radius before the test drive ends.           |
| `Config.TestDrive.bucket`                 | boolean | `true`  | Run each test drive in its own routing bucket (no traffic, no other players). |
| `Config.TestDrive.cooldownSeconds`        | number  | `300`   | Time between two test drives of the same player.                              |
| `Config.TestDrive.maxActivePerDealership` | number  | `5`     | Simultaneous test drives per dealership.                                      |
| `Config.TestDrive.fuel`                   | number  | `100.0` | Fuel of the test vehicle.                                                     |

#### Sell-back and pre-owned

| Config key                                 | Type    | Default  | Description                                                                                                                                                        |
| ------------------------------------------ | ------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Config.SellBack.enabled`                  | boolean | `true`   | Master switch.                                                                                                                                                     |
| `Config.SellBack.percent`                  | number  | `60`     | Offer as a percentage of the dealership price.                                                                                                                     |
| `Config.SellBack.mileageDepreciationPerKm` | number  | `0.02`   | Percentage points removed per kilometre recorded in the `mileage` column.                                                                                          |
| `Config.SellBack.minPercent`               | number  | `25`     | Lowest possible offer.                                                                                                                                             |
| `Config.SellBack.cooldownMinutes`          | number  | `30`     | Time between two sales of the same player.                                                                                                                         |
| `Config.SellBack.requireNearDealership`    | boolean | `true`   | Require the player to be at the dealership.                                                                                                                        |
| `Config.SellBack.toUsedStock`              | boolean | `true`   | Put sold vehicles on sale as pre-owned in the same dealership.                                                                                                     |
| `Config.SellBack.usedMarkupPercent`        | number  | `15`     | Markup applied to the amount paid to the seller.                                                                                                                   |
| `Config.SellBack.serverFunds`              | string  | `'mint'` | Server dealerships: `mint` creates the money paid to the seller, `society` takes it from the dealership account. Player dealerships always pay from their society. |

#### Stock

| Config key                    | Type    | Default | Description                                                                                                                                           |
| ----------------------------- | ------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Stock.enabled`        | boolean | `false` | When `false` every vehicle is unlimited (prices and visibility per dealership still apply). When `true` the stock set in the admin panel is enforced. |
| `Config.Stock.restockMinutes` | number  | `60`    | Restock interval.                                                                                                                                     |
| `Config.Stock.restockAmount`  | number  | `1`     | Units added per vehicle on each restock, up to its maximum.                                                                                           |

#### Showroom and interface

| Config key                                                | Type    | Default                | Description                                                                                                               |
| --------------------------------------------------------- | ------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `Config.Showroom.cameraDistance` / `cameraHeight` / `fov` | number  | `6.0` / `1.4` / `42.0` | Camera framing.                                                                                                           |
| `Config.Showroom.rotateSpeed`                             | number  | `0.35`                 | Degrees per pixel dragged.                                                                                                |
| `Config.Showroom.hideRadar`                               | boolean | `true`                 | Hide the minimap while browsing.                                                                                          |
| `Config.Showroom.fallbackPreview`                         | vector4 | Simeon's floor         | Showroom point used by dealerships without their own.                                                                     |
| `Config.Showroom.displayDistance`                         | number  | `80.0`                 | Distance at which display vehicles are spawned.                                                                           |
| `Config.Showroom.mixUsed`                                 | boolean | `false`                | Also show pre-owned vehicles inside the regular categories.                                                               |
| `Config.Showroom.autoFrame`                               | boolean | `true`                 | Adjust the camera distance to the size of each model.                                                                     |
| `Config.Showroom.transitionMs`                            | number  | `500`                  | Duration of the camera glide between vehicles.                                                                            |
| `Config.Showroom.brandChip`                               | boolean | `true`                 | Show the dealership chip with the logo and name.                                                                          |
| `Config.CustomColors`                                     | table   | enabled                | `enabled`, `price` (per custom colour, `0` = free), `secondary`, `pearl`.                                                 |
| `Config.Interface.reducedMotion`                          | string  | `'auto'`               | `auto` follows the system setting, `on` disables animations, `off` always animates.                                       |
| `Config.Interface.sounds`                                 | boolean | `true`                 | Interface sound effects.                                                                                                  |
| `Config.Interface.scale`                                  | number  | `1.0`                  | Extra multiplier on top of the automatic resolution scaling.                                                              |
| `Config.Interface.panelScale`                             | number  | `1.08`                 | Size of the admin, management and financing panels (0.8 to 1.4).                                                          |
| `Config.Interface.compareMax`                             | number  | `3`                    | Vehicles in the comparator (2 to 4).                                                                                      |
| `Config.Currencies`                                       | table   | empty                  | Custom currencies `{ label, icon, symbol }` that a dealership can sell in. Charged through `Functions.CustomCurrency`.    |
| `Config.Thumbnails`                                       | table   | enabled                | `resource`, `studio` (vector4), `camera`, `width`, `height`, `quality`, `chromaKey`, `folder`.                            |
| `Config.Blips`                                            | table   | —                      | Default blip per vehicle class (each dealership can change it).                                                           |
| `Config.DefaultInteraction`                               | table   | —                      | Default mode (`marker`, `npc`, `target`), NPC model, radius and marker style.                                             |
| `Config.Logs.discord`                                     | table   | disabled               | `enabled`, `webhook`, `username`, `color`.                                                                                |
| `Config.Logs.console`                                     | boolean | `true`                 | One console line per logged event.                                                                                        |
| `Config.Palette`                                          | table   | 35 colours             | `{ id, label, hex }` entries. `id` is the GTA colour index stored on the vehicle; `hex` is only used to paint the swatch. |

### `config/catalog.lua` — categories and vehicles

```lua
Catalog.Categories = {
    { id = 'sports', label = 'Sports', icon = 'gauge-high', class = 'car' },
    { id = 'boats', label = 'Boats', icon = 'ship', class = 'sea' },
    { id = 'helicopters', label = 'Helicopters', icon = 'helicopter', class = 'air' },
}

Catalog.Vehicles = {
    { model = 'sultan', price = 58000, category = 'sports' },
    { model = 'bati', price = 23000, category = 'motorcycles', type = 'bike' },
    { model = 'squalo', price = 64000, category = 'boats', class = 'sea', type = 'boat' },
    { model = 'dodo', price = 480000, category = 'planes', class = 'air', type = 'plane', label = 'Dodo', brand = 'Mammoth' },
}
```

| Field            | Required | Description                                                                                                                                                                                                                  |
| ---------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model`          | yes      | Spawn name. Lowercased automatically.                                                                                                                                                                                        |
| `price`          | yes      | Base price. Each dealership can override it in the admin panel.                                                                                                                                                              |
| `category`       | yes      | ID of a category from `Catalog.Categories`.                                                                                                                                                                                  |
| `class`          | no       | `car`, `air` or `sea`. Defaults to the category class.                                                                                                                                                                       |
| `type`           | no       | Spawn type for server-created vehicles: `automobile`, `bike`, `boat`, `heli`, `plane`, `submarine`. Defaults to `boat` for sea, `heli` for air and `automobile` for land. Set `bike` for motorcycles and `plane` for planes. |
| `label`, `brand` | no       | Display names. When omitted the game's own labels are used.                                                                                                                                                                  |
| `image`          | no       | Thumbnail URL. When omitted a generated thumbnail, `Functions.VehicleImage` or `docs.fivem.net` is used.                                                                                                                     |
| `hidden`         | no       | Keep the entry but hide it everywhere.                                                                                                                                                                                       |

Category fields: `id`, `label` (or a locale key `cat_<id>` inside the `ui` table), `icon` (FontAwesome solid icon name) and `class`. The catalogue is validated on start: entries with a missing model, a duplicated model, an invalid price or an unknown category are hidden and reported in the console, without breaking the shop. Models that do not exist on a player's game build are hidden for that player and reported in their F8 console and in the server console.

The admin panel **Catalog** tab edits the same list in-game. Its changes are stored in `nexus_vs_catalog` and applied on top of the file: the file is the base, the database entry wins for the same model, and a removed entry is hidden without deleting it from the file. **Reset** discards the database entry and returns to the file values.

### `config/finance.lua` — financing

| Config key                                 | Type    | Default              | Description                                                                                                                                                                                           |
| ------------------------------------------ | ------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Finance.Enabled`                          | boolean | `true`               | Master switch.                                                                                                                                                                                        |
| `Finance.Account`                          | string  | `'bank'`             | Account charged for instalments.                                                                                                                                                                      |
| `Finance.DownPaymentAccount`               | string  | `'bank'`             | Account charged for the down payment.                                                                                                                                                                 |
| `Finance.MaxActivePerPlayer`               | number  | `3`                  | Open loans (active, overdue or repossessed) allowed per character.                                                                                                                                    |
| `Finance.MinPrice`                         | number  | `10000`              | Vehicles cheaper than this cannot be financed.                                                                                                                                                        |
| `Finance.TickSeconds`                      | number  | `60`                 | How often due instalments are processed.                                                                                                                                                              |
| `Finance.WarnBeforeMinutes`                | number  | `30`                 | Warning sent to online owners before a payment.                                                                                                                                                       |
| `Finance.GraceMinutes`                     | number  | `120`                | Time given after a failed payment before the next attempt.                                                                                                                                            |
| `Finance.MaxMissed`                        | number  | `1`                  | Failed attempts tolerated; the next one repossesses the vehicle.                                                                                                                                      |
| `Finance.ChargeOffline`                    | boolean | `true`               | Charge offline owners directly in the database.                                                                                                                                                       |
| `Finance.Repossession.enabled`             | boolean | `true`               | Apply repossession through the garage adapter.                                                                                                                                                        |
| `Finance.Repossession.mode`                | string  | `'auto'`             | `impound`, `custody`, `delete` or `none`. `auto` uses `impound` with nexus\_garage, the custom adapter, QBCore and Qbox, and `custody` otherwise. When an impound cannot be applied, custody is used. |
| `Finance.Repossession.impoundId`           | number  | `1`                  | nexus\_garage impound used for repossessions.                                                                                                                                                         |
| `Finance.Repossession.qbDepotState`        | number  | `2`                  | State written by the QBCore and Qbox adapters.                                                                                                                                                        |
| `Finance.Repossession.custodyOwner`        | string  | `'nexus_vs_custody'` | Owner written on the vehicle while it is held in custody.                                                                                                                                             |
| `Finance.Repossession.allowRedeem`         | boolean | `true`               | Let the owner redeem a vehicle in custody from `/myfinances`.                                                                                                                                         |
| `Finance.Repossession.redeemFeePercent`    | number  | `10`                 | Fee added to the outstanding debt when redeeming.                                                                                                                                                     |
| `Finance.Repossession.releaseCheckSeconds` | number  | `300`                | How often impounded vehicles are checked for release, to close their loans.                                                                                                                           |
| `Finance.Plans`                            | table   | 3 plans              | `{ id, installments, interest, minDown, intervalMinutes }`. `interest` and `minDown` are percentages; `intervalMinutes` is real time (`1440` = one day).                                              |

**How the numbers are calculated** — the down payment must be at least `minDown` percent of the price and lower than the price. Interest is `financed × interest%`, rounded to the nearest unit with integer arithmetic. The instalment is the total divided by the number of instalments, rounded up; the last instalment is whatever remains, so the total is always exact and there is never a division by zero. The same formula runs in the interface simulator and on the server.

### `config/plates.lua` — plates

| Config key                              | Type    | Default          | Description                                                                                                                        |
| --------------------------------------- | ------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `Plates.Pattern`                        | string  | `'11AAA111'`     | `A` letter, `1` digit, `.` letter or digit, `^X` the fixed character `X`, any other character is kept as is. Maximum 8 characters. |
| `Plates.Letters` / `Plates.Digits`      | string  | —                | Character sets used by the generator (ambiguous letters such as I and O are excluded by default).                                  |
| `Plates.Custom.enabled`                 | boolean | `true`           | Allow paid custom plates.                                                                                                          |
| `Plates.Custom.price`                   | number  | `15000`          | Extra cost of a custom plate.                                                                                                      |
| `Plates.Custom.minLength` / `maxLength` | number  | `2` / `8`        | Length limits.                                                                                                                     |
| `Plates.Custom.allowed`                 | string  | `'^[A-Z0-9 ]+$'` | Lua pattern of allowed characters. Double spaces are always rejected.                                                              |
| `Plates.Blacklist`                      | table   | —                | Words that cannot appear anywhere in a custom plate (checked without spaces).                                                      |
| `Plates.TestPrefix`                     | string  | `'TEST'`         | Prefix of test drive plates. Plates starting with it can never be bought or given.                                                 |
| `Plates.ReservationMinutes`             | number  | `10`             | Lifetime of unused plate reservations.                                                                                             |

### `config/dealers.lua` — player dealerships

| Config key                            | Type    | Default         | Description                                                                                                                                                                                                                                                            |
| ------------------------------------- | ------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PlayerDealers.Enabled`               | boolean | `true`          | Master switch for player dealerships.                                                                                                                                                                                                                                  |
| `PlayerDealers.Command`               | string  | `'vshopmanage'` | Command that opens the management menu.                                                                                                                                                                                                                                |
| `PlayerDealers.ManagementPointRadius` | number  | `1.6`           | Interaction radius of the management point.                                                                                                                                                                                                                            |
| `PlayerDealers.UseJobGrades`          | boolean | `true`          | Members of the dealership job get a rank from their job grade.                                                                                                                                                                                                         |
| `PlayerDealers.JobGrades`             | table   | —               | Job grade → rank.                                                                                                                                                                                                                                                      |
| `PlayerDealers.SetJobOnHire`          | boolean | `false`         | Also set the framework job when hiring (`HireGrade`).                                                                                                                                                                                                                  |
| `PlayerDealers.MaxEmployees`          | number  | `25`            | Employees per dealership.                                                                                                                                                                                                                                              |
| `PlayerDealers.HireDistance`          | number  | `8.0`           | Distance to the player being hired.                                                                                                                                                                                                                                    |
| `PlayerDealers.RequireStock`          | boolean | `true`          | Player dealerships only sell what they have in stock.                                                                                                                                                                                                                  |
| `PlayerDealers.WholesalePercent`      | number  | `70`            | Cost of stock orders as a percentage of the list price.                                                                                                                                                                                                                |
| `PlayerDealers.DeliveryMinutes`       | number  | `0`             | Delay before ordered stock arrives (`0` = instant).                                                                                                                                                                                                                    |
| `PlayerDealers.MaxOrderQuantity`      | number  | `10`            | Units per order.                                                                                                                                                                                                                                                       |
| `PlayerDealers.LowStock`              | number  | `2`             | Threshold for low stock warnings.                                                                                                                                                                                                                                      |
| `PlayerDealers.DefaultCommission`     | number  | `10`            | Commission percentage on assisted sales, paid by the society.                                                                                                                                                                                                          |
| `PlayerDealers.SellbackFromSociety`   | boolean | `true`          | Sell-backs at player dealerships are paid by the society.                                                                                                                                                                                                              |
| `PlayerDealers.Offer`                 | table   | —               | Assisted sales: `distance`, `seconds` to accept and `maxDiscount` percentage per rank.                                                                                                                                                                                 |
| `PlayerDealers.Ranks`                 | table   | 3 ranks         | `{ id, name, perms }`. Permissions: `sell`, `viewSales`, `viewStock`, `orderStock`, `viewFinances`, `manageEmployees`, `viewSociety`, `deposit`, `withdraw`, `settings`.                                                                                               |
| `PlayerDealers.Editable`              | table   | —               | Whitelist of settings the staff can change, each with a minimum rank and limits: `name`, `categories`, `priceModifier`, `modelPrice`, `testdrive`, `testdriveSeconds`, `sellback`, `sellbackPercent`, `financePlans`, `delivery`, `commission`, `selfService`, `blip`. |

The owner of a player dealership always has the highest rank. Employees are stored in `nexus_vs_employees`; with `UseJobGrades` the members of the dealership job are also recognised. `Functions.CanManageDealership` can override any permission check.

***

## Locales & Editable Strings

Seven complete translations live in `locales/` (`en`, `es`, `de`, `fr`, `it`, `pt`, `zh`), all outside the escrow, with identical key sets. Each file has two parts:

* the top-level keys for game messages, notifications, prompts, error codes, command suggestions and log titles;
* a `ui` table with every text shown in the interface, keyed by a stable identifier (`buy`, `test_drive`, `finance_from`, `confirm_purchase_text`, …). Placeholders use `{name}` in the `ui` table and `%s` in game messages and must be kept as they are.

Error codes returned by the server (`plate_taken`, `not_enough_money`, `finance_limit`, …) are locale keys, so every error a player can see is translatable. Category names can be translated by adding `cat_<categoryId>` to the `ui` table; plan names use `plan_<planId>`.

To add a language, copy `locales/en.lua` to `locales/<code>.lua`, change `Locales['en']` to `Locales['<code>']`, translate the values and set `Config.Locale = '<code>'`. No manifest change is needed.

***

## Compatibility

### Frameworks

| Framework  | Support                                                                                                                                                                                                                                                                                                                                                               |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ESX**    | `getSharedObject`, `GetPlayerFromId`, `GetPlayerFromIdentifier`, accounts `money` and `bank`, table `owned_vehicles` (`owner`, `plate`, `vehicle`, `type`, `stored`).                                                                                                                                                                                                 |
| **QBCore** | `GetCoreObject`, `Functions.GetPlayer`, `GetPlayerByCitizenId`, `PlayerData.money`, table `player_vehicles` (`license`, `citizenid`, `vehicle`, `hash`, `mods`, `plate`, `garage`, `state`). While a loan is open the native `balance`, `paymentamount`, `paymentsleft` and `financetime` columns are kept up to date so phones and MDTs that read them keep working. |
| **Qbox**   | `qbx_core` exports (`GetPlayer`, `GetPlayerByCitizenId`, `GetPlayerData`), `qbx_vehicles:CreatePlayerVehicle` for new vehicles, same finance columns as QBCore.                                                                                                                                                                                                       |

### Garages — `Config.Systems.Garage`

Every adapter implements the same interface and lives in `bridge/garages/` (outside the escrow). The SQL adapters only write the columns that actually exist in your vehicle table, so older or customised schemas do not break the insert.

| Adapter     | How the vehicle is written                                                                                         | Door delivery                                                                  | Repossession                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `nexus`     | `exports.nexus_garage:AddVehicle` (props, colours, class and garage), verified with `IsPlateTaken` and a read-back | `SetVehicleOut`, so nexus\_garage tracks the vehicle as out and syncs the keys | `SendVehicleToImpound(nil, plate, impoundId, reason, debt, true)`; the loan closes on `nexus_garage:server:vehicleReleased` |
| `default`   | Native table (`owned_vehicles` / `player_vehicles`)                                                                | —                                                                              | QBCore: `state = 2` and `depotprice = debt` when the column exists. ESX: custody                                            |
| `qbox`      | `qbx_vehicles:CreatePlayerVehicle({ model, citizenid, garage, props })`                                            | —                                                                              | `state = 2` and `depotprice`                                                                                                |
| `jg`        | Native table plus `garage_id` / `in_garage` when present                                                           | `jg-advancedgarages:registerVehicleOutside(plate, netId)`                      | Custody; the vehicle is removed from the world with `deleteOutsideVehicle`                                                  |
| `cd_garage` | Native table plus `garage_id`, `in_garage`, `garage_type` when present; keys with `AddOwnedKeys`                   | Persistent vehicle registration                                                | Custody, or impound state with `mode = 'impound'`                                                                           |
| `qs`        | Native table                                                                                                       | —                                                                              | Custody                                                                                                                     |
| `okok`      | Native table                                                                                                       | —                                                                              | Custody                                                                                                                     |
| `custom`    | `Functions.InsertVehicle`                                                                                          | `Functions.OnSpawnedOutside`                                                   | `Functions.Repossess`                                                                                                       |

In custody the owner column of the vehicle is set to `Finance.Repossession.custodyOwner`, so it disappears from the player's garage without being deleted; redeeming or cancelling the loan gives it back. When no mode can be applied, the loan is still marked as repossessed, a console warning is printed and `Functions.OnRepossessed` fires, so the server can handle it in its own way. `/vshop diag` shows exactly what the active adapter writes and reads back.

### Keys — `Config.Systems.Keys`

| Value            | On delivery and test drive                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------------------------------ |
| `nexus`          | Keys follow ownership in nexus\_garage (synced on purchase and when the vehicle is marked as out)                  |
| `qb-vehiclekeys` | `TriggerEvent('vehiclekeys:client:SetOwner', plate)`                                                               |
| `qs-vehiclekeys` | `exports['qs-vehiclekeys']:GiveKeys(plate, model, true)`                                                           |
| `cd_garage`      | `TriggerEvent('cd_garage:AddKeys', plate)`                                                                         |
| `okokGarage`     | `TriggerServerEvent('okokGarage:GiveKeys', plate)`                                                                 |
| `t1ger_keys`     | `exports['t1ger_keys']:GiveTemporaryKeys(plate, model, 'nexus_vehicleshop')`                                       |
| `Renewed`        | `exports['Renewed-Vehiclekeys']:addKey(plate)`                                                                     |
| `wasabi_carlock` | `exports.wasabi_carlock:GiveKey(plate)`                                                                            |
| `custom`         | `Functions.GiveKeys(vehicle, plate)`                                                                               |
| `auto`           | nexus\_garage, then qb-vehiclekeys, qs-vehiclekeys, Renewed, wasabi\_carlock and t1ger\_keys, whichever is started |

### Fuel, notifications, prompts, target and banking

The fuel bridge, notification bridge, prompt bridge and target bridge use the same values and calls as nexus\_garage and live in `bridge/client/systems.lua`, outside the escrow. With `nexus_TextUI` the dealership prompt and the display vehicle price cards are floating 3D prompts that handle the key press themselves; with any other system a marker and a help prompt are drawn. Society accounts use `Functions.GetSocietyMoney`, `Functions.AddSocietyMoney` and `Functions.RemoveSocietyMoney`, which ship with `esx_addonaccount`, `qb-banking`, `qb-management`, `okokBanking` and `Renewed-Banking` branches and an `auto` probe. When no banking resource is running, the balance is kept in an internal ledger in `nexus_vs_dealerships`. Every movement is written to `nexus_vs_transactions`.

***

## Security

* **Server-authoritative purchase.** The client sends a catalogue reference (`n:<dealershipId>:<model>` for new vehicles, `u:<id>` for pre-owned) and colour IDs. Price, stock, plate, plan, down payment limits and delivery are resolved on the server. Colour IDs outside the configured palette are rejected, and custom RGB channels must be integers from 0 to 255.
* **Distance checks.** Purchases, test drives and sell-backs are only accepted near the dealership.
* **Atomic operations.** Plates are reserved with a primary key before charging; limited stock is decremented with a conditional update; pre-owned vehicles are removed with a single delete so only one buyer can get them; offline charges use a conditional update that only succeeds when the balance is sufficient.
* **Charge, write, verify, refund.** The money is taken, the vehicle is written through the adapter and read back; if anything fails the vehicle is removed, the money refunded, the stock returned, the plate released and a `refund` log is written.
* **Per-player lock and rate limiting** on every money-moving action, plus a global request limiter.
* **Test drive isolation.** Server-owned vehicles, server timer and geofence, routing bucket, reserved plate prefix, exit blocked during the drive and cleanup on every exit path.
* **Ownership checked in the database** for sell-backs, never on the client. Financed and impounded vehicles cannot be sold back, and nexus\_garage refuses to change the owner of a financed vehicle.
* **Admin actions** require the ACE on the server for every callback, not only for opening the panel.
* **Management actions** check the rank and permission of the player on the server for every request. Settings outside the `PlayerDealers.Editable` whitelist or its limits are rejected. Society withdrawals and stock orders are atomic against the balance.
* **Thumbnail uploads** are only accepted from admins and limited in size.

***

## Commands & Permissions

| Command           | Permission                                  | Description                                                                         |
| ----------------- | ------------------------------------------- | ----------------------------------------------------------------------------------- |
| `/vshopadmin`     | `nexus.vehicleshop.admin`                   | Opens the admin panel.                                                              |
| `/vshop diag`     | `nexus.vehicleshop.admin` or server console | Runs the self-test and prints the result in the server console and the player's F8. |
| `/vshop reload`   | `nexus.vehicleshop.admin` or server console | Reloads the catalogue, stock, dealerships and loans without restarting.             |
| `/myfinances`     | everyone                                    | Opens the player's financing panel.                                                 |
| `/vshopmanage`    | dealership staff                            | Opens the management menu of a player dealership (`PlayerDealers.Command`).         |
| `X` (key mapping) | everyone                                    | Ends the current test drive. Rebindable in the FiveM key bindings.                  |

***

## Developer API

Server exports live in `server/api.lua`, client exports in `client/exports.lua`. Wait for `nexus_vehicleshop:server:ready` before calling server exports during startup.

### Client Exports

| Export           | Parameters                               | Returns   | Description                                                                                                                                     |
| ---------------- | ---------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `OpenShowroom`   | `dealershipId: number`, `model?: string` | —         | Opens the showroom of that dealership, optionally preselecting a model. Runs `Functions.CanOpenShowroom` first and respects the distance check. |
| `IsInTestDrive`  | —                                        | `boolean` | Whether the local player is currently on a test drive.                                                                                          |
| `CloseUI`        | —                                        | —         | Closes any open vehicleshop interface and restores the camera.                                                                                  |
| `OpenFinances`   | —                                        | —         | Opens the player's financing panel.                                                                                                             |
| `OpenManagement` | `dealershipId?: number`                  | —         | Opens the management menu of a player dealership the local player works at.                                                                     |

### Server Exports

| Export                                         | Parameters                                                               | Returns                                   | Description                                                                                                                                                                                                                                           |
| ---------------------------------------------- | ------------------------------------------------------------------------ | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GetCatalog`                                   | `dealershipId?: number`                                                  | `table[]`                                 | Without an ID: every valid catalogue entry (`model`, `label`, `brand`, `price`, `category`, `class`, `hidden`). With an ID: what that dealership sells, with its prices, stock and pre-owned entries (`id`, `condition`, `mileage`).                  |
| `GetVehiclePrice`                              | `model: string`, `dealershipId?: number`                                 | `number` or `nil`                         | Base price, or the dealership price when an ID is given.                                                                                                                                                                                              |
| `GeneratePlate`                                | `pattern?: string`                                                       | `string` or `nil`                         | Generates a free plate (default pattern or the one given) and reserves it for `Plates.ReservationMinutes`.                                                                                                                                            |
| `IsPlateAvailable`                             | `plate: string`                                                          | `boolean`                                 | Not registered in the garage, not reserved and not using the test prefix.                                                                                                                                                                             |
| `GiveVehicle`                                  | `identifier: string or number`, `model: string`, `opts?: table`          | `true, plate` or `false, error`           | Registers a vehicle for a character through the active garage adapter and verifies it. `opts`: `plate`, `garage`, `class`, `stored` (default `true`), `colors = { primary, secondary }` (palette IDs), `props`. For rewards, VIP packages and events. |
| `IsFinanced`                                   | `plate: string`                                                          | `boolean, table or nil`                   | Whether the plate has an open loan (active, overdue or repossessed) and its public data.                                                                                                                                                              |
| `GetFinances`                                  | `identifier: string or number`                                           | `table[]`                                 | Up to the 50 most recent loans of a character.                                                                                                                                                                                                        |
| `PayFinance`                                   | `src: number`, `plate: string`, `amount?: number or 'all'`               | `boolean, error?`                         | Pays the next instalment for that player, or settles the loan when `amount` is `'all'` or covers the remaining debt.                                                                                                                                  |
| `Repossess`                                    | `plate: string`, `reason?: string`                                       | `boolean, applied: boolean, mode: string` | Repossesses a financed vehicle immediately. `applied` tells whether the mode could be applied and `mode` which one was used.                                                                                                                          |
| `AddStock`                                     | `dealershipId: number`, `model: string`, `amount: number`                | `boolean`                                 | Adds units (turns unlimited stock into limited).                                                                                                                                                                                                      |
| `SetStock`                                     | `dealershipId: number`, `model: string`, `stock: number`, `max?: number` | `boolean`                                 | Sets stock and optionally the restock maximum. `-1` means unlimited.                                                                                                                                                                                  |
| `GetSales`                                     | `dealershipId?: number`, `fromTs?: number`, `toTs?: number`              | `table[]`                                 | Up to 500 sales with `plate`, `model`, `buyer`, `buyer_name`, `price`, `method` and `ts`.                                                                                                                                                             |
| `GetDealerships`                               | —                                                                        | `table[]`                                 | Enabled dealerships with their public data and display vehicles.                                                                                                                                                                                      |
| `GetDealershipEmployees`                       | `dealershipId: number`                                                   | `table[]`                                 | `identifier`, `name`, `rank`, `commission` of each employee.                                                                                                                                                                                          |
| `GetEmployeeRank`                              | `dealershipId: number`, `who: source or identifier`                      | `number`                                  | Rank of that player (`0` = not staff). Includes the owner and job grades.                                                                                                                                                                             |
| `IsEmployee`                                   | `dealershipId: number`, `who: source or identifier`                      | `boolean`                                 | Whether the player has any rank in that dealership.                                                                                                                                                                                                   |
| `HireEmployee`                                 | `dealershipId: number`, `who`, `rank?: number`, `commission?: number`    | `boolean, error?`                         | Adds or updates an employee of a player dealership.                                                                                                                                                                                                   |
| `FireEmployee`                                 | `dealershipId: number`, `who`                                            | `boolean, error?`                         | Removes an employee.                                                                                                                                                                                                                                  |
| `GetSocietyBalance`                            | `dealershipId: number`                                                   | `number` or `nil`                         | Current society balance of the dealership.                                                                                                                                                                                                            |
| `AddDealershipMoney` / `RemoveDealershipMoney` | `dealershipId: number`, `amount: number`, `note?: string`                | `boolean, error?`                         | Credits or debits the society account and records the transaction.                                                                                                                                                                                    |
| `GetTransactions`                              | `dealershipId: number`, `page?: number`                                  | `table[], total`                          | Society movements, newest first.                                                                                                                                                                                                                      |
| `GetDealershipStats`                           | `dealershipId?: number`, `range?: 'day' or 'week' or 'month'`            | `table`                                   | The data behind the dashboards: revenue and sales series, payment methods, top models, totals against the previous period, financing, employees and society.                                                                                          |
| `GetDealershipSettings`                        | `dealershipId: number`                                                   | `table`                                   | Effective settings of the dealership (price modifier, test drive, sell-back, delivery, commission, self-service, currency, finance plans).                                                                                                            |
| `SetDealershipSetting`                         | `dealershipId: number`, `key: string`, `value: any`                      | `boolean, error?`                         | Changes one setting. Values are validated and clamped.                                                                                                                                                                                                |
| `GetRepossessionMode`                          | —                                                                        | `string`                                  | The effective repossession mode (`impound`, `custody`, `delete`, `none`).                                                                                                                                                                             |
| `GetVehicleImage`                              | `model: string`                                                          | `string` or `nil`                         | URL of the generated thumbnail of a model, if any.                                                                                                                                                                                                    |

```lua
-- Give a vehicle as a battle pass reward
local ok, plateOrError = exports.nexus_vehicleshop:GiveVehicle(source, 'sultan', {
    colors = { primary = 145, secondary = 0 },
})
if not ok then print('reward failed: ' .. plateOrError) end

-- Block a sale in your own script while the loan is open
local financed = exports.nexus_vehicleshop:IsFinanced(plate)
if financed then
    return TriggerClientEvent('chat:addMessage', source, { args = { 'This vehicle is financed.' } })
end
```

### Events — Emitted (server, local)

| Event                                          | Payload                                               | When                                                                                                                                                                                         |
| ---------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_vehicleshop:server:ready`               | —                                                     | Tables installed, catalogue, dealerships, stock and loans loaded.                                                                                                                            |
| `nexus_vehicleshop:server:purchased`           | `src`, `data`                                         | After a successful purchase. `data`: `plate`, `model`, `price`, `method`, `paid`, `condition`, `dealership`, `dealershipName`, `financeId`, `garage`, `delivery`, `netId`, `props`, `quote`. |
| `nexus_vehicleshop:server:financePayment`      | `identifier`, `plate`, `amount`, `remaining`          | After every instalment charged.                                                                                                                                                              |
| `nexus_vehicleshop:server:financePaid`         | `identifier`, `plate`, `data`                         | When a loan is fully paid, settled early or closed by paying the impound fine.                                                                                                               |
| `nexus_vehicleshop:server:repossessed`         | `plate`, `identifier`, `debt`, `applied`, `mode`      | When a vehicle is repossessed.                                                                                                                                                               |
| `nexus_vehicleshop:server:soldBack`            | `src`, `plate`, `model`, `amount`                     | After a sell-back.                                                                                                                                                                           |
| `nexus_vehicleshop:server:testDriveEnded`      | `src`, `data`, `reason`                               | When a test drive ends. `reason`: `time`, `stopped`, `left_area`, `vehicle_lost`, `dropped`.                                                                                                 |
| `nexus_vehicleshop:server:employeeHired`       | `dealershipId`, `identifier`, `rank`, `actor`         | An employee is hired.                                                                                                                                                                        |
| `nexus_vehicleshop:server:employeeFired`       | `dealershipId`, `identifier`, `actor`                 | An employee is fired.                                                                                                                                                                        |
| `nexus_vehicleshop:server:employeeRankChanged` | `dealershipId`, `identifier`, `rank`, `actor`         | An employee's rank changes.                                                                                                                                                                  |
| `nexus_vehicleshop:server:commissionPaid`      | `dealershipId`, `identifier`, `amount`, `sale`        | A seller receives a commission.                                                                                                                                                              |
| `nexus_vehicleshop:server:offerAccepted`       | `dealershipId`, `sellerSource`, `buyerSource`, `data` | A buyer accepts an assisted sale.                                                                                                                                                            |
| `nexus_vehicleshop:server:societyTransaction`  | `dealershipId`, `kind`, `amount`, `balance`, `actor`  | Any society movement.                                                                                                                                                                        |
| `nexus_vehicleshop:server:stockOrdered`        | `dealershipId`, `model`, `quantity`, `cost`, `actor`  | A stock order is placed.                                                                                                                                                                     |
| `nexus_vehicleshop:server:stockDelivered`      | `dealershipId`, `model`, `quantity`                   | Ordered stock arrives.                                                                                                                                                                       |
| `nexus_vehicleshop:server:settingsChanged`     | `dealershipId`, `key`, `value`, `actor`               | A dealership setting changes.                                                                                                                                                                |

### Events — Listened

| Event                                       | Side                    | Payload                            | Purpose                                                                                                 |
| ------------------------------------------- | ----------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `nexus_vehicleshop:client:openShowroom`     | Client (net)            | `dealershipId`, `model?`           | Event equivalent of `OpenShowroom`.                                                                     |
| `nexus_garage:server:vehicleReleased`       | Server (local)          | `src`, `plate`, `impoundId`, `fee` | Emitted by nexus\_garage 1.0.1 when an impound fine is paid; closes the repossessed loan of that plate. |
| `nexus_vehicleshop:client:testDriveStarted` | Client (local, emitted) | `data`                             | Raised on the client when the player is seated in the test vehicle, for HUD or radio scripts.           |

### Editable Functions (`functions.lua`)

`functions.lua` is outside the escrow and split by `IsDuplicityVersion()`. Every hook is called from the protected code.

#### Server side

| Function                                                                                                                           | Signature                                                  | Called                                          | Purpose                                                                                                                     |
| ---------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `Functions.CanPurchase`                                                                                                            | `(source, dealership, model, price) -> allowed, reasonKey` | Before charging                                 | Licences, VIP-only models, garage limits of third-party garages. Return `false, 'your_locale_key'` to block with a message. |
| `Functions.HasLicense`                                                                                                             | `(source, vehicleClass) -> boolean`                        | Before test drives and purchases                | Plug in esx\_license or QBCore metadata.                                                                                    |
| `Functions.OnVehiclePurchased`                                                                                                     | `(source, data)`                                           | After a purchase                                | Same `data` as the `purchased` event.                                                                                       |
| `Functions.OnFinancePayment`                                                                                                       | `(identifier, plate, amount, remaining, data)`             | After each instalment                           | —                                                                                                                           |
| `Functions.OnFinanceMissed`                                                                                                        | `(identifier, plate, missed, data)`                        | After a failed charge                           | —                                                                                                                           |
| `Functions.OnFinancePaid`                                                                                                          | `(identifier, plate, data)`                                | Loan closed                                     | —                                                                                                                           |
| `Functions.OnRepossessed`                                                                                                          | `(plate, identifier, debt, data)`                          | After a repossession                            | Handle repossession yourself on garages without impound support.                                                            |
| `Functions.OnSellBack`                                                                                                             | `(source, plate, model, amount)`                           | After a sell-back                               | —                                                                                                                           |
| `Functions.OnTestDriveStart` / `OnTestDriveEnd`                                                                                    | `(source, data)` / `(source, data, reason)`                | Test drive lifecycle                            | —                                                                                                                           |
| `Functions.RemoveMoney` / `Functions.AddMoney`                                                                                     | `(source, amount, account, reason) -> handled, result`     | Before the built-in money logic                 | Return `true, true` to take over (custom currencies, VIP coins). The default returns `false`.                               |
| `Functions.ChargeOffline`                                                                                                          | `(identifier, amount, account) -> handled, result`         | Before the built-in offline charge              | For custom banking resources.                                                                                               |
| `Functions.CustomCurrency`                                                                                                         | `(action, source, currency, amount)`                       | Purchases at dealerships with a custom currency | `action` is `balance` (return the amount), `remove` or `add` (return `true` on success).                                    |
| `Functions.SocietySystem`                                                                                                          | `() -> string`                                             | On start and in `/vshop diag`                   | Picks the banking resource; returns `internal` when none is running.                                                        |
| `Functions.GetSocietyMoney` / `AddSocietyMoney` / `RemoveSocietyMoney`                                                             | `(society) -> number` / `(society, amount) -> boolean`     | Every society movement                          | Banking bridges with an `auto` probe.                                                                                       |
| `Functions.CanManageDealership`                                                                                                    | `(source, dealership, permission) -> boolean or nil`       | Every management request                        | Return `nil` for the default rank rules, `true` or `false` to override.                                                     |
| `Functions.OnEmployeeHired` / `OnEmployeeFired` / `OnEmployeeRankChanged`                                                          | same payload as the events                                 | Staff changes                                   | —                                                                                                                           |
| `Functions.OnCommissionPaid`, `OnSocietyTransaction`, `OnStockOrdered`, `OnStockDelivered`, `OnSettingsChanged`, `OnOfferAccepted` | same payload as the events                                 | Player dealership activity                      | —                                                                                                                           |
| `Functions.OnThumbnailSaved`                                                                                                       | `(model, path)`                                            | After a thumbnail is written                    | —                                                                                                                           |
| `Functions.InsertVehicle`, `VerifyVehicle`, `DeleteVehicle`, `GetVehicleOwner`, `DefaultGarage`, `OnSpawnedOutside`, `Repossess`   | see the adapter interface                                  | `custom` garage adapter                         | Integrate any garage without touching protected code.                                                                       |

#### Client side

| Function                                          | Signature                                              | Purpose                                                                         |
| ------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------- |
| `Functions.Notify`                                | `(message, kind)`                                      | Custom notifications.                                                           |
| `Functions.ShowTextUI` / `HideTextUI`             | `(message)` / `()`                                     | Custom prompts.                                                                 |
| `Functions.AddTarget` / `RemoveTarget`            | `(id, coords, radius, options) -> handle` / `(handle)` | Custom target.                                                                  |
| `Functions.GiveKeys`                              | `(vehicle, plate)`                                     | Custom keys.                                                                    |
| `Functions.SetFuel`                               | `(vehicle, fuel)`                                      | Custom fuel.                                                                    |
| `Functions.VehicleImage`                          | `(model) -> url or nil`                                | Point thumbnails to your own image pack (for example `nui://my_images/%s.png`). |
| `Functions.CanOpenShowroom`                       | `(dealershipId) -> boolean`                            | Block the showroom (handcuffed, dead, wanted…).                                 |
| `Functions.OnShowroomOpened` / `OnShowroomClosed` | `(dealershipId)`                                       | Hide your HUD, pause music, etc.                                                |
| `Functions.OnVehicleDelivered`                    | `(vehicle, plate)`                                     | Extras on the delivered vehicle.                                                |
| `Functions.OnManagementOpened`                    | `(dealershipId)`                                       | Hide your HUD while the management menu is open.                                |

### Garage adapter interface

```lua
VS.Garage.Register('my_garage', {
    insert = function(identifier, model, plate, props, garageKey, vehicleClass, stored) return true end,
    verify = function(plate, identifier) return true end,
    delete = function(plate) return true end,
    defaultGarage = function(vehicleClass) return 'legion' end,
    onSpawnedOutside = function(plate, netId, garageKey) end,
    getOwner = function(plate) return identifier, model, mileage end,
    isImpounded = function(plate) return false end,
    isPlateTaken = function(plate) return false end,
    repossess = function(plate, identifier, debt, reason) return false end,
    isReleased = function(plate) return false end,
    removeOutside = function(plate) end,
})
```

Add a new file to `bridge/garages/` (and to `server_scripts` in the manifest) or simply implement the `custom` adapter through `functions.lua`. `VS.Garage.NativeInsert`, `NativeGet`, `NativeVerify`, `NativeDelete` and `NativeSetState` are available as helpers for SQL-based garages. `isReleased` lets impounded loans close when the vehicle leaves the impound, and `removeOutside` removes a repossessed vehicle from the world; both are optional.

### Database Schema

| Table                   | Purpose                                                                                                                                                                                                                                                                                                        |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_vs_dealerships`  | Dealerships created in-game: type (`server` or `player`), owner, class, categories, job restriction, society, interaction, management point, delivery spawns, showroom point, test drive settings, display slots, sell-back point, blip, settings, internal balance, enabled.                                  |
| `nexus_vs_stock`        | Per-dealership overrides: price, stock (`-1` unlimited), restock maximum, hidden.                                                                                                                                                                                                                              |
| `nexus_vs_used`         | Pre-owned vehicles: dealership, model, price, mileage, properties, seller.                                                                                                                                                                                                                                     |
| `nexus_vs_plates`       | Short-lived plate reservations (primary key on the plate).                                                                                                                                                                                                                                                     |
| `nexus_vs_finance`      | Loans: plate (unique), owner, dealership, model, plan, price, down payment, total, paid, instalment, instalments total and left, interval, next due date, missed payments, custody data, status (`active`, `grace`, `repossessed`, `seized`, `paid`, `cancelled`). Indexed by status, due date and dealership. |
| `nexus_vs_sales`        | Sales history with buyer, seller, price, method, discount, extras and commission.                                                                                                                                                                                                                              |
| `nexus_vs_employees`    | Employees of player dealerships: rank, commission, name and who hired them.                                                                                                                                                                                                                                    |
| `nexus_vs_transactions` | Society movements of every dealership: kind, amount, balance after, actor and note.                                                                                                                                                                                                                            |
| `nexus_vs_orders`       | Stock orders: model, quantity, cost, who ordered, delivery time and status.                                                                                                                                                                                                                                    |
| `nexus_vs_catalog`      | Catalogue entries edited in-game, applied on top of `config/catalog.lua`.                                                                                                                                                                                                                                      |
| `nexus_vs_logs`         | Event log mirrored to Discord.                                                                                                                                                                                                                                                                                 |

The schema is upgraded on start by adding missing columns and indexes only; existing rows are never rewritten. `/vshop diag` reports whether the schema is up to date. Vehicles themselves are always stored in your garage or framework table, never duplicated. Plates are stored uppercase and trimmed, and matched against both the trimmed and the 8-character padded form.

### Integration Example

A gang script that sells armoured vans only to its members, logs every sale to its own channel and greets the buyer in chat.

```lua
-- nexus_vehicleshop/functions.lua (server half)
function Functions.CanPurchase(source, dealership, model, price)
    if model == 'stockade' then
        local gang = exports['my_gangs']:GetGang(source)
        if not gang then return false, 'purchase_blocked' end
    end
    return true
end

function Functions.OnVehiclePurchased(source, data)
    if data.model == 'stockade' then
        TriggerClientEvent('chat:addMessage', source, { args = { 'Gang', ('Van %s registered to the crew'):format(data.plate) } })
    end
end
```

```lua
-- my_gangs/server.lua
AddEventHandler('nexus_vehicleshop:server:purchased', function(src, data)
    if data.method == 'finance' then
        print(('%s financed a %s (%s), instalment %d'):format(GetPlayerName(src), data.model, data.plate, data.quote.installment))
    end
end)

AddEventHandler('nexus_vehicleshop:server:repossessed', function(plate, identifier, debt)
    TriggerEvent('my_gangs:server:alert', identifier, ('Your vehicle %s was repossessed'):format(plate))
end)
```

***

## FAQ

**The console says the folder must be called `nexus_vehicleshop`.** Rename the folder exactly like that, without version suffixes or capitals.

**Which garage is being used?** The start line prints it (`garage adapter: nexus`). With `Config.Systems.Garage = 'auto'` the script picks nexus\_garage when it is running, `qbx_vehicles` on Qbox, then jg, cd\_garage, qs and okok, and the framework tables otherwise. Make sure your garage starts before `nexus_vehicleshop`; if it starts later the adapter is resolved again automatically.

**`/vshop diag` fails on "nexus\_garage 1.0.1 exports".** The installed nexus\_garage is older than 1.0.1 and does not expose `AddVehicle`. Update it.

**Bought vehicles do not appear in my third-party garage.** Run `/vshop diag` and check which columns are written and read back. Set `Config.Delivery.defaultGarages` to the garage names your resource expects, or implement the `custom` adapter in `functions.lua`.

**Can players spawn inside the vehicle instead of receiving it in the garage?** That is the default (`Config.Delivery.mode = 'door'`). Place enough delivery spawns; when all of them are blocked the vehicle goes to the garage and the player is told so.

**The showroom shows the wrong place or the vehicle falls through the floor.** Set a **Showroom camera** point for that dealership in the admin panel, on open, flat ground. Dealerships without one use `Config.Showroom.fallbackPreview`.

**Boats or planes do not spawn for test drives.** Give each catalogue entry the right `type` (`boat`, `plane`, `heli`) and place the dealership's delivery or test drive point on water or on a runway.

**Financing charges are not happening.** Instalments are processed every `Finance.TickSeconds` in real time. Players need enough money in `Finance.Account`. Offline charges require `Finance.ChargeOffline = true`. For testing, create a plan with `intervalMinutes = 2`.

**A repossessed vehicle never went to the impound.** The impound is used with the `nexus`, `default` (QBCore) and `qbox` adapters. Other garages use custody by default: the vehicle leaves the player's garage and can be redeemed from `/myfinances`. Check the active mode with `/vshop diag`. With nexus\_garage check that `Finance.Repossession.impoundId` matches an existing impound.

**Who receives the impound fine of a repossessed vehicle?** The society configured on that nexus\_garage impound. Assign the impound to the dealership job if the dealership should collect it.

**Can a financed vehicle be sold or transferred?** No. Sell-back refuses it, and nexus\_garage 1.0.1 refuses to change its owner (admin panel and `SetVehicleOwner`) while the loan is open.

**A model is missing from the showroom.** Check the server console on start: invalid catalogue entries are listed and hidden. Addon models that are not streamed are hidden on the clients that cannot load them and reported in their F8 console.

**Vehicle thumbnails are empty.** Without generated thumbnails they come from `docs.fivem.net`, which has no images for addon vehicles. Generate them in `/vshopadmin` → **Catalog** → **Generate missing** (requires `screenshot-basic`) and restart the resource, or return your own URLs from `Functions.VehicleImage`.

**Can I use my own currency or a VIP dealership?** Define the currency in `Config.Currencies`, select it in the dealership settings and implement `Functions.CustomCurrency`. Restrict the dealership to a job or use `Functions.CanPurchase` for eligibility.

**How do player dealerships get vehicles?** Staff with the `orderStock` permission order units in the **Stock** tab at `PlayerDealers.WholesalePercent` of the list price, paid from the society. Units arrive after `PlayerDealers.DeliveryMinutes`. Automatic restocking does not apply to player dealerships.

**Where does the money of a player dealership go?** Every sale and instalment is credited to its society account; commissions, stock orders and sell-backs are paid from it. Staff with `deposit` or `withdraw` can move money in the **Society** tab, and every movement is listed there.

**Do server dealerships create money on sell-backs?** By default yes (`Config.SellBack.serverFunds = 'mint'`). Set it to `'society'` to pay sell-backs from the dealership account instead.

**Does the database upgrade touch existing data?** No. It only adds missing tables, columns and indexes. Back up the database before updating anyway.

**Are display vehicles networked?** No. They are local entities created only near the dealership, so they cost nothing to the server and cannot be stolen or deleted by other players.

### Before opening a ticket

* Make sure the folder is named exactly `nexus_vehicleshop` and you are on the latest version.
* Make sure oxmysql, the framework and your garage start before `nexus_vehicleshop`.
* Run `/vshop diag` and include its output.
* Set `Config.Debug = true`, reproduce the problem and include the full server and F8 console output.
