---
icon: building-columns
---

# Nexus Banking

> A secure, drop-in banking system for ESX, QBCore and Qbox: personal, shared, business, gang and savings accounts, ATMs and branches created in game, cards with PIN, bills, standing orders and optional loans, all behind a card-first React interface and a transaction engine that validates and executes every operation atomically on the server.

**Version:** 1.0.0 · **Framework(s):** ESX, QBCore, Qbox · **Database:** MySQL / MariaDB (oxmysql)

***

## 📝 Description

Nexus Banking replaces your banking resource with one built around a single rule: **the client only asks, the server decides**. Every deposit, withdrawal, transfer, bill, fee or payout goes through one transaction engine that validates the amount, checks the player's permission on that account, checks the player is physically at a branch or ATM, takes a lock on the accounts involved, moves the money with a conditional SQL update and writes the history. If any step fails, nothing is charged.

The **personal balance stays in your framework** (`xPlayer` bank account on ESX, `PlayerData.money.bank` on QBCore/Qbox). That keeps every other script that calls `removeAccountMoney('bank')` or `RemoveMoney('bank')` working exactly as before; Nexus Banking adds the history, the interface and the rules on top. Shared, business (society), gang, savings and legacy addon accounts live in the resource's own tables.

Branches and ATMs are stored in the database and managed from an in-game admin panel (`/bankadmin`). Eight default branches are created on first start; every ATM prop on the map also works out of the box.

For servers coming from another bank, the resource ships **compatibility folders** that impersonate Renewed-Banking, qb-banking, qb-management, esx\_addonaccount, okokBanking and ps-banking, plus a migrator that imports existing balances.

***

## ✨ Features

