---
name: get-started
description: Set up a Kryptos workspace — connect blockchain wallets by public address, wait for the import to finish, and give a first look at the portfolio. Use when the user is new to Kryptos, has nothing connected yet, asks to add or connect a wallet, or asks how to get started.
---

# Get started with Kryptos

Help the user connect their wallets and see their portfolio for the first time. Explicit instructions from the user take priority over this workflow. Discover chains, providers and portfolios from the tools; don't assume them. Errors return a `code` (for example `NO_TAX_YEAR_PLAN`, `PERIOD_LOCKED`) and a `message`; use both to tell why a call failed.

## Workflow

1. **See what's already there.** Call `get_integrations`. If wallets are connected, summarise them (chain, alias, last sync) and ask what the user wants to add. Call `get_accounting_settings` and tell the user the country (`countryCode`), base currency and cost-basis method in force. They drive every tax figure, and they're changed in the Kryptos web app, not here.
2. **Pick the portfolio.** Every wallet belongs to a portfolio, in retail and Enterprise workspaces alike. Call `list_portfolios`:
   - Enterprise: ask which portfolio to use;
   - otherwise: use the one with `isDefault: true`;
   - if there are none: offer `create_portfolio`.

   Pass its id as `primaryPortfolioId` on every `add_wallet` call.

3. **Collect the wallet.** Ask for the public wallet address and the chain. Wallets are added by **public address only**:
   - Never ask for, accept or repeat a private key, seed phrase, API key, secret or password. If the user pastes one, tell them not to share it, don't use it, and suggest they treat it as exposed.
   - Exchange accounts (API key or OAuth) and CSV files are connected in the Kryptos web app; say so and move on.
4. **Pick the chains.**
   - **EVM address** (`0x` + 40 hex characters): call `detect_wallet_chains` and show the chains with activity. Skip any whose `isWorking` is false, and ask whether to add all of them or only some. The `id` of each result is the `providerId` to use. An empty result means no activity was found; ask the user which chain they use instead of guessing.
   - **Any other address** (Bitcoin, Solana and others): find the provider with `get_integration_metadata(search: <chain name>, type: "blockchain", isWorking: true)`. Search matches loosely ("bitcoin" also finds Bitcoin Cash and Bitcoin SV), so confirm the exact provider with the user.
5. **Add it.** After the user confirms the address, chains and portfolio, call `add_wallet` once per chain with `address`, the chain's `providerId`, `primaryPortfolioId`, and an `alias` the user will recognise, such as "Main wallet (Ethereum)".
   - Each call starts an import and returns it as `sync`. If `sync` is empty, start one with `sync_integration`.
   - If the message says the credentials are already present, that wallet is already connected on that chain; tell the user and move on. If it says the alias exists, pick another alias.
6. **Wait for the import.** Track each sync with `get_sync_job` (or `list_sync_jobs`). `pending`, `queued` and `in_progress` mean it's still running; `completed`, `partially_synced`, `failed` and `cancelled` are final. Imports usually take a few minutes; report progress briefly rather than calling in a tight loop.
   - `failed`: the job explains why; report it, and don't retry more than once without asking.
   - `partially_synced` with `limitReached: true`: the import stopped at the workspace's transaction limit, so the history is incomplete. Tell the user the limit is raised through their plan or Kryptos support.
7. **First look.** After the imports, a recalculation runs automatically. Wait until `list_tasks(operationType: "workspace-recompute", limit: 1)` is no longer `pending` or `in_progress`; cost basis and gains lag until it finishes. Then:
   - `get_dashboard_totals` for net worth, cost basis and unrealised gain (never add up a page of holdings);
   - `get_holdings(sortBy: "value", sortOrder: "desc", limit: 10)` for the largest holdings;
   - `get_transaction_summary` for how many transactions were imported.
8. **Point to the review.** Call `get_reconciliation_summary` and report the review percentage. If there are action items, explain in one sentence that some transactions need attention before the tax figures can be trusted, and offer to go through them (the `review-and-reconcile` skill).

## Boundaries

- Ask before adding each wallet; show the address, chains and portfolio you'll use.
- Don't guess a chain from an address that could belong to several; ask, or use `detect_wallet_chains` for EVM addresses.
- A freshly imported wallet often shows missing prices and balance mismatches, many of them from airdropped spam tokens. That's expected; don't present it as an error.
- Describe the user's data; don't give investment or tax advice.
