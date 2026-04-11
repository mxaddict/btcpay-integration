# BTCPayServer Integration for Navio

Workspace for the Navio BTCPayServer integration.

This repo is a coordination root, not the application code itself. It tracks the integration plan and pins the forked repositories needed to add Navio support to BTCPayServer.

## Contents

| Path | Purpose |
|------|---------|
| `BTCPAY.md` | Full integration plan and implementation notes |
| `PRs.md` | Draft PR tracking for the forked repos |
| `NBitcoin/` | Navio-enabled fork of `MetacoSA/NBitcoin` |
| `NBXplorer/` | Navio-enabled fork of `dgarage/NBXplorer` |
| `btcpayserver/` | Navio-enabled fork of `btcpayserver/btcpayserver` |
| `btcpayserver-docker/` | Navio-enabled fork of `btcpayserver/btcpayserver-docker` |
| `navio-core/` | Read-only reference submodule for Navio chain params and RPC docs |

## Current State

- Testnet-only integration plan
- Submodules checked out on `navio-support`
- `navio-core` included as the chain-parameter source of truth
- PR tracking kept in `PRs.md`

## Workflow

1. Read `BTCPAY.md` for the implementation plan.
2. Make code changes inside the relevant submodule.
3. Push the `navio-support` branch from that submodule.
4. Open or update the matching draft PR.
5. Keep `navio-core` aligned with Navio chain parameter changes when needed.

## Notes

- This repo does not build BTCPayServer directly.
- The root repo mainly documents and coordinates the integration effort.
- `navio-core` is reference-only for chain params, genesis data, and RPC docs.
