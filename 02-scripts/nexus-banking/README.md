---
icon: building-columns
---

# Nexus Banking

> A secure, drop-in banking system for ESX, QBCore and Qbox: personal, shared, business, gang and savings accounts, a real credit score, custom loans secured with cars or houses, vehicle financing for nexus\_vehicleshop, ATMs and branches created in game, cards with PIN, bills and standing orders, all behind a card-first React interface and a transaction engine that validates and executes every operation atomically on the server.

**Version:** 1.1.0 · **Framework(s):** ESX, QBCore, Qbox · **Database:** MySQL / MariaDB (oxmysql)

***

## 📝 Description

Nexus Banking replaces your banking resource with one built around a single rule: **the client only asks, the server decides**. Every deposit, withdrawal, transfer, bill, loan payment or payout goes through one transaction engine. The engine validates the amount, checks the player's permission on that account, checks the player is physically at a branch or ATM, locks the accounts involved, moves the money with a conditional SQL update and writes the history. If any step fails, nothing is charged.

The **personal balance stays in your framework** (`xPlayer` bank account on ESX, `PlayerData.money.bank` on QBCore/Qbox). Every other script that calls `removeAccountMoney('bank')` or `RemoveMoney('bank')` keeps working exactly as before; Nexus Banking adds the history, the interface and the rules on top. Shared, business (society), gang, savings and legacy addon accounts live in the resource's own tables.

Every player has a **credit score from 300 to 850** built from their real behaviour: paying on time, debt, income, account age, bills and savings. The score decides which loans they can get, their limit and their rate. **Custom loans** (personal, vehicle, property and business) can be secured with a car or a house. When a player stops paying, the bank seizes the collateral, and it only ever clears as much debt as the asset is worth. With **nexus\_vehicleshop**, car financing runs through a real bank loan that pays the dealership as instalments come in.

Branches and ATMs are stored in the database and managed from an in-game admin panel (`/bankadmin`). Eight default branches are created on first start, and every ATM prop on the map works out of the box.

For servers coming from another bank, the resource ships **compatibility folders** that impersonate Renewed-Banking, qb-banking, qb-management, esx\_addonaccount, okokBanking and ps-banking, plus a migrator that imports existing balances.

***

## ✨ Features

