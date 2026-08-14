# Navio BTCPayServer Integration PRs

## Pull Requests

| #   | Repository                   | PR Link                                                  | Status | Notes                                 |
| --- | ---------------------------- | -------------------------------------------------------- | ------ | ------------------------------------- |
| 0   | nav-io/libblsct-bindings     | <https://github.com/nav-io/libblsct-bindings/pull/231>   | Merged | C# SWIG bindings; merged 2026-05-24   |
| 1   | mxaddict/NBitcoin            | <https://github.com/mxaddict/NBitcoin/pull/1>            | Draft  | Added BlsctDerivationStrategy         |
| 2   | mxaddict/NBXplorer           | <https://github.com/mxaddict/NBXplorer/pull/1>           | Draft  | Added GenerateBlsctAddressesCore      |
| 3   | mxaddict/btcpayserver        | <https://github.com/mxaddict/btcpayserver/pull/1>        | Draft  | Navio plugin + audit key wallet setup |
| 4   | mxaddict/btcpayserver-docker | <https://github.com/mxaddict/btcpayserver-docker/pull/1> | Draft  | Docker deployment + navio-cli.sh      |

## Recent Changes

### NBitcoin (navio-support branch)

- Added `Navio.cs` network definition with testnet genesis hex, RPC remappings
- Added BLSCT RPC operations to `RPCOperations.cs`
- Added `RPCMethodOverrides` dictionary to `RPCClient.cs` for RPC remapping
- Added `Blsct` parameter to `CreateWalletOptions`
- `ConfigureBLSCTOverrides()` static method with 8 RPC remappings

### NBXplorer (navio-support branch)

- `NavioNBXplorerNetwork` with `BlsctDerivationStrategyFactory`
- `BlsctDerivationStrategy` + `BlsctDerivationStrategyFactory` in
  NBXplorer.Client
- BLSCT RPC overrides in `RPCClientExtensions.cs` (`blsct=true` wallet creation)
- Descriptor import skip for BLSCT wallets in `Repository.cs`
- `GenerateBlsctAddressesCore` — native P/Invoke address derivation
- `scantxoutset` skip for Navio in `ScanUTXOSetService.cs`
- PSBT endpoints blocked for Navio in `MainController.PSBT.cs` (HTTP 400)
- 9 BLSCT methods added to RPC proxy whitelist

### BTCPayServer (navio-support branch)

- `AltcoinsPlugin.Navio.cs` — plugin with `IsBLSCT = true`, rate rules, icon
- `BTCPayNetwork.IsBLSCT` property for BLSCT-specific UI behavior
- Wallet setup UI: hides hardware/file/scan/seed import for BLSCT
- Xpub view: "BLSCT Audit Key" label, `navio-cli` help text, hides xpub examples
- Confirm addresses: skips address preview for BLSCT (native derivation only)
- Controller: auto-converts raw 160-char hex to `blsct:VIEW:SPEND` format
- Controller: reflection dispatch to `BlsctDerivationStrategyFactory.Parse()`
- Hides "Create new wallet" option for BLSCT (daemon creates wallets)

### btcpayserver-docker (navio-support branch)

- `navio.yml` docker fragment with `naviod` container, RPC/P2P ports, env vars
- `navio_libblsct` shared volume for native P/Invoke (`LD_LIBRARY_PATH` set)
- `crypto-definitions.json` entry for NAV
- `navio-cli.sh` wrapper script

## Upstream Targets

| Fork                           | Upstream                                      |
| ------------------------------ | --------------------------------------------- |
| `mxaddict/NBitcoin`            | `MetacoSA/NBitcoin`                           |
| `mxaddict/NBXplorer`           | `btcpayserver/NBXplorer` (moved from dgarage) |
| `mxaddict/btcpayserver`        | `btcpayserver/btcpayserver`                   |
| `mxaddict/btcpayserver-docker` | `btcpayserver/btcpayserver-docker`            |
| `mxaddict/libblsct-bindings`   | `nav-io/libblsct-bindings`                    |

## Submission Order

0. **Publish `NavioBlsct` 0.1.0 to nuget.org** — a `workflow_dispatch` run of
   `C#: Publish package` on `nav-io/libblsct-bindings`. Nothing below can build
   for anyone else until this happens; see [docs/backlog.md](docs/backlog.md).
1. **NBitcoin** → Must merge and ship a release first (defines Navio network +
   RPC remapping)
2. **NBXplorer** → Depends on a published NBitcoin; its `NBXplorer.Client`
   project references have to go back to `PackageReference` at that point
3. **BTCPayServer** → Depends on NBXplorer (uses audit key parsing)
4. **btcpayserver-docker** → Independent, can go in parallel

## Integration Repo

<https://github.com/mxaddict/btcpay-integration>

## Current Status

Rebased onto current upstream on 2026-08-14; unit tests green in all four repos.
The blockers are no longer about writing code:

1. `NavioBlsct` 0.1.0 is not on nuget.org, so no upstream PR builds off our
   machines
2. End-to-end testing against a running Navio testnet daemon has not started —
   phases 3–5 of [E2E.md](E2E.md)
3. `NBXplorer.Client` still carries local `ProjectReference` entries that cannot
   be reverted until a published NBitcoin has the Navio network

See [docs/backlog.md](docs/backlog.md) for the full list, including the genesis
hash NBitcoin cannot represent.
