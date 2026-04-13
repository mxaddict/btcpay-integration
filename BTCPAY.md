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
