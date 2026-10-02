# Choosing a label for an uncategorised transaction

Always pick the final label from `get_supported_labels`, using its exact name; a misspelt label isn't rejected, it's stored as is. For what a label means **in this workspace**, read `tagsException` and `advancedTxSettings` from `get_accounting_settings`. They can differ by country and settings. Use this guide to narrow the choice, then confirm it with the user.

## First: is it one half of a move between the user's own accounts?

If a withdrawal from one of the user's wallets matches a deposit into another (same asset, similar amount, within a day), merge them with `merge_transactions`. Kryptos labels the result Transfer, Bridge Transfer, Trade or Cross Chain Swaps, and cost basis and holding period carry over. Merge **before** relabelling; a relabelled transaction can't be merged. Label the halves separately only when the other side isn't in the workspace.

## Incoming (was a plain Deposit)

1. Moved from another wallet the user owns → merge (above), or leave it as a Deposit and dismiss it from the review.
2. Earned → a reward label (staking, farming, mining, interest).
3. Received for free → a gift or airdrop label.
4. Borrowed → a borrow or loan label.
5. Capital coming back → unstake, lend redeem or collateral withdrawal.
6. Payment for goods or services → an income label.

## Outgoing (was a plain Withdrawal)

1. Moved to another wallet or exchange account the user owns → merge (above).
2. Exchanged for another asset → a trade or swap label.
3. Lent or staked → a lend or stake label.
4. Paid for something, or a fee → a payment or fee label.
5. Given away → a gift or donation label.
6. Lost or stolen → the lost or stolen label.

## On-chain activity that is often left as a plain Deposit/Withdrawal

These are examples. Read what each transaction did from its `protocol.functionName` and source data rather than relying on a fixed list.

| What the transaction did                                                     | Label                   |
| ---------------------------------------------------------------------------- | ----------------------- |
| Sent to or received from a bridge, with the other side also in the workspace | Merge → Bridge Transfer |
| Sent to a bridge, other side not in the workspace                            | Bridge Send             |
| Received from a bridge, other side not in the workspace                      | Bridge Receive          |
| Added assets to a pool (`addLiquidity…`)                                     | Add Liquidity           |
| Removed assets from a pool                                                   | Liquidity Withdrawal    |
| Collected LP fees or farm rewards (`collect`, `harvest`)                     | Reward                  |
| Bought or sold an NFT (`buyNFT`)                                             | NFT Buy / NFT Sell      |
| Paid a merchant (`pay`, a payment processor)                                 | Payment                 |
| Wrapped or unwrapped a token (ETH ↔ WETH)                                    | Wrap / Unwrap           |

A one-sided swap leg (only the incoming or only the outgoing side was imported) means the other leg is missing. Don't label one side as a Swap; look for the missing leg first.

## What the label changes

Check `tagsException` for this workspace. Commonly:

- a plain Withdrawal is a taxable disposal, while Bridge Send and Transfer are not;
- a plain Deposit is an acquisition at market value, not income;
- income labels are taxed as income at market value when received;
- trade labels are a disposal, so a capital gain or loss;
- a Bridge Send plus a separate Bridge Receive resets the cost basis and holding period, whereas a merged Bridge Transfer keeps them.

Tax rules differ by country. Explain what a label means; don't tell the user what they owe.
