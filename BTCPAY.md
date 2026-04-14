# BTCPayServer Integration Plan for Navio

## Overview

Add Navio (a Bitcoin Core fork with BLSCT) as a supported cryptocurrency in
BTCPayServer. Navio uses Bitcoin-compatible blockchain/network RPCs but requires
**BLSCT-specific wallet RPCs** for address generation, balance queries, UTXO
listing, and transaction creation/signing. It follows the **NBXplorer path**
(same as Litecoin, Groestlcoin, Dogecoin, etc.) but with BLSCT wallet RPC
overrides. This requires changes across 4 repositories, each submitted as a
separate PR.

**Scope: Testnet only.** Mainnet chainparams are not finalized (`fBLSCT = false`
on mainnet currently). Testnet has `fBLSCT = true` and a confirmed genesis hash.
Mainnet integration will follow once mainnet chainparams are complete.

## Architecture

```
┌─────────────────────┐
│    BTCPayServer     │  ← Navio plugin (AltcoinsPlugin partial class)
│  (btcpayserver)     │
└────────┬────────────┘
         │ uses
┌────────▼────────────┐
│     NBXplorer       │  ← Navio chain registration
│  (dgarage/NBXplorer)│
└────────┬────────────┘
         │ uses
┌────────▼────────────┐
│  NBitcoin.Altcoins  │  ← Navio network definition (consensus, addresses, ports)
│ (MetacoSA/NBitcoin) │
└─────────────────────┘
```

BTCPayServer's on-chain wallet infrastructure is shared across all
NBXplorer-based coins. However, Navio requires BLSCT-specific RPC calls for
wallet operations (balance, send, UTXO listing, transaction creation/signing)
instead of the standard Bitcoin wallet RPCs. NBXplorer and NBitcoin's RPC layer
need Navio-specific overrides to route wallet calls through the BLSCT RPC
variants (e.g. `getblsctbalance` instead of `getbalance`, `sendtoblsctaddress`
instead of `sendtoaddress`).

## Navio Testnet Chain Parameters

Source: `src/kernel/chainparams.cpp` (CTestNetParams, line 261)

| Parameter                | Value                                                                | Source Line             |
| ------------------------ | -------------------------------------------------------------------- | ----------------------- |
| CryptoCode               | `NAV`                                                                | —                       |
| Chain Name               | Navio                                                                | —                       |
| P2P Port                 | 33570                                                                | chainparams.cpp:314     |
| RPC Port                 | 33577                                                                | chainparamsbase.cpp:48  |
| Message Start            | `0xb9, 0x3c, 0x0e, 0xdf`                                             | chainparams.cpp:310-313 |
| Genesis Block Hash       | `0x57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b` | chainparams.cpp:321     |
| Genesis Merkle Root      | `0x86054bea52edd87ef7366fd103a66f263557736fd006f1a1942dc50ca40c507f` | chainparams.cpp:322     |
| Subsidy Halving Interval | 210000                                                               | chainparams.cpp:269     |
| PoW Target Spacing       | 600s (10 min)                                                        | chainparams.cpp:285     |
| PoS Target Spacing       | 60s (1 min)                                                          | chainparams.cpp:287     |
| BIP34 Height             | 21111                                                                | chainparams.cpp:272     |
| BIP65 Height             | 581885                                                               | chainparams.cpp:274     |
| BIP66 Height             | 330776                                                               | chainparams.cpp:275     |
| CSV Height               | 770112                                                               | chainparams.cpp:276     |
| Segwit Height            | 834624                                                               | chainparams.cpp:277     |
| BLSCT Enabled            | `true`                                                               | chainparams.cpp:278     |
| Bech32 HRP               | `tb`                                                                 | chainparams.cpp:337     |
| BLSCT bech32_mod_hrp     | `tnv`                                                                | chainparams.cpp:338     |
| Base58 P2PKH             | `0x6f` (111)                                                         | chainparams.cpp:330     |
| Base58 P2SH              | `0xc4` (196)                                                         | chainparams.cpp:332     |
| Base58 Secret Key        | `0xef` (239)                                                         | chainparams.cpp:333     |
| EXT_PUBLIC_KEY           | `0x043587CF`                                                         | chainparams.cpp:334     |
| EXT_SECRET_KEY           | `0x04358394`                                                         | chainparams.cpp:335     |

> Signet and Regtest both have `fBLSCT = false` and use standard Bitcoin genesis
> blocks — they do not represent Navio's consensus and are excluded from this
> integration.

## RPC Compatibility (Verified Present)

Navio uses BLSCT (Boneh-Lynn-Shacham Confidential Transactions) for wallet
operations. All wallet-related RPCs must use the BLSCT variants — standard
Bitcoin wallet RPCs operate on transparent (non-BLSCT) outputs and are **not**
suitable for Navio's privacy layer.

### Blockchain / Network RPCs (standard — same as Bitcoin)

| RPC Method           | File                       | Line |
| -------------------- | -------------------------- | ---- |
| `getblockchaininfo`  | src/rpc/blockchain.cpp     | 1236 |
| `getblock`           | src/rpc/blockchain.cpp     | 662  |
| `getbestblockhash`   | src/rpc/blockchain.cpp     | 242  |
| `getrawtransaction`  | src/rpc/rawtransaction.cpp | 291  |
| `getnetworkinfo`     | src/rpc/net.cpp            | 613  |
| `sendrawtransaction` | src/rpc/mempool.cpp        | 37   |
| `getmempoolinfo`     | src/rpc/mempool.cpp        | 682  |

### Wallet RPCs (BLSCT variants — required for Navio)

| RPC Method                         | Replaces (Bitcoin standard)       | File                      | Line |
| ---------------------------------- | --------------------------------- | ------------------------- | ---- |
| `createwallet` (with `blsct=true`) | `createwallet`                    | src/wallet/rpc/wallet.cpp | 348  |
| `getwalletinfo`                    | — (same, reports `"blsct": true`) | src/wallet/rpc/wallet.cpp | 44   |
| `getblsctbalance`                  | `getbalance`                      | src/blsct/wallet/rpc.cpp  | 349  |
| `sendtoblsctaddress`               | `sendtoaddress`                   | src/blsct/wallet/rpc.cpp  | 535  |
| `listblsctunspent`                 | `listunspent`                     | src/blsct/wallet/rpc.cpp  | 862  |
| `listblscttransactions`            | `listtransactions`                | src/blsct/wallet/rpc.cpp  | 1014 |
| `createblsctrawtransaction`        | `createrawtransaction`            | src/blsct/wallet/rpc.cpp  | 1253 |
| `fundblsctrawtransaction`          | `fundrawtransaction`              | src/blsct/wallet/rpc.cpp  | 1636 |
| `signblsctrawtransaction`          | `signrawtransactionwithwallet`    | src/blsct/wallet/rpc.cpp  | 1867 |
| `decodeblsctrawtransaction`        | `decoderawtransaction`            | src/blsct/wallet/rpc.cpp  | 1908 |

### Additional BLSCT RPCs (available for extended functionality)

| RPC Method                | Description                              | File                      | Line |
| ------------------------- | ---------------------------------------- | ------------------------- | ---- |
| `setblsctseed`            | Set/restore BLSCT wallet seed            | src/blsct/wallet/rpc.cpp  | 1131 |
| `getblsctseed`            | Export BLSCT wallet seed                 | src/wallet/rpc/backup.cpp | 691  |
| `getblsctauditkey`        | Get audit (view) key for watch-only      | src/wallet/rpc/backup.cpp | 721  |
| `createblsctbalanceproof` | Create proof of balance (reserve proof)  | src/blsct/wallet/rpc.cpp  | 1187 |
| `unlockblsctoutpoint`     | Unlock a locked BLSCT outpoint           | src/blsct/wallet/rpc.cpp  | 1821 |
| `getblsctrecoverydata`    | Get recovery data for BLSCT outputs      | src/blsct/wallet/rpc.cpp  | 1991 |
| `generatetoblsctaddress`  | Mine blocks to a BLSCT address (testnet) | src/rpc/mining.cpp        | 280  |

