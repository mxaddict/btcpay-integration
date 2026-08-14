# Backlog

Working notes for the Navio → BTCPayServer integration. Everything below was
verified on 2026-08-14 by running the suites, reading navio-core, and querying
GitHub and nuget.org. Re-verify before acting on anything months from now.

## Verified state (2026-08-14)

All four fork branches were rebased onto current upstream this session, and all
unit suites are green afterwards. Figures are what the runners reported:

- `NBitcoin.Tests` `~Navio`, net8.0 — Failed: 0, Passed: 37, Skipped: 1
- `NBXplorer.Tests` `~Navio|~Blsct`, net10.0 — Failed: 0, Passed: 24, Skipped: 4
- `BTCPayServer.Tests` `~Navio`, net10.0 — Failed: 0, Passed: 6, Skipped: 2

Phases 1–2 of [E2E.md](../E2E.md) pass. Phases 3–5 have not been started.

Upstream remotes are now configured on every fork (`upstream`, plus `navio` on
`libblsct-bindings`), so drift is measurable from here on.

## Blockers on upstream PRs

### `NavioBlsct` 0.1.0 was never published to nuget.org

The hardest blocker, and the one nobody outside `nav-io` can clear.
`nav-io/libblsct-bindings#231` **merged** 2026-05-24, and its
`C#: Publish package` run (`26356460712`) reports success — but enumerating that
run's jobs shows `pack` and all three `verify` jobs succeeded while the
`publish` job was **skipped**. It is gated
`if: github.event_name == 'workflow_dispatch'`, so a merge to `main` can never
publish. The registry agrees: the nuget.org search API returns `totalHits: 0`
for `NavioBlsct`, and `v2/package/NavioBlsct/0.1.0` resolves to 404.

The package exists here only as `local-nuget/NavioBlsct.0.1.0.nupkg` (46 MB,
native binaries for linux-x64/win-x64/osx-arm64), wired up by the root
`nuget.config`. `NBXplorer.Client` references it, so **no upstream PR builds for
anyone but us** until someone with `nav-io` write access triggers the workflow
by hand.

Needs: a `workflow_dispatch` run on `nav-io/libblsct-bindings` `main`, then
confirming the package actually appears on nuget.org — not just that the run
went green, since a green run with a skipped publish is precisely what happened
last time. Worth proposing a tag-triggered publish upstream so it cannot recur.

### `NBXplorer.Client` still points at local project paths

`NBXplorer/NBXplorer.Client/NBXplorer.Client.csproj` carries `ProjectReference`
entries into `../../NBitcoin/`. Upstream needs `PackageReference` on published
versions — as of the rebase, upstream is on NBitcoin 10.0.9 / NBitcoin.Altcoins
6.0.4, which is what those entries revert to.

Chicken-and-egg: the published NBitcoin has no Navio network, so the revert
cannot happen until MetacoSA merges and ships. Sequence is `NavioBlsct` publish
→ NBitcoin merge + release → revert here → NBXplorer PR. Until then the local
refs must stay and the NBXplorer PR cannot leave draft.

### Dev-only NuGet plumbing must be kept out of the PRs

The root `nuget.config` adds a `local-nuget` source, and the `btcpayserver`
branch carries a `chore: add local NuGet config` commit. Neither belongs
upstream. Drop or fixup that commit when the BTCPayServer branch is prepared for
review.

## Navio network parameters — fixed, with one hole left

navio-core **re-genesised its testnet** between our old submodule pointer
(April) and current master. Found by diffing `src/kernel/chainparams.cpp`:

|                      | old pointer    | current master  |
| -------------------- | -------------- | --------------- |
| testnet nTime        | 1743259590     | 1777481682      |
| testnet nNonce       | 2              | 0               |
| testnet P2P port     | 33570          | 33670           |
| testnet RPC port     | 33577          | 33677           |
| testnet magic        | `b9 3c 0e df`  | `24 67 d2 c1`   |
| testnet genesis hash | (not asserted) | `7a04d0211de9…` |

Everything we had was from the pre-re-genesis chain, so the P2P magic would have
been rejected by every peer and the RPC port pointed at nothing. Fixed this
session in `Navio.cs`, `NavioTests.cs`, the `navio.yml` docker fragment,
`E2E.md` and `BTCPAY.md`, and a `NavioTestnetMagicMatchesChainparams` test now
guards the magic (confirmed it goes red on a flipped byte). The locally
installed `navio-cli` v0.1.3 already defaults `-testnet` to 33677, which is
independent confirmation the new params are what is deployed.

**Still open — the genesis hash cannot be represented.** `Navio.cs` sets a
_synthetic_ genesis: real header fields, but a minimal standard coinbase,
because the real genesis carries a BLSCT transaction NBitcoin has no parser for.
So `network.GenesisHash` is not the chain's genesis hash, and
`NavioGenesisHashIsCorrect` stays skipped. This is not cosmetic:
`DbConnectionHelper.CreateGenesis` in NBXplorer seeds the chain with
`Network.NBitcoinNetwork.Consensus.HashGenesisBlock`, so NBXplorer's height-0
block will not match the daemon's. Whether that breaks indexing is **unverified
and only phase 3 can answer it**. The real fix is either a BLSCT-aware block
parser in NBitcoin or a way to set a network's genesis hash without parsing a
block; both are upstream design decisions worth raising before the NBitcoin PR.

Note the previously documented "expected" genesis hash,
`57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b`, appears
**nowhere** in navio-core at either pointer. It was wrong, and every doc that
quoted it has been corrected to `7a04d0211de9…`.

### Mainnet parameters are still stale — deliberately left

`Navio.cs` `CreateMainnet` carries magic `acb1d2db`, port 8333 and RPC
port 48484. Current navio-core mainnet is magic `bd 5f c3 00`, port 48470, RPC
48471 — 48484 is `blsctregtest`'s RPC port, not mainnet's. Mainnet genesis also
moved to `0af3c23ae1ac…` with an nTime in the future, so the chain has not
launched. Left alone because the integration targets testnet and touching
mainnet would be scope creep, but it must be fixed before anyone points this at
mainnet.

### `blsctregtest` now exists in navio-core

There is a `ChainType::BLSCTREGTEST` with `consensus.fBLSCT = true`, P2P 18444,
RPC 48484. Earlier notes recorded that Navio regtest was excluded because
`fBLSCT=false` and `CreateRegtest()` returns null — that reason no longer holds.
A BLSCT-capable regtest would let the daemon tests run in CI instead of needing
a live testnet, which is the single biggest lever on phases 3–5. Not
investigated further; worth costing before building more testnet-only
scaffolding.

## Phase 3 (live daemon) — blocked on a usable node

- `/usr/bin/naviod` is Navio Core v0.1.3 running as system user `navcoin` on
  ports 48470–48472 — those are **mainnet** ports under current chainparams. Its
  datadir and cookie are not readable by `mxaddict`. Unusable for testnet work.
- `~/.navio/` holds only `wallets/` and an empty `settings.json` — no chain
  data, no cookie.
- The `navio-core` submodule is not built; there is no `src/naviod` in the tree.

Pick one and do it: build `navio-core` from the submodule, or get credentials
for the existing service and run a second instance with `-testnet`. Then set
`NBXPLORER_NAVTESTNET_RPCURL`/`_RPCUSER`/`_RPCPASSWORD` and un-skip the four
`NavioDaemonTests`.

## Test vectors prove less than they look like they prove

`NBXplorer.Tests/Data/blsct_vectors.json` is committed and
`DeriveBlsctAddress_MatchesFixture` passes against it. But the fixture was
generated by `BlsctDerivationStrategy.DeriveBlsctAddress` — the same code path
the test exercises. It is a regression guard, not a correctness check: a
derivation that is wrong the same way twice passes.

Real cross-implementation vectors have to come from the C++ side
(`external_api_tests.cpp` seeds) and do not exist. Until they do, "our BLSCT
address derivation matches navio-core" is **unverified**, and the first evidence
either way will be a testnet payment landing (or not) in phase 5.

## Skipped tests — coverage gaps, stated plainly

- **`NavioGenesisHashIsCorrect` (skipped)** — see the genesis section above. Now
  at least asserts the real hash, so enabling it is a one-line change once
  NBitcoin can represent it.
- **`NBitcoin.Tests` net472 target does not run at all.** The run aborts with
  `Could not find 'mono' host`; only net8.0 executes. Decide: install mono, or
  narrow the test project's `TargetFrameworks`. "37 passed" describes one of two
  configured targets.
- **`NavioDaemonTests` ×4 (skipped)** —
  `EnsureWalletCreated_ForNavio_CreatesBlsctWallet`,
  `GetBalance_RoutesTo_Getblsctbalance`,
  `Listunspent_RoutesTo_Listblsctunspent`, `NavioChain_IndexesBlocks`. Gated on
  the daemon env vars above.
- **`NavioPluginTests` ×2 (skipped)** — `SetupWallet_ForNavio_SetsIsBLSCT_True`
  and `ImportWallet_ForNavio_SetsIsBLSCT_True`, both marked
  `Skip = "requires full BTCPayServer WebApplicationFactory integration stack"`.
  These are the only tests that would exercise the audit-key wallet setup path,
  so that UI flow is covered by nothing.

## Known limitation to carry into every PR

`listblscttransactions` is a stub in navio-core (`src/blsct/wallet/rpc.cpp`,
implementation commented out) and returns an empty array. BTCPayServer's
transaction-history tab for Navio will be empty even after confirmed payments.
This is a navio-core limitation, not a BTCPay bug — say so in each PR
description or it reads as a defect in our code.

## Considered and declined

- **Rebasing `libblsct-bindings` onto upstream.** Not needed: upstream
  squash-merged our C# work as `3e7943c`. The submodule now tracks `nav-io/main`
  directly and the fork's `csharp-support` branch is just pre-squash history.
- **Fixing up the "add then remove" of the `NavioBlsct` reference in NBitcoin's
  history.** Left as a separate honest commit rather than rewriting the commit
  that introduced it; upstream will squash anyway.

## Not reviewed

The implementation diffs have had no line-level review this session — the state
above comes from test runs, git and GitHub state, reading navio-core's
chainparams, and targeted greps. The BTCPayServer UI changes, the `navio.yml`
fragment beyond its ports, and `navio-cli.sh` have had no such review since they
were written in April 2026. The btcpayserver rebase resolved five conflicts by
hand (upstream moved `UIStoresController.Onchain.cs` into
`Plugins/Wallets/Controllers/`); the tests pass, but the UI has not been opened
in a browser since.
