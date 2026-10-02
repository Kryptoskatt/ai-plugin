---
name: review-and-reconcile
description: Review a Kryptos workspace for data problems that make crypto tax figures wrong — uncategorised transactions, missing purchase history, missing prices, balance mismatches, high P&L transactions and unidentified addresses — and fix them with the user. Use when the user asks to review, reconcile, clean up or check their data, asks why a tax figure or balance looks wrong, or wants to get ready to file.
---

# Review and reconcile a Kryptos workspace

This mirrors the **Review** page in the Kryptos web app (retail and Enterprise): a progress score and six checks. Work through them with the user, fix what the data supports, and ask about everything else. Explicit instructions from the user take priority over this workflow.

Discover everything from the tools: counts, thresholds, labels and how this workspace taxes each label. Don't assume them. Errors return a `code` (for example `NO_TAX_YEAR_PLAN`, `PERIOD_LOCKED`) and a `message`; use both to tell why a call failed. If one says a permission or scope is missing, tell the user to reconnect Kryptos and grant it.

## The six checks

| Check                      | Kind        | How to list it                                                                                                       | Usual fix                                                                               |
| -------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Missing Integrations       | Suggestion  | `get_missing_integrations`                                                                                           | Connect the wallet or exchange, name the address, or ignore it                          |
| Missing Balances           | Action item | `get_missing_balances`                                                                                               | Mark scam tokens as spam; find the missing deposit/withdrawal                           |
| Missing Purchase History   | Action item | `get_missing_purchases`; rows via `get_transactions(isMissingTransaction: true)`                                     | Connect the source wallet, merge an unmatched transfer, relabel, or add the acquisition |
| Missing Price              | Action item | `get_missing_prices`; rows via `get_transactions(hasMissingPrice: true, isMissingPriceIgnored: false)`               | Set a per-unit price, mark spam, or dismiss                                             |
| Uncategorised Transactions | Action item | `get_transactions(types: ["deposit","withdrawal"], labels: ["Deposit","Withdrawal"], isUncategorisedIgnored: false)` | Give each a specific label, merge, or dismiss a pure transfer                           |
| High P&L Transactions      | Suggestion  | `get_transactions(minTotalGains: <highPnlThreshold>, isHighPnLReviewed: false)`                                      | Check the legs; fix them or mark reviewed                                               |

Action items must be resolved before the figures can be trusted. Suggestions are worth checking but may be correct as they are. Don't pass `isMissingTransaction` and `hasMissingPrice` in the same call; they are combined with OR, not AND.

## Workflow

1. **Check the data is settled.** Call `list_sync_jobs`. If any sync is `pending`, `queued` or `in_progress`, the counts will change; offer to wait. A sync that ended `partially_synced` with `limitReached: true` hit the workspace's transaction limit, so its history is incomplete; say so.
2. **Learn how this workspace is taxed.** Call `get_accounting_settings`. Its `tagsException` gives the tax treatment of each label here (for example `CAPITAL_GAIN`, `INCOME`, or none), and `advancedTxSettings` switches categories on or off (crypto-to-crypto, airdrops, rewards, LP tokens…). Use these, not general assumptions, when you explain what a label change does.
3. **Summarise.** Call `get_reconciliation_summary`. Report `percentage` and a table of the six checks with their counts and kind.
   - The price, purchase, uncategorised and high-P&L counts are **transactions**; their lists return **groups** (asset + wallet), so the totals won't match.
   - The integrations and balances counts should equal their lists' `meta.totalCount`. If they don't, the data changed between calls (a sync, a recompute, or a spam change finishing); read the summary again.
   - `percentage` only moves with prices, purchases, uncategorised and high-P&L items. Fixing balances or integrations is still necessary but won't change it.
4. **Fix root causes first**, in this order, skipping checks at zero:
   1. Missing Integrations and Missing Balances — an unconnected wallet or spam tokens are the most common cause of everything else.
   2. Missing Purchase History.
   3. Missing Price.
   4. Uncategorised Transactions.
   5. High P&L Transactions.
