---
name: crypto-bookkeeping
description: Run crypto bookkeeping for a Kryptos Enterprise workspace — connect Xero, QuickBooks or Softledger, sync the chart of accounts, map transactions to GL accounts with rules, check the generated journals and push them to the ERP. Use when the user mentions a chart of accounts, GL codes, journals, month-end close, Xero, QuickBooks or Softledger.
---

# Crypto bookkeeping (Enterprise)

Map crypto activity to the chart of accounts and send clean journals to the user's accounting system. Explicit instructions from the user take priority over this workflow. Discover everything from the tools: platform, organisation, account codes, rules, task state. Never assume them. Errors return a `code` (for example `NO_TAX_YEAR_PLAN`, `PERIOD_LOCKED`) and a `message`; use both to tell why a call failed.

## Before you start

Call `list_chart_of_accounts`. If it fails with `ENTERPRISE_REQUIRED` (the feature requires an enterprise workspace), the workspace is retail: explain that bookkeeping and ERP sync are Enterprise features, and stop. Two more gates can appear later, and the error message says which one applies:

- an ERP may be disabled for the workspace (`ERP_DISABLED`);
- pushing to an ERP needs a paid Enterprise plan (the trial doesn't include it), and is refused when the period has more transactions than the plan allows (`ENTERPRISE_PLAN_REQUIRED`, `PLAN_TRANSACTION_LIMIT_EXCEEDED`).

State the gate plainly when you hit one.

## How accounts get assigned

Every ledger leg gets a debit and a credit account:

- **Asset rules** (category `ASSET_RULES`) set the asset side. Workspaces start with catch-alls ("Digital Assets" for every token, "Cash Equivalents" for fiat) at the lowest ranks. Within a category the lowest rank that matches wins, so a rule for a specific asset needs a lower rank than the catch-alls; use `reorder_rules` if needed.
- **Custom rules** (`CUSTOM_RULES`) set the other side, for deposits, withdrawals and payments only — for example a staking-reward label to an income account. Trades and transfers use asset rules only.
- **Default rules** map capital gains, capital losses, fees and rounding ("Small Errors") to accounts. They aren't matched against legs; to move them to different accounts, use `update_rule` with a new `coa`.
- **Manual assignments** (`assign_ledger_accounts`, `bulk_assign_ledger_accounts`) are kept. Rule runs and recomputes don't overwrite them.
- Rule changes take effect when `apply_rules` runs, and also on the next workspace recompute (for example after a sync).

## Workflow

1. **Connect the ERP.** Call `list_erps` (shows `isConnected`), then `get_erp_connection_status(platform)`.
   - **Not connected, Xero or QuickBooks:** `connect_erp(platform)` returns an `authUrl` for the user to open and approve; you can't do this for them. Check the status again afterwards.
   - **Softledger** uses client credentials and is connected in the Kryptos web app. Never ask for or accept those in the chat.
   - **Xero:** if `tenantInfo` is empty, no organisation is selected, and syncing, creating accounts and pushing will all fail. Call `list_erp_tenants`, ask which organisation, then `select_erp_tenant`.
   - **QuickBooks** connects to one company; there is nothing to select. **Softledger:** `select_erp_tenant` only sets the fallback location for journals whose accounts carry none.
   - **Wrong organisation:** switching does not reset anything. After selecting the right one, run `sync_coa_from_erp`, and warn that journals already pushed went to the previous organisation.
   - **Broken connection:** the status can still say connected after a token stops refreshing; it shows up as errors or failed tasks. Run `connect_erp` again (and, for Xero with several organisations, select the organisation again).
   - **No ERP:** the chart of accounts can be imported from CSV in the web app (ERP Integrations → Custom), or created one account at a time with `create_chart_of_account` (`code`, `name`, `class` = ASSET, EQUITY, EXPENSE, LIABILITY or REVENUE). Journals then stay in Kryptos.
2. **Chart of accounts.** `sync_coa_from_erp(platform)` pulls the ERP's accounts (for Xero, active accounts that have a code). Accounts deleted in the ERP are deleted in Kryptos too. Legs tagged with them lose that account, including manual tags, and rules pointing at them stop matching, so say this before running it. (If the ERP returns no accounts, or fewer than half of those held, nothing is deleted.) Read the result with `list_chart_of_accounts`.
   - **Only accounts synced from the ERP can be pushed.** A journal that uses a Kryptos-only account — including the starter accounts the default and catch-all rules point to — is rejected by the ERP adapter. After syncing, point those rules at ERP accounts with `update_rule`, then run `apply_rules`.
   - Need a new account in the ERP: `list_erp_account_classes(platform)` for the valid classes, then `create_erp_accounts` with `accounts: [{ code, name, class }]` (up to 500; existing codes are updated). Check the result for per-account `errors`.
   - Softledger account codes are internal ids; the human account number is in the name, for example "(1000) Cash".
3. **Rules.** `list_rules` returns rules by category, then rank. `create_default_rules` only seeds a workspace that has no default rules; it won't restore one that was deleted. For a custom or asset rule, resolve every reference first, because rules match ids:
   - `assets`: `[{ id, symbol, name }]` from `search_assets` (prefer the contract address), or the string `"All Assets"`;
   - `label`: `[{ label }]` using names from `get_supported_labels`;
   - `accounts` / `counterParty`: `[{ walletId }]` from `get_integrations`, or `{ address, name }` / `{ alias, name }` where `name` is the provider name. A bare address never matches;
   - `ledgerType`: `Incoming`, `Outgoing` or `All` (fee legs count as outgoing);
   - `coa`: a code from `list_chart_of_accounts`.

   Call `create_custom_rule` with the right `category` and a deliberate `rank`, check its position with `list_rules`, then `apply_rules` (asynchronous; returns a task id — poll `get_task`).

4. **Fill the gaps.** `list_coa_ledgers(missingCoa: true)` lists legs still missing an account (add `startTime`/`endTime` to work on one period, such as the month being closed) (`meta.total` for the count, up to 500 per page). A transaction with any untagged leg gets **no journal at all**, so it won't show in `list_journals` until it's fixed.
   - Many legs following one pattern → write a rule (step 3).
   - A batch of transactions of the **same type** → `bulk_assign_ledger_accounts(trxIds, debitCoaCode, creditCoaCode)`, up to 100 ids per call; poll the returned task. Trades and transfers only take the code for their direction.
   - One leg → `assign_ledger_accounts(ledgerId, debitCoaCode, creditCoaCode)`. It rebuilds that journal immediately and marks the leg as manual; pass `null` to hand it back to the rules.
5. **Wait for journals to settle.** After `apply_rules`, a bulk assignment or `trigger_recalculation`, poll `get_recalculation_status(taskId)` or `get_task` with the returned task id. Without one, `list_tasks(operationType: "coa-pipeline", limit: 1)`: `pending` or `in_progress` means journals are still being rebuilt.
6. **Check the journals.** `list_journals(includeLines: true)` shows each journal's `syncedToERP` and `erpError`; narrow it with `startTime`/`endTime` (the transaction date) and `syncedToERP: false` to see what a period still has to push. A line whose account is missing means the code no longer exists. Small rounding differences are balanced to the "Small Errors" account automatically. Tell the user how many journals are unsynced and which have errors.
7. **Roll-ups.** If the business posts monthly roll-up journals (Roll-up Accounting in the web app; there's no tool for it here), don't push individual journals for those months. Pushing everything also sends unpushed journals inside a roll-up period, which then post twice when the roll-up is pushed. Push explicit `transactionIds` outside rolled-up periods instead. A push of ids inside a closed roll-up is refused with `COVERED_BY_ROLLUP`.
8. **Push.** Before pushing, say how many journals will go, to which organisation, and whether as drafts. Then call `sync_to_erp(platform)`:
   - `transactionIds` for a specific set (up to 1000), or omit them to push every unsynced journal. **Softledger needs explicit ids, at most 50 per push.**
   - `pushType`: `DRAFT` or `POSTED` (Xero and Softledger default to draft; QuickBooks ignores it).
   - The workspace base currency (`get_accounting_settings`) must match the ERP organisation's currency, otherwise the push fails with `CURRENCY_MISMATCH`.
   - `JOURNALS_REGENERATING` means step 5 isn't finished; wait and retry.
   - Poll the returned task. A `partial_success` result lists which journals failed; their `erpError` says why. Fix them and push those ids again.
   - Journals that were already pushed are not re-sent when rules change later. Push them again with explicit `transactionIds` if the ERP copy must be updated.
9. **Lock (optional).** `lock_period` is a tax lock on a quarter or a year (see `get_lock_boundaries`). It blocks transaction and ledger edits on or before that date. It doesn't block account tagging or ERP pushes, and it isn't the same as closing a month in the ERP.

## Boundaries

- Ask before every write, and say clearly when it goes to the ERP rather than only to Kryptos.
- Don't call `disconnect_erp` unless the user asks for exactly that. It deletes that ERP's accounts from Kryptos (legs and rules lose them, including manual tags) and clears the journals' ERP references, so reconnecting and pushing again would duplicate entries in the ERP.
- `trigger_recalculation` re-derives every rule-assigned leg and rebuilds all journals (manual assignments are kept). Pass `updateLedgerCoa: false` to rebuild journals only. Run it only when the user asks.
- Never guess an account code; take it from `list_chart_of_accounts`, or ask.
- Explain the accounting treatment Kryptos applies; don't give accounting or tax advice beyond that.
