# nexus\_banking Integration

> With nexus\_banking 1.1.0 running, buyers pay from any bank account they can use, and financing becomes a real bank loan whose rate and down payment depend on the buyer's credit score. The dealership is paid as the instalments come in.

**Requires:** nexus\_vehicleshop 1.1.0+ and nexus\_banking 1.1.0+ · Optional: everything keeps working without nexus\_banking.

***

## 📝 How it works

### Paying from the bank

At checkout and in **My Financing**, the buyer picks the account to pay from:

* their personal account;
* a shared account where they are allowed to withdraw;
* a business or gang account where their grade is allowed to withdraw.

The list and every permission are checked by nexus\_banking on the server. Accounts the player can't use never appear, and a forged account id is rejected.

### Financing through the bank

1. The shop asks the bank for the buyer's terms: credit tier, extra interest and minimum down payment.
2. The plan shown to the buyer is the dealership plan adjusted for that tier. The quote and the bank loan use the same formula, so the numbers always match.
3. The bank checks affordability and reserves the loan. The shop takes the down payment and delivers the car.
4. Only after a successful delivery is the loan activated. If delivery fails, the loan is cancelled and the buyer gets the down payment back.
5. From then on the **bank is the source of truth**: it charges every instalment (online or offline), applies late fees, updates the credit score and handles default.
6. The shop's finance row keeps a link to the loan (`bank_loan_id`) and follows it: instalments paid, loan paid off, car released or repossessed.

If the bank refuses (blocked tier, not affordable, previous default), the purchase is refused with the reason. The shop does **not** fall back to its own financing.

### Dealer payout

| `dealerPayout` (in nexus\_banking) | Dealership receives                                                                                 |
| ---------------------------------- | --------------------------------------------------------------------------------------------------- |
| `installments` (default)           | The down payment at the sale, then the principal part of each instalment when the bank collects it. |
| `upfront`                          | The down payment plus the full financed amount at the sale.                                         |

Staff can't finance a car through the bank at their own dealership.

### Missed payments and repossession

The bank retries a failed instalment, then counts it as missed and adds a late fee. After the configured number of missed instalments, the loan defaults and the bank calls `Repossess(plate, reason, 'custody')`. The repossessed car can be redeemed at the dealership with the shop's usual rules.

A car that backs a loan can't be sold back to the dealership. Players also can't pledge a car that is still financed at the shop as collateral for a bank loan.

***

## 🔧 Configuration

### nexus\_vehicleshop: `config/config.lua`

| Option                                 | Default  | Description                                                                                                                                   |
| -------------------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.Systems.Banking`               | `'auto'` | `auto` now picks nexus\_banking first when it is running. Use `nexus_banking` to force it, or `default` to keep the framework society system. |
| `Config.BankIntegration.payments`      | `true`   | Purchases and down payments go through the bank.                                                                                              |
| `Config.BankIntegration.accountPicker` | `true`   | Shows the "Pay from" selector with shared and business accounts.                                                                              |
| `Config.BankIntegration.finance`       | `true`   | Financing becomes a real bank loan. Set to `false` to keep the shop's own financing.                                                          |

### nexus\_banking: `config/lending.lua` → `Config.Lending.vehicleShop`

| Option                                     | Default                                                 | Description                                                                              |
| ------------------------------------------ | ------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `enabled`                                  | `true`                                                  | Bank financing for the shop. Needs `Config.Modules.loans = true` in nexus\_banking.      |
| `dealerPayout`                             | `'installments'`                                        | `installments` or `upfront`.                                                             |
| `checkAffordability` / `noIncomeAllowance` | `true` / 60000                                          | Instalments must fit the buyer's income; without income data, up to this monthly amount. |
| `repossessMode`                            | `'custody'`                                             | Mode passed to `Repossess`.                                                              |
| `interestByTier`                           | poor +6, fair +3, good 0, very\_good -1.5, excellent -3 | Points added to the plan interest.                                                       |
| `minDownByTier`                            | poor 35, fair 20, others 0                              | Minimum down payment (%) by tier. The higher of this and the plan's minimum wins.        |
| `blockedTiers`                             | `{}`                                                    | Tiers that can't finance, for example `{ poor = true }`.                                 |

{% hint style="warning" %}
Updating with `Config.Systems.Banking = 'auto'`: dealership company accounts now live in nexus\_banking. If you had dealership money in esx\_addonaccount or qb-management, move it with `/bankaudit migrate <source>` or set `Config.Systems.Banking` explicitly.
{% endhint %}

***

## ⚙️ Installation

1. Update both resources and start the bank first:

```
ensure nexus_banking
ensure nexus_vehicleshop
```

2. In nexus\_banking, set `Config.Modules.loans = true` and fund the reserve account (`/bankadmin` → Accounts → `bank_reserve`).
3. Restart. The shop adds the `bank_loan_id` column to `nexus_vs_finance` automatically if it doesn't exist.
4. Run `/vshop diag`. These three checks must pass:
   * nexus\_banking integration;
   * bank vehicle finance terms;
   * bank plan matches shop quote.

Existing loans created before the update keep running on the shop's own financing until they finish.

***

## 🌐 Locales & Editable Strings

New keys in all 7 languages:

* the `bank_*` errors and labels;
* `credit_tier_*`;
* `invalid_account`, `vehicle_pledged`, `bank_self_finance`, `bank_affordability`;
* the `ui` key `pay_from`.

The bank integration lives in `bridge/server/banking.lua`, outside the escrow.

***

## 💻 Developer API

| Export (nexus\_vehicleshop) | Parameters             | Returns                                                                                                       |
| --------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------- |
| `Repossess`                 | `plate, reason, mode?` | `ok, applied` or `false, 'finance_not_found'`. `mode`: `custody`, or the shop's configured mode when omitted. |

The shop listens to these nexus\_banking events for loans with `origin = 'vehicleshop'`:

| Event                                | Shop reaction                                                            |
| ------------------------------------ | ------------------------------------------------------------------------ |
| `nexus_banking:server:onLoanPayment` | Pays the dealer share (`installments` mode) and updates the finance row. |
| `nexus_banking:server:onLoanMissed`  | Updates the missed count and calls `Functions.OnFinanceMissed`.          |
| `nexus_banking:server:onLoanPaid`    | Releases the car (respects the redeem rules for repossessed cars).       |

Useful nexus\_banking exports: `GetVehicleFinanceTerms`, `GetEffectivePlan`, `GetLoanByRef('vehicleshop', plate)`, `PayLoan`, `GetPaymentAccounts`, `Charge`, `Refund`, `HasLien('vehicle', plate)`. See the [Nexus Banking](../nexus-banking.md) page.

***

## ❓ FAQ

**The shop still uses its own financing.** Check that nexus\_banking starts first, `Config.Modules.loans` is on in the bank, and `Config.BankIntegration.finance` is `true`. `/vshop diag` tells you which step fails.

**A buyer is refused with "bank" errors.** Their credit tier is blocked, the instalments don't fit their income, or they have a defaulted loan. The message says which.

**The dealership only got the down payment.** That's `installments` mode: the rest arrives with every instalment. Switch to `upfront` in nexus\_banking if you prefer, after funding the reserve.

**Can a buyer pay with the company account?** Only if their grade is allowed to withdraw from it in nexus\_banking.

**The "Pay from" selector doesn't appear.** nexus\_banking isn't running, or `Config.BankIntegration.accountPicker` is `false`.