5. **For each check:** list the items (up to 200 per page, biggest groups first), explain the likely cause in plain language, and propose specific fixes as a table. **Wait for the user to confirm** before any write; one confirmation can cover a batch the user has seen. If a write is rejected because the transaction is in a locked period, check `list_period_locks` and tell the user.
6. **Re-check.** Writes (labels, edits, merges, new transactions) trigger a recalculation automatically. Watch `list_tasks(operationType: "workspace-recompute", limit: 1)` until it is no longer `pending` or `in_progress`, then call `get_reconciliation_summary` again and report the new percentage. If `get_accounting_status` still shows a `lastCalculation` from before your fixes (or after `mark_spam`, which doesn't trigger one), call `trigger_recompute` and poll `get_task` before quoting any tax figure.

Pass the `false` review flags shown in the table so items already dismissed or marked reviewed drop out of the lists, matching the summary counts.

## How to fix each check

**Missing Integrations.** Rows are groups. When `is_exchange_address` is true the row is a whole exchange (`provider_name` set): tell the user to connect that exchange in the Kryptos web app; never ask for or accept API keys or secrets in the chat. An unrecognised `address` (listed once it has at least 100 transactions; `provider` is its chain) — ask whose it is:

- the user's own wallet → add it with `add_wallet` (public address, the `provider` as `providerId`, and a `primaryPortfolioId` from `list_portfolios`);
- a known person or company → `create_contact`, then `assign_address_to_contact` (the chain as `chainName`); this also removes the row;
- never going to be named (burn address, dust sender) → `ignore_counterparty(ignored: true)` for **each** id in the row's `counter_party_ids` (not the row `id`).

**Missing Balances.** `diff` is calculated minus reported. `missing_deposit` means the history is missing an acquisition or deposit; `missing_withdrawal` means a missing disposal or withdrawal. Only wallets whose provider reports holdings are compared, so CSV-imported and custom wallets never appear here.

- In on-chain wallets most rows are usually **airdropped scam tokens**: a URL, "claim" or "visit" in the name, no price, a very large quantity, a calculated quantity of 0. Offer `mark_spam` with `entries: [{ assetId, isSpam: true }]` (up to 500; leaving out `chainId` marks it on every chain). A scam token can copy a real ticker such as USDT, so check its `assetId` differs from the one on the user's priced transactions, and use `get_asset_transaction_counts` to show how much marking hides. `mark_spam` returns a task: poll `get_task`, then `trigger_recompute`, then read the list again. Rows without an `assetId` can't be marked.
- For the rest, check Missing Integrations first. If an integration may simply be behind, call `resync_integration` with the default incremental mode, or `startTime` to backfill. Never pass `syncMode: "resync_from_start"` unless the user explicitly asks; it deletes the integration's data first. If the user can describe the missing transaction, record it with `create_manual_transaction` (see below).

**Missing Purchase History.** A sale with no earlier acquisition. Common causes, in order of likelihood:

- the coins came from a wallet that isn't connected → connect it;
- a transfer between the user's own wallets wasn't matched → merge the two sides (see Merging below);
- an incoming gift, airdrop or reward is still a plain Deposit → relabel it (see Uncategorised);
- a purchase from before the user started tracking → `create_manual_transaction` with the user's date, quantity and cost.

Never invent a cost basis, date or quantity; ask.

**Missing Price.** Read the transaction's legs with `get_transaction_ledgers`, then set the price on the unpriced leg with `update_ledger(id, ledgerId, price, priceCurrency, priceTimestamp)`: `price` is a **per-unit** price (value is quantity × price, so a total is wrong), `priceCurrency` its currency (a price in another currency is converted onto the workspace's), and `priceTimestamp` the leg's own timestamp.

