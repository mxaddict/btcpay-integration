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
- Added `BlsctDerivationStrategy.cs` with:
  - `BlsctAddressDeriver` for BLSCT sub-address derivation via C# P/Invoke
  - `BlsctDerivationStrategy` class implementing the derivation strategy pattern
  - `BlsctDerivationStrategyFactory` for parsing `blsct:VIEW_HEX:SPEND_HEX` strings
- Added project reference to `libblsct-bindings/ffi/csharp/NavioBlsct.csproj`

### NBXplorer (navio-support branch)
- Updated `NBXplorerNetworkProvider.Navio.cs` to set `DerivationStrategyFactory` to `BlsctDerivationStrategyFactory`
- Added BLSCT path in `GenerateAddressesCore` that detects `BlsctDerivationStrategy` and routes to `GenerateBlsctAddressesCore`
- Implemented `GenerateBlsctAddressesCore` method that derives BLSCT addresses using `BlsctAddressDeriver.Derive()`

## Submission Order

1. **NBitcoin** → Must merge first (defines Navio network + BlsctDerivationStrategy)
2. **NBXplorer** → Depends on NBitcoin (uses BlsctDerivationStrategy)
3. **BTCPayServer** → Depends on NBXplorer
4. **btcpayserver-docker** → Independent, can go in parallel

## Integration Repo

<https://github.com/mxaddict/btcpay-integration>

## Current Status

All core integration code has been implemented. The remaining work is:
1. End-to-end testing with a running Navio testnet daemon
2. Optional: UI enhancement for Navio-specific audit key import instructions
3. Upstream PRs review and merge