> **Important:** `getnewaddress` works natively with BLSCT wallets — it does NOT
> need a BLSCT variant or remapping. When a wallet is created with `blsct=true`,
> `getnewaddress` auto-defaults to `OutputType::BLSCT` and returns a BLSCT
> bech32m address (e.g., `tnv1...` on testnet). You can also pass
> `address_type="blsct"` explicitly. Source: `src/wallet/rpc/addresses.cpp:47`.

> **Warning:** `listblscttransactions` is currently a **stub** in navio-core —
> the implementation is commented out (`src/blsct/wallet/rpc.cpp:1102-1112`) and
> returns an empty array. Do not rely on it for transaction history.
> `listblsctunspent` works correctly for confirmed UTXOs.

> **Warning:** BLSCT has **no unconfirmed UTXO visibility**.
> `AvailableBlsctCoins()` in navio-core hardcodes skip for `nDepth == 0`
> (`src/wallet/spend.cpp:502-505`). Payments are only visible after 1
> confirmation.

Wallet features confirmed: BLSCT wallets, confidential transactions, sub-address
derivation from seed, balance proofs, atomic swaps.

## Implementation Status

| Step  | Description                                     | Status                        |
| ----- | ----------------------------------------------- | ----------------------------- |
| 0     | Integration repo + submodules                   | **DONE**                      |
| 0b    | libblsct-bindings C# P/Invoke                   | **DONE**                      |
| 1a–1f | NBitcoin: Navio network + RPC remapping         | **DONE**                      |
| 1g    | BLSCT address derivation                        | **DONE** (moved to NBXplorer) |
| 2a–2e | NBXplorer: chain registration + BLSCT overrides | **DONE**                      |
| 2f    | NBXplorer: PSBT skip for Navio                  | **DONE**                      |
| 2g    | NBXplorer: RPC whitelist                        | **DONE**                      |
| 2h    | NBXplorer: `GenerateBlsctAddressesCore`         | **DONE**                      |
| 3a–3c | BTCPayServer: Navio plugin + icon               | **DONE**                      |
| 3d    | BTCPayServer: audit key wallet setup UI         | **DONE**                      |
| 4a–4c | btcpayserver-docker: fragment + CLI             | **DONE**                      |
| —     | `libblsct.so` volume mount in Docker            | **DONE**                      |
| —     | End-to-end testing with Navio testnet           | **TODO**                      |

## Implementation Steps

### Step 0: Create Integration Repo, Fork and Clone All Repositories

**Status: DONE.** Repo created, all submodules added, branches created.

<details>
<summary>Completed setup details (click to expand)</summary>

Create a new repo `mxaddict/btcpay-integration` to house the integration plan
and all work as git submodules. This repo is the single workspace for the entire
integration. It includes:

- **This plan** (`BTCPAY.md`) as the root document, committed first before any
  other work
- **Submodules** for the 4 BTCPay repo forks where code changes happen
- **`navio-core` submodule** (read-only reference) for looking up chain
  parameters when updating testnet details or implementing mainnet later

Use the GitHub CLI (`gh`) for all repo creation and forking.

```bash
# 1. Create the integration repo on GitHub
gh repo create mxaddict/btcpay-integration --public \
  --description "Navio BTCPayServer integration — submodules for all required repos"

# 2. Initialize the repo locally in the current working directory
#    with the plan as the first commit
git init
cp /path/to/BTCPAY.md ./BTCPAY.md   # or the plan is already here
git remote add origin git@github.com:mxaddict/btcpay-integration.git
git add BTCPAY.md
git commit -m "Add BTCPayServer integration plan for Navio"
git push -u origin main
```

```bash
# 3. Fork upstream repos under mxaddict
gh repo fork MetacoSA/NBitcoin --clone=false
gh repo fork dgarage/NBXplorer --clone=false
gh repo fork btcpayserver/btcpayserver --clone=false
gh repo fork btcpayserver/btcpayserver-docker --clone=false

# 4. Add all forks as submodules
git submodule add git@github.com:mxaddict/NBitcoin.git NBitcoin
git submodule add git@github.com:mxaddict/NBXplorer.git NBXplorer
git submodule add git@github.com:mxaddict/btcpayserver.git btcpayserver
git submodule add git@github.com:mxaddict/btcpayserver-docker.git btcpayserver-docker

# 5. Add navio-core as a reference submodule (chain params source of truth)
git submodule add git@github.com:mxaddict/navio-core.git navio-core

# 6. Create navio-support branch in each fork submodule
cd NBitcoin && git checkout -b navio-support && cd ..
cd NBXplorer && git checkout -b navio-support && cd ..
cd btcpayserver && git checkout -b navio-support && cd ..
cd btcpayserver-docker && git checkout -b navio-support && cd ..

# 7. Commit the submodule layout
git add .
git commit -m "Add submodules for BTCPay Navio integration"
git push
```

</details>

The resulting repo structure:

```
btcpay-integration/
├── BTCPAY.md                  ← This integration plan
├── NBitcoin/                  ← mxaddict/NBitcoin (fork of MetacoSA/NBitcoin)
├── NBXplorer/                 ← mxaddict/NBXplorer (fork of dgarage/NBXplorer)
├── btcpayserver/              ← mxaddict/btcpayserver (fork of btcpayserver/btcpayserver)
├── btcpayserver-docker/       ← mxaddict/btcpayserver-docker (fork of btcpayserver/btcpayserver-docker)
├── navio-core/                ← mxaddict/navio-core (chain params reference + external C API source)
└── libblsct-bindings/         ← mxaddict/libblsct-bindings (fork of nav-io/libblsct-bindings)
```

The `navio-core` submodule is read-only reference. When testnet parameters
change or mainnet is ready, update the submodule pointer, read the new values
from `src/kernel/chainparams.cpp`, and propagate them to the 4 fork submodules.

Draft PRs are open in all fork repos — see **PR Submission Order** section and
[PRs.md](PRs.md) for links.

---

### Step 0b: libblsct-bindings — Add C# P/Invoke Layer