*   **Atomic, server-validated transactions**:

    * one validator for every amount: a whole number, greater than zero, below the limit, never NaN, infinite or text;
    * authorization recalculated on the server;
    * conditional `UPDATE ... WHERE balance >= ?`;
    * per-account locks, one request at a time per player and idempotent requests.

    See [Security](./#security).
* **Credit score** (300-850):
  * six weighted factors and five tiers;
  * daily and weekly caps on how fast it can rise, plus anti-farming rules;
  * a dedicated screen with an animated gauge, the factors, tips and history.
* **Custom loans**:
  * personal, vehicle, property and business types, each with its own amounts, terms, frequencies, rates and fees;
  * fixed-instalment, simple-interest and interest-only-start methods;
  * a live simulator with a payment chart;
  * early repayment, partial or full.
* **Collateral**:
  * pledge a car (valued from nexus\_vehicleshop prices with depreciation and mileage) or a house (ps-housing, qb-houses, qs-housing or your own housing script);
  * pledges are locked in the database and checked every few minutes;
  * the collateral is seized on default.
* **Vehicle financing for nexus\_vehicleshop**:
  * real bank loans with rates and down payments based on the credit tier;
  * dealers are paid per instalment;
  * repossession on default;
  * payments from any account the buyer can use.
* **Account types**:
  * personal (framework balance);
  * shared with roles (owner, manager, contributor, viewer);
  * business accounts per job and gang accounts, with permissions by grade;
  * savings jars, and addon accounts for legacy ESX scripts.
* **Transfers by IBAN**:
  * recipients are looked up and confirmed before sending;
  * recent contacts are suggested;
  * offline players are supported;
  * an optional fee can be charged.
* **Branches and ATMs**:
  * branches with marker, NPC, target or nexus\_TextUI prompts, with per-branch colour themes and bank logos;
  * map ATMs detected by prop model;
  * ATM limits per withdrawal and per day.
* **ATM experience**:
  * the card slides into the machine while the server checks it in parallel;
  * the animation lasts under a second and can be skipped;
  * a two-column PIN pad that also works with the physical keyboard.
* **Cards and PIN** (optional):
  * virtual cards or inventory items with metadata;
  * six designs (standard, gold, black, platinum, business, junior) with item images included;
  * tiers with daily limits and salted SHA-256 PINs;
  * blocking, freezing, PIN change, replacement and cancellation.
* **Bills**:
  * invoice nearby players (`/invoice`) and pay from anywhere (`/bills`);
  * due dates, late fees and optional automatic charge;
  * paid and overdue bills from businesses affect the credit score.
* **Savings jars**: goals and capped interest for active players only.
* **Standing orders**: recurring payments that re-check permissions on every run.
* **Plan loans** (v1 quick plans) are still available next to the custom loans.
* **Illustrated empty states** on every screen, so new players never see a blank panel.
* **Admin panel**: statistics, selftest, account search, balance adjustments with a mandatory reason, freezing, CSV export, audit log and a branch/ATM editor.
* **Police tools**: freeze and unfreeze accounts by IBAN for configured jobs.
* **7 languages**, a configurable currency, five colour themes, three built-in bank logos or your own SVG, and every text editable.

***

## 📋 Dependencies

### Required

| Dependency                                                               | Notes                                                                                                                           |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| **ESX** (`es_extended`), **QBCore** (`qb-core`) or **Qbox** (`qbx_core`) | Auto-detected, or forced with `Config.Framework`.                                                                               |
| **oxmysql**                                                              | The script uses `MySQL.*.await`. mysql-async is not supported.                                                                  |
| **MySQL 8 / MariaDB 10.2+**                                              | The `nexus_bank_*` tables are created and upgraded automatically. The balance `CHECK` constraint is enforced on these versions. |

### Optional

| Dependency                                                      | What it's for                                               | Selected with                                                         |
| --------------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------- |
| `nexus_vehicleshop` 1.1.0+                                      | Bank financing, vehicle prices for collateral, repossession | `Config.Lending.vehicleShop`, `Config.Collateral.vehicle.priceSource` |
| `ps-housing`, `qb-houses`, `qs-housing`                         | Houses as collateral                                        | `Config.Collateral.property.system`                                   |
| `nexus_notify`                                                  | Notifications (default)                                     | `Config.Systems.Notify = 'nexus'`                                     |
| `nexus_TextUI`                                                  | Prompts at branches and ATMs (default)                      | `Config.Systems.TextUI = 'nexus'`                                     |
| `okokNotify`, `ox_lib`, `mythic_notify`, `ps-ui`, ESX/QB notify | Alternative notifications                                   | `Config.Systems.Notify`                                               |
| `okokTextUI`, `ox_lib`, `ps-ui`, ESX/QB TextUI                  | Alternative prompts                                         | `Config.Systems.TextUI`                                               |
| `ox_target` / `qb-target`                                       | Target mode on branches and ATM models                      | `Config.Systems.Target`                                               |
| `ox_inventory`, `qb-inventory` (and forks), `qs-inventory`      | Cards as items                                              | `Config.Systems.Inventory`, `Config.Cards.mode = 'item'`              |

***

## ⚙️ Installation

1. **Download** the resource from the cfx.re portal (Keymaster) and drop it into your `resources` folder.
2. **Keep the folder name exactly `nexus_banking`.** The resource validates its own name and stops otherwise.
3. **Remove or stop your previous bank** (qb-banking, Renewed-Banking, okokBanking, esx\_banking...). Two banks must never run at the same time.
4. **Optional compatibility**: copy the folders you need from `nexus_banking/compat/` (for example `compat/qb-banking`) into your resources. See [Compatibility](./#compatibility).
5. **server.cfg order** (the bank must start before nexus\_vehicleshop):

```
ensure oxmysql
ensure es_extended        # or qb-core / qbx_core
ensure nexus_notify
ensure nexus_TextUI
ensure nexus_banking
ensure qb-banking         # only if you use the compatibility folder
ensure nexus_vehicleshop  # optional, after the bank
```

6. **Permissions** for the admin panel and the selftest:

```
add_ace group.admin nexus.banking.admin allow
```

7. **Recommended convars**:

```
set nexus_banking:pepper "a-long-random-secret"     # used to hash card PINs
set nexus_banking:webhook "https://discord.com/api/webhooks/..."
```

8. **Card item images** (only with `Config.Cards.mode = 'item'`): copy the PNG files from `install/images/` into your inventory's image folder. `install/images/README.md` lists the path for each inventory.
9. **Start the server.** The tables are created on first start and upgraded automatically on updates. New columns are only added if they don't exist yet, so restarting is always safe. `sql/install.sql` is also included if you prefer to import it by hand.
10. Run `/bankaudit selftest` from the server console. Every check must say PASS.
11. **Loans**: turn on `Config.Modules.loans`, then fund the reserve account. Search `bank_reserve` in `/bankadmin` → Accounts and adjust its balance.
12. **Coming from another bank?** Run `/bankaudit migrate <source>` first (dry run), review the output, then run `/bankaudit migrate <source> apply`.

{% hint style="warning" %}
Set the `nexus_banking:pepper` convar before players create cards. Changing it later invalidates every existing PIN. The console warns you at start-up if cards exist and the convar is missing.
{% endhint %}

{% hint style="info" %}
Updating from 1.0? Replace the files and restart. The update adds four tables (`nexus_bank_credit`, `nexus_bank_credit_events`, `nexus_bank_credit_snapshots`, `nexus_bank_liens`) and new columns to `nexus_bank_loans`. Existing loans keep working as plan loans. If your database user has no `ALTER` permission, the console tells you which columns are missing.
{% endhint %}

***

## 🔧 Configuration

All configuration lives in `config/` (outside the escrow).

### `config/config.lua`

| Option                                   | Default                                      | Description                                                                                                                                                                                                                                            |
| ---------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Config.Framework`                       | `'auto'`                                     | `auto`, `esx`, `qbcore` or `qbox`.                                                                                                                                                                                                                     |
| `Config.Locale`                          | `'en'`                                       | `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`.                                                                                                                                                                                                              |
| `Config.Bank`                            | `Trading Monarchy` / `TM`                    | Fictional bank name, short brand shown on cards, default theme and IBAN prefix.                                                                                                                                                                        |
| `Config.Themes`                          | 5 themes                                     | `monarchy`, `obsidian`, `emerald`, `crimson`, `ocean`. Each theme has `accent`, `accentSoft`, `chip`, a 3-colour `card` gradient and a `logo` (`monarch`, `pillar` or `laurel`). Set `logoSvg` to use your own SVG. Branches can use different themes. |
| `Config.Systems`                         | nexus / nexus / none / auto                  | Notify, TextUI, Target and Inventory bridges.                                                                                                                                                                                                          |
| `Config.Commands`                        | `bankadmin`, `bankaudit`, `bills`, `invoice` | Command names.                                                                                                                                                                                                                                         |
| `Config.AdminAce`                        | `nexus.banking.admin`                        | ACE for the admin panel, selftest and migrator.                                                                                                                                                                                                        |
| `Config.Currency`, `Config.NumberLocale` | `$`, `en-US`                                 | Currency symbol and number format in the interface.                                                                                                                                                                                                    |
| `Config.Modules`                         | see below                                    | Turns each feature on or off: `shared`, `society`, `gang`, `cards` (off), `bills`, `savings`, `standingOrders`, `loans` (off), `credit` (on), `police`.                                                                                                |
| `Config.Limits`                          |                                              | `maxTransaction` (10,000,000), transfer fee, offline transfers, shared account limits, message length.                                                                                                                                                 |
| `Config.Security`                        |                                              | Rate limit (5 requests every 3 s), branch and ATM distances, maximum movement while the interface is open, session length, `strictAtms`, abuse threshold.                                                                                              |
| `Config.Society` / `Config.Gangs`        |                                              | Who can `view`, `deposit`, `withdraw`, `transfer`, send `bills` and create standing `orders`: `'boss'`, `'employee'` or a minimum grade, with per-job overrides.                                                                                       |
| `Config.History`                         | 25 / 60 days                                 | Page size, retention before archiving, statement days. Old credit events and snapshots are cleaned up with the same retention.                                                                                                                         |
| `Config.Interaction`, `Config.Blips`     |                                              | Branch interaction mode, NPC, marker and blips.                                                                                                                                                                                                        |
| `Config.Bills`                           |                                              | Maximum amount, pending limit, due hours, overdue fee, automatic charge, issuer distance, `payAnywhere`.                                                                                                                                               |
| `Config.StandingOrders`                  |                                              | Minimum interval, max per account, failures before stopping.                                                                                                                                                                                           |
| `Config.Police`                          | police 2, sheriff 2                          | Jobs and minimum grade allowed to freeze accounts.                                                                                                                                                                                                     |
| `Config.Logs`                            |                                              | Console logs and Discord embeds. The webhook URL comes from the `nexus_banking:webhook` convar.                                                                                                                                                        |
| `Config.Interface`                       |                                              | Scale, sounds, reduced motion, quick amounts.                                                                                                                                                                                                          |

### `config/credit.lua`

| Option                                          | Default                                                                      | Description                                                                                                  |
| ----------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `min` / `max` / `initial`                       | 300 / 850 / 580                                                              | Score range and the score of a new player.                                                                   |
| `maxGainPerDay` / `maxGainPerWeek`              | 15 / 45                                                                      | How fast the score can rise. Drops are not capped.                                                           |
| `cacheMinutes`                                  |                                                                              | How long a computed score is reused.                                                                         |
| `lookbackDays`                                  | 120                                                                          | How far back events count.                                                                                   |
| `tiers`                                         | poor 300, fair 580, good 670, very\_good 740, excellent 800                  | Tier thresholds (`from`) and colours.                                                                        |
| `weights`                                       | payments 35, utilization 15, debt\_income 15, age 10, records 15, balance 10 | Weight of each factor. Set one to 0 to ignore it.                                                            |
| `events`                                        |                                                                              | Weight of each event: `loan_ontime`, `loan_missed`, `loan_paid`, `loan_default`, `bill_ontime`, `bill_late`. |
| `maxEventsPerDay`                               | 4                                                                            | Positive events that can count per player per day.                                                           |
| `billMinAmount`                                 | 250                                                                          | Smaller bills don't affect the score.                                                                        |
| `minLoanForEvents`                              | 1000                                                                         | Smaller loans don't affect the score.                                                                        |
| `ageFullDays`, `balanceTarget`, `maxDebtIncome` |                                                                              | Targets for the age, balance and debt-to-income factors.                                                     |

How it works:

* A payment only counts as "on time" in the second half of its period, so paying the moment a loan is taken does not farm points.
* `loan_paid` only counts if the loan lived at least two periods.
* A business can lower a player's score with a late bill at most once a day.
* Income is computed from salaries and payments sent by scripts or businesses, never from transfers between players. Feed real salaries with `Functions.CreditIncome` for the best results.

### `config/lending.lua`

`Config.Lending` (needs `Config.Modules.loans = true`):

| Option                     | Default                               | Description                                                                                                                                                                                                                                      |
| -------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`, `requireBranch` | true, true                            | Custom loans on, and only at a branch.                                                                                                                                                                                                           |
| `maxActive`                | 2                                     | Custom loans per player at once.                                                                                                                                                                                                                 |
| `maxDebtIncome`            | 0.5                                   | Max share of income that can go to instalments.                                                                                                                                                                                                  |
| `noIncomeAllowance`        | 15000                                 | Monthly instalments allowed when the player has no income data.                                                                                                                                                                                  |
| `dailyServerCap`           | 2,000,000                             | Money lent per day by the whole server.                                                                                                                                                                                                          |
| `frequencies`              | daily 1440, weekly 10080              | Instalment intervals in minutes.                                                                                                                                                                                                                 |
| `types`                    | personal, vehicle, property, business | Per type: `min`, `max`, `minPeriods`, `maxPeriods`, `frequencies`, `methods`, `rate` (% per real day), `fee` (%), `collateral` (`none`, `vehicle`, `property`, `any`), `requireCollateral`, `ltv` (%), `graceMax`, `minScore`, `incomeMultiple`. |
| `tiers`                    |                                       | Per credit tier: `rateAdd`, `limit` (share of the type maximum), `ltv` multiplier, `allowed`.                                                                                                                                                    |
| `securedDiscount`          | 0.15                                  | Rate discount when collateral is pledged.                                                                                                                                                                                                        |
| `late`                     | 60 min, 5 %, 2, 3                     | `graceMinutes` to retry a failed charge, `feePercent`, missed instalments in a row before the score drops (`scoreDropAt`), missed instalments before default (`maxMissed`).                                                                      |
| `prepay`                   |                                       | `allowed`, `partial`, `feePercent` (1), `minPartial` (500), `mode` (`reduce_installment` or `reduce_term`).                                                                                                                                      |
| `seizure`                  |                                       | `vehicle` mode (`custody`, `impound`, `delete`, `stock`, `event`), `custodyOwner`, `stockDealership`, `property` (`adapter` or `event`), `onAssetLost` (`accelerate`, `default`, `flag`).                                                        |
| `vehicleShop`              |                                       | See [Vehicle financing](./#vehicle-financing-with-nexus_vehicleshop).                                                                                                                                                                            |

`Config.Collateral`:

| Option                                                         | Description                                                                          |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `vehicle.priceSource`                                          | `nexus_vehicleshop` (catalogue prices) or `config` (`prices` table).                 |
| `vehicle.depreciation`, `mileagePerKm`, `minPercent`           | Value = price minus depreciation and mileage, never below `minPercent` of the price. |
| `vehicle.allowUnknown`, `fallbackValue`                        | Models without a known price are refused by default.                                 |
| `vehicle.maxValue`                                             | Cap on any single vehicle's value.                                                   |
| `vehicle.blockFinanced`                                        | Cars still financed at the dealership can't be pledged.                              |
| `property.system`                                              | `auto`, `ps-housing`, `qb-houses`, `qs-housing`, `custom` or `none`.                 |
| `property.depreciation`, `fallbackValue`, `maxValue`, `prices` | Property valuation.                                                                  |

### `config/cards.lua`

`mode` (`virtual` or `item`), item name, card requirement at ATMs and branches, PIN length, maximum attempts, cards per player and fees. Tiers set the daily limit, price and `design`. `itemImages = true` gives each item the image `bank_card_<design>`. `designs` defines the six built-in designs (base colour, accent, text, chip, pattern, finish, foil, sheen and edge) so you can restyle them or add your own.

### `config/atms.lua`

`Config.Atm` sets the ATM prop models, use distance, `maxWithdraw`, `dailyWithdraw`, deposits and transfers. `cardAnimation` controls the card insert animation:

* `enabled`: turn the animation on or off;
* `speed`: 1 is under one second;
* `skippable`: click or press a key to skip it.

### `config/savings.lua`, `config/loans.lua`, `config/compat.lua`

Savings jar interest and limits; v1 plan loans (`funding`, `fundingAccount`, plans, `maxDebt`, cooldowns, `dailyCap`); compatibility folder options.

***

## 🖥️ Commands

| Command                               | Who                          | Description                                                                                                                                   |
| ------------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `/bankadmin`                          | ACE                          | Opens the admin panel: overview, selftest, accounts, branches and ATMs, audit.                                                                |
| `/bankaudit selftest`                 | ACE / console                | Runs the security and loan maths selftest and prints PASS/FAIL.                                                                               |
| `/bankaudit migrate <source> [apply]` | ACE / console                | Imports balances from `esx_addonaccount`, `qb-banking`, `qb-management`, `Renewed-Banking` or `okokBanking`. Without `apply` it is a dry run. |
| `/bankaudit archive`                  | ACE / console                | Archives old transactions now.                                                                                                                |
| `/bills`                              | Everyone                     | Opens your bills from anywhere.                                                                                                               |
| `/invoice`                            | Players with bill permission | Sends a bill to a nearby player.                                                                                                              |
| `/bankfreeze <IBAN> <reason>`         | Police / ACE                 | Freezes an account.                                                                                                                           |
| `/bankunfreeze <IBAN>`                | Police / ACE                 | Unfreezes an account (admin freezes need ACE).                                                                                                |

***

## 🏦 Branches and ATMs in game

Open `/bankadmin` → **Branches**:

* **New point**: choose **Branch** or **ATM**, name it and set its position. You can use **My position**, **Capture** (walk to the spot and press `E`) or, for ATMs, **Nearest ATM**.
* Branches can override the interaction mode, the NPC model, the colour theme (with its logo) and their blip.
* Every point can be edited, deleted, teleported to or marked on the map. Changes reach every player instantly.

Map ATMs work without placing anything. With `Config.Security.strictAtms = true`, only ATMs placed from the panel are accepted.

{% hint style="warning" %}
With `strictAtms = false` (default), a modified client could open an ATM session anywhere. The damage is limited to the player's own accounts and the ATM limits, but serious servers should place their ATMs with the panel and turn `strictAtms` on.
{% endhint %}

***

## 📈 Credit score

Players open the **Credit** tab at any branch. It shows:

* the score on an animated gauge with the five tiers;
* whether the score is trending up or down, and the daily and weekly caps;
* each factor with its weight and a short explanation;
* up to three tips for the weakest factors;
* the history of events that moved the score.

| Factor         | What it measures                                      |
| -------------- | ----------------------------------------------------- |
| Payments       | On-time vs missed instalments and bills.              |
| Utilization    | Current debt compared with the player's credit limit. |
| Debt to income | Monthly instalments compared with monthly income.     |
| Age            | Days since the player's first account.                |
| Records        | Paid loans, defaults and late bills.                  |
| Balance        | Average balance over the last weeks.                  |

Other scripts can read the score with `GetCreditScore` (a number) or `GetCreditProfile` (score, tier and factors). They can add their own events with `AddCreditEvent`, which is capped at ±10 points per event and counts towards the daily limit. You can add custom factors with `Functions.CreditFactors`.

***

## 💳 Custom loans and collateral

Players open **Loans** at a branch, pick a type and move the sliders. The simulator asks the server for a real quote (rate, fee, instalment, total cost and the full schedule) and shows a payment chart. Accepting creates the loan, moves the money from the bank reserve and locks any collateral in the same step.

**Methods**

| Method    | How it pays                                                                      |
| --------- | -------------------------------------------------------------------------------- |
| `annuity` | Same instalment every period.                                                    |
| `simple`  | Equal principal every period plus interest on what's left.                       |
| `grace`   | Interest only for up to `graceMax` periods, then annuity.                        |
| `flat`    | Used only by nexus\_vehicleshop financing: same formula as the dealership quote. |

**Limits and rates**

* The limit comes from the type maximum × the tier `limit` × the type `ltv`, plus the value of the collateral.
* The part not covered by collateral also counts towards `Config.Loans.maxDebt`.
* The rate is the type rate + the tier `rateAdd`, minus `securedDiscount` when collateral is pledged, and never below `minRate`.
* A player can't take more than `maxDebtIncome` of their income in instalments.

**Paying**

* Instalments are charged automatically from the personal account, online or offline.
* From the loan card, players can pay the next instalment, make a partial early payment or pay everything off.
* Early payments include the interest accrued so far and a small fee. The rest is rescheduled with `reduce_installment` or `reduce_term`.

**Missed payments**

1. If the charge fails, the bank retries for `graceMinutes`.
2. Then the instalment is missed: a late fee of `feePercent` plus that period's interest is added.
3. After `scoreDropAt` missed instalments in a row, the score drops.
4. After `maxMissed`, the loan defaults and the collateral is seized. Seizing clears debt up to the collateral's value; the rest stays as defaulted debt.

**Collateral**

* Players can only pledge cars and houses they own. Ownership is checked on the server.
* Each asset can only back one loan at a time (unique lock in `nexus_bank_liens`).
* A pledged car can't be sold back to nexus\_vehicleshop.
* Every five minutes the bank checks that pledged assets still belong to the borrower. If one is gone, `onAssetLost` decides: demand the full amount (`accelerate`), default straight away or only flag the loan.
* Seizing never touches a vehicle that now belongs to someone else.

Vehicle seizure modes:

| Mode      | What happens                                                                          |
| --------- | ------------------------------------------------------------------------------------- |
| `custody` | The car is moved to the `custodyOwner` identifier (removed from the player's garage). |
| `impound` | Sent to the nexus\_garage impound (falls back to custody).                            |
| `delete`  | Removed from the database.                                                            |
| `stock`   | Returned to a nexus\_vehicleshop dealership as stock.                                 |
| `event`   | Nothing is done; handle `onCollateralSeized` yourself.                                |

{% hint style="info" %}
ps-housing and qb-houses keep properties in memory. After a house is seized, those scripts may only show the change after they restart. Use `property = 'event'` and your own handler if you need it to be instant.
{% endhint %}

***

## 🚗 Vehicle financing with nexus\_vehicleshop

With nexus\_vehicleshop 1.1.0 or newer and `Config.Lending.vehicleShop.enabled = true`, financed purchases become real bank loans:

1. The shop asks the bank for the buyer's terms: tier, extra interest (`interestByTier`) and minimum down payment (`minDownByTier`). Tiers in `blockedTiers` can't finance.
2. The bank checks affordability (`checkAffordability`, `noIncomeAllowance`) and reserves the loan.
3. The shop takes the down payment and delivers the car. Only then is the loan activated; if anything fails, the loan is cancelled and the money returned.
4. The bank collects the instalments. With `dealerPayout = 'installments'` (default), the dealership account receives the principal part of every instalment as it is paid. With `'upfront'`, it receives the full financed amount at the sale.
5. On default the bank calls `exports.nexus_vehicleshop:Repossess(plate, reason, repossessMode)`.

Staff can't finance a car through the bank at their own dealership. Players can also pay instalments and pay off the loan from the bank's Loans tab. A car that has already been repossessed must be redeemed at the dealership.

{% hint style="warning" %}
`upfront` is easier for player-run dealerships but pays the whole loan from the bank reserve at the moment of the sale. Keep `installments` unless you fund the reserve generously.
{% endhint %}

***

## 🌐 Locales & Editable Strings

* Languages: English (default), Spanish, German, French, Italian, Portuguese and Chinese in `locales/*.lua`.
* Server texts are top-level keys with `%s` placeholders. Error codes returned by the server are locale keys.
* Interface texts are in the `ui` table of each locale, with `{name}` placeholders.
* To add a language, copy `locales/en.lua`, change `Locales['en']` and set `Config.Locale`.
* The interface (`html/`) is outside the escrow. The React source is in `web/` (`npm install`, then `npm run build`).

***

## 🔗 Compatibility

### Frameworks

ESX Legacy, QBCore and Qbox. Gangs come from `PlayerData.gang` on QBCore/Qbox; on ESX, fill in `Functions.GetGang`. Vehicle collateral reads `owned_vehicles` (ESX) or `player_vehicles` (QBCore/Qbox).

### Housing

| System        | Table used                                         | Status                                    |
| ------------- | -------------------------------------------------- | ----------------------------------------- |
| ps-housing    | `properties` (`owner_citizenid`)                   | Checked against the official repository   |
| qb-houses     | `player_houses` + `houselocations`                 | Checked against the official repository   |
| qs-housing    | `player_houses` + `houselocations`                 | Assumed to match qb-houses, not confirmed |
| Anything else | `Functions.HousingList` / `Functions.HousingSeize` | Your adapter                              |

### Compatibility folders

Each folder inside `compat/` is a tiny resource with the **same name as the original bank**. Remove the original, copy the folder and `ensure` it after `nexus_banking`.

| Folder             | Exports / events provided                                                                                                                                                                                      |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Renewed-Banking`  | `getAccountMoney`, `addAccountMoney`, `removeAccountMoney`, `handleTransaction`, `getAccountTransactions`, `GetJobAccount`, `CreateJobAccount`, `addAccountMember`, `removeAccountMember`, `changeAccountName` |
| `qb-banking`       | `AddMoney`, `AddGangMoney`, `RemoveMoney`, `RemoveGangMoney`, `GetAccount`, `GetGangAccount`, `GetAccountBalance`, `CreateJobAccount`, `CreateGangAccount`, `CreatePlayerAccount`, `CreateBankStatement`       |
| `qb-management`    | `AddMoney`, `RemoveMoney`, `GetAccount`, `AddGangMoney`, `RemoveGangMoney`, `GetGangAccount`                                                                                                                   |
| `esx_addonaccount` | `getSharedAccount`, `getAccount`, `GetSharedAccount`, `GetAccount`                                                                                                                                             |
| `okokBanking`      | `GetAccount`, `AddMoney`, `RemoveMoney`, `AddTransaction`, `GetPlayerTransactions`                                                                                                                             |
| `ps-banking`       | `createBill`                                                                                                                                                                                                   |

`CreateJobAccount` and `CreateGangAccount` only give the starting balance when the account didn't exist yet, so calling them on every start never creates money.

### Migration

`/bankaudit migrate <source>` reads `addon_account_data`, `bank_accounts`, `management_funds`, `bank_accounts_new` or `okokbanking_societies`. Accounts already migrated are skipped, so running `apply` twice never duplicates money. Back up your database before using `apply`.

***

## 🛡️ Security

1. **One amount validator** for network events and exports.
2. **Authorization on every action**, recalculated on the server. Account ids sent by the interface are never trusted.
3. **Atomic SQL** with a `CHECK (balance >= 0)` constraint and automatic refund if the credit side fails.
4. **Idempotency**: every confirmed action and every loan, payment and payout carries a unique key in `nexus_bank_ops`.
5. **Locks and rate limit**: per-account mutex in a fixed order, one money action at a time per player, 5 requests every 3 seconds.
6. **The client never writes history, balances, quotes or loan terms.** Every quote is recalculated on the server when the player accepts.
7. **PINs are hashed** with SHA-256, a per-card salt and a server pepper.
8. **Distance**: sessions only open near a branch or ATM.
9. **Offline money safety**: money is not taken from or given to a player's offline balance in the 45 seconds after they join or leave. This avoids a race with the framework's own save, and scheduled charges simply retry.
10. **Lending safety**:
    * collateral valued conservatively with caps;
    * unknown models refused;
    * seizure only clears the collateral's value;
    * a daily lending cap for the whole server;
    * dealers paid per instalment by default.
11. **Admin actions** need the ACE and a written reason; everything goes to `nexus_bank_audit`. CSV exports neutralise spreadsheet formulas.
12. **`/bankaudit selftest`** checks on your own server:
    * amounts, overdraw, double spend, idempotency, frozen accounts and permissions;
    * the loan maths for every method, tier limits, LTV and depreciation;
    * that the bank and vehicleshop quotes match.

***

## 💻 Developer API

### Account ids

| Prefix | Type                         | Example          |
| ------ | ---------------------------- | ---------------- |
| `p:`   | Personal (framework balance) | `p:ABC12345`     |
| `s:`   | Business (job)               | `s:police`       |
| `g:`   | Gang                         | `g:ballas`       |
| `x:`   | Shared                       | `x:7KQ2M9PZ`     |
| `v:`   | Savings jar                  | `v:3HDT8W2N`     |
| `e:`   | Addon / system               | `e:bank_reserve` |

Most exports also accept a job name, a gang name, `society_<job>` or an IBAN.

### Server exports: accounts and payments

| Export                                                                            | Parameters                                | Returns                                                                                                                                                |
| --------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `IsReady`                                                                         | —                                         | `boolean`                                                                                                                                              |
| `GetAccount` / `GetAccountByIban`                                                 | `id` / `iban`                             | `{ id, type, label, owner, balance, frozen, iban }` or `nil`                                                                                           |
| `GetBalance` / `GetIban`                                                          | `id`                                      | `number` / `string`                                                                                                                                    |
| `AddMoney` / `RemoveMoney`                                                        | `id, amount, title, opts?`                | `ok, txId or error`                                                                                                                                    |
| `Transfer`                                                                        | `fromId, toId, amount, title, opts?`      | `ok, txId or error`                                                                                                                                    |
| `AddHistory`                                                                      | `id, signedAmount, title, opts?`          | `ok, id`                                                                                                                                               |
| `GetSocietyAccount` / `GetGangAccount` / `GetPersonalAccount`                     | `name` / `identifier`                     | account id                                                                                                                                             |
| `ResolveAccount`                                                                  | `name, create?`                           | account id                                                                                                                                             |
| `CreateSharedAccount`, `AddMember`, `RemoveMember`, `GetMembers`, `RenameAccount` | ...                                       |                                                                                                                                                        |
| `FreezeAccount` / `UnfreezeAccount` / `IsFrozen`                                  | `id, reason?, actor?`                     | `boolean`                                                                                                                                              |
| `GetTransactions`                                                                 | `id, page?, perPage?, filters?`           | `rows, total`                                                                                                                                          |
| `GetPlayerAccounts`                                                               | `source`                                  | accounts with permissions                                                                                                                              |
| `GetPaymentAccounts`                                                              | `source`                                  | accounts the player can pay from: `{ id, type, label, balance, iban }`                                                                                 |
| `Charge`                                                                          | `source, accountId, amount, title, opts?` | `ok, txId or error`. `accountId` is `nil` (personal), `'cash'` or an account id the player can withdraw from. `opts.to` sends the money to an account. |
| `Refund`                                                                          | `source, accountId, amount, title, opts?` | `ok, txId or error`                                                                                                                                    |
| `CanAccess`                                                                       | `source, id, action`                      | `boolean`                                                                                                                                              |
| `CreateBill`, `GetBills`, `PayBill`, `CancelBill`                                 | ...                                       |                                                                                                                                                        |
| `ValidateAmount`                                                                  | `value, max?`                             | `integer` or `nil, error`                                                                                                                              |

### Server exports: credit, loans and collateral

| Export                                                             | Parameters                        | Returns                                                                                                        |
| ------------------------------------------------------------------ | --------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `GetCreditScore`                                                   | `source` or `identifier`          | `number` (same as 1.0)                                                                                         |
| `GetCreditScoreValue`                                              | `source` or `identifier`          | `number` or `nil`                                                                                              |
| `GetCreditProfile`                                                 | `source` or `identifier`          | `{ identifier, score, tier, factors }`                                                                         |
| `AddCreditEvent`                                                   | `who, kind, ref?, weight?`        | `boolean`. `kind` up to 20 characters, `weight` from -10 to 10, `ref` makes it count once.                     |
| `GetLoans`                                                         | `source` or `identifier`          | active, pending and defaulted loans                                                                            |
| `GetLoan`                                                          | `loanId`                          | loan or `nil`                                                                                                  |
| `GetLoanByRef`                                                     | `origin, ref`                     | loan or `nil` (for example `'vehicleshop', plate`)                                                             |
| `PayLoan`                                                          | `source, loanId, mode, opts?`     | `ok, amount or error`. `mode`: `next`, `partial` (with `opts.amount`) or `all`. `opts.accountId`, `opts.uuid`. |
| `HasLien`                                                          | `assetType, assetId`              | `false` or `true, { loanId, identifier, status, value }`                                                       |
| `GetLiens`                                                         | `identifier`                      | list of active pledges                                                                                         |
| `GetCollateralValue`                                               | `assetType, assetId, identifier?` | `number`                                                                                                       |
| `GetVehicleFinanceTerms`                                           | `source` or `identifier`          | `{ ok, reason, score, tier, interestAdd, minDown, payout, bank }`                                              |
| `GetEffectivePlan`                                                 | `plan, terms`                     | plan with the tier adjustments                                                                                 |
| `ReserveVehicleLoan` / `ActivateVehicleLoan` / `CancelVehicleLoan` | used by nexus\_vehicleshop        |                                                                                                                |

`opts` for money exports:

* `message`;
* `actor`;
* `uuid`: idempotency key, strongly recommended;
* `force`;
* `allowOffline`;
* `fromLabel`, `toLabel`.

Error codes are locale keys such as `insufficient_funds`, `account_frozen`, `invalid_amount`, `duplicate`, `no_permission`, `invalid_account`.

### Client exports

| Export                          | Description                                    |
| ------------------------------- | ---------------------------------------------- |
| `OpenBank(branchId?)`           | Opens the nearest branch, or the given one.    |
| `OpenATM()`                     | Opens an ATM session at the player's position. |
| `OpenBills()` / `OpenInvoice()` | Bills screens.                                 |
| `CloseUI()` / `IsOpen()`        |                                                |

### Events (server, local)

| Event                                                   | Payload                                                             |
| ------------------------------------------------------- | ------------------------------------------------------------------- |
| `nexus_banking:server:ready`                            | —                                                                   |
| `nexus_banking:server:onTransaction`                    | `tx`                                                                |
| `nexus_banking:server:onBillCreated` / `onBillPaid`     | `bill`                                                              |
| `nexus_banking:server:onLoanIssued`                     | `loan`                                                              |
| `nexus_banking:server:onLoanPayment`                    | `loan, amount, share` (`share` = dealer part for vehicleshop loans) |
| `nexus_banking:server:onLoanMissed`                     | `loan, missed`                                                      |
| `nexus_banking:server:onLoanPaid`                       | `loan`                                                              |
| `nexus_banking:server:onLoanDefault`                    | `loan`                                                              |
| `nexus_banking:server:onLoanAtRisk`                     | `loan` (collateral no longer owned)                                 |
| `nexus_banking:server:onLienCreated` / `onLienReleased` | `{ id, type, asset, loan, identifier, reason? }`                    |
| `nexus_banking:server:onCollateralSeized`               | `info`                                                              |
| `nexus_banking:server:onCreditTierChanged`              | `identifier, oldTier, newTier, score`                               |
| `nexus_banking:server:onAccountFrozen`                  | `id, frozen, reason`                                                |
| `nexus_banking:server:onMemberChanged`                  | `id, identifier, role`                                              |

### Editable functions (`functions.lua`)

Server:

* **Access and sessions**: `CanAccess`, `CanOpenBank`, `OnSessionOpened`, `CanFreeze`.
* **Accounts and transactions**: `OnTransaction`, `OnAccountCreated`, `OnAccountDeleted`, `OnAccountFrozen`, `OnMemberChanged`.
* **Bills and cards**: `OnBillCreated`, `OnBillPaid`, `OnBillCancelled`, `OnCardIssued`, `OnCardBlocked`.
* **Recurring money**: `OnStandingOrderRun`, `OnSavingsInterest`.
* **Loans**: `OnLoanIssued`, `OnLoanPayment`, `OnLoanPaid`, `OnLoanDefault`, `OnLoanMissed`, `LoanEligible`.
* **Credit**: `CreditScore`, `CreditFactors`, `CreditIncome`, `OnCreditTierChanged`.
* **Collateral and housing**: `OnLienReleased`, `OnCollateralSeized`, `HousingList`, `HousingSeize`.
* **Framework and inventory**: `GetGang`, `RemoveMoney`, `AddMoney`, `GetMoney`, `HasCardItem`, `GiveCardItem`, `RemoveCardItem`.
* **Admin**: `OnAdminAction`.

Client: `Notify`, `ShowTextUI`, `HideTextUI`, `AddTarget`, `RemoveTarget`, `CanUseBank`, `OnBankOpened`, `OnBankClosed`, `AtmAnimation`.

### Database schema

| Table                                                                   | Content                                                                                                                                                                                                                                  |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus_bank_accounts`                                                   | accounts, balance (`CHECK >= 0`), IBAN, frozen state                                                                                                                                                                                     |
| `nexus_bank_members`                                                    | shared account members and roles                                                                                                                                                                                                         |
| `nexus_bank_transactions` / `_archive`                                  | history                                                                                                                                                                                                                                  |
| `nexus_bank_ops`                                                        | idempotency keys                                                                                                                                                                                                                         |
| `nexus_bank_cards`                                                      | cards and PIN hashes                                                                                                                                                                                                                     |
| `nexus_bank_bills`                                                      | bills                                                                                                                                                                                                                                    |
| `nexus_bank_loans`                                                      | plan and custom loans. Columns: `kind` (`plan`, `custom`, `vehicleshop`), `loan_type`, `method`, `rate_ppm`, `schedule` (JSON), `paid_count`, `fee`, `late_fees`, `grace_until`, `origin`, `external_ref`, `tier`, `score_at`, `at_risk` |
| `nexus_bank_liens`                                                      | pledged assets with status `pending`, `active`, `lost`, `seized`, `released` or `failed`                                                                                                                                                 |
| `nexus_bank_credit`                                                     | cached score per player                                                                                                                                                                                                                  |
| `nexus_bank_credit_events`                                              | score events (unique per player, kind and reference)                                                                                                                                                                                     |
| `nexus_bank_credit_snapshots`                                           | balance snapshots for the balance factor                                                                                                                                                                                                 |
| `nexus_bank_standing_orders`, `nexus_bank_branches`, `nexus_bank_audit` | standing orders, branches and ATMs, audit log                                                                                                                                                                                            |

### Integration examples

```lua
-- charge a monthly membership, safe against double charges
local ok, err = exports.nexus_banking:Transfer('p:' .. citizenid, 'police', 500, 'Gym membership', {
    uuid = ('gym:%s:%s'):format(citizenid, os.date('%Y-%m')),
})

-- reward a player for paying rent on time in your housing script
exports.nexus_banking:AddCreditEvent(src, 'rent_paid', ('rent:%s:%s'):format(houseId, os.date('%Y-%m')), 3)

-- block selling a house that backs a loan
local pledged = exports.nexus_banking:HasLien('property', houseId)
if pledged then return false, 'This house backs a bank loan' end

-- let a player pay with any account they can use
local accounts = exports.nexus_banking:GetPaymentAccounts(src)
local ok, res = exports.nexus_banking:Charge(src, accounts[1].id, 2500, 'Tuning', { uuid = orderId, to = 'mechanic' })
```

***

## ❓ FAQ

**The resource does not start and says the folder must be called `nexus_banking`.** Rename the folder exactly, without version suffixes.

**Where is the player's bank balance?** In the framework, as always.

**My jobs pay into a society but the money does not appear.** Use the matching compatibility folder or call `exports.nexus_banking:AddMoney('police', amount, 'title')`.

**A boss cannot see the business account.** On ESX the grade must be named `boss`; on QBCore/Qbox it must have `isboss = true`. You can also use grade numbers in `Config.Society.jobs`.

**The ATM says I am too far.** Stand in front of it. If you use custom ATM props, add their models to `Config.Atm.models`.

**The card animation is too slow / I don't want it.** Raise `Config.Atm.cardAnimation.speed` or set `enabled = false`. Players can always skip it with a click.

**Loans say the bank cannot grant loans right now.** The reserve account has no money, or the daily server cap was reached. Search `bank_reserve` in `/bankadmin` → Accounts and add funds.

**A player says their score doesn't go up after paying.** Payments only count in the second half of the period, positive events are capped per day, and very small loans don't count. The Credit tab shows the caps.

**Can a player pledge a car worth nothing to borrow a lot?** No. Only the value of the collateral raises the limit, unknown models are refused, and seizing only clears the collateral's value.

**A seized house still shows as owned.** ps-housing and qb-houses cache properties; restart them or handle `onCollateralSeized` yourself.

**nexus\_vehicleshop still uses its own financing.** Start nexus\_banking before the shop, turn on `Config.Modules.loans` and check `/vshop diag`.

**PINs stopped working after a restart.** The `nexus_banking:pepper` convar changed. Restore it.

**Discord logs do not arrive.** Set the `nexus_banking:webhook` convar in `server.cfg`.

### Before opening a ticket

* Folder named exactly `nexus_banking`, latest version.
* `oxmysql` started before the framework, Nexus Banking before nexus\_vehicleshop.
* Only one bank running.
* Output of `/bankaudit selftest`.
* `Config.Debug = true`, full server and F8 console output and steps to reproduce.