- Only use a price the user gives you, or one they accept from a source you name.
- If a major asset has no price — a chain's native coin, a large stablecoin, wrapped BTC or ETH — that's a gap in Kryptos's price data: tell the user to report it to Kryptos support rather than typing a price in.
- For an airdropped scam token, offer `mark_spam` as above.
- If the user wants to leave it, dismiss it with `update_transaction_label(isMissingPriceIgnored: true)`, one transaction at a time.

**Uncategorised Transactions.** Plain Deposit/Withdrawal labels with no specific meaning. Check `tagsException`: a plain Withdrawal is commonly a taxable disposal and a plain Deposit an acquisition at market value, so a bridge move or a transfer to the user's own exchange account left as a Withdrawal can overstate gains.

- Group the transactions by what they did: the transaction's `protocol.functionName` (for example `bridge`, `addLiquidityETH`, `execTransaction`, `buyNFT`) and the protocol or exchange in its source data. Propose one label per group, using the **exact** name from `get_supported_labels`. Labels aren't validated, so a misspelt one is stored as is.
- **Merge before you relabel** (see Merging). A merged transaction keeps cost basis and holding date across the move; two separately labelled halves don't.
- Apply a label to a group with `bulk_update_transaction_labels` (`{ ids, data: { label } }`). `data` only accepts `label`, `description`, `notes` and `tags`, and one id in a locked period rejects the whole batch.
- A genuine transfer of the user's own funds with no other activity can stay a Deposit/Withdrawal. Dismiss it with `update_transaction_label(isUncategorisedIgnored: true)`, one at a time (the bulk tool can't set this flag).
- To flag one for the user to check later, add a tag; `tags` replaces the whole list, so read the current tags first.

See `references/label-guide.md`.

**High P&L Transactions.** Pass `minTotalGains` exactly equal to `highPnlThreshold` from the summary; any other value, or sorting by gains, is refused. It matches large gains and large losses. If the threshold is very low (it defaults to a single unit of currency when no one has set it), the list can be most of the workspace: say so, and review the largest by `sortBy: "netValue"` first. Gains, cost basis and proceeds are always null on this connection, so judge each transaction by its legs (`get_transaction_ledgers`): is the price plausible for that date, is the quantity right, is an earlier acquisition missing? Fix a wrong price with `update_ledger`, and wrong assets or quantities with `edit_transaction`. If the user confirms it's correct, mark it with `update_transaction_label(isHighPnLReviewed: true)`. A large gain is often just a large gain.

**Merging.** `merge_transactions(trxsIDs)` joins at least two transactions into one. It works for Deposit/Withdrawal (and fiat deposit/withdrawal) transactions no more than a day apart, with one asset on each side, and usually one deposit with one withdrawal. Kryptos sets the result's label itself (Transfer, Bridge Transfer, Trade or Cross Chain Swaps). A transaction already relabelled (for example to Bridge Send) can no longer be merged. `split_transaction` undoes a merge.

**Recording a missing transaction.** `create_manual_transaction` needs `timestamp` (epoch ms), a `label`, and legs `{ assetId, quantity, price?, walletId? }`: incoming only for a deposit, outgoing only for a withdrawal, both for a trade. Resolve each `assetId` with `search_assets` (prefer the contract address; a bare ticker can match a counterfeit) and each `walletId` with `get_integrations`.

## Boundaries

- Ask before every write. Show what will change and on which transactions.
- Never guess prices, cost basis, dates, quantities or a counterparty's identity. If the data doesn't show it, ask the user.
- Don't delete transactions, resync from scratch or unlock periods unless the user explicitly asks for that specific action.
- `get_transactions` and the summary leave out Spam- and Ignore-labelled transactions; the missing-price and missing-purchase lists may still include them.
- Describe what the data shows and what each label means for tax in this workspace. Don't give personal tax or investment advice.

## What to tell the user at the end

The starting and new percentage, what was fixed (counts per check), what is left and why, and whether gains have been recalculated since the fixes.