* **Atomic, server-validated transactions** — one validator for every amount (whole number, greater than zero, below the limit, never NaN, infinite or text), authorization recalculated on the server, conditional `UPDATE ... WHERE balance >= ?`, per-account locks, one request at a time per player and idempotent requests. See [Security](nexus-banking.md#security).
* **Account types** — personal (framework balance), shared with roles (owner, manager, contributor, viewer), business accounts per job and gang accounts with permissions by grade, savings jars and addon accounts for legacy ESX scripts.
* **Transfers by IBAN** — every account has an IBAN. Recipients are looked up and confirmed before sending, recent contacts are suggested, transfers to offline players are supported and an optional fee can be charged (money sink).
* **Branches and ATMs** — branches with marker, NPC, target or nexus\_TextUI prompts and per-branch colour themes; map ATMs detected by prop model; ATM limits per withdrawal and per day.
* **Cards and PIN** (optional) — virtual cards or inventory items with metadata, tiers with daily limits, salted SHA-256 PIN, blocking after failed attempts, freeze, change PIN, replace and cancel.
* **Bills** — invoice nearby players from any account you are allowed to bill from (`/invoice`), pay from anywhere (`/bills`), due dates, overdue status, optional late fee and optional automatic charge for business and gang bills.
* **Savings jars** — goals, animated jars and optional interest capped by balance, by payout and only for active players. Money deposited during the current period does not earn interest, so parking money just before the payout is useless.
* **Standing orders** — recurring payments (rent, salaries) that re-check the creator's permission on every run and stop after repeated failures.
* **Loans** (optional, off by default) — credit score, plans with minimum score, limits per player and per day, cooldowns, origination and late fees, automatic collection and default handling. Funded from a bank reserve account so no money is created.
* **Statements** — filterable, paginated history and a printable monthly statement view.
* **Admin panel** — server statistics, security selftest, account search, balance adjustments with mandatory reason, freeze, CSV export, full audit log and branch/ATM editor.
* **Police tools** — freeze and unfreeze accounts by IBAN for configured jobs. Freezes made by an admin can only be lifted by an admin.
* **7 languages**, configurable currency and number format, five colour themes, every text editable.

***

## 📋 Dependencies

### Required

| Dependency                                                               | Notes                                                                                                                 |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| **ESX** (`es_extended`), **QBCore** (`qb-core`) or **Qbox** (`qbx_core`) | Auto-detected, or forced with `Config.Framework`.                                                                     |
| **oxmysql**                                                              | The script uses `MySQL.*.await`. mysql-async is not supported.                                                        |
| **MySQL 8 / MariaDB 10.2+**                                              | Eleven `nexus_bank_*` tables are created automatically. The balance `CHECK` constraint is enforced on these versions. |

### Optional

| Dependency                                                      | What it's for                          | Selected with                                            |
| --------------------------------------------------------------- | -------------------------------------- | -------------------------------------------------------- |
| `nexus_notify`                                                  | Notifications (default)                | `Config.Systems.Notify = 'nexus'`                        |
| `nexus_TextUI`                                                  | Prompts at branches and ATMs (default) | `Config.Systems.TextUI = 'nexus'`                        |
| `okokNotify`, `ox_lib`, `mythic_notify`, `ps-ui`, ESX/QB notify | Alternative notifications              | `Config.Systems.Notify`                                  |
| `okokTextUI`, `ox_lib`, `ps-ui`, ESX/QB TextUI                  | Alternative prompts                    | `Config.Systems.TextUI`                                  |
| `ox_target` / `qb-target`                                       | Target mode on branches and ATM models | `Config.Systems.Target`                                  |
| `ox_inventory`, `qb-inventory` (and forks), `qs-inventory`      | Cards as items                         | `Config.Systems.Inventory`, `Config.Cards.mode = 'item'` |

***

## ⚙️ Installation

1. **Download** the resource from the cfx.re portal (Keymaster) and drop it into your `resources` folder.
2. **Keep the folder name exactly `nexus_banking`.** The resource validates its own name and stops otherwise.
3. **Remove or stop your previous bank** (qb-banking, Renewed-Banking, okokBanking, esx\_banking...). Two banks must never run at the same time.
4. **Optional compatibility**: copy the folders you need from `nexus_banking/compat/` (for example `compat/qb-banking`) into your resources. Each one has the same name as the original resource and forwards every call to Nexus Banking. See [Compatibility](nexus-banking.md#compatibility).
5. **server.cfg order**:

```
ensure oxmysql
ensure es_extended        # or qb-core / qbx_core
ensure nexus_notify
ensure nexus_TextUI
ensure nexus_banking
ensure qb-banking         # only if you use the compatibility folder
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

8. **Start the server.** The tables are created on first start (`sql/install.sql` is also included if you prefer to import it by hand) and eight default branches are seeded.
9. Run `/bankaudit selftest` from the server console. Every check must say PASS.
10. **Coming from another bank?** Run `/bankaudit migrate <source>` first (dry run), review the output, then `/bankaudit migrate <source> apply`.

{% hint style="warning" %}
Set the `nexus_banking:pepper` convar before players create cards. Changing it later invalidates every existing PIN.
{% endhint %}

{% hint style="info" %}
The Discord webhook is read **only** from the `nexus_banking:webhook` convar. It is not in `config.lua` on purpose: the config files are shared with clients, so a webhook placed there could be read by any player.
{% endhint %}

***

## 🔧 Configuration

All configuration lives in `config/` (outside the escrow).

### `config/config.lua`

| Option                                   | Default                                      | Description                                                                                                                                                                                                                                                                                   |
| ---------------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Framework`                       | `'auto'`                                     | `auto`, `esx`, `qbcore` or `qbox`.                                                                                                                                                                                                                                                            |
| `Config.Locale`                          | `'en'`                                       | `en`, `es`, `de`, `fr`, `it`, `pt`, `zh`.                                                                                                                                                                                                                                                     |
| `Config.Bank`                            | `Trading Monarchy` / `TM`                    | Fictional bank name, short brand shown on cards, default theme and IBAN prefix.                                                                                                                                                                                                               |
| `Config.Themes`                          | 5 themes                                     | `monarchy`, `obsidian`, `emerald`, `crimson`, `ocean`. Each has `accent`, `accentSoft`, `chip` and a 3 colour `card` gradient. Branches can use different themes.                                                                                                                             |
| `Config.Systems`                         | nexus / nexus / none / auto                  | Notify, TextUI, Target and Inventory bridges.                                                                                                                                                                                                                                                 |
| `Config.Commands`                        | `bankadmin`, `bankaudit`, `bills`, `invoice` | Command names.                                                                                                                                                                                                                                                                                |
| `Config.AdminAce`                        | `nexus.banking.admin`                        | ACE for the admin panel, selftest and migrator.                                                                                                                                                                                                                                               |
| `Config.Currency`, `Config.NumberLocale` | `$`, `en-US`                                 | Currency symbol and number format in the interface.                                                                                                                                                                                                                                           |
| `Config.Modules`                         | see below                                    | Turns each feature on or off: `shared`, `society`, `gang`, `cards` (off), `bills`, `savings`, `standingOrders`, `loans` (off), `police`.                                                                                                                                                      |
| `Config.Limits`                          |                                              | `maxTransaction` (10,000,000), transfer fee percent/min/max, offline transfers, max shared accounts owned, max members, message length.                                                                                                                                                       |
| `Config.Security`                        |                                              | Rate limit (5 requests every 3 s), branch and ATM distances, maximum movement while the interface is open, session length, `strictAtms`, abuse warning threshold.                                                                                                                             |
| `Config.Society` / `Config.Gangs`        |                                              | Auto creation, ignored jobs, and who can `view`, `deposit`, `withdraw`, `transfer`, send `bills` and create standing `orders`: `'boss'`, `'employee'` or a minimum grade number, with per job overrides. If `orders` is not set it follows the `transfer` rule (including per job overrides). |
| `Config.History`                         | 25 / 60 days                                 | Page size, retention days before archiving to `nexus_bank_transactions_archive`, statement days.                                                                                                                                                                                              |
| `Config.Interaction`                     | marker                                       | Default branch interaction (`marker`, `npc`, `target`), NPC model and scenario, marker style.                                                                                                                                                                                                 |
| `Config.Blips`                           |                                              | Branch and ATM blips.                                                                                                                                                                                                                                                                         |
| `Config.Bills`                           |                                              | Maximum amount, pending limit per player, default due hours, overdue fee, automatic charge (business and gang bills only), issuer distance, `payAnywhere` (allow paying from `/bills` away from a branch).                                                                                    |
| `Config.StandingOrders`                  |                                              | Minimum interval, max per account, failures before stopping.                                                                                                                                                                                                                                  |
| `Config.Police`                          | police 2, sheriff 2                          | Jobs and minimum grade allowed to freeze accounts; whether a frozen account can still receive money.                                                                                                                                                                                          |
| `Config.Logs`                            |                                              | Console logs and Discord embeds (`minAmount` for large transactions). The webhook URL comes from the `nexus_banking:webhook` convar.                                                                                                                                                          |
| `Config.Interface`                       |                                              | Scale, sounds, reduced motion, quick amounts.                                                                                                                                                                                                                                                 |

### `config/atms.lua`

`Config.Atm` sets the ATM prop models, use distance, `maxWithdraw` per operation, `dailyWithdraw` per player, whether ATMs accept deposits or transfers and the ATM animation. `Config.DefaultBranches` is only used the first time the resource starts; after that, branches are managed with `/bankadmin`.

### `config/cards.lua`

`mode` (`virtual` or `item`), item name, whether a card is required at ATMs and branches, PIN length, maximum attempts, cards per player, issue and replacement fees and tiers (`standard`, `gold`, `black`) with their daily limit and price. Enable the module with `Config.Modules.cards = true`. At an ATM, a card only gives access to the account it was issued for.

### `config/savings.lua`

Maximum jars, interest percent per payout, payout interval, the part of the balance that earns interest, maximum interest per payout, days of activity required and withdrawal cooldown. Money that entered the jar during the last interval is excluded from the interest base.

### `config/loans.lua`

See [Loans and the economy](nexus-banking.md#loans-and-the-economy).

### `config/compat.lua`

Enables each compatibility folder, sets the society prefix (`society_`) and `psBankingAccount` (default `e:bank_reserve`), the account that receives payments of `ps-banking` bills created without an issuer.

***

## 🖥️ Commands

| Command                               | Who                          | Description                                                                                                                                   |
| ------------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `/bankadmin`                          | ACE                          | Opens the admin panel: overview, selftest, accounts, branches and ATMs, audit.                                                                |
| `/bankaudit selftest`                 | ACE / console                | Runs the security selftest against temporary accounts and prints PASS/FAIL.                                                                   |
| `/bankaudit migrate <source> [apply]` | ACE / console                | Imports balances from `esx_addonaccount`, `qb-banking`, `qb-management`, `Renewed-Banking` or `okokBanking`. Without `apply` it is a dry run. |
| `/bankaudit archive`                  | ACE / console                | Archives old transactions now.                                                                                                                |
| `/bills`                              | Everyone                     | Opens your bills from anywhere.                                                                                                               |
| `/invoice`                            | Players with bill permission | Sends a bill to a nearby player.                                                                                                              |
| `/bankfreeze <IBAN> <reason>`         | Police / ACE                 | Freezes an account.                                                                                                                           |
| `/bankunfreeze <IBAN>`                | Police / ACE                 | Unfreezes an account (admin freezes need ACE).                                                                                                |

***

## 🏦 Branches and ATMs in game

Open `/bankadmin` → **Branches**:

* **New point** → choose **Branch** or **ATM**, name it and set its position with **My position**, **Capture** (hides the panel, walk to the spot, `E` to save, `Esc` to cancel) or, for ATMs, **Nearest ATM** (snaps to the closest ATM prop).
* Branches can override the interaction mode (marker, NPC, target), the NPC model and the colour theme, and can hide their blip. Editing one field keeps the others as they were.
* Every point can be edited, deleted (with confirmation), teleported to or marked on the map.
* Changes are broadcast to every player instantly.

Map ATMs work without placing anything. With `Config.Security.strictAtms = true` only ATMs placed from the panel are accepted, which removes any chance of opening an ATM session away from a real ATM.

{% hint style="warning" %}
With `strictAtms = false` (default) the server cannot know where every ATM prop of the map is, so a modified client could open an ATM session anywhere. The damage is limited to the player's own accounts and the ATM limits, but serious servers should place their ATMs with the panel and turn `strictAtms` on.
{% endhint %}

***

## 🌐 Locales & Editable Strings

* Languages: English (default), Spanish, German, French, Italian, Portuguese and Chinese in `locales/*.lua`.
* Server and notification texts are top-level keys and use `%s` placeholders. Error codes returned by the server are locale keys.
* Interface texts are in the `ui` table of each locale and use `{name}` placeholders. They are sent to the interface every time it opens, so editing a locale file only needs a resource restart.
* To add a language, copy `locales/en.lua`, change `Locales['en']` to your code and set `Config.Locale`.
* The interface (`html/`) is outside the escrow. The React source is in `web/` (`npm install`, then `npm run build`).

***

## 🔗 Compatibility

### Frameworks

ESX Legacy, QBCore and Qbox. Gangs come from `PlayerData.gang` on QBCore/Qbox; on ESX fill in `Functions.GetGang` in `functions.lua`.

### Compatibility folders

Each folder inside `compat/` is a tiny resource with the **same name as the original bank**. Remove the original, copy the folder and `ensure` it after `nexus_banking`. Scripts that call the original exports keep working and the money ends up in Nexus Banking.

| Folder             | Exports / events provided                                                                                                                                                                                                           |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Renewed-Banking`  | `getAccountMoney`, `addAccountMoney`, `removeAccountMoney`, `handleTransaction`, `getAccountTransactions`, `GetJobAccount`, `CreateJobAccount` (job table or name), `addAccountMember`, `removeAccountMember`, `changeAccountName`  |
| `qb-banking`       | `AddMoney`, `AddGangMoney`, `RemoveMoney`, `RemoveGangMoney`, `GetAccount`, `GetGangAccount`, `GetAccountBalance`, `CreateJobAccount`, `CreateGangAccount`, `CreatePlayerAccount`, `CreateBankStatement`                            |
| `qb-management`    | `AddMoney`, `RemoveMoney`, `GetAccount`, `AddGangMoney`, `RemoveGangMoney`, `GetGangAccount` (legacy qb-management; current QBCore uses qb-banking for this)                                                                        |
| `esx_addonaccount` | `esx_addonaccount:getSharedAccount`, `esx_addonaccount:getAccount` and `GetSharedAccount` / `GetAccount`, returning objects with `money`, `addMoney`, `removeMoney`, `setMoney`, `save` (used by esx\_society and property scripts) |
| `okokBanking`      | `GetAccount`, `AddMoney`, `RemoveMoney`, `AddTransaction(identifier, data, source)`, `GetPlayerTransactions(identifier, limit)`                                                                                                     |
| `ps-banking`       | `createBill({ identifier, description, type, amount })`. ps-banking bills have no issuer, so they are paid into `Config.Compat.psBankingAccount`.                                                                                   |

Account names are resolved in this order: account id, IBAN, `society_` prefix removed, existing business account, existing gang account, framework job, framework gang, player identifier.

{% hint style="info" %}
Why folders and not `provide`? Several banks export functions with the same name and different signatures (`AddMoney` in qb-banking, qb-management and okokBanking). A single resource cannot expose two different `AddMoney`, so each emulation lives in its own folder. The console warns if an original bank is running at the same time.
{% endhint %}

### Migration

`/bankaudit migrate <source>` reads `addon_account_data` (esx\_addonaccount), `bank_accounts` (qb-banking), `management_funds` (legacy qb-management), `bank_accounts_new` (Renewed-Banking) or `okokbanking_societies` (okokBanking). Shared accounts keep their owner and members. Accounts that already have a balance in Nexus Banking, or that were already migrated once, are skipped, so running `apply` twice never duplicates money. Back up your database before using `apply`.

***

## 🛡️ Security

1. **One amount validator** (`ValidateAmount`) for network events and exports: whole numbers only, greater than zero, below `Config.Limits.maxTransaction`, never NaN, infinite, scientific notation or text.
2. **Authorization on every action**, recalculated on the server from the player's identifier, job grade, gang grade or account role. Account ids sent by the interface are never trusted.
3. **Atomic SQL**: `UPDATE ... SET balance = balance - ? WHERE id = ? AND balance >= ? AND frozen = 0`, checking that exactly one row changed. If the credit side fails, the debit is refunded, and a failed refund is written to the audit. A `CHECK (balance >= 0)` constraint protects the table.
4. **Idempotency**: every confirmed action carries a unique nonce; the server records it in `nexus_bank_ops` and rejects repeats. Scripts can pass their own `uuid`.
5. **Locks and rate limit**: per-account mutex taken in a fixed order (no deadlocks), every money action of a player is processed one at a time (no parallel requests to dodge daily or account limits), 5 requests every 3 seconds per player, abuse counter written to the audit.
6. **The client never writes history or balances.** There is no network event that logs a transaction.
7. **PINs are hashed** with SHA-256, a per-card salt and a server pepper; cards block after the configured attempts.
8. **Distance**: the server opens a session only near a branch or ATM and closes it when the player moves away or after the session time.
9. **Admin actions** need the ACE and a written reason, and everything goes to `nexus_bank_audit` (and Discord if enabled).
10. **Disconnect safety**: money is never moved for a player who is leaving the server, so a disconnect in the middle of an operation cannot duplicate money.
11. **Bills in progress** are recovered automatically: a bill interrupted by a crash is marked paid only if the payment really happened, otherwise it goes back to pending.
12. **`/bankaudit selftest`** proves it on your own server: negative and decimal amounts, overdraw, concurrent double spend, idempotency, frozen accounts, society permissions, standing order permissions, system credits and history consistency.

***

## 💳 Loans and the economy

Loans are **off by default** (`Config.Modules.loans`). They were designed not to inflate the economy:

* **Funding**: `Config.Loans.funding = 'account'` (default) lends money from a real account and repayments go back to it, so no money is created. The default `fundingAccount` is `e:bank_reserve` (" Reserve"), created automatically and invisible to players: no job boss can withdraw it. Fund it from `/bankadmin` → Accounts → adjust balance, or point `fundingAccount` at a business account (`s:<job>`) if a player-run bank should finance the loans. `'mint'` creates the money and repayments destroy it.
* **Sinks**: origination fee (kept by the bank), late fees, optional transfer fees, card fees.
* **Limits**: maximum active loans, maximum total debt per player, cooldown between loans, minimum account age, **daily cap for the whole server**, minimum credit score per plan. Players with a frozen account cannot ask for loans.
* **Collection**: installments are charged automatically, online or offline. Each missed installment adds a late fee; after `maxMissed` the loan defaults, the player cannot ask for new loans and, with `garnish`, the bank keeps collecting what is available. A loan paid off early is never charged again.
* **Credit score** (300-850): base score, on-time payments, missed payments, defaults, paid off loans and account age. `Functions.CreditScore` lets you change it and `GetCreditScore` exposes it to other scripts.

***

## 💻 Developer API

### Account ids

| Prefix | Type                                               | Example                                    |
| ------ | -------------------------------------------------- | ------------------------------------------ |
| `p:`   | Personal (framework balance)                       | `p:ABC12345` / `p:char1:abc...`            |
| `s:`   | Business (job)                                     | `s:police`                                 |
| `g:`   | Gang                                               | `g:ballas`                                 |
| `x:`   | Shared                                             | `x:7KQ2M9PZ`                               |
| `v:`   | Savings jar                                        | `v:3HDT8W2N`                               |
| `e:`   | Addon / system (legacy ESX accounts, bank reserve) | `e:property_black_money`, `e:bank_reserve` |

Most exports also accept a job name, a gang name, `society_<job>` or an IBAN.

### Server exports

| Export                                           | Parameters                                                       | Returns                                                      |
| ------------------------------------------------ | ---------------------------------------------------------------- | ------------------------------------------------------------ |
| `IsReady`                                        | —                                                                | `boolean`                                                    |
| `GetAccount`                                     | `id`                                                             | `{ id, type, label, owner, balance, frozen, iban }` or `nil` |
| `GetAccountByIban`                               | `iban`                                                           | same as above                                                |
| `GetBalance`                                     | `id`                                                             | `number`                                                     |
| `GetIban`                                        | `id`                                                             | `string`                                                     |
| `AddMoney`                                       | `id, amount, title, opts?`                                       | `ok, txId or error`                                          |
| `RemoveMoney`                                    | `id, amount, title, opts?`                                       | `ok, txId or error`                                          |
| `Transfer`                                       | `fromId, toId, amount, title, opts?`                             | `ok, txId or error`                                          |
| `AddHistory`                                     | `id, signedAmount, title, opts?`                                 | `ok, id` (history only)                                      |
| `GetSocietyAccount` / `GetGangAccount`           | `name`                                                           | account id (created if needed)                               |
| `GetPersonalAccount`                             | `identifier`                                                     | account id                                                   |
| `ResolveAccount`                                 | `name, create?`                                                  | account id                                                   |
| `CreateSharedAccount`                            | `ownerIdentifier, label`                                         | account id                                                   |
| `AddMember` / `RemoveMember` / `GetMembers`      | `id, identifier, role?`                                          | `boolean` / list                                             |
| `RenameAccount`                                  | `id, label`                                                      | `boolean`                                                    |
| `FreezeAccount` / `UnfreezeAccount` / `IsFrozen` | `id, reason?, actor?`                                            | `boolean`                                                    |
| `GetTransactions`                                | `id, page?, perPage?, filters?`                                  | `rows, total`                                                |
| `GetPlayerAccounts`                              | `source`                                                         | accounts the player can see, with permissions                |
| `CanAccess`                                      | `source, id, action`                                             | `boolean`                                                    |
| `CreateBill`                                     | `issuerAccount, targetIdentifier, amount, reason, dueAt?, opts?` | `billId` or `nil, error`                                     |
| `GetBills` / `PayBill` / `CancelBill`            | ...                                                              |                                                              |
| `GetCreditScore`                                 | `identifier`                                                     | `number`                                                     |
| `ValidateAmount`                                 | `value, max?`                                                    | `integer` or `nil, error`                                    |

`opts`: `message`, `actor`, `uuid` (idempotency key, strongly recommended for payments), `force` (ignore frozen), `allowOffline` (default `true` for exports), `fromLabel`, `toLabel`. Error codes are locale keys such as `insufficient_funds`, `account_frozen`, `invalid_amount`, `duplicate`.

### Client exports

| Export                          | Description                                                                        |
| ------------------------------- | ---------------------------------------------------------------------------------- |
| `OpenBank(branchId?)`           | Opens the nearest branch, or the given one (the server still checks the distance). |
| `OpenATM()`                     | Opens an ATM session at the player's position (rejected with `strictAtms`).        |
| `OpenBills()` / `OpenInvoice()` | Bills screens.                                                                     |
| `CloseUI()` / `IsOpen()`        |                                                                                    |

### Events (server, local)

| Event                                                 | Payload                                                                                                          |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `nexus_banking:server:ready`                          | —                                                                                                                |
| `nexus_banking:server:onTransaction`                  | `tx` = `{ id, type, amount, fee, from, to, fromLabel, toLabel, title, message, actor, channel, resource, time }` |
| `nexus_banking:server:onBillCreated` / `onBillPaid`   | `bill`                                                                                                           |
| `nexus_banking:server:onLoanIssued` / `onLoanDefault` | `loan`                                                                                                           |
| `nexus_banking:server:onAccountFrozen`                | `id, frozen, reason`                                                                                             |
| `nexus_banking:server:onMemberChanged`                | `id, identifier, role`                                                                                           |

### Editable functions (`functions.lua`)

Server: `CanAccess`, `CanOpenBank`, `OnSessionOpened`, `OnTransaction`, `OnAccountCreated`, `OnAccountDeleted`, `OnAccountFrozen`, `OnMemberChanged`, `OnBillCreated`, `OnBillPaid`, `OnBillCancelled`, `OnCardIssued`, `OnCardBlocked`, `OnStandingOrderRun`, `OnSavingsInterest`, `OnLoanIssued`, `OnLoanPayment`, `OnLoanPaid`, `OnLoanDefault`, `OnAdminAction`, `LoanEligible`, `CreditScore`, `CanFreeze`, `GetGang`, `RemoveMoney` / `AddMoney` / `GetMoney` (framework overrides, for example cash as item), `HasCardItem` / `GiveCardItem` / `RemoveCardItem` (custom inventory).

Client: `Notify`, `ShowTextUI`, `HideTextUI`, `AddTarget`, `RemoveTarget`, `CanUseBank`, `OnBankOpened`, `OnBankClosed`, `AtmAnimation`.

### Database schema

| Table                                  | Content                                                                                                  |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `nexus_bank_accounts`                  | id, type, label, owner, balance (`CHECK >= 0`), frozen, reason, IBAN, meta, interest schedule, last seen |
| `nexus_bank_members`                   | shared account members and roles                                                                         |
| `nexus_bank_transactions` / `_archive` | history with uuid, counterpart, amount, balance after, title, message, actor, channel, resource          |
| `nexus_bank_ops`                       | idempotency keys (kept 3 days; migration keys are kept forever)                                          |
| `nexus_bank_cards`                     | cards, salted PIN hash, tier, limit, attempts, status                                                    |
| `nexus_bank_bills`                     | bills                                                                                                    |
| `nexus_bank_loans`                     | loans                                                                                                    |
| `nexus_bank_standing_orders`           | recurring payments                                                                                       |
| `nexus_bank_branches`                  | branches and placed ATMs                                                                                 |
| `nexus_bank_audit`                     | audit log                                                                                                |

### Integration example

```lua
-- charge a monthly membership from a player's bank, safe against double charges
local ok, err = exports.nexus_banking:Transfer('p:' .. citizenid, 'police', 500, 'Gym membership', {
    uuid = ('gym:%s:%s'):format(citizenid, os.date('%Y-%m')),
})
if not ok then print('payment failed: ' .. err) end

-- react to every transaction
AddEventHandler('nexus_banking:server:onTransaction', function(tx)
    if tx.amount >= 100000 then print(('large %s of %d by %s'):format(tx.type, tx.amount, tx.actor)) end
end)
```

***

## ❓ FAQ

**The resource does not start and says the folder must be called `nexus_banking`.** Rename the folder exactly, without version suffixes.

**Where is the player's bank balance?** In the framework, as always. Nexus Banking reads and writes it through the framework, so other scripts keep working.

**My jobs or vehicleshop pay into a society but the money does not appear.** Make sure the script calls a bank that exists: either use the matching compatibility folder (for example `compat/qb-banking`) or call `exports.nexus_banking:AddMoney('police', amount, 'title')`.

**A boss cannot see the business account.** On ESX the grade must be named `boss`; on QBCore/Qbox the grade must have `isboss = true`. You can also use grade numbers in `Config.Society.jobs`.

**Employees can only deposit.** That is the default. Change `Config.Society.access` or add a per job override.

**The ATM says I am too far.** The server checks the distance to the ATM. Stand in front of it; if you use custom ATM props, add their models to `Config.Atm.models`.

**I used a card at the ATM and cannot pick another account.** That is intended: a card only opens the account it belongs to. Use a branch to manage other accounts.

**My session expires while using the bank.** You moved more than `Config.Security.maxMoveFromOpen` metres or the session lasted longer than `sessionMinutes`. Open it again.

**I changed `Config.DefaultBranches` and nothing happened.** That list is only used on the first start. Use `/bankadmin` → Branches.

**PINs stopped working after a restart.** The `nexus_banking:pepper` convar changed. Restore it.

**Loans say the bank cannot grant loans right now.** The reserve account has no money, or the daily server cap was reached. Search `bank_reserve` in `/bankadmin` → Accounts and add funds.

**Discord logs do not arrive.** Set the `nexus_banking:webhook` convar in `server.cfg`; the webhook is not read from `config.lua`.

**Can I run the original qb-banking and the compatibility folder together?** No. Only one resource can have that name.

### Before opening a ticket

* Folder named exactly `nexus_banking`, latest version.
* `oxmysql` started before the framework and Nexus Banking.
* Only one bank running.
* Output of `/bankaudit selftest`.
* `Config.Debug = true`, full server and F8 console output and steps to reproduce.
