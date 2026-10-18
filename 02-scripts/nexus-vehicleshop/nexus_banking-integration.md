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

1. The shop asks the bank for the buyer's terms: credit tier, extra interest, minimum down payment, the monthly instalments the bank still lets them take on, and the room left in the bank's daily lending limit.
2. The plan shown to the buyer is the dealership plan adjusted for that tier. The quote and the bank loan use the same formula, so the numbers always match.
3. The bank checks affordability and reserves the loan. The shop takes the down payment and delivers the car.
4. Only after a successful delivery is the loan activated. If delivery fails, the loan is cancelled and the buyer gets the down payment back.
5. From then on the **bank is the source of truth**: it charges every instalment (online or offline), applies late fees, updates the credit score and handles default.
6. The shop's finance row keeps a link to the loan (`bank_loan_id`) and follows it: instalments paid, loan paid off, car released or repossessed.

If the bank refuses, the purchase is refused with the exact reason (see below). The shop does **not** fall back to its own financing.

### What the buyer sees before paying

* A note with the monthly amount the bank lets them take on. If the chosen plan costs more per month, it turns into a warning that suggests a bigger down payment or a longer plan.
* A **How financing works** button that opens a short modal: what the bank lends, how the credit score sets rate and down payment, the income check, the vehicle as collateral and repossession.

### Refusal messages

| Cause                               | Message                                                                 |
| ----------------------------------- | ----------------------------------------------------------------------- |
| Credit score in a blocked tier      | Your credit score (value) is too low for financing.                     |
| A loan in default                   | You have a loan in default.                                             |
| Instalments too high for the income | The bank only lets you take on the amount shown more per month.         |
| Down payment below the minimum      | The minimum down payment for your credit score is the amount shown.     |
| Bank daily lending limit reached    | The bank has reached its daily lending limit, try again tomorrow.       |
| Bank reserve without funds          | The bank does not have enough funds to lend right now.                  |
| Vehicle already pledged             | This vehicle is pledged as collateral for a bank loan.                  |
| Terms changed during checkout       | Your financing terms changed, check the new offer.                      |
| Bank not running                    | The bank is not available right now.                                    |
| Anything else                       | The bank could not process the financing, with an error code to report. |

### Dealer payout

| `dealerPayout` (in nexus\_banking) | Dealership receives                                                                                 |
| ---------------------------------- | --------------------------------------------------------------------------------------------------- |
| `installments` (default)           | The down payment at the sale, then the principal part of each instalment when the bank collects it. |
| `upfront`                          | The down payment plus the full financed amount at the sale.                                         |

Staff can't finance a car through the bank at their own dealership.

### Credit history

Opening a dealership financing and paying it off both appear in the buyer's credit history in the bank. Events the anti-farming rules don't count are shown as **No impact**.

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

The bank's `Config.Lending.dailyServerCap` also applies to dealership financing.

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

New keys:

* the `bank_*` errors and labels, including `bank_daily_cap`, `bank_no_liquidity` and `bank_finance_failed`;
* `credit_tier_*`;
* `invalid_account`, `vehicle_pledged`, `bank_self_finance`, `bank_affordability`, `down_too_low`;
* the `ui` keys `pay_from`, `fin_guide*`, `bank_capacity` and `bank_capacity_warn`.

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

Useful nexus\_banking exports: `GetVehicleFinanceTerms` (now also returns `capacity` and `room`), `GetEffectivePlan`, `GetLoanByRef('vehicleshop', plate)`, `PayLoan`, `GetPaymentAccounts`, `Charge`, `Refund`, `HasLien('vehicle', plate)`. `ReserveVehicleLoan` returns `nil, errorKey, extra` on failure. See the [Nexus Banking](../nexus-banking/) page and [Help, guide and multiple collateral](../nexus-banking/help-guide-and-multiple-collateral.md).

***

## ❓ FAQ

**The shop still uses its own financing.** Check that nexus\_banking starts first, `Config.Modules.loans` is on in the bank, and `Config.BankIntegration.finance` is `true`. `/vshop diag` tells you which step fails.

**A buyer is refused with "bank" errors.** The message names the cause: blocked credit tier, instalments too high for their income, a defaulted loan, a down payment below the minimum, or the bank's daily limit or reserve. See the table above.

**The dealership only got the down payment.** That's `installments` mode: the rest arrives with every instalment. Switch to `upfront` in nexus\_banking if you prefer, after funding the reserve.

**Can a buyer pay with the company account?** Only if their grade is allowed to withdraw from it in nexus\_banking.

**The "Pay from" selector doesn't appear.** nexus\_banking isn't running, or `Config.BankIntegration.accountPicker` is `false`.
