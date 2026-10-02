# Kryptos for Claude

Kryptos brings your crypto portfolio, transaction history, tax figures and accounting ledger into Claude, across 5,000+ exchanges, wallets and blockchains.

This plugin connects Claude to the **Kryptos MCP server** and adds four skills that guide Claude through common workflows.

## What's included

| Component | What it does |
| --- | --- |
| MCP server `kryptos` (`https://mcp.kryptos.io/`) | Tools to read your holdings, transactions, ledgers, tax figures and chart of accounts, and to fix data with your confirmation. |
| Skill `get-started` | Connect blockchain wallets by public address, wait for the import, and get a first look at the portfolio. |
| Skill `review-and-reconcile` | Work through the Kryptos Review checks — uncategorised transactions, missing purchase history, missing prices, balance mismatches, high P&L and unidentified addresses — and fix them with you. |
| Skill `prepare-tax-year` | Check the data and gains are current, summarise a tax year, generate the tax report, and optionally lock the period once filed. |
| Skill `crypto-bookkeeping` | For Kryptos Enterprise: sync the chart of accounts with Xero, QuickBooks or Softledger, map transactions with rules, and push journals. |

## Install

In Claude Code:

```
/plugin marketplace add Kryptoskatt/ai-plugin
/plugin install kryptos@kryptos
```

Then run `/mcp`, choose **kryptos** and sign in with your Kryptos account.

## Authentication and data

- You sign in to Kryptos with OAuth 2.1 and choose one Kryptos workspace. Claude can only act on that workspace, within the permissions your Kryptos role allows. You can revoke access from your Kryptos account at any time.
- The only service this plugin talks to is the Kryptos MCP server at `https://mcp.kryptos.io/`, which reads and writes your Kryptos workspace data. The plugin itself contains no scripts, hooks or credentials.
- Kryptos never asks for exchange API keys, wallet private keys, seed phrases or passwords in the conversation. Wallets are added by public address only; exchanges are connected in the Kryptos web app.
- No tool can send, swap, withdraw, deposit or trade any asset. The skills ask for your confirmation before changing Kryptos data (labels, prices, contacts, rules, journals), and Claude asks before running a tool that writes.

Privacy policy: https://kryptos.io/privacy-policy · Terms of service: https://kryptos.io/terms-of-services

## Requirements

A Kryptos account. Bookkeeping and ERP features require a Kryptos Enterprise workspace.

## Support

https://kryptos.io/contact-us
