# Help, guide and multiple collateral

> What players see in the Loans, Credit and Savings tabs to understand how everything works, how several assets can back one loan, and how savings goals are enforced.

**Applies to:** nexus\_banking 1.1.0 (update of 18 Oct 2026)

***

## ❔ Help tooltips

Every field that needs explaining in **Loans**, **Credit** and **Savings** has a small **?** icon. Hovering it shows one or two lines: what _Fixed installment_, _Simple interest_ and _Grace_ do, what the rate and the opening fee are, how the loan-to-value works, what each credit factor measures, and what each tier means. The texts live in the `ui` table of the locale files (`tip_*`, `credit_ft_*`, `tier_tip_*`, `loan_type_tip_*`), so you can edit them without touching the NUI.

## 📖 "How it works" guide

A **How it works** button in Loans and Credit opens a five-step modal: score, loans, collateral, late payments and dealership financing. It stays highlighted until the player opens it once. The numbers in the text (grace minutes, late fee, maximum missed instalments, number of collaterals, prepayment fee) come from your config, so the guide always matches your server.

## 🛡️ Several assets as collateral

A loan can be backed by more than one asset, for example four cars of 50,000 instead of one of 200,000.

| Option (`config/lending.lua`) | Default | Description                              |
| ----------------------------- | ------- | ---------------------------------------- |
| `maxCollaterals`              | `5`     | Maximum assets pledged in the same loan. |

* The values add up and the limit and loan-to-value are calculated on the total.
* The same asset can't appear twice in one loan, and an asset already pledged anywhere else is refused.
* Vehicle and mortgage loans require collateral. Business loans accept vehicles and properties. Personal loans accept none.
* **Mortgages only accept properties.** Vehicles never appear in the mortgage list; the panel says so.
* If the borrower defaults, **all** the assets pledged in that loan are seized, not only the ones needed to cover the debt.
* The server revalidates every asset, the quote and the total when the loan is accepted, and locks each asset so two loans can't pledge the same one at the same time.

## 📋 Credit history

Events that used to be discarded by the anti-farming rules (very small loans, a payment too early in the period, a loan repaid too soon, the daily limit of positive events) are now recorded with weight 0 and shown as **No impact**, with a tooltip explaining why. Opening a loan, including dealership financing, also appears in the history.

## 🐷 Savings jar goals

* A jar with a goal doesn't accept money above it. The deposit is limited to the room left, and a full jar shows **Goal reached**.
* The cap is enforced in the transaction engine, so no other script or forged request can go over it.
* The goal can't be lowered below what the jar already holds.
* Interest never takes a jar over its goal.
* Admin adjustments from `/bankadmin` are exempt.

## ❌ Why a loan is refused

The messages now name the cause: daily lending limit reached, bank without enough funds, too many assets, loan above the limit, instalments too high for the player's income, and so on. Dealership financing has its own list on the [nexus\_banking Integration](../nexus-vehicleshop/nexus_banking-integration.md) page.

## 🌐 New strings

All new keys are in the 7 languages. Server keys: `loan_daily_cap`, `loan_no_liquidity`, `collateral_too_many`, `savings_goal_full`, `savings_goal_room`, `savings_goal_below`, `savings_goal_exceeded`.
