---
name: prepare-tax-year
description: Get a tax year ready to file in Kryptos — check the data is complete and the gains are up to date, summarise capital gains, income and expenses for the year, generate the tax report, and optionally lock the period once filed. Use when the user asks for their tax report, their gains or income for a year, or wants to get ready to file.
---

# Prepare a tax year

Take the user from "is my data ready?" to a generated report for one tax year. Explicit instructions from the user take priority over this workflow. Read every setting from the tools — country, fiscal year, cost-basis method, available reports. Don't assume them.

Errors return a `code` (for example `NO_TAX_YEAR_PLAN`, `PERIOD_LOCKED`) and a `message`; use both to tell why a call failed.

## Workflow

1. **Pin down the year and the rules in force.** Call `get_accounting_settings`. It returns:
   - `countryCode` (ISO-2), `method` (cost basis), `baseCurrency` and `timezone`;
   - `fiscalStart` / `fiscalEnd` as day/month;
   - `advancedTxSettings`, where `cost_basis_per_wallet` says whether cost basis is tracked per wallet or across all wallets;
   - `tagsException`, the workspace's tax treatment per label.

   **`year` is the calendar year in which the tax year starts.** Work out the exact dates from `fiscalStart` in the workspace `timezone` and confirm them with the user. Examples: a UK 2024/25 tax year (6 Apr 2024 – 5 Apr 2025) is `year: 2024`; an Australian FY2025 (1 Jul 2024 – 30 Jun 2025) is `year: 2024`; a calendar-year country's 2024 is `year: 2024`.

2. **Check the data is settled.**
   - `list_sync_jobs`: any sync still `pending`, `queued` or `in_progress` will change the figures — offer to wait. A sync that ended `partially_synced` with `limitReached: true` stopped at the workspace's transaction limit, so the history is incomplete; say so.
   - `get_reconciliation_summary` covers the whole workspace, not just one year. If it shows action items (uncategorised transactions, missing purchase history, missing prices, missing balances), say how many and that they can make the figures wrong, and offer to fix them first (the `review-and-reconcile` skill). If the user wants to carry on, continue and repeat the caveat with the figures.
3. **Make sure gains are current.** Imports and edits trigger a recalculation automatically.
   - `list_tasks(operationType: "workspace-recompute", limit: 1)`: if it is `pending` or `in_progress`, wait for it.
   - Then `get_accounting_status`. Only if `hasOutput` is false, or `lastCalculation` is older than the last import or edit, call `trigger_recompute` (it returns a `taskId`) and poll `get_task` until it finishes.
   - Don't quote figures from a stale calculation.
4. **Summarise the year.**
   - **Capital gains:** `list_tax_reports(year)` gives proceeds, cost basis, gains, the number of disposals, and a summary of gains and losses (short- and long-term where the country splits them). It also has `missingPurchaseCount` — mention it if it is above zero. A year the plan doesn't cover comes back as a stub with `locked: true` and a `lockReason` instead of figures; tell the user plainly.
   - **Income and expenses** aren't in that report. Call `get_accounting_breakdown(from, to)` with the tax year's start and end in epoch ms. It is refused, with the reason, if the range touches a tax year the plan doesn't cover, so query one tax year at a time.
   - Name the cost-basis method the figures use. Figures fetched under different methods aren't comparable; pass `method` explicitly when the user asks to compare.
   - `list_tax_report_years` only lists years that have disposals. A year with only income won't appear there, but still has figures.
5. **Drill down if asked.** `get_tax_report_disposals(year)` lists each sale with the acquisitions it was matched against (pages start at 0). It is only available for tax years the workspace's plan covers. If it's refused, the `code` says why: `NO_TAX_YEAR_PLAN` (the plan doesn't cover that tax year), `ENTERPRISE_PLAN_REQUIRED`, or `PLAN_TRANSACTION_LIMIT_EXCEEDED` (the year has more transactions than the plan allows; `data` has the counts). Tell the user plainly. `get_year_end_lots` is gated the same way.
6. **Generate the report.** `get_report_catalog(year)` lists the reports for the user's country, each with `title`, `reportKey`, `type` (pdf, csv…) and its state:
   - `locked: true` means it can't be generated for that year; `lockReason` says why (for example the plan doesn't cover the year, or the transaction count in `lockData.current` is over `lockData.limit`). Don't call `generate_report` for a locked report. Note that `available: true` alone doesn't mean it can be generated.
   - A non-null `status` or `taskId` means it was already generated or is in progress; check `list_reports(year)` before generating again.
   - `accountingDependent: true` reports need current gains (step 3).
   - If the catalog is empty, no reports are offered for that country and year; say so.

   Let the user pick an unlocked report and call `generate_report(year, reportKey)`; it returns a `taskId`. Poll `get_task` until it completes. It can also fail because there's no data for the year, or because the accounting isn't ready yet (go back to step 3). Then call `get_report_link(year, reportKey)` for the download link. The link expires after about an hour, so fetch it when the user is ready; if it says the report isn't ready, wait and try again.

7. **Lock the period (optional, only once filed).** Offer this only after the user says they have filed, every wallet for the year is imported, and gains are current. Recalculations never go back before a lock, so anything imported later for a locked period is ignored.
   - `get_lock_boundaries` returns the period ends a lock can go on (`periodEnd` in epoch ms, the inclusive end; `periodKind` `quarter` or `year`; `locked`), for the workspace's cost-basis method.
   - A lock blocks edits to every transaction dated **on or before** the latest locked period end, not only the last period.
   - Confirm the boundary with the user, then call `lock_period(periodEnd, periodKind, method)`.

## Boundaries

- Ask before recomputing, generating a report or locking a period.
- Never unlock a period, change tax settings or edit transactions as part of this workflow. If figures look wrong, go back to the review.
- `get_tax_loss_harvesting` is a simulation. If the user asks about it, present it as such, and note that wash-sale or superficial-loss rules may not all be modelled.
- Report what Kryptos calculated. Don't tell the user what they owe or how to file, and don't give personal tax advice.