**Status: DONE.** PR
[#231](https://github.com/nav-io/libblsct-bindings/pull/231) open at
`nav-io/libblsct-bindings`.

**Repo:** `mxaddict/libblsct-bindings` (fork of `nav-io/libblsct-bindings`)
**Branch:** `csharp-support` **PR target:** `nav-io/libblsct-bindings`

`libblsct-bindings` already has Python, TypeScript, and Rust FFI wrappers over
navio-core's external C API (`src/blsct/external_api/blsct.h`). This step adds a
minimal C# P/Invoke layer covering only what address generation needs.

<details>
<summary>Completed implementation details (click to expand)</summary>

#### 0b-i. Fork and add submodule

Done. Submodule at `libblsct-bindings/` on branch `csharp-support`.

#### 0b-ii. C API functions used

All functions from `navio-core/src/blsct/external_api/blsct.h`:

| C function           | Signature                                                                       | Purpose                                 |
| -------------------- | ------------------------------------------------------------------------------- | --------------------------------------- |
| `gen_sub_addr_id`    | `BlsctSubAddrId* gen_sub_addr_id(int64_t account, uint64_t address)`            | Create sub-address identifier           |
| `derive_sub_address` | `BlsctSubAddr* derive_sub_address(BlsctScalar*, BlsctPubKey*, BlsctSubAddrId*)` | Derive sub-address from view+spend keys |
| `encode_address`     | `BlsctRetVal* encode_address(BlsctSubAddr*, const char* hrp)`                   | Encode as bech32m string                |
| `decode_address`     | `BlsctRetVal* decode_address(const char* addr, const char* hrp)`                | Decode bech32m string                   |
| `free_obj`           | `void free_obj(void*)`                                                          | Free C heap objects                     |

#### 0b-iii. `ffi/csharp/NavioBlsct.csproj`

Targets `net8.0;net10.0;netstandard2.1`. `AllowUnsafeBlocks = true`.
`PackageId = NavioBlsct`, version `0.1.0`. Excludes `tests/**/*.cs`.

#### 0b-iv. `ffi/csharp/Blsct.cs`

Implements `NavioBlsct.Blsct` static unsafe class with:

- P/Invoke declarations for all C API functions
- Managed API: `GenSubAddrId`, `DeriveSubAddress`, `EncodeAddress`,
  `DecodeAddress`, `FreeObj`
- `AddressEncoding` enum (`Bech32 = 0`, `Bech32M = 1`)
- Internal helpers: `EnsureSuccess`, `ReadResultCode`, `ReadValuePtr`,
  `ReadRetValStringAndFreeValue`
- 7 xUnit tests in `ffi/csharp/tests/BlsctTests.cs`

#### 0b-v. Native library distribution

The `libblsct.so` / `blsct.dll` shared library is built from navio-core. In the
Docker deployment, the `naviod` container already has the MCL/blsct shared
libraries. NBXplorer's container must have access to them.

**Docker approach:** Copy `libblsct.so` into the NBXplorer container image
during the btcpayserver-docker build.

**DONE:** `navio.yml` uses a shared named volume `navio_libblsct` mounted in
both `naviod` (`/opt/libblsct`, read-write) and `nbxplorer` (`/opt/libblsct`,
read-only). `LD_LIBRARY_PATH=/opt/libblsct` set in NBXplorer. The `naviod` image
entrypoint copies `libblsct.so` to the shared volume on startup.

Longer term: publish `NavioBlsct` as a NuGet package that includes pre-built
native binaries for Linux x64, macOS arm64, Windows x64.

</details>

---

### Step 1: NBitcoin — Add Navio Network Definition

**Status: DONE.** All sub-steps (1a–1f) implemented. 4 commits on
`mxaddict/NBitcoin:navio-support`. Draft PR
[#1](https://github.com/mxaddict/NBitcoin/pull/1).

**Repo:** `mxaddict/NBitcoin` (fork of `MetacoSA/NBitcoin`) **Branch:**
`navio-support`

<details>
<summary>Completed implementation details (click to expand)</summary>

#### 1a. `NBitcoin.Altcoins/Navio.cs`

Done. Full network definition with real genesis hex for testnet, mainnet, and
regtest. Includes `ConfigureBLSCTOverrides()` static method with 8 RPC
remappings.

#### 1b. Register in `NBitcoin.Altcoins/AltcoinNetworkSets.cs`

Done. `Navio` property registered, yielded in `GetAll()`.

#### 1c. BLSCT RPC operations in `NBitcoin/RPC/RPCOperations.cs`

Done. 16 BLSCT operations added to enum: `getblsctbalance`,
`sendtoblsctaddress`, `listblsctunspent`, `listblscttransactions`,
`createblsctrawtransaction`, `fundblsctrawtransaction`,
`signblsctrawtransaction`, `decodeblsctrawtransaction`, `setblsctseed`,
`getblsctseed`, `getblsctauditkey`, `createblsctbalanceproof`,
`unlockblsctoutpoint`, `getblsctrecoverydata`, `generatetoblsctaddress`.

#### 1d. RPC method name remapping in `NBitcoin/RPC/RPCClient.cs`

Done. `RPCMethodOverrides` dictionary property added. Remapping applied in both
`SendCommandAsync` and `SendCommandWithNamedArgsAsync`.

#### 1e. `blsct` parameter in `CreateWalletOptions`

Done. `public bool? Blsct` property in `CreateWalletOptions.cs`. Parameter
passed in `CreateWalletAsync` in `RPCClient.Wallet.cs`.

#### 1f. `ConfigureBLSCTOverrides` in `Navio.cs`

Done. Static method configures 8 RPC remappings:

| Standard RPC                   | BLSCT RPC                   |
| ------------------------------ | --------------------------- |
| `getbalance`                   | `getblsctbalance`           |
| `sendtoaddress`                | `sendtoblsctaddress`        |
| `listunspent`                  | `listblsctunspent`          |
| `listtransactions`             | `listblscttransactions`     |
| `createrawtransaction`         | `createblsctrawtransaction` |
| `fundrawtransaction`           | `fundblsctrawtransaction`   |
| `signrawtransactionwithwallet` | `signblsctrawtransaction`   |
| `decoderawtransaction`         | `decodeblsctrawtransaction` |

> `createwallet` and `getnewaddress` are NOT remapped. `createwallet` uses the
> `blsct=true` parameter. `getnewaddress` auto-detects `WALLET_FLAG_BLSCT`.

</details>

#### 1g. BLSCT Client-Side Address Derivation

**Status: DONE.** Moved to NBXplorer (§2h). `BlsctDerivationStrategy` in
`NBXplorer.Client/DerivationStrategy/BlsctDerivationStrategy.cs` derives BLSCT
sub-addresses from `(viewKey, spendKey)` via native P/Invoke to the C API.
String format: `blsct:VIEW_KEY_HEX:SPEND_KEY_HEX`. HRP: `tnv` (testnet), `nav`
(mainnet).

---

### Step 2: NBXplorer — Register Navio Chain

**Status: DONE.** All sub-steps (2a–2h) implemented. Commits on
`mxaddict/NBXplorer:navio-support`. Draft PR
[#1](https://github.com/mxaddict/NBXplorer/pull/1).

**Repo:** `mxaddict/NBXplorer` (fork of `dgarage/NBXplorer`) **Branch:**
`navio-support`

<details>
<summary>Completed implementation details for 2a–2d (click to expand)</summary>

#### 2a. `NBXplorer.Client/NBXplorerNetworkProvider.Navio.cs`

Done. Custom `NavioNBXplorerNetwork` subclass overrides
`CreateStrategyFactory()` to return `BlsctDerivationStrategyFactory`.
`InitNavio()` sets `MinRPCVersion = 220000`, coin type `0'` (mainnet) / `1'`
(testnet). `GetNAV()` helper method.

#### 2b. Register in `NBXplorer.Client/NBXplorerNetworkProvider.cs`

Done. `InitNavio(networkType)` call registered.

#### 2c. BLSCT RPC overrides in `RPCClientExtensions.cs`

Done. `EnsureWalletCreated` applies `Navio.ConfigureBLSCTOverrides(client)` and
passes `Blsct = true` for Navio wallet creation.

#### 2d. BLSCT-aware address generation — skip descriptor import

Done. `ImportDescriptorToRPCIfNeeded` returns early for Navio in
`NBXplorer/Backend/Repository.cs`. Both descriptor import and Legacy
`ImportAddressAsync` fallback are skipped — the daemon's `blsct::KeyMan` handles
address tracking via the private view key.

</details>

#### 2h. `GenerateAddressesCore` BLSCT derivation path

**Status: DONE.** `GenerateAddressesCore` in `NBXplorer/Backend/Repository.cs`
detects `BlsctDerivationStrategy` and routes to `GenerateBlsctAddressesCore`,
which derives addresses via native P/Invoke (`BlsctAddressDeriver.Derive()`).
Uses gap-limit logic identical to BIP32 path. Change addresses use account `-1`,
receive uses account `0`. No `importaddress` call — the daemon's `blsct::KeyMan`
handles payment detection via the private view key.

#### 2e. BLSCT-aware UTXO scanning — use `listblsctunspent`, skip `scantxoutset`

**Status: DONE.** `GetScannedItems` in `ScanUTXOSetService.cs` returns null for
Navio, skipping `scantxoutset`. UTXO discovery relies on the daemon wallet's
`listblsctunspent` (via remapped `listunspent`). Confirmed-only — no unconfirmed
BLSCT UTXO visibility (`spend.cpp:502-505`). Payments appear after 1
confirmation (~1 min PoS).

#### 2f. BLSCT transaction creation — use `sendtoblsctaddress`, skip PSBT entirely

**Status: DONE.** Both `CreatePSBT` and `UpdatePSBT` endpoints in
`NBXplorer/Controllers/MainController.PSBT.cs` throw `NBXplorerException` (HTTP
400, code `psbt-not-supported`) for Navio. No BLSCT PSBT support in navio-core.
Sending uses `sendtoblsctaddress` (remapped from `sendtoaddress` via step 1f),
which is a single-call create+fund+sign+broadcast. Hot wallet mode only for
testnet.

#### 2g. Add BLSCT methods to RPC proxy whitelist

**Status: DONE.** 9 BLSCT methods added to `WhitelistedRPCMethods` in
`NBXplorer/Controllers/MainController.cs`: `getblsctbalance`,
`sendtoblsctaddress`, `listblsctunspent`, `listblscttransactions`,
`createblsctrawtransaction`, `fundblsctrawtransaction`,
`signblsctrawtransaction`, `decodeblsctrawtransaction`,
`createblsctbalanceproof`.

---

### Step 3: BTCPayServer — Add Navio Plugin

**Status: DONE.** All sub-steps (3a–3d) implemented. Commits on
`mxaddict/btcpayserver:navio-support`. Draft PR
[#1](https://github.com/mxaddict/btcpayserver/pull/1).

**Repo:** `mxaddict/btcpayserver` (fork of `btcpayserver/btcpayserver`)
**Branch:** `navio-support`

<details>
<summary>Completed implementation details for 3a–3c (click to expand)</summary>

#### 3a. `AltcoinsPlugin.Navio.cs`

Done. Full plugin following Groestlcoin pattern: `CryptoCode = "NAV"`,
`DisplayName = "Navio"`, rate rules via CoinGecko, icon path
`imlegacy/navio.svg`, block explorer links, `SupportRBF = true`,
`SupportPayJoin = false`.

#### 3b. Register in `AltcoinsPlugin.cs`

Done. `if (selectedChains.Contains("NAV")) InitNavio(services);`

#### 3c. Coin icon

Done. SVG icon at `BTCPayServer/wwwroot/imlegacy/navio.svg`.

</details>

#### 3d. BLSCT wallet setup — audit key as derivation strategy

**Status: DONE.** Full UI integration for BLSCT audit key wallet import.

Changes across 7 files:

- **`BTCPayNetwork.cs`** — added `IsBLSCT` property (default `false`)
- **`AltcoinsPlugin.Navio.cs`** — set `IsBLSCT = true`, `VaultSupported = false`
- **`DerivationSchemeViewModel.cs`** — added `IsBLSCT` property
- **`SetupWallet.cshtml`** — hides "Create a new wallet" for BLSCT (wallet
  creation happens on the daemon)
- **`ImportWalletOptions.cshtml`** — hides hardware wallet, file import, QR
  scan, and seed import for BLSCT; shows only "Enter BLSCT audit key" with
  "Recommended" badge
- **`ImportWallet/Xpub.cshtml`** — BLSCT-specific label ("BLSCT audit key"),
  help text with `navio-cli` instructions, placeholder, hides xpub examples
  table
- **`ImportWallet/ConfirmAddresses.cshtml`** — skips address preview table for
  BLSCT (native P/Invoke derivation not available client-side), shows audit key
  confirmation instead
- **`UIStoresController.Onchain.cs`** — sets `IsBLSCT` in `SetupWallet`,
  `ImportWallet`, and `UpdateWallet` actions; `ParseDerivationStrategy` auto-
  converts raw 160-char hex from `getblsctauditkey` to
  `blsct:VIEW_HEX:SPEND_HEX` format; uses reflection to dispatch to
  `BlsctDerivationStrategyFactory.Parse()` (works around `new` vs `override`);
  `ConfirmAddresses` skips address preview for BLSCT

User flow: run `navio-cli -testnet getblsctauditkey` → paste the raw 160-char
hex (or `blsct:VIEW_HEX:SPEND_HEX` format) → BTCPayServer auto-splits into view
key (64 chars) + spend key (96 chars) → stored as `blsct:VIEW_HEX:SPEND_HEX`.
Hot wallet spending uses remapped `sendtoaddress` → `sendtoblsctaddress`.

---

### Step 4: btcpayserver-docker — Docker Fragment

**Status: DONE.** All sub-steps (4a–4c) implemented. 2 commits on
`mxaddict/btcpayserver-docker:navio-support`. Draft PR
[#1](https://github.com/mxaddict/btcpayserver-docker/pull/1).

**Repo:** `mxaddict/btcpayserver-docker` (fork of
`btcpayserver/btcpayserver-docker`) **Branch:** `navio-support`

<details>
<summary>Completed implementation details (click to expand)</summary>

#### 4a. `docker-compose-generator/docker-fragments/navio.yml`

Done. Docker fragment with `naviod` container (`navio/naviod:latest`), RPC port
33577, P2P port 33570, NBXplorer env vars (`NBXPLORER_CHAINS=nav`,
`NBXPLORER_NAVRPCURL`), BTCPayServer env vars (`BTCPAY_CHAINS=nav`), volumes.

#### 4b. `docker-compose-generator/crypto-definitions.json`

Done. Entry added: `"Crypto": "nav"`, `"CryptoFragment": "navio"`, no Lightning
fragments.

#### 4c. `navio-cli.sh`

Done. Wrapper script:
`docker exec btcpayserver_naviod navio-cli -datadir="/data" "$@"`

</details>

**DONE:** `libblsct.so` shared volume added to `navio.yml`. Named volume
`navio_libblsct` is mounted read-write in `naviod` at `/opt/libblsct` and
read-only in `nbxplorer` at `/opt/libblsct`. `LD_LIBRARY_PATH` set in NBXplorer.
The `navio/naviod` image entrypoint must copy `libblsct.so` to `/opt/libblsct/`
on startup so NBXplorer can load it via P/Invoke.

---

## Navio Daemon File Locations

| Item           | Path                          |
| -------------- | ----------------------------- |
| Daemon binary  | `./src/naviod`                |
| CLI binary     | `./src/navio-cli`             |
| Config file    | `~/.navio/navio.conf`         |
| Data directory | `~/.navio/`                   |
| Testnet data   | `~/.navio/testnet5/`          |
| Log file       | `~/.navio/testnet5/debug.log` |

## Building Navio Core

```bash
./autogen.sh
./configure --disable-bench
make -j$(nproc)
```

Binaries appear in `./src/`.

## Local Build Setup (Development)

The submodules form a dependency chain. To build locally against the fork
changes, use **local project references** rather than NuGet packages for the
repos you've changed.

### NBitcoin.Altcoins — builds standalone

```bash
cd NBitcoin
dotnet build NBitcoin.Altcoins/NBitcoin.Altcoins.csproj
```

### NBXplorer — switch to local NBitcoin project refs

`NBXplorer.Client/NBXplorer.Client.csproj` must reference local NBitcoin instead
of NuGet:

```xml
<!-- Replace PackageReference for NBitcoin and NBitcoin.Altcoins with: -->
<ProjectReference Include="..\..\NBitcoin\NBitcoin\NBitcoin.csproj" />
<ProjectReference Include="..\..\NBitcoin\NBitcoin.Altcoins\NBitcoin.Altcoins.csproj" />
```

Then build:

```bash
cd NBXplorer
dotnet build NBXplorer/NBXplorer.csproj -p:AllowMissingPrunePackageData=true
```

> **Note:** Revert `NBXplorer.Client.csproj` back to `PackageReference` before
> submitting the upstream PR. The PR must reference the published NuGet
> versions, not local paths.

### BTCPayServer — builds against published NuGet packages

BTCPayServer targets NBitcoin 9.0.5 (NuGet). Our forks use NBitcoin 10.0.1.
Direct project reference substitution causes version conflicts. BTCPayServer
builds fine with upstream NuGet packages because:

- `AltcoinsPlugin.Navio.cs` uses `GetFromCryptoCode("NAV")` which exists in
  published NBXplorer.Client
- The `GetNAV()` helper in our NBXplorer fork is only needed for
  NBXplorer-internal use

```bash
cd btcpayserver
dotnet build BTCPayServer/BTCPayServer.csproj -p:AllowMissingPrunePackageData=true
```

> **Upstream PR dependency:** BTCPayServer's PR can only be fully validated
> end-to-end once NBitcoin and NBXplorer PRs are merged and published to NuGet.
> The compile check above only verifies BTCPayServer's Navio code compiles
> against the existing published NBXplorer.Client.

## Testing Checklist

### libblsct-bindings C# layer

- [ ] `Blsct.GenSubAddrId(account, index)` returns valid handle
- [ ] `Blsct.DeriveSubAddress(viewKey, spendKey, id)` produces correct 96-byte
      sub-address
- [ ] `Blsct.EncodeAddress(subAddr, "tnv")` produces correct `tnv1…` string
      matching daemon output
- [ ] `Blsct.DecodeAddress("tnv1…", "tnv")` round-trips correctly
- [ ] Native library loads on Linux x64 (Docker target platform)

> 7 xUnit tests exist in `libblsct-bindings/ffi/csharp/tests/BlsctTests.cs`
> (input validation, enum values, null safety). Require native `libblsct.so` for
> full integration tests.

### Foundation (code done, needs E2E validation)

- [ ] NBitcoin: Navio testnet network parses BLSCT `tnv` addresses correctly
- [ ] NBitcoin: RPC method remapping works (`getbalance` → `getblsctbalance`,
      etc.)
- [ ] NBitcoin: `CreateWalletAsync` passes `blsct=true` for Navio
- [x] NBitcoin: `BlsctAddressDeriver.Derive()` produces correct `tnv1…`
      addresses matching daemon output (verify against `keyman_tests.cpp`
      expected values)
- [x] NBitcoin: `BlsctDerivationStrategy` round-trips correctly through
      `ToString()` / `Parse()`

### NBXplorer ↔ Daemon (code done, needs E2E validation)

- [ ] NBXplorer: creates BLSCT wallet on Navio daemon (`createwallet` with
      `blsct=true`)
- [ ] NBXplorer: BLSCT RPC overrides are active for Navio connections
- [ ] NBXplorer: `getblsctbalance` returns correct balance via remapped
      `getbalance`
- [ ] NBXplorer: `listblsctunspent` returns BLSCT UTXOs via remapped
      `listunspent`
- [ ] NBXplorer: `listblscttransactions` returns transaction history (blocked:
      navio-core stub)
- [ ] NBXplorer: indexes Navio testnet blocks and tracks BLSCT transactions
- [x] NBXplorer: descriptor import is correctly skipped for Navio (BLSCT
      seed-based)
- [x] NBXplorer: `GenerateAddressesCore` with `BlsctDerivationStrategy` produces
      correct `tnv1…` addresses
- [x] NBXplorer: `ImportAddressAsync` (Legacy fallback) is skipped for Navio

### BTCPayServer (code done, needs E2E validation)

- [ ] BTCPayServer: Navio appears as available cryptocurrency
- [ ] BTCPayServer: can create store with Navio payment method
- [ ] BTCPayServer: generates valid BLSCT `tnv` testnet addresses
- [ ] BTCPayServer: receives testnet BLSCT payment and confirms invoice
- [ ] BTCPayServer: webhook fires on payment confirmation
- [ ] BTCPayServer: refund flow works with BLSCT transactions

### Transaction Flow

- [ ] `sendtoblsctaddress` works via remapped `sendtoaddress`
- [ ] `fundblsctrawtransaction` + `signblsctrawtransaction` produce valid BLSCT
      transactions
- [ ] `sendrawtransaction` broadcasts BLSCT transactions successfully

### Docker

- [ ] Docker: `naviod` container starts and syncs testnet
- [ ] Docker: NBXplorer connects to naviod and indexes chain
- [ ] Docker: `navio-cli.sh` wrapper works correctly

## Unit & Integration Test Plan

These tests need to be written. Each section lists: test file (which already has
a corresponding `.csproj`), required usings, and specific test cases. Tests are
grouped by what they can run without — pure unit tests first, then tests
requiring `libblsct.so`, then tests requiring a live daemon.

**Key implementation notes:**

- `BlsctDerivationStrategy.Parse()` returns `null` on failure — it does NOT
  throw. All invalid-input tests must assert `Assert.Null(result)`.
- `BlsctDerivationStrategyFactory.Parse()` uses `new` (not `override`). Declare
  the factory variable as `BlsctDerivationStrategyFactory`, not the base class,
  so the correct overload is called.
- `Navio.ConfigureBLSCTOverrides(client)` is a `static` method on
  `NBitcoin.Altcoins.Navio`. It sets `client.RPCMethodOverrides`
  (`Dictionary<string, string>`) in-place. A dummy `RPCClient` (fake URI,
  no live connection) is sufficient to test the dictionary contents.
- `UIStoresController.ParseDerivationStrategy` is `internal static` in
  `BTCPayServer`. It is accessible from `BTCPayServer.Tests` because
  `[assembly: InternalsVisibleTo("BTCPayServer.Tests")]` is already declared
  in `BTCPayServer/Program.cs`.
- Test vectors cannot be static hex strings derived from `keyman_tests.cpp`
  because that file does not exist. The C++ external API tests
  (`navio-core/src/test/blsct/external_api/external_api_tests.cpp`) use
  `gen_scalar(11)` and `gen_scalar(12)` as view and spend keys — deterministic
  but only computable by running the native library. **Approach:** write a
  one-time fixture generator (see Test Vectors section) that runs against the
  built `libblsct.so`, captures outputs, and stores them as JSON. Tests then
  load that JSON. Round-trip and differentiation tests do not need pre-computed
  vectors.

---

### NBitcoin — Unit Tests

**File:** `NBitcoin/NBitcoin.Tests/NavioTests.cs`  
**Framework:** xUnit (project: `NBitcoin/NBitcoin.Tests/NBitcoin.Tests.csproj`)

```csharp
// Required usings:
using NBitcoin;
using NBitcoin.Altcoins;
using NBitcoin.DataEncoders;
using NBitcoin.RPC;
using System.Collections.Generic;
using Xunit;
```

#### Navio Network Definition

```csharp
[Fact]
NavioNetwork_Testnet_HasCorrectGenesisHash()
// var network = AltNetworkSets.Navio.Testnet;  // NBitcoin.Altcoins.AltNetworkSets
// Assert.Equal(
//   uint256.Parse("57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b"),
//   network.GetGenesis().GetHash());

[Fact]
NavioNetwork_Testnet_HasCorrectPorts()
// var network = AltNetworkSets.Navio.Testnet;
// Assert.Equal(33570, network.DefaultPort);
// Assert.Equal(33577, network.RPCPort);

[Fact]
NavioNetwork_Testnet_HasCorrectBlsctBech32HRP()
// var network = AltNetworkSets.Navio.Testnet;
// Find the Bech32Encoder for the BLSCT HRP "tnv" among network.Bech32Encoders
// Assert it exists and its HRP == "tnv"

[Fact]
NavioNetwork_Testnet_HasCorrectBase58Prefixes()
// var network = AltNetworkSets.Navio.Testnet;
// Assert.Equal(0x6f, network.GetVersionBytes(Base58Type.PUBKEY_ADDRESS, false)[0]);
// Assert.Equal(0xc4, network.GetVersionBytes(Base58Type.SCRIPT_ADDRESS, false)[0]);

[Fact]
NavioNetwork_RegisteredInAltNetworkSets()
// Assert.NotNull(AltNetworkSets.Navio);
// Assert.Contains(AltNetworkSets.Navio, AltNetworkSets.GetAll());
```

#### RPC Method Remapping

```csharp
// Helper: create a dummy RPCClient (no live connection needed)
// var client = new RPCClient(
//     new RPCCredentialString { UserPassword = new NetworkCredential("u","p") },
//     new Uri("http://127.0.0.1:33577/"),
//     AltNetworkSets.Navio.Testnet);

[Fact]
ConfigureBLSCTOverrides_SetsRPCMethodOverridesDictionary()
// NBitcoin.Altcoins.Navio.ConfigureBLSCTOverrides(client);
// Assert.NotNull(client.RPCMethodOverrides);
// Assert.IsType<Dictionary<string, string>>(client.RPCMethodOverrides);

[Fact]
ConfigureBLSCTOverrides_RemapsAllEightMethods()
// NBitcoin.Altcoins.Navio.ConfigureBLSCTOverrides(client);
// var m = client.RPCMethodOverrides;
// Assert.Equal("getblsctbalance",           m["getbalance"]);
// Assert.Equal("sendtoblsctaddress",        m["sendtoaddress"]);
// Assert.Equal("listblsctunspent",          m["listunspent"]);
// Assert.Equal("listblscttransactions",     m["listtransactions"]);
// Assert.Equal("createblsctrawtransaction", m["createrawtransaction"]);
// Assert.Equal("fundblsctrawtransaction",   m["fundrawtransaction"]);
// Assert.Equal("signblsctrawtransaction",   m["signrawtransactionwithwallet"]);
// Assert.Equal("decodeblsctrawtransaction", m["decoderawtransaction"]);

[Fact]
ConfigureBLSCTOverrides_DoesNotRemapCreatewallet()
// NBitcoin.Altcoins.Navio.ConfigureBLSCTOverrides(client);
// Assert.False(client.RPCMethodOverrides.ContainsKey("createwallet"));

[Fact]
ConfigureBLSCTOverrides_DoesNotRemapGetnewaddress()
// NBitcoin.Altcoins.Navio.ConfigureBLSCTOverrides(client);
// Assert.False(client.RPCMethodOverrides.ContainsKey("getnewaddress"));

[Fact]
ConfigureBLSCTOverrides_ExactlyEightMappings()
// NBitcoin.Altcoins.Navio.ConfigureBLSCTOverrides(client);
// Assert.Equal(8, client.RPCMethodOverrides.Count);
```

#### CreateWalletOptions.Blsct

```csharp
// Required using: using NBitcoin.RPC;

[Fact]
CreateWalletOptions_BlsctProperty_DefaultsToNull()
// Assert.Null(new CreateWalletOptions().Blsct);

[Fact]
CreateWalletOptions_BlsctTrue_IncludedInSerialization()
// Serialize CreateWalletOptions { Blsct = true } via NBitcoin's JSON serializer
// Assert JSON contains the key "blsct" with value true
// (Inspect RPCClient.CreateWalletAsync source for serialization path)

[Fact]
CreateWalletOptions_BlsctNull_OmittedFromSerialization()
// Serialize CreateWalletOptions { Blsct = null }
// Assert JSON does NOT contain "blsct" key
```

#### BLSCT RPCOperations Enum

```csharp
// Required using: using NBitcoin.RPC;

[Theory]
[InlineData(RPCOperations.getblsctbalance,           "getblsctbalance")]
[InlineData(RPCOperations.sendtoblsctaddress,        "sendtoblsctaddress")]
[InlineData(RPCOperations.listblsctunspent,          "listblsctunspent")]
[InlineData(RPCOperations.listblscttransactions,     "listblscttransactions")]
[InlineData(RPCOperations.createblsctrawtransaction, "createblsctrawtransaction")]
[InlineData(RPCOperations.fundblsctrawtransaction,   "fundblsctrawtransaction")]
[InlineData(RPCOperations.signblsctrawtransaction,   "signblsctrawtransaction")]
[InlineData(RPCOperations.decodeblsctrawtransaction, "decodeblsctrawtransaction")]
[InlineData(RPCOperations.setblsctseed,              "setblsctseed")]
[InlineData(RPCOperations.getblsctseed,              "getblsctseed")]
[InlineData(RPCOperations.getblsctauditkey,          "getblsctauditkey")]
[InlineData(RPCOperations.createblsctbalanceproof,   "createblsctbalanceproof")]
[InlineData(RPCOperations.unlockblsctoutpoint,       "unlockblsctoutpoint")]
[InlineData(RPCOperations.getblsctrecoverydata,      "getblsctrecoverydata")]
[InlineData(RPCOperations.generatetoblsctaddress,    "generatetoblsctaddress")]
RPCOperations_BlsctEnums_HaveCorrectStringValues(RPCOperations op, string expected)
// Assert.Equal(expected, op.ToString());
```

---

### NBXplorer — Unit Tests

**File:** `NBXplorer/NBXplorer.Tests/NavioTests.cs`  
**Framework:** xUnit (project: `NBXplorer/NBXplorer.Tests/NBXplorer.Tests.csproj`)

```csharp
// Required usings:
using NBitcoin;
using NBitcoin.Altcoins;
using NBXplorer;
using NBXplorer.DerivationStrategy;
using Xunit;
```

#### BlsctDerivationStrategy — Parse and Round-trip

> **Important:** `BlsctDerivationStrategy.Parse()` returns `null` on invalid
> input — it does NOT throw. Use `Assert.Null(result)` for failure cases.

```csharp
// Test constants (put in a shared helper):
// const string ValidView  = new string('a', 64);  // 64 hex chars = 32 bytes
// const string ValidSpend = new string('b', 96);  // 96 hex chars = 48 bytes
// const string ValidStr   = $"blsct:{ValidView}:{ValidSpend}";

[Fact]
BlsctDerivationStrategy_Parse_ValidString_ReturnsStrategy()
// var s = BlsctDerivationStrategy.Parse(ValidStr);
// Assert.NotNull(s);
// Assert.Equal(Encoders.Hex.DecodeData(ValidView),  s.ViewKey);
// Assert.Equal(Encoders.Hex.DecodeData(ValidSpend), s.SpendKey);

[Fact]
BlsctDerivationStrategy_Parse_MissingPrefix_ReturnsNull()
// Assert.Null(BlsctDerivationStrategy.Parse($"{ValidView}:{ValidSpend}"));

[Fact]
BlsctDerivationStrategy_Parse_WrongViewKeyLength_ReturnsNull()
// Assert.Null(BlsctDerivationStrategy.Parse($"blsct:{new string('a', 62)}:{ValidSpend}"));

[Fact]
BlsctDerivationStrategy_Parse_WrongSpendKeyLength_ReturnsNull()
// Assert.Null(BlsctDerivationStrategy.Parse($"blsct:{ValidView}:{new string('b', 94)}"));

[Fact]
BlsctDerivationStrategy_Parse_NonHexViewKey_ReturnsNull()
// Assert.Null(BlsctDerivationStrategy.Parse($"blsct:{new string('Z', 64)}:{ValidSpend}"));

[Fact]
BlsctDerivationStrategy_Parse_NullInput_ReturnsNull()
// Assert.Null(BlsctDerivationStrategy.Parse(null));

[Fact]
BlsctDerivationStrategy_ToString_RoundTrips()
// var s = BlsctDerivationStrategy.Parse(ValidStr);
// Assert.Equal(ValidStr, s.ToString());  // exact case preservation

[Fact]
BlsctDerivationStrategy_ToString_Parse_RoundTrip()
// var s1 = BlsctDerivationStrategy.Parse(ValidStr);
// var s2 = BlsctDerivationStrategy.Parse(s1.ToString());
// Assert.Equal(s1.ViewKey,  s2.ViewKey);
// Assert.Equal(s1.SpendKey, s2.SpendKey);
```

#### BlsctDerivationStrategyFactory

> **Important:** `BlsctDerivationStrategyFactory.Parse()` uses `new`, not
> `override`. Declare factory as `BlsctDerivationStrategyFactory` (not the
> base `DerivationStrategyFactory`) so the correct overload resolves.

```csharp
[Fact]
BlsctDerivationStrategyFactory_Parse_BlsctString_ReturnsBlsctStrategy()
// var factory = new BlsctDerivationStrategyFactory(AltNetworkSets.Navio.Testnet);
// var result = factory.Parse(ValidStr);
// Assert.IsType<BlsctDerivationStrategy>(result);

[Fact]
BlsctDerivationStrategyFactory_Parse_BlsctString_KeysMatchInput()
// var factory = new BlsctDerivationStrategyFactory(AltNetworkSets.Navio.Testnet);
// var result = (BlsctDerivationStrategy)factory.Parse(ValidStr);
// Assert.Equal(Encoders.Hex.DecodeData(ValidView),  result.ViewKey);
// Assert.Equal(Encoders.Hex.DecodeData(ValidSpend), result.SpendKey);

[Fact]
BlsctDerivationStrategyFactory_Parse_NonBlsctString_FallsBackToBase()
// var factory = new BlsctDerivationStrategyFactory(AltNetworkSets.Navio.Testnet);
// var result = factory.Parse("xpub...");  // use a real xpub for testnet
// Assert.IsNotType<BlsctDerivationStrategy>(result);
```

#### NavioNBXplorerNetwork

```csharp
// Required using: using NBXplorer.Models;  // for NetworkType

[Fact]
NavioNBXplorerNetwork_Testnet_HasCorrectCryptoCode()
// var provider = new NBXplorerNetworkProvider(NetworkType.Testnet);
// Assert.Equal("NAV", provider.GetFromCryptoCode("NAV").CryptoCode);

[Fact]
NavioNBXplorerNetwork_Testnet_HasCorrectMinRPCVersion()
// var network = new NBXplorerNetworkProvider(NetworkType.Testnet).GetFromCryptoCode("NAV");
// Assert.Equal(220000, network.MinRPCVersion);

[Fact]
NavioNBXplorerNetwork_Testnet_CreateStrategyFactory_ReturnsBlsctFactory()
// var network = new NBXplorerNetworkProvider(NetworkType.Testnet).GetFromCryptoCode("NAV");
// Assert.IsType<BlsctDerivationStrategyFactory>(network.DerivationStrategyFactory);

[Fact]
NavioNBXplorerNetwork_Mainnet_CoinTypeIs0()
// var network = new NBXplorerNetworkProvider(NetworkType.Mainnet).GetFromCryptoCode("NAV");
// Assert.Equal(0u, network.CoinType);

[Fact]
NavioNBXplorerNetwork_Testnet_CoinTypeIs1()
// var network = new NBXplorerNetworkProvider(NetworkType.Testnet).GetFromCryptoCode("NAV");
// Assert.Equal(1u, network.CoinType);
```

---

### NBXplorer — Integration Tests (require `libblsct.so`)

**File:** `NBXplorer/NBXplorer.Tests/BlsctDerivationTests.cs`  
**Framework:** xUnit  
**Skip guard:** Check `File.Exists(Environment.GetEnvironmentVariable("LIBBLSCT_SO_PATH") ?? "")`
at the start of each test; skip via `Skip.If(...)` (using the xunit.v3.skip
package) or `[Fact(Skip = "requires libblsct.so")]` when the env var is absent.

The method under test is the static method on `BlsctDerivationStrategy`:
```csharp
// BlsctDerivationStrategy.DeriveBlsctAddress(
//     byte[] viewKey, byte[] spendKey, long account, ulong index, string hrp)
// → string  (bech32m-encoded address)
```

```csharp
[Fact]
DeriveBlsctAddress_IsDeterministic()
// Same inputs twice → same output
// var addr1 = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "tnv");
// var addr2 = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "tnv");
// Assert.Equal(addr1, addr2);
// (Use any 32-byte viewKey and 48-byte spendKey — random bytes are fine here)

[Fact]
DeriveBlsctAddress_Testnet_StartsWithTnv1()
// var addr = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "tnv");
// Assert.StartsWith("tnv1", addr);

[Fact]
DeriveBlsctAddress_Mainnet_StartsWithNav1()
// var addr = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "nav");
// Assert.StartsWith("nav1", addr);

[Fact]
DeriveBlsctAddress_DifferentAccount_ProducesDifferentAddress()
// var a0 = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "tnv");
// var a1 = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 1, 0, "tnv");
// Assert.NotEqual(a0, a1);

[Fact]
DeriveBlsctAddress_DifferentIndex_ProducesDifferentAddress()
// var a0 = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "tnv");
// var a1 = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 1, "tnv");
// Assert.NotEqual(a0, a1);

[Fact]
DeriveBlsctAddress_ChangeAccount_NegativeOne_Differs()
// long changeAccount = BlsctDerivationStrategy.ChangeAccount;  // = -1
// var receive = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, 0, 0, "tnv");
// var change  = BlsctDerivationStrategy.DeriveBlsctAddress(viewKey, spendKey, changeAccount, 0, "tnv");
// Assert.NotEqual(receive, change);

[Fact]
DeriveBlsctAddress_MatchesFixture()
// Load blsct_vectors.json (generated by fixture tool — see Test Vectors section)
// For each vector: Assert.Equal(vector.ExpectedAddress,
//   BlsctDerivationStrategy.DeriveBlsctAddress(
//     Encoders.Hex.DecodeData(vector.ViewKey),
//     Encoders.Hex.DecodeData(vector.SpendKey),
//     vector.Account, (ulong)vector.Index, vector.Hrp));
```

---

### NBXplorer — Integration Tests (require live Navio daemon)

**File:** `NBXplorer/NBXplorer.Tests/NavioDaemonTests.cs`  
**Framework:** xUnit  
**Skip guard:** Custom `NavioDaemonFactAttribute` — skips unless env vars
`NBXPLORER_NAVTESTNET_RPCURL`, `NBXPLORER_NAVTESTNET_RPCUSER`,
`NBXPLORER_NAVTESTNET_RPCPASSWORD` are all set.

```csharp
[NavioDaemonFact]
EnsureWalletCreated_ForNavio_CreatesBlsctWallet()
// Build RPCClient from env vars with Navio.ConfigureBLSCTOverrides applied
// Call EnsureWalletCreated (or createwallet with Blsct=true directly)
// Call getwalletinfo → Assert result["blsct"].Value<bool>() == true

[NavioDaemonFact]
GetBalance_RoutesTo_Getblsctbalance()
// After wallet creation, call client.SendCommand("getbalance")
// Assert: no RPCException (confirms remapping routes to getblsctbalance)

[NavioDaemonFact]
Listunspent_RoutesTo_Listblsctunspent()
// client.SendCommand("listunspent")
// Assert: result is JArray (may be empty), no RPCException

[NavioDaemonFact]
NavioChain_IndexesBlocks()
// Start NBXplorer with Navio testnet config
// Poll GetStatusAsync() until Navio chain Height > 0 (timeout 30s)
// Assert: status.ChainHeight > 0
```

---

### BTCPayServer — Unit Tests

**File:** `btcpayserver/BTCPayServer.Tests/NavioPluginTests.cs`  
**Framework:** xUnit (project: `btcpayserver/BTCPayServer.Tests/BTCPayServer.Tests.csproj`)

```csharp
// Required usings:
using BTCPayServer;
using BTCPayServer.Controllers;
using BTCPayServer.Models.StoreViewModels;
using Xunit;
```

#### IsBLSCT Property

```csharp
// Resolve BTCPayNetwork via BTCPayNetworkProvider (use DI or instantiate directly)

[Fact]
NavioBTCPayNetwork_IsBLSCT_IsTrue()
// var provider = CreateTestNetworkProvider();  // helper that registers Navio
// var network = provider.GetNetwork<BTCPayNetwork>("NAV");
// Assert.True(network.IsBLSCT);

[Fact]
BitcoinBTCPayNetwork_IsBLSCT_IsFalse()
// Assert.False(provider.GetNetwork<BTCPayNetwork>("BTC").IsBLSCT);

[Fact]
DerivationSchemeViewModel_IsBLSCT_DefaultsFalse()
// Assert.False(new DerivationSchemeViewModel().IsBLSCT);
```

#### UIStoresController — Audit Key Parsing

> `ParseDerivationStrategy` is `internal static` in `UIStoresController`.
> Accessible from `BTCPayServer.Tests` via `InternalsVisibleTo` already set
> in `BTCPayServer/Program.cs`. Call as:
> `UIStoresController.ParseDerivationStrategy(input, navioNetwork)`

```csharp
[Fact]
ParseDerivationStrategy_RawAuditKey_160HexChars_ConvertsToBlsctFormat()
// string viewHex  = new string('a', 64);
// string spendHex = new string('b', 96);
// string raw = viewHex + spendHex;  // 160 chars
// var result = UIStoresController.ParseDerivationStrategy(raw, navioNetwork);
// Assert.Equal($"blsct:{viewHex}:{spendHex}", result.AccountDerivation.ToString());

[Fact]
ParseDerivationStrategy_AlreadyBlsctFormat_PassesThrough()
// string input = $"blsct:{new string('a', 64)}:{new string('b', 96)}";
// var result = UIStoresController.ParseDerivationStrategy(input, navioNetwork);
// Assert.Equal(input, result.AccountDerivation.ToString());

[Fact]
ParseDerivationStrategy_WrongLength_IsHandled()
// 100-char hex is neither 160 chars nor "blsct:..." format
// Assert an appropriate error or null result — check what the controller
// returns for non-BLSCT-format input on a BLSCT network

[Fact]
SetupWallet_ForNavio_SetsIsBLSCT_True()
// Use TestServer / WebApplicationFactory to call GET /stores/{id}/onchain/NAV
// Assert: viewModel.IsBLSCT == true in the rendered SetupWallet view model

[Fact]
ImportWallet_ForNavio_SetsIsBLSCT_True()
// GET /stores/{id}/onchain/NAV/import
// Assert: viewModel.IsBLSCT == true
```

---

### libblsct-bindings — Integration Tests (require `libblsct.so`)

**File:** `libblsct-bindings/ffi/csharp/tests/BlsctIntegrationTests.cs`  
**Framework:** xUnit (project already exists alongside `BlsctTests.cs`)  
**Skip guard:** Same `LIBBLSCT_SO_PATH` env var pattern as NBXplorer tests above.

```csharp
// Required usings: using NavioBlsct; using Xunit;

[Fact]
GenSubAddrId_ReturnsNonZeroHandle()
// var handle = Blsct.GenSubAddrId(0, 0);
// Assert.NotEqual(IntPtr.Zero, handle);
// Blsct.FreeObj(handle);

[Fact]
DeriveSubAddress_ValidKeys_ReturnsNonZeroHandle()
// (Use random 32-byte viewKey, 48-byte spendKey — sizes matter, values don't
//  for this test since we're checking the call doesn't crash / returns non-null)
// var id   = Blsct.GenSubAddrId(0, 0);
// var addr = Blsct.DeriveSubAddress(viewKey, spendKey, id);
// Assert.NotEqual(IntPtr.Zero, addr);
// Blsct.FreeObj(addr); Blsct.FreeObj(id);

[Fact]
EncodeAddress_Testnet_StartsWithTnv1()
// Derive sub-address then encode with hrp = "tnv"
// Assert.StartsWith("tnv1", encoded);

[Fact]
EncodeAddress_Mainnet_StartsWithNav1()
// Encode with hrp = "nav"
// Assert.StartsWith("nav1", encoded);

[Fact]
DecodeAddress_RoundTrips()
// encode then decode — decoded bytes must equal the original sub-address bytes

[Fact]
FullPipeline_MatchesFixture()
// Load blsct_vectors.json (same file used by NBXplorer integration tests)
// For each vector: encode via C# P/Invoke → Assert.Equal(vector.ExpectedAddress)
```

---

### Test Vectors

`keyman_tests.cpp` does not exist in navio-core. The C++ test reference is
`navio-core/src/test/blsct/external_api/external_api_tests.cpp`, which uses
`gen_scalar(11)` (view) and `gen_scalar(12)` (spend) — deterministic from
integer seeds but only computable by running the native library.

**Approach — fixture generator script:**

Write a small C# console app (or xUnit fixture class) that:
1. Calls `BlsctDerivationStrategy.DeriveBlsctAddress` for the test inputs below
2. Writes results to JSON

Run once after building `libblsct.so`, commit the JSON output.

```csharp
// Inputs to generate vectors for:
// viewKey  = bytes from gen_scalar(11) — run external_api_tests to get hex, OR
//            use a known 32-byte seed: 0x0b repeated 32 times as a starting point
// spendKey = bytes from gen_scalar(12) — OR 0x0c repeated 48 times
// Derive: (account=0, index=0, hrp="tnv")
//         (account=0, index=1, hrp="tnv")
//         (account=-1, index=0, hrp="tnv")  // change address
//         (account=0, index=0, hrp="nav")   // mainnet
```

Store JSON at:
- `NBXplorer/NBXplorer.Tests/TestData/blsct_vectors.json`
- `libblsct-bindings/ffi/csharp/tests/TestData/blsct_vectors.json`

```json
[
  { "viewKey": "<64 hex>", "spendKey": "<96 hex>",
    "account": 0, "index": 0, "hrp": "tnv",
    "expectedAddress": "tnv1..." },
  ...
]
```

Until the fixture is generated, the `MatchesFixture` tests should be marked
`[Fact(Skip = "fixture not yet generated")]` and the round-trip / prefix tests
used as the primary coverage.

---

### Test Execution Matrix

| Test Suite | Needs `libblsct.so` | Needs Daemon | Run in CI |
|------------|---------------------|--------------|-----------|
| NBitcoin NavioTests | No | No | Yes |
| NBXplorer NavioTests (unit) | No | No | Yes |
| NBXplorer BlsctDerivationTests | Yes | No | Conditional (`LIBBLSCT_SO_PATH` set) |
| NBXplorer NavioDaemonTests | Yes | Yes | No (manual) |
| BTCPayServer NavioPluginTests | No | No | Yes |
| libblsct-bindings BlsctTests (existing) | No | No | Yes |
| libblsct-bindings BlsctIntegrationTests | Yes | No | Conditional (`LIBBLSCT_SO_PATH` set) |

CI runs all "No" rows unconditionally. "Conditional" rows run only when
`LIBBLSCT_SO_PATH` env var points to a valid `libblsct.so`. "Manual" rows
are for pre-PR validation against a live testnet node.

---

## PR Submission Order

> **Draft PRs open in mxaddict forks.** See [PRs.md](PRs.md) for links. Upstream
> PRs to `MetacoSA/NBitcoin`, `dgarage/NBXplorer`, etc. have **not** been opened
> yet — waiting for E2E testing and remaining TODO items.

| #   | Repo                | Fork PR                                                                                  | Status | Upstream PR                      |
| --- | ------------------- | ---------------------------------------------------------------------------------------- | ------ | -------------------------------- |
| 0   | libblsct-bindings   | [nav-io#231](https://github.com/nav-io/libblsct-bindings/pull/231)                       | Open   | Same (targets upstream directly) |
| 1   | NBitcoin            | [mxaddict/NBitcoin#1](https://github.com/mxaddict/NBitcoin/pull/1)                       | Draft  | Not yet                          |
| 2   | NBXplorer           | [mxaddict/NBXplorer#1](https://github.com/mxaddict/NBXplorer/pull/1)                     | Draft  | Not yet                          |
| 3   | BTCPayServer        | [mxaddict/btcpayserver#1](https://github.com/mxaddict/btcpayserver/pull/1)               | Draft  | Not yet                          |
| 4   | btcpayserver-docker | [mxaddict/btcpayserver-docker#1](https://github.com/mxaddict/btcpayserver-docker/pull/1) | Draft  | Not yet                          |

**Merge order:** #0 (independent) → #1 → #2 → #3 (sequential dependency). #4 is
independent and can go in parallel with #1–#3.

## Future Work (Mainnet)

- Finalize mainnet chainparams (`fBLSCT` activation, genesis block)
- Register a BIP44 coin type via
  [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)
- Update NBitcoin mainnet genesis hex
- Add Navio to CoinGecko/exchange rate providers for `DefaultRateRules`
- Set up block explorer and update `blockExplorerLink`
- Build and publish Docker image for `navio/naviod`
- Add Lightning support if applicable

## References

- [NBitcoin.Altcoins pattern](https://github.com/MetacoSA/NBitcoin/tree/master/NBitcoin.Altcoins)
  — Litecoin.cs, Groestlcoin.cs
- [NBXplorer chain registration](https://github.com/dgarage/NBXplorer/tree/master/NBXplorer.Client)
  — NBXplorerNetworkProvider.\*.cs
- [BTCPayServer AltcoinsPlugin](https://github.com/btcpayserver/btcpayserver/tree/master/BTCPayServer/Plugins/Altcoins)
  — AltcoinsPlugin.Groestlcoin.cs
- [BTCPayServer Docker fragments](https://github.com/btcpayserver/btcpayserver-docker/tree/master/docker-compose-generator/docker-fragments)
- [Navio Chain Parameters](navio-core/src/kernel/chainparams.cpp)
- [Navio RPC Interface](navio-core/doc/JSON-RPC-interface.md)
- [Navio BLSCT Wallet RPCs](navio-core/src/blsct/wallet/rpc.cpp) — all BLSCT
  wallet RPC implementations
