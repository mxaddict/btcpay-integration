# Navio BTCPayServer Integration PRs

## Pull Requests

| #   | Repository                   | PR Link                                                  | Status | Notes                                      |
| --- | ---------------------------- | -------------------------------------------------------- | ------ | ------------------------------------------ |
| 0   | nav-io/libblsct-bindings     | <https://github.com/nav-io/libblsct-bindings/pull/231>   | Open   | C# P/Invoke layer                          |
| 1   | mxaddict/NBitcoin            | <https://github.com/mxaddict/NBitcoin/pull/1>            | Draft  | Added BlsctDerivationStrategy              |
| 2   | mxaddict/NBXplorer           | <https://github.com/mxaddict/NBXplorer/pull/1>           | Draft  | Added GenerateBlsctAddressesCore           |
| 3   | mxaddict/btcpayserver        | <https://github.com/mxaddict/btcpayserver/pull/1>        | Draft  | Navio plugin + audit key wallet setup      |
| 4   | mxaddict/btcpayserver-docker | <https://github.com/mxaddict/btcpayserver-docker/pull/1> | Draft  | Docker deployment + navio-cli.sh           |

## Recent Changes

### NBitcoin (navio-support branch)
- Added `Navio.cs` network definition with testnet genesis hex, RPC remappings
- Added BLSCT RPC operations to `RPCOperations.cs`
- Added `RPCMethodOverrides` dictionary to `RPCClient.cs` for RPC remapping
- Added `Blsct` parameter to `CreateWalletOptions`
- `ConfigureBLSCTOverrides()` static method with 8 RPC remappings

### NBXplorer (navio-support branch)
- `NavioNBXplorerNetwork` with `BlsctDerivationStrategyFactory`
- `BlsctDerivationStrategy` + `BlsctDerivationStrategyFactory` in NBXplorer.Client
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

## Submission Order

1. **libblsct-bindings** → Independent (C# P/Invoke layer)
2. **NBitcoin** → Must merge first (defines Navio network + RPC remapping)
3. **NBXplorer** → Depends on NBitcoin (uses BlsctDerivationStrategy + remapping)
4. **BTCPayServer** → Depends on NBXplorer (uses audit key parsing)
5. **btcpayserver-docker** → Independent, can go in parallel with 1–4

## Integration Repo

<https://github.com/mxaddict/btcpay-integration>

## Current Status

All code across all 4 repos is complete. The only remaining work is:
1. End-to-end testing with a running Navio testnet daemon
2. Upstream PRs to `MetacoSA/NBitcoin`, `dgarage/NBXplorer`, etc. (waiting on E2E validation)
