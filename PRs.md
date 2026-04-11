# Navio BTCPayServer Integration PRs

## Pull Requests

| # | Repository | PR Link | Status |
|---|------------|---------|--------|
| 1 | mxaddict/NBitcoin | https://github.com/mxaddict/NBitcoin/pull/1 | Draft |
| 2 | mxaddict/NBXplorer | https://github.com/mxaddict/NBXplorer/pull/1 | Draft |
| 3 | mxaddict/btcpayserver | https://github.com/mxaddict/btcpayserver/pull/1 | Draft |
| 4 | mxaddict/btcpayserver-docker | https://github.com/mxaddict/btcpayserver-docker/pull/1 | Draft |

## Submission Order

1. **NBitcoin** → Must merge first (defines Navio network)
2. **NBXplorer** → Depends on NBitcoin (NuGet package)
3. **BTCPayServer** → Depends on NBXplorer
4. **btcpayserver-docker** → Independent, can go in parallel

## Integration Repo

https://github.com/mxaddict/btcpay-integration