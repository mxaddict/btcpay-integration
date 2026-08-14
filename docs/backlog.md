# Backlog

Working notes for the Navio → BTCPayServer integration. Everything below was
verified on 2026-08-14 by running the suites, reading navio-core, and querying
GitHub and nuget.org. Re-verify before acting on anything months from now.

## Verified state (2026-08-14)

All four fork branches were rebased onto current upstream this session, and all
unit suites are green afterwards. Figures are what the runners reported:

- `NBitcoin.Tests` `~Navio`, net8.0 — Failed: 0, Passed: 41, Skipped: 0
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

## Navio network parameters — done, both networks

navio-core **re-genesised its testnet** between our old submodule pointer
(April) and current master, and mainnet has since launched. Everything we had
was from the pre-re-genesis chain, so the P2P magic would have been rejected by
every peer and the RPC port pointed at nothing:

|                   | what we had    | navio-core master |
| ----------------- | -------------- | ----------------- |
| testnet nTime     | 1743259590     | 1777481682        |
| testnet nNonce    | 2              | 1 (after grind)   |
| testnet P2P / RPC | 33570 / 33577  | 33670 / 33677     |
| testnet magic     | `b9 3c 0e df`  | `24 67 d2 c1`     |
| testnet genesis   | synthetic      | `7a04d0211de9…`   |
| testnet datadir   | `testnet5`     | `testnet7`        |
| mainnet P2P / RPC | 8333 / 48484   | 48470 / 48471     |
| mainnet magic     | `acb1d2db`     | `bd 5f c3 00`     |
| mainnet genesis   | a 2019 fiction | `0af3c23ae1ac…`   |

All corrected in `Navio.cs`, `NavioTests.cs`, the `navio.yml` fragment, `E2E.md`
and `BTCPAY.md`. The old mainnet RPC port 48484 was `blsctregtest`'s. Magic
bytes for both networks are now asserted by tests (verified red on a flipped
byte), and `NavioGenesisHashIsCorrect` /`NavioMainnetGenesisHashIsCorrect` are
enabled and green — nothing in the NBitcoin Navio suite is skipped any more.

The previously documented "expected" genesis hash
`57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b` appears
**nowhere** in navio-core at either pointer. It was simply wrong.

### How the genesis hashes became representable

`Consensus.HashGenesisBlock` in NBitcoin parses the configured genesis bytes and
returns `block.GetHash()` — which reads only the 80-byte header. So the genesis
entries now carry the **real header**, testnet's post-grind nonce included, with
a placeholder body. The hash is exact; `GetGenesis().Transactions` is fiction
and must not be read. Both hashes were derived independently in Python from the
chainparams fields and matched navio-core's own asserts before being written
into `Navio.cs`.

This closes the NBXplorer concern from the previous session:
`DbConnectionHelper.CreateGenesis` seeds height 0 from
`Consensus.HashGenesisBlock`, which now matches the daemon.

### Open: NBitcoin cannot parse any Navio transaction or block

Found while reconstructing the genesis: navio-core's `COutPoint` serializes
**only the output id — there is no output index**
(`src/primitives/transaction.h`). This is not a BLSCT detail; it applies to
every Navio transaction. `CBlock` also splices a PoS proof between the header
and `vtx`, and `CTxOut` carries range proofs, token ids and predicates behind a
flags byte.

So anything downstream that deserializes a Navio transaction or block body with
NBitcoin is wrong today. Not yet established **whether NBXplorer does that** for
Navio on the indexing path — the derivation and RPC paths were built to avoid
it, but block indexing was never traced. Trace it before phase 3, because a
failure there looks like "sync is stuck" rather than a parser bug.

If it does need parsing, the fix is a `NavioConsensusFactory` overriding
`CreateTxIn` with a `TxIn` whose `ReadWrite` omits the index — `TxIn.ReadWrite`
and `ConsensusFactory.CreateTxIn` are both virtual, so the hook exists. The
BLSCT output fields are a much larger job.

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

- **`NBitcoin.Tests` net472 target does not run at all.** The run aborts with
  `Could not find 'mono' host`; only net8.0 executes. Decide: install mono, or
  narrow the test project's `TargetFrameworks`. "41 passed" describes one of two
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
- **Making the docker fragment pick ports per network.** Not needed: BTCPay's
  fragments pass `rpcport`/`port` explicitly and point NBXplorer at the same
  numbers, so one set works on mainnet and testnet alike. That is also why the
  33577 → 33677 renumbering was cosmetic, not a fix — both sides always agreed.
  An earlier commit message here claimed otherwise and was corrected.
- **Writing a `NavioConsensusFactory` now.** The hook exists and the genesis no
  longer needs it. Worth building only if NBXplorer turns out to parse Navio
  block bodies — see the open item above.

## Not reviewed

The implementation diffs have had no line-level review this session — the state
above comes from test runs, git and GitHub state, reading navio-core's
chainparams, and targeted greps. The BTCPayServer UI changes, the `navio.yml`
fragment beyond its ports, and `navio-cli.sh` have had no such review since they
were written in April 2026. The btcpayserver rebase resolved five conflicts by
hand (upstream moved `UIStoresController.Onchain.cs` into
`Plugins/Wallets/Controllers/`); the tests pass, but the UI has not been opened
in a browser since.
