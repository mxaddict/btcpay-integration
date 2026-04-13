# BTCPayServer Integration Plan for Navio

## Overview

Add Navio (a Bitcoin Core fork with BLSCT) as a supported cryptocurrency in BTCPayServer. Navio uses Bitcoin-compatible blockchain/network RPCs but requires **BLSCT-specific wallet RPCs** for address generation, balance queries, UTXO listing, and transaction creation/signing. It follows the **NBXplorer path** (same as Litecoin, Groestlcoin, Dogecoin, etc.) but with BLSCT wallet RPC overrides. This requires changes across 4 repositories, each submitted as a separate PR.

**Scope: Testnet only.** Mainnet chainparams are not finalized (`fBLSCT = false` on mainnet currently). Testnet has `fBLSCT = true` and a confirmed genesis hash. Mainnet integration will follow once mainnet chainparams are complete.

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

BTCPayServer's on-chain wallet infrastructure is shared across all NBXplorer-based coins. However, Navio requires BLSCT-specific RPC calls for wallet operations (balance, send, UTXO listing, transaction creation/signing) instead of the standard Bitcoin wallet RPCs. NBXplorer and NBitcoin's RPC layer need Navio-specific overrides to route wallet calls through the BLSCT RPC variants (e.g. `getblsctbalance` instead of `getbalance`, `sendtoblsctaddress` instead of `sendtoaddress`).

## Navio Testnet Chain Parameters

Source: `src/kernel/chainparams.cpp` (CTestNetParams, line 261)

| Parameter | Value | Source Line |
|-----------|-------|-------------|
| CryptoCode | `NAV` | — |
| Chain Name | Navio | — |
| P2P Port | 33570 | chainparams.cpp:314 |
| RPC Port | 33577 | chainparamsbase.cpp:48 |
| Message Start | `0xb9, 0x3c, 0x0e, 0xdf` | chainparams.cpp:310-313 |
| Genesis Block Hash | `0x57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b` | chainparams.cpp:321 |
| Genesis Merkle Root | `0x86054bea52edd87ef7366fd103a66f263557736fd006f1a1942dc50ca40c507f` | chainparams.cpp:322 |
| Subsidy Halving Interval | 210000 | chainparams.cpp:269 |
| PoW Target Spacing | 600s (10 min) | chainparams.cpp:285 |
| PoS Target Spacing | 60s (1 min) | chainparams.cpp:287 |
| BIP34 Height | 21111 | chainparams.cpp:272 |
| BIP65 Height | 581885 | chainparams.cpp:274 |
| BIP66 Height | 330776 | chainparams.cpp:275 |
| CSV Height | 770112 | chainparams.cpp:276 |
| Segwit Height | 834624 | chainparams.cpp:277 |
| BLSCT Enabled | `true` | chainparams.cpp:278 |
| Bech32 HRP | `tb` | chainparams.cpp:337 |
| BLSCT bech32_mod_hrp | `tnv` | chainparams.cpp:338 |
| Base58 P2PKH | `0x6f` (111) | chainparams.cpp:330 |
| Base58 P2SH | `0xc4` (196) | chainparams.cpp:332 |
| Base58 Secret Key | `0xef` (239) | chainparams.cpp:333 |
| EXT_PUBLIC_KEY | `0x043587CF` | chainparams.cpp:334 |
| EXT_SECRET_KEY | `0x04358394` | chainparams.cpp:335 |

> Signet and Regtest both have `fBLSCT = false` and use standard Bitcoin genesis blocks — they do not represent Navio's consensus and are excluded from this integration.

## RPC Compatibility (Verified Present)

Navio uses BLSCT (Boneh-Lynn-Shacham Confidential Transactions) for wallet operations. All wallet-related RPCs must use the BLSCT variants — standard Bitcoin wallet RPCs operate on transparent (non-BLSCT) outputs and are **not** suitable for Navio's privacy layer.

### Blockchain / Network RPCs (standard — same as Bitcoin)

| RPC Method | File | Line |
|------------|------|------|
| `getblockchaininfo` | src/rpc/blockchain.cpp | 1236 |
| `getblock` | src/rpc/blockchain.cpp | 662 |
| `getbestblockhash` | src/rpc/blockchain.cpp | 242 |
| `getrawtransaction` | src/rpc/rawtransaction.cpp | 291 |
| `getnetworkinfo` | src/rpc/net.cpp | 613 |
| `sendrawtransaction` | src/rpc/mempool.cpp | 37 |
| `getmempoolinfo` | src/rpc/mempool.cpp | 682 |

### Wallet RPCs (BLSCT variants — required for Navio)

| RPC Method | Replaces (Bitcoin standard) | File | Line |
|------------|---------------------------|------|------|
| `createwallet` (with `blsct=true`) | `createwallet` | src/wallet/rpc/wallet.cpp | 348 |
| `getwalletinfo` | — (same, reports `"blsct": true`) | src/wallet/rpc/wallet.cpp | 44 |
| `getblsctbalance` | `getbalance` | src/blsct/wallet/rpc.cpp | 349 |
| `sendtoblsctaddress` | `sendtoaddress` | src/blsct/wallet/rpc.cpp | 535 |
| `listblsctunspent` | `listunspent` | src/blsct/wallet/rpc.cpp | 862 |
| `listblscttransactions` | `listtransactions` | src/blsct/wallet/rpc.cpp | 1014 |
| `createblsctrawtransaction` | `createrawtransaction` | src/blsct/wallet/rpc.cpp | 1253 |
| `fundblsctrawtransaction` | `fundrawtransaction` | src/blsct/wallet/rpc.cpp | 1636 |
| `signblsctrawtransaction` | `signrawtransactionwithwallet` | src/blsct/wallet/rpc.cpp | 1867 |
| `decodeblsctrawtransaction` | `decoderawtransaction` | src/blsct/wallet/rpc.cpp | 1908 |

### Additional BLSCT RPCs (available for extended functionality)

| RPC Method | Description | File | Line |
|------------|-------------|------|------|
| `setblsctseed` | Set/restore BLSCT wallet seed | src/blsct/wallet/rpc.cpp | 1131 |
| `getblsctseed` | Export BLSCT wallet seed | src/wallet/rpc/backup.cpp | 691 |
| `getblsctauditkey` | Get audit (view) key for watch-only | src/wallet/rpc/backup.cpp | 721 |
| `createblsctbalanceproof` | Create proof of balance (reserve proof) | src/blsct/wallet/rpc.cpp | 1187 |
| `unlockblsctoutpoint` | Unlock a locked BLSCT outpoint | src/blsct/wallet/rpc.cpp | 1821 |
| `getblsctrecoverydata` | Get recovery data for BLSCT outputs | src/blsct/wallet/rpc.cpp | 1991 |
| `generatetoblsctaddress` | Mine blocks to a BLSCT address (testnet) | src/rpc/mining.cpp | 280 |

> **Important:** `getnewaddress` works natively with BLSCT wallets — it does NOT need a BLSCT variant or remapping. When a wallet is created with `blsct=true`, `getnewaddress` auto-defaults to `OutputType::BLSCT` and returns a BLSCT bech32m address (e.g., `tnv1...` on testnet). You can also pass `address_type="blsct"` explicitly. Source: `src/wallet/rpc/addresses.cpp:47`.

> **Warning:** `listblscttransactions` is currently a **stub** in navio-core — the implementation is commented out (`src/blsct/wallet/rpc.cpp:1102-1112`) and returns an empty array. Do not rely on it for transaction history. `listblsctunspent` works correctly for confirmed UTXOs.

> **Warning:** BLSCT has **no unconfirmed UTXO visibility**. `AvailableBlsctCoins()` in navio-core hardcodes skip for `nDepth == 0` (`src/wallet/spend.cpp:502-505`). Payments are only visible after 1 confirmation.

Wallet features confirmed: BLSCT wallets, confidential transactions, sub-address derivation from seed, balance proofs, atomic swaps.

## Implementation Steps

### Step 0: Create Integration Repo, Fork and Clone All Repositories

Create a new repo `mxaddict/btcpay-integration` to house the integration plan and all work as git submodules. This repo is the single workspace for the entire integration. It includes:

- **This plan** (`BTCPAY.md`) as the root document, committed first before any other work
- **Submodules** for the 4 BTCPay repo forks where code changes happen
- **`navio-core` submodule** (read-only reference) for looking up chain parameters when updating testnet details or implementing mainnet later

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

**Stop here and verify the plan is pushed before continuing.** The plan must be committed and available at `mxaddict/btcpay-integration` before any implementation work begins.

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
git submodule add git@github.com:nav-io/navio-core.git navio-core

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

The resulting repo structure:

```
btcpay-integration/
├── BTCPAY.md                  ← This integration plan
├── NBitcoin/                  ← mxaddict/NBitcoin (fork of MetacoSA/NBitcoin)
├── NBXplorer/                 ← mxaddict/NBXplorer (fork of dgarage/NBXplorer)
├── btcpayserver/              ← mxaddict/btcpayserver (fork of btcpayserver/btcpayserver)
├── btcpayserver-docker/       ← mxaddict/btcpayserver-docker (fork of btcpayserver/btcpayserver-docker)
├── navio-core/                ← nav-io/navio-core (chain params reference + external C API source)
└── libblsct-bindings/         ← mxaddict/libblsct-bindings (fork of nav-io/libblsct-bindings)
```

The `navio-core` submodule is read-only reference. When testnet parameters change or mainnet is ready, update the submodule pointer, read the new values from `src/kernel/chainparams.cpp`, and propagate them to the 4 fork submodules.

After implementation, push each fork's `navio-support` branch and open draft PRs — see **PR Submission Order** section below.

---

### Step 0b: libblsct-bindings — Add C# P/Invoke Layer

**Repo:** `mxaddict/libblsct-bindings` (fork of `nav-io/libblsct-bindings`)
**Branch:** `csharp-support`
**PR target:** `nav-io/libblsct-bindings`

`libblsct-bindings` already has Python, TypeScript, and Rust FFI wrappers over navio-core's external C API (`src/blsct/external_api/blsct.h`). This step adds a minimal C# P/Invoke layer covering only what address generation needs.

**Status:** Not yet implemented.

#### 0b-i. Fork and add submodule

```bash
gh repo fork nav-io/libblsct-bindings --clone=false
git submodule add git@github.com:mxaddict/libblsct-bindings.git libblsct-bindings
cd libblsct-bindings && git checkout -b csharp-support && cd ..
```

#### 0b-ii. Existing C API functions needed

All required functions already exist in `navio-core/src/blsct/external_api/blsct.h`:

| C function | Signature | Purpose |
|------------|-----------|---------|
| `gen_sub_addr_id` | `BlsctSubAddrId* gen_sub_addr_id(int64_t account, uint64_t address)` | Create sub-address identifier |
| `derive_sub_address` | `BlsctSubAddr* derive_sub_address(BlsctScalar*, BlsctPubKey*, BlsctSubAddrId*)` | Derive sub-address from view+spend keys |
| `encode_address` | `BlsctRetVal* encode_address(BlsctSubAddr*, const char* hrp)` | Encode as bech32m string |
| `decode_address` | `BlsctRetVal* decode_address(const char* addr, const char* hrp)` | Decode bech32m string |
| `free_obj` | `void free_obj(void*)` | Free C heap objects |

Type sizes (from `blsct.h` constants):
- `BlsctScalar`   = 32 bytes (`SCALAR_SIZE`)
- `BlsctPubKey`   = 48 bytes (`PUBLIC_KEY_SIZE`)
- `BlsctSubAddr`  = 96 bytes (`SUB_ADDR_SIZE`)
- `BlsctSubAddrId` = 16 bytes (`SUB_ADDR_ID_SIZE`) — int64 account + uint64 address

#### 0b-iii. Create `ffi/csharp/NavioBlsct.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <PackageId>NavioBlsct</PackageId>
    <Version>0.1.0</Version>
  </PropertyGroup>
</Project>
```

> **Note:** Keep the C# binding on `net8.0` to match BTCPayServer's C# stack. Local builds and CI should install the .NET 8 runtime/SDK.

#### 0b-iv. Create `ffi/csharp/Blsct.cs`

```csharp
using System;
using System.Runtime.InteropServices;

namespace NavioBlsct
{
    /// <summary>
    /// P/Invoke bindings to navio-core's blsct external C API.
    /// The native library (libblsct.so / blsct.dll) must be on the library path.
    /// </summary>
    public static unsafe class Blsct
    {
        private const string Lib = "blsct"; // libblsct.so or blsct.dll

        // --- Raw P/Invoke declarations ---

        [DllImport(Lib)] private static extern IntPtr gen_sub_addr_id(long account, ulong address);
        [DllImport(Lib)] private static extern IntPtr derive_sub_address(byte* viewKey, byte* spendKey, IntPtr subAddrId);
        [DllImport(Lib)] private static extern IntPtr encode_address(IntPtr subAddr, [MarshalAs(UnmanagedType.LPStr)] string hrp);
        [DllImport(Lib)] private static extern IntPtr decode_address([MarshalAs(UnmanagedType.LPStr)] string addr, [MarshalAs(UnmanagedType.LPStr)] string hrp);
        [DllImport(Lib)] private static extern void   free_obj(IntPtr obj);

        // BlsctRetVal has a char* str field at offset 0 (see blsct.h)
        [DllImport(Lib)] private static extern IntPtr get_blsct_ret_val_str(IntPtr retVal);

        // --- Managed API ---

        /// <summary>Creates a sub-address identifier for (account, index).</summary>
        public static IntPtr GenSubAddrId(long account, ulong index) =>
            gen_sub_addr_id(account, index);

        /// <summary>
        /// Derives a BLSCT sub-address from view key + spending key + identifier.
        /// </summary>
        /// <param name="viewKeyBytes">32 bytes (Fr scalar)</param>
        /// <param name="spendKeyBytes">48 bytes (G1 point)</param>
        /// <param name="subAddrId">Handle from GenSubAddrId</param>
        public static IntPtr DeriveSubAddress(byte[] viewKeyBytes, byte[] spendKeyBytes, IntPtr subAddrId)
        {
            if (viewKeyBytes.Length  != 32) throw new ArgumentException("View key must be 32 bytes");
            if (spendKeyBytes.Length != 48) throw new ArgumentException("Spend key must be 48 bytes");
            fixed (byte* vk = viewKeyBytes, sk = spendKeyBytes)
                return derive_sub_address(vk, sk, subAddrId);
        }

        /// <summary>Encodes a derived sub-address as a bech32m string.</summary>
        /// <param name="subAddr">Handle from DeriveSubAddress</param>
        /// <param name="hrp">"tnv" (testnet) or "nav" (mainnet)</param>
        public static string EncodeAddress(IntPtr subAddr, string hrp)
        {
            var retVal = encode_address(subAddr, hrp);
            try
            {
                var strPtr = get_blsct_ret_val_str(retVal);
                return Marshal.PtrToStringAnsi(strPtr)
                    ?? throw new InvalidOperationException("encode_address returned null string");
            }
            finally { free_obj(retVal); }
        }

        /// <summary>Decodes a bech32m address string to raw bytes (96 bytes: C‖D).</summary>
        public static byte[] DecodeAddress(string addr, string hrp)
        {
            var retVal = decode_address(addr, hrp);
            try
            {
                // BlsctRetVal contains a byte* data field; extract 96 bytes
                // (exact field layout: check blsct.h BlsctRetVal struct)
                // TODO: confirm BlsctRetVal layout and extract bytes
                throw new NotImplementedException("Verify BlsctRetVal byte extraction");
            }
            finally { free_obj(retVal); }
        }

        /// <summary>Frees a handle returned by the C API.</summary>
        public static void FreeObj(IntPtr handle) => free_obj(handle);
    }
}
```

> **TODO before finalising:** Verify the `BlsctRetVal` struct layout in `blsct.h` to correctly extract the string and byte data fields. Check how Python/TypeScript bindings read the return value — use the same field accessors pattern.

#### 0b-v. Native library distribution

The `libblsct.so` / `blsct.dll` shared library is built from navio-core. In the Docker deployment, the `naviod` container already has the MCL/blsct shared libraries. NBXplorer's container must have access to them.

**Docker approach:** Copy `libblsct.so` into the NBXplorer container image during the btcpayserver-docker build. Add to `btcpayserver-docker/navio.yml`:

```yaml
nbxplorer:
  volumes:
    - "navio_datadir:/root/.navio"
    - "/path/to/libblsct.so:/usr/local/lib/libblsct.so:ro"
  environment:
    LD_LIBRARY_PATH: /usr/local/lib
```

Longer term: publish `NavioBlsct` as a NuGet package that includes pre-built native binaries for Linux x64, macOS arm64, Windows x64 (same approach as other native NuGet packages).

#### 0b-vi. PR to upstream

Once tested, open PR from `mxaddict/libblsct-bindings:csharp-support` → `nav-io/libblsct-bindings:main`.

---

### Step 1: NBitcoin — Add Navio Network Definition

**Repo:** `mxaddict/NBitcoin` (fork of `MetacoSA/NBitcoin`)
**Branch:** `navio-support`

#### 1a. `NBitcoin.Altcoins/Navio.cs` — already created

**Status:** Done. File exists at `NBitcoin/NBitcoin.Altcoins/Navio.cs` with real genesis hex for testnet, mainnet, and regtest. No changes needed unless chain parameters change.

#### 1b. Register in `NBitcoin.Altcoins/AltcoinNetworkSets.cs`

Add to the static properties and the `GetAll()` method:

```csharp
public static Navio Navio { get; } = Navio.Instance;
```

And in `GetAll()`:
```csharp
yield return Navio;
```

#### 1c. Add BLSCT RPC operations to `NBitcoin/RPC/RPCOperations.cs`

**File:** `NBitcoin/NBitcoin/RPC/RPCOperations.cs`

The enum currently ends at line 130-131:
```csharp
		disconnectnode,
		importdescriptors
	}
}
```

Add the BLSCT entries **before the closing brace**, after `importdescriptors`. Add a trailing comma to `importdescriptors`:

```csharp
		disconnectnode,
		importdescriptors,

		// BLSCT wallet operations (Navio)
		getblsctbalance,
		sendtoblsctaddress,
		listblsctunspent,
		listblscttransactions,
		createblsctrawtransaction,
		fundblsctrawtransaction,
		signblsctrawtransaction,
		decodeblsctrawtransaction,
		setblsctseed,
		getblsctseed,
		getblsctauditkey,
		createblsctbalanceproof,
		unlockblsctoutpoint,
		getblsctrecoverydata,
		generatetoblsctaddress
	}
}
```

#### 1d. Add RPC method name remapping to `NBitcoin/RPC/RPCClient.cs`

**File:** `NBitcoin/NBitcoin/RPC/RPCClient.cs`

NBitcoin's `RPCClient` has **no per-chain RPC override mechanism**. All wallet methods are non-virtual and hardcode standard Bitcoin RPC names. The fix is a `Dictionary<string, string>` remapping layer.

**Step 1: Add the property.** Find the class body (it's a `partial class RPCClient`). Add this property near the other public properties at the top of the class:

```csharp
/// <summary>
/// Per-chain RPC method name overrides. When set, SendCommandAsync
/// will remap method names before sending to the daemon.
/// Example: {"getbalance": "getblsctbalance"} causes all getbalance
/// calls to be sent as getblsctbalance to the daemon.
/// </summary>
public Dictionary<string, string> RPCMethodOverrides { get; set; }
```

**Step 2: Modify `SendCommandAsync` (line 672-675).** Current code:
```csharp
public Task<RPCResponse> SendCommandAsync(string commandName, CancellationToken cancellationToken, params object[] parameters)
{
    return SendCommandAsync(new RPCRequest(commandName, parameters), cancellationToken: cancellationToken);
}
```

Replace with:
```csharp
public Task<RPCResponse> SendCommandAsync(string commandName, CancellationToken cancellationToken, params object[] parameters)
{
    if (RPCMethodOverrides != null && RPCMethodOverrides.TryGetValue(commandName, out var remapped))
        commandName = remapped;
    return SendCommandAsync(new RPCRequest(commandName, parameters), cancellationToken: cancellationToken);
}
```

**Step 3: Modify `SendCommandWithNamedArgsAsync` (line 661-664).** Current code:
```csharp
public Task<RPCResponse> SendCommandWithNamedArgsAsync(string commandName, Dictionary<string, object> parameters, CancellationToken cancellationToken)
{
    return SendCommandAsync(new RPCRequest() { Method = commandName, NamedParams = parameters }, cancellationToken);
}
```

Replace with:
```csharp
public Task<RPCResponse> SendCommandWithNamedArgsAsync(string commandName, Dictionary<string, object> parameters, CancellationToken cancellationToken)
{
    if (RPCMethodOverrides != null && RPCMethodOverrides.TryGetValue(commandName, out var remapped))
        commandName = remapped;
    return SendCommandAsync(new RPCRequest() { Method = commandName, NamedParams = parameters }, cancellationToken);
}
```

> **Note:** The sync variant `SendCommandWithNamedArgs` (line 656-659) also goes through `SendCommandAsync`, so it will pick up the remapping automatically. No separate change needed.

#### 1e. Add `blsct` parameter to `CreateWalletOptions`

**File:** `NBitcoin/NBitcoin/RPC/CreateWalletOptions.cs`

Current full file content (16 lines):
```csharp
#nullable enable

namespace NBitcoin.RPC
{
	public class CreateWalletOptions
	{
		public bool? DisablePrivateKeys { get; set; } 
		public bool? Blank { get; set; } 
		public string? Passphrase { get; set; } 
		public bool? AvoidReuse { get; set; }
		public bool? Descriptors { get; set; }
		public bool? LoadOnStartup { get; set; }
	}
}

#nullable restore
```

Add `Blsct` property after `LoadOnStartup`:
```csharp
		public bool? LoadOnStartup { get; set; }
		public bool? Blsct { get; set; }
```

**File:** `NBitcoin/NBitcoin/RPC/RPCClient.Wallet.cs`

In `CreateWalletAsync` (line 154-175), add the `blsct` parameter handling. Current code at lines 169-173:
```csharp
			if (options?.Descriptors is bool descriptors)
				parameters.Add("descriptors", descriptors);
			if (options?.LoadOnStartup is bool loadOnStartup)
				parameters.Add("load_on_startup", loadOnStartup);
			var result = await SendCommandWithNamedArgsAsync(RPCOperations.createwallet.ToString(), parameters, cancellationToken).ConfigureAwait(false);
```

Add after the `LoadOnStartup` check, before the `SendCommand`:
```csharp
			if (options?.Descriptors is bool descriptors)
				parameters.Add("descriptors", descriptors);
			if (options?.LoadOnStartup is bool loadOnStartup)
				parameters.Add("load_on_startup", loadOnStartup);
			if (options?.Blsct is bool blsct)
				parameters.Add("blsct", blsct);
			var result = await SendCommandWithNamedArgsAsync(RPCOperations.createwallet.ToString(), parameters, cancellationToken).ConfigureAwait(false);
```

#### 1f. Configure Navio network with BLSCT RPC remappings

**File:** `NBitcoin/NBitcoin.Altcoins/Navio.cs`

Add this static method inside the `Navio` class body, after the `PostInit()` method and before `CreateTestnet()`. The file needs `using NBitcoin.RPC;` and `using System.Collections.Generic;` added to the imports.

```csharp
using NBitcoin.RPC;           // add to top of file
using System.Collections.Generic;  // add to top of file
```

Insert after `PostInit()`:
```csharp
        /// <summary>
        /// Configures an RPCClient with BLSCT method name overrides for Navio.
        /// Call this after creating an RPCClient connected to a Navio daemon.
        /// </summary>
        public static void ConfigureBLSCTOverrides(RPCClient client)
        {
            client.RPCMethodOverrides = new Dictionary<string, string>
            {
                { "getbalance",                    "getblsctbalance" },
                { "sendtoaddress",                 "sendtoblsctaddress" },
                { "listunspent",                   "listblsctunspent" },
                { "listtransactions",              "listblscttransactions" },
                { "createrawtransaction",          "createblsctrawtransaction" },
                { "fundrawtransaction",            "fundblsctrawtransaction" },
                { "signrawtransactionwithwallet",  "signblsctrawtransaction" },
                { "decoderawtransaction",          "decodeblsctrawtransaction" },
            };
        }
```

> **Important:** `createwallet` and `getnewaddress` are NOT in this remap dict. `createwallet` stays as-is — BLSCT is enabled via the `blsct=true` parameter. `getnewaddress` auto-detects `WALLET_FLAG_BLSCT` and defaults to `OutputType::BLSCT` — no remapping needed.

NBXplorer will call `Navio.ConfigureBLSCTOverrides(client)` when initializing the RPC connection for Navio.

#### 1g. BLSCT Client-Side Address Derivation

**Status:** Not yet implemented. This is required for payment address generation to work.

**Prerequisite:** Step 0b (libblsct-bindings C# P/Invoke layer) must be completed first.

**Background:** BLSCT uses a Monero-style sub-address scheme over BLS12-381 instead of Bitcoin's BIP32/xpub. The `getblsctauditkey` RPC returns a 160-char hex string:
- First 64 chars = view key scalar (32 bytes, Fr field element)
- Next 96 chars = spending public key (48 bytes, G1 point)

Together these act as the BLSCT equivalent of an xpub. The C API (`navio-core/src/blsct/external_api/blsct.h`) already implements the full derivation — we call it via P/Invoke rather than reimplementing BLS12-381 math in C#.

**Derivation algorithm** (from `navio-core/src/blsct/wallet/address.cpp:34-58`, wrapped by external API):

```
Input:  viewKey  (32-byte Fr scalar)   — BlsctScalar in C API
        spendKey (48-byte G1 point)    — BlsctPubKey in C API
        account  (int64)               — part of BlsctSubAddrId
        index    (uint64)              — part of BlsctSubAddrId

C API call sequence:
  id   = gen_sub_addr_id(account, index)     → BlsctSubAddrId (16 bytes)
  addr = derive_sub_address(viewKey, spendKey, id) → BlsctSubAddr (96 bytes: C‖D)
  str  = encode_address(addr, hrp)           → "tnv1…" bech32m string

HRP: "tnv" (testnet), "nav" (mainnet)
```

##### 1g-i. Add libblsct-bindings NuGet reference

After Step 0b produces a NuGet package (or local project reference during dev):

```xml
<!-- NBitcoin.Altcoins/NBitcoin.Altcoins.csproj -->
<PackageReference Include="NavioBlsct" Version="0.1.0" />
```

During local development, use a project reference to `libblsct-bindings/ffi/csharp/`:
```xml
<ProjectReference Include="..\..\libblsct-bindings\ffi\csharp\NavioBlsct.csproj" />
```

##### 1g-ii. Create `NBitcoin.Altcoins/BlsctDerivationStrategy.cs`

```csharp
using System;
using NBitcoin.DataEncoders;
using NavioBlsct; // from libblsct-bindings C# layer (Step 0b)

namespace NBitcoin.Altcoins
{
    /// <summary>
    /// Client-side BLSCT sub-address derivation via navio-core external C API.
    /// Calls: gen_sub_addr_id → derive_sub_address → encode_address.
    /// </summary>
    public static class BlsctAddressDeriver
    {
        /// <param name="viewKeyBytes">32 bytes (from getblsctauditkey, first 64 hex chars)</param>
        /// <param name="spendKeyBytes">48 bytes (from getblsctauditkey, next 96 hex chars)</param>
        /// <param name="account">0 = receive, -1 = change, -2 = staking</param>
        /// <param name="index">Increments per address</param>
        /// <param name="hrp">"tnv" (testnet) or "nav" (mainnet)</param>
        /// <returns>"tnv1…" or "nav1…" bech32m address string</returns>
        public static string Derive(
            byte[] viewKeyBytes,
            byte[] spendKeyBytes,
            long account,
            ulong index,
            string hrp)
        {
            using var id   = Blsct.GenSubAddrId(account, index);
            using var addr = Blsct.DeriveSubAddress(viewKeyBytes, spendKeyBytes, id);
            return Blsct.EncodeAddress(addr, hrp);
        }
    }

    /// <summary>
    /// NBitcoin DerivationStrategyBase for BLSCT wallets.
    /// String format: "blsct:VIEW_KEY_HEX:SPEND_KEY_HEX"
    ///   VIEW_KEY_HEX  = 64 hex chars (32 bytes, Fr scalar)
    ///   SPEND_KEY_HEX = 96 hex chars (48 bytes, G1 point)
    /// Obtained by: navio-cli getblsctauditkey
    /// </summary>
    public class BlsctDerivationStrategy : DerivationStrategyBase
    {
        public const string Prefix = "blsct:";
        public const long ChangeAccount  = -1;
        public const long StakingAccount = -2;

        public byte[] ViewKey  { get; } // 32 bytes
        public byte[] SpendKey { get; } // 48 bytes

        public BlsctDerivationStrategy(byte[] viewKey, byte[] spendKey)
            : base(null)
        {
            if (viewKey.Length  != 32) throw new ArgumentException("View key must be 32 bytes");
            if (spendKey.Length != 48) throw new ArgumentException("Spend key must be 48 bytes");
            ViewKey  = viewKey;
            SpendKey = spendKey;
        }

        public static BlsctDerivationStrategy Parse(string s)
        {
            if (!s.StartsWith(Prefix, StringComparison.OrdinalIgnoreCase)) return null;
            var parts = s.Substring(Prefix.Length).Split(':');
            if (parts.Length != 2 || parts[0].Length != 64 || parts[1].Length != 96) return null;
            return new BlsctDerivationStrategy(
                Encoders.Hex.DecodeData(parts[0]),
                Encoders.Hex.DecodeData(parts[1]));
        }

        public override string ToString() =>
            Prefix + Encoders.Hex.EncodeData(ViewKey) + ":" + Encoders.Hex.EncodeData(SpendKey);

        public override DerivationStrategyBase GetChild(int i) => this;
        public override Derivation Derive(uint index) =>
            throw new NotSupportedException(
                "Use BlsctAddressDeriver.Derive() for BLSCT address generation");
    }
}
```

> **Note:** `DerivationStrategyBase.Derive(uint index)` is intentionally not supported — BLSCT derivation takes `(account, index)` not a flat uint. NBXplorer's `GenerateAddressesCore` detects `BlsctDerivationStrategy` and calls `BlsctAddressDeriver.Derive()` directly.

---

### Step 2: NBXplorer — Register Navio Chain

**Repo:** `mxaddict/NBXplorer` (fork of `dgarage/NBXplorer`)
**Branch:** `navio-support`

#### 2a. Create `NBXplorer.Client/NBXplorerNetworkProvider.Navio.cs`

```csharp
using NBitcoin;

namespace NBXplorer
{
    public partial class NBXplorerNetworkProvider
    {
        private void InitNavio(ChainName networkType)
        {
            Add(new NBXplorerNetwork(NBitcoin.Altcoins.Navio.Instance, networkType)
            {
                MinRPCVersion = 220000,
                CoinType = networkType == ChainName.Mainnet
                    ? new KeyPath("0'")   // No registered BIP44 coin type yet; use 0' for now
                    : new KeyPath("1'"),
            });
        }

        public NBXplorerNetwork GetNAV()
        {
            return GetFromCryptoCode(
                NBitcoin.Altcoins.Navio.Instance.CryptoCode);
        }
    }
}
```

#### 2b. Register in `NBXplorer.Client/NBXplorerNetworkProvider.cs`

In the constructor, add after the other `Init` calls:

```csharp
InitNavio(networkType);
```

#### 2c. Apply BLSCT RPC overrides on Navio RPC client initialization

**File:** `NBXplorer/NBXplorer/RPCClientExtensions.cs`

The `EnsureWalletCreated` method starts at line 249. Current code (lines 249-261):
```csharp
public static async Task EnsureWalletCreated(this RPCClient client, ILogger logger)
{
    var network = client.Network.NetworkSet;
    var walletName = client.CredentialString.WalletName ?? "";
    bool created = false;
    try
    {
        await client.CreateWalletAsync(walletName, new CreateWalletOptions()
        {
            LoadOnStartup = true,
            Blank = client.Network.ChainName != ChainName.Regtest
        });
        logger.LogInformation($"{network.CryptoCode}: Created RPC wallet \"{walletName}\"");
        created = true;
    }
```

Make two changes:

**Change 1:** Add BLSCT override application at the start of the method (after `var network =`, before `var walletName =`):
```csharp
    var network = client.Network.NetworkSet;
    // Apply BLSCT RPC overrides for Navio
    if (network is NBitcoin.Altcoins.Navio)
    {
        NBitcoin.Altcoins.Navio.ConfigureBLSCTOverrides(client);
    }
    var walletName = client.CredentialString.WalletName ?? "";
```

**Change 2:** Add `Blsct = true` to the `CreateWalletOptions` for Navio:
```csharp
        await client.CreateWalletAsync(walletName, new CreateWalletOptions()
        {
            LoadOnStartup = true,
            Blank = client.Network.ChainName != ChainName.Regtest,
            Blsct = network is NBitcoin.Altcoins.Navio ? true : null
        });
```

> **Note:** `createwallet` is NOT in the remap dict, so it goes to the daemon as `createwallet` with the `blsct=true` parameter. The remapping only affects subsequent wallet RPC calls (`getbalance` → `getblsctbalance`, etc.).

#### 2d. BLSCT-aware address generation — skip descriptor import, use BlsctDerivationStrategy

**Status:** `ImportDescriptorToRPCIfNeeded` skip — done. `GenerateAddressesCore` BLSCT path — see §2h.

**`getnewaddress`** works natively on BLSCT wallets (auto-selects `OutputType::BLSCT`). Not remapped.

**`importdescriptors` must be skipped.** `WALLET_FLAG_BLSCT` clears `WALLET_FLAG_DESCRIPTORS` — standard descriptors are incompatible. Already implemented in `NBXplorer/NBXplorer/Backend/Repository.cs:250`.

**`ImportAddressAsync` (Legacy fallback) must also be skipped for Navio.** After skipping descriptors, NBXplorer falls back to `importaddress` per address — which also fails on BLSCT wallets. The daemon already tracks all addresses generated by its own `blsct::KeyMan`; no import is needed.

**File:** `NBXplorer/NBXplorer/Backend/Repository.cs`

In `ImportDescriptorToRPCIfNeeded`, extend the existing Navio early-return to also skip the Legacy fallback path. The current code already returns early for Navio — verify the Legacy `ImportAddressAsync` calls further down the method are also unreachable for Navio.

#### 2h. `GenerateAddressesCore` BLSCT derivation path

**Status:** Not yet implemented. Depends on §1g being implemented first.

**File:** `NBXplorer/NBXplorer/Backend/Repository.cs`

`GenerateAddressesCore` (~line 174) currently always derives addresses via BIP32 client-side. Add a BLSCT path at the top of the method that fires when the strategy is `BlsctDerivationStrategy`:

```csharp
internal async Task<int> GenerateAddressesCore(
    DbConnection connection,
    DerivationStrategyBase strategy,
    DerivationLine derivationLine,
    GenerateAddressQuery query)
{
    // BLSCT path: derive addresses from (viewKey, spendKey) instead of BIP32
    if (strategy is NBitcoin.Altcoins.BlsctDerivationStrategy blsct)
    {
        return await GenerateBlsctAddressesCore(
            connection, blsct, derivationLine.Feature, query);
    }

    // ... existing BIP32 path unchanged ...
}

private async Task<int> GenerateBlsctAddressesCore(
    DbConnection connection,
    BlsctDerivationStrategy strategy,
    DerivationFeature feature,
    GenerateAddressQuery query)
{
    var walletKey = GetWalletKey(strategy, Network);
    var descriptorKey = GetDescriptorKey(strategy, feature);
    var gapNextIndex = await GetGapAndNextIdx(connection, descriptorKey);
    long toGenerate = ToGenerateCount(query, gapNextIndex?.gap);
    if (gapNextIndex is not null && toGenerate == 0)
        return 0;

    // Initial wallet/descriptor row creation (same as BIP32 path)
    if (gapNextIndex is null)
    {
        await connection.ExecuteAsync(
            "INSERT INTO wallets VALUES (@wid, @metadata::JSONB) ON CONFLICT DO NOTHING",
            walletKey);
        await connection.ExecuteAsync(
            "INSERT INTO descriptors VALUES (@code, @descriptor, @metadata::JSONB) " +
            "ON CONFLICT DO NOTHING; " +
            "INSERT INTO wallets_descriptors (code, descriptor, wallet_id) " +
            "VALUES (@code, @descriptor, @wallet_id) ON CONFLICT DO NOTHING;",
            new
            {
                descriptorKey.code,
                descriptorKey.descriptor,
                metadata = Serializer.ToString(new LegacyDescriptorMetadata()
                {
                    Derivation = strategy,
                    Feature = feature,
                    Type = LegacyDescriptorMetadata.TypeName
                }),
                wallet_id = walletKey.wid
            });
        gapNextIndex = await GetGapAndNextIdx(connection, descriptorKey);
        toGenerate = ToGenerateCount(query, gapNextIndex?.gap);
    }
    if (gapNextIndex is null) return 0;

    long totalGenerated = 0;
    var hrp = Network.NBitcoinNetwork.NetworkSet is NBitcoin.Altcoins.Navio
        ? (Network.NBitcoinNetwork == Network.NBitcoinNetwork.GetNetwork("nav-test")
            ? "tnv" : "nav")
        : throw new InvalidOperationException("BlsctDerivationStrategy used on non-Navio network");

    // Map DerivationFeature to BLSCT account index
    long account = feature == DerivationFeature.Change
        ? NBitcoin.Altcoins.BlsctDerivationStrategy.ChangeAccount   // -1
        : 0; // receive

    do
    {
        var nextIndex = gapNextIndex.next_idx;
        var inserts = new DescriptorScriptInsert[toGenerate];

        // Derive BLSCT addresses for indices [nextIndex, nextIndex + toGenerate)
        for (long i = 0; i < toGenerate; i++)
        {
            var addrStr = NBitcoin.Altcoins.BlsctAddressDeriver.Derive(
                strategy.ViewKey,
                strategy.SpendKey,
                account,
                (ulong)(nextIndex + i),
                hrp);
            var addr = BitcoinAddress.Create(addrStr, Network.NBitcoinNetwork);
            inserts[i] = new DescriptorScriptInsert(
                descriptorKey.descriptor,
                nextIndex + i,
                addr.ScriptPubKey,
                addr.ScriptPubKey.Hash.ToString(),
                addr,
                feature);
        }

        await InsertDescriptorScripts(connection, inserts);
        totalGenerated += toGenerate;
        gapNextIndex = await GetGapAndNextIdx(connection, descriptorKey);
        toGenerate = ToGenerateCount(query, gapNextIndex?.gap);
    } while (toGenerate > 0);

    return (int)totalGenerated;
}
```

> **Note on `importaddress`:** Do not call `ImportAddressAsync` for Navio after address generation. The daemon's BLSCT wallet (`blsct::KeyMan`) scans all outputs using the private view key — it does not need explicit address import. NBXplorer's DB is the source of truth for address tracking; the daemon handles payment detection independently via `walletnotify`.

#### 2e. BLSCT-aware UTXO scanning — use `listblsctunspent`, skip `scantxoutset`

**Finding:** BLSCT output detection requires the daemon's private view key + sub-address pool. External tools cannot identify BLSCT outputs. NBXplorer **must rely on the daemon wallet** for all UTXO discovery.

**`listblsctunspent`** (via remapped `listunspent`) is the primary discovery mechanism. It returns:
- `outid` — output hash (not txid — BLSCT uses per-output identifiers)
- `address` — decoded BLSCT address
- `amount` — decrypted from range proof
- `confirmations`, `spendable`, `scriptPubKey`

**Limitations:**
- **Confirmed only** — `AvailableBlsctCoins()` in navio-core hardcodes skip for `nDepth == 0` (`spend.cpp:502-505`). Unconfirmed BLSCT UTXOs are never returned, even with `minconf=0`.
- **`listblscttransactions` is a stub** — the implementation is commented out (`rpc.cpp:1102-1112`), returns empty array. Cannot be used for transaction history until navio-core fixes this.

**`-walletnotify`** fires for new BLSCT outputs with `%s` = outpoint hash (not txid). NBXplorer can use this for push notifications, then poll `listblsctunspent` for details.

**File:** `NBXplorer/NBXplorer/ScanUTXOSetService.cs`

In `GetScannedItems` (~line 294), skip `scantxoutset` for Navio and use `listblsctunspent` instead:
```csharp
if (network.NBitcoinNetwork.NetworkSet is NBitcoin.Altcoins.Navio)
{
    // BLSCT outputs are blinded — scantxoutset cannot identify them.
    // The daemon wallet handles detection via the private view key.
    // NBXplorer should poll listblsctunspent (remapped from listunspent)
    // for confirmed UTXOs instead.
    return null; // Skip scantxoutset for Navio
}
```

> **Impact on payment detection:** BTCPayServer invoices typically watch for unconfirmed payments first, then track confirmations. Since BLSCT has no unconfirmed UTXO visibility, Navio payments will only appear after 1 confirmation (~1 min with PoS). This is acceptable for testnet but may need a navio-core fix for production use.

#### 2f. BLSCT transaction creation — use `sendtoblsctaddress`, skip PSBT entirely

**Finding:** `sendtoblsctaddress` is a **single-call create+fund+sign+broadcast operation**, identical in pattern to Bitcoin's `sendtoaddress`. It handles coin selection, change address generation, BLSCT range proof creation, BLS signature aggregation, and broadcast — all in one RPC call. Returns a txid hex string (same format as `sendtoaddress`).

There is **no BLSCT PSBT support** in navio-core. The `UnsignedTransaction` format exists for advanced use cases (external signing, atomic swaps) but is NOT needed for BTCPayServer's standard payment flow.

**This is the simplest integration path:** Since `sendtoblsctaddress` is remapped from `sendtoaddress` (via step 1f), any code that calls `sendtoaddress` on the Navio RPC client will automatically route to `sendtoblsctaddress`. The return format (txid) is identical.

**File:** `NBXplorer/Controllers/MainController.PSBT.cs`

For the PSBT creation endpoint, skip the PSBT flow for Navio since BTCPayServer's hot wallet can use `sendtoblsctaddress` directly:

```csharp
if (network.NBitcoinNetwork.NetworkSet is NBitcoin.Altcoins.Navio)
{
    // BLSCT does not support PSBT. Use sendtoblsctaddress
    // (remapped from sendtoaddress) for direct send instead.
    // The multi-step create+fund+sign flow is not needed.
    throw new NotSupportedException(
        "PSBT is not supported for Navio BLSCT transactions. " +
        "Use the direct send API (sendtoaddress → sendtoblsctaddress) instead.");
}
```

**For the multi-step flow** (advanced use cases only, not needed for initial integration):
1. `createblsctrawtransaction` (remapped from `createrawtransaction`) — creates `UnsignedTransaction` hex
2. `fundblsctrawtransaction` (remapped from `fundrawtransaction`) — adds inputs + change, returns funded `UnsignedTransaction` hex
3. `signblsctrawtransaction` (remapped from `signrawtransactionwithwallet`) — signs and returns standard `CTransaction` hex
4. `sendrawtransaction` — broadcasts (standard, no remapping needed)

> **Note:** BTCPayServer's wallet UI has both a "hot wallet" mode (server-side signing) and a "watch-only + external signer" mode. The hot wallet mode can use `sendtoblsctaddress` directly. The watch-only mode would need the multi-step flow with external signing, but BLSCT external signing requires sharing BLSCT private keys, which defeats the purpose. For initial testnet integration, hot wallet mode is sufficient.

#### 2g. Add BLSCT methods to RPC proxy whitelist

**File:** `NBXplorer/Controllers/MainController.cs`

Add BLSCT wallet methods to `WhitelistedRPCMethods` (~line 83) so they can be called through NBXplorer's RPC proxy:

```csharp
static HashSet<string> WhitelistedRPCMethods = new HashSet<string>()
{
    // ... existing methods ...
    // BLSCT wallet methods (Navio)
    "getblsctbalance",
    "sendtoblsctaddress",
    "listblsctunspent",
    "listblscttransactions",
    "createblsctrawtransaction",
    "fundblsctrawtransaction",
    "signblsctrawtransaction",
    "decodeblsctrawtransaction",
    "createblsctbalanceproof",
};
```

---

### Step 3: BTCPayServer — Add Navio Plugin

**Repo:** `mxaddict/btcpayserver` (fork of `btcpayserver/btcpayserver`)
**Branch:** `navio-support`

#### 3a. Create `BTCPayServer/Plugins/Altcoins/AltcoinsPlugin.Navio.cs`

Follow the exact pattern of `AltcoinsPlugin.Groestlcoin.cs`:

```csharp
using BTCPayServer.Common;
using BTCPayServer.Payments;
using BTCPayServer.Services;
using Microsoft.Extensions.DependencyInjection;
using NBitcoin;

namespace BTCPayServer.Plugins.Altcoins;

public partial class AltcoinsPlugin
{
    public void InitNavio(IServiceCollection services)
    {
        var nbxplorerNetwork = NBXplorerNetworkProvider.GetFromCryptoCode("NAV");
        var network = new BTCPayNetwork()
        {
            CryptoCode = nbxplorerNetwork.CryptoCode,
            DisplayName = "Navio",
            NBXplorerNetwork = nbxplorerNetwork,
            DefaultRateRules = new[]
            {
                "NAV_X = NAV_BTC * BTC_X",
                "NAV_BTC = coingecko(NAV_BTC)"
            },
            CryptoImagePath = "imlegacy/navio.svg",
            DefaultSettings =
                BTCPayDefaultSettings.GetDefaultSettings(ChainName),
            CoinType = ChainName == ChainName.Mainnet
                ? new KeyPath("0'")
                : new KeyPath("1'"),
            SupportRBF = true,
            SupportPayJoin = false,
            VaultSupported = true,
        }.SetDefaultElectrumMapping(ChainName);

        var blockExplorerLink = ChainName == ChainName.Mainnet
            ? "https://explorer.navio.org/tx/{0}"      // Update when explorer exists
            : "https://testnet.explorer.navio.org/tx/{0}";

        services.AddBTCPayNetwork(network)
            .AddTransactionLinkProvider(
                PaymentTypes.CHAIN.GetPaymentMethodId(
                    nbxplorerNetwork.CryptoCode),
                new DefaultTransactionLinkProvider(blockExplorerLink));
    }
}
```

#### 3b. Register in `BTCPayServer/Plugins/Altcoins/AltcoinsPlugin.cs`

In the `Execute()` method, add alongside the other altcoin checks:

```csharp
if (selectedChains.Contains("NAV"))  InitNavio(services);
```

#### 3c. Add coin icon

Place an SVG icon at `BTCPayServer/wwwroot/imlegacy/navio.svg`.

#### 3d. BLSCT wallet setup — audit key as derivation strategy

**Status:** Not yet implemented. Depends on §1g and §2h.

Instead of an xpub, Navio uses `(viewKey, spendingPubKey)` from `getblsctauditkey` as the wallet's derivation strategy. The user flow:

**Step 1: Export audit key from running daemon**

```bash
navio-cli -testnet getblsctauditkey
# Returns 160-char hex string:
# First 64 chars  = view key scalar (32 bytes)
# Next 96 chars   = spending public key (48 bytes)
```

**Step 2: Construct BlsctDerivationStrategy string**

Split the 160-char output and format as:
```
blsct:VIEW_KEY_HEX:SPEND_KEY_HEX
```
Example:
```
blsct:a3f1...c2e9:04ab...7f3d
```

**Step 3: Register BlsctDerivationStrategy parser in BTCPayServer**

**File:** `BTCPayServer/Services/Wallets/BTCPayWalletProvider.cs` or the derivation strategy parsing path.

BTCPayServer calls `network.NBXplorerNetwork.ParseDerivationStrategy(string)` when a user pastes a strategy. For Navio, this must recognize `blsct:` prefix and return a `BlsctDerivationStrategy`.

**File:** `NBXplorer/NBXplorer.Client/NBXplorerNetworkProvider.Navio.cs`

Add strategy parsing to `InitNavio`:
```csharp
private void InitNavio(ChainName networkType)
{
    Add(new NBXplorerNetwork(NBitcoin.Altcoins.Navio.Instance, networkType)
    {
        MinRPCVersion = 220000,
        CoinType = networkType == ChainName.Mainnet
            ? new KeyPath("0'")
            : new KeyPath("1'"),
        // Custom strategy parser: recognize "blsct:..." strings
        DerivationStrategyFactory = new BlsctDerivationStrategyFactory(),
    });
}
```

Alternatively, override `ParseDerivationStrategy` on the NBXplorerNetwork for Navio to intercept `blsct:...` strings before the standard BIP32 parser runs.

**Step 4: Wallet import UI in BTCPayServer (minimal change)**

In the "Import wallet" flow for Navio, show a text field labelled "BLSCT Audit Key" with instructions to run `getblsctauditkey`. The value is stored internally as `blsct:VIEW_HEX:SPEND_HEX`.

No other UI changes required for testnet — address generation, payment detection, and invoice confirmation all flow through the standard NBXplorer path once §1g and §2h are implemented.

> **Spending:** The daemon's hot wallet holds the private BLSCT seed. NBXplorer/BTCPayServer request sends via the remapped `sendtoaddress` → `sendtoblsctaddress` RPC. No private key export or PSBT needed.

---

### Step 4: btcpayserver-docker — Docker Fragment

**Repo:** `mxaddict/btcpayserver-docker` (fork of `btcpayserver/btcpayserver-docker`)
**Branch:** `navio-support`

#### 4a. Create `docker-compose-generator/docker-fragments/navio.yml`

```yaml
services:
  naviod:
    restart: unless-stopped
    container_name: btcpayserver_naviod
    image: navio/naviod:latest
    environment:
      NAVIO_NETWORK: ${NBITCOIN_NETWORK:-testnet}
      NAVIO_EXTRA_ARGS: |
        rpcport=33577
        rpcbind=0.0.0.0:33577
        rpcallowip=0.0.0.0/0
        port=33570
        whitelist=0.0.0.0/0
        testnet=1
    expose:
      - "33577"
      - "33570"
    volumes:
      - "navio_datadir:/data"
      - "navio_wallet_datadir:/walletdata"

  nbxplorer:
    environment:
      NBXPLORER_CHAINS: "nav"
      NBXPLORER_NAVRPCURL: http://naviod:33577/
      NBXPLORER_NAVNODEENDPOINT: naviod:33570
    volumes:
      - "navio_datadir:/root/.navio"

  btcpayserver:
    environment:
      BTCPAY_CHAINS: "nav"
      BTCPAY_NAVEXPLORERURL: http://nbxplorer:24445/

volumes:
  navio_datadir:
  navio_wallet_datadir:

required:
  - "nbxplorer"
```

#### 4b. Add entry to `docker-compose-generator/crypto-definitions.json`

```json
{
  "Crypto": "nav",
  "CryptoFragment": "navio",
  "CLightningFragment": null,
  "LNDFragment": null,
  "EclairFragment": null,
  "PhoenixdFragment": null
}
```

#### 4c. Create `navio-cli.sh` wrapper script

Every altcoin in btcpayserver-docker has a CLI wrapper script in the repo root. Create `navio-cli.sh`:

```bash
#!/bin/bash

docker exec btcpayserver_naviod navio-cli -datadir="/data" "$@"
```

Make executable: `chmod +x navio-cli.sh`

This follows the exact pattern of `litecoin-cli.sh`, `dogecoin-cli.sh`, `groestlcoin-cli.sh`, etc.

---

## Navio Daemon File Locations

| Item | Path |
|------|------|
| Daemon binary | `./src/naviod` |
| CLI binary | `./src/navio-cli` |
| Config file | `~/.navio/navio.conf` |
| Data directory | `~/.navio/` |
| Testnet data | `~/.navio/testnet5/` |
| Log file | `~/.navio/testnet5/debug.log` |

## Building Navio Core

```bash
./autogen.sh
./configure --disable-bench
make -j$(nproc)
```

Binaries appear in `./src/`.

## Local Build Setup (Development)

The submodules form a dependency chain. To build locally against the fork changes, use **local project references** rather than NuGet packages for the repos you've changed.

### NBitcoin.Altcoins — builds standalone
```bash
cd NBitcoin
dotnet build NBitcoin.Altcoins/NBitcoin.Altcoins.csproj
```

### NBXplorer — switch to local NBitcoin project refs

`NBXplorer.Client/NBXplorer.Client.csproj` must reference local NBitcoin instead of NuGet:

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

> **Note:** Revert `NBXplorer.Client.csproj` back to `PackageReference` before submitting the upstream PR. The PR must reference the published NuGet versions, not local paths.

### BTCPayServer — builds against published NuGet packages

BTCPayServer targets NBitcoin 9.0.5 (NuGet). Our forks use NBitcoin 10.0.1. Direct project reference substitution causes version conflicts. BTCPayServer builds fine with upstream NuGet packages because:
- `AltcoinsPlugin.Navio.cs` uses `GetFromCryptoCode("NAV")` which exists in published NBXplorer.Client
- The `GetNAV()` helper in our NBXplorer fork is only needed for NBXplorer-internal use

```bash
cd btcpayserver
dotnet build BTCPayServer/BTCPayServer.csproj -p:AllowMissingPrunePackageData=true
```

> **Upstream PR dependency:** BTCPayServer's PR can only be fully validated end-to-end once NBitcoin and NBXplorer PRs are merged and published to NuGet. The compile check above only verifies BTCPayServer's Navio code compiles against the existing published NBXplorer.Client.

## Testing Checklist

### libblsct-bindings C# layer
- [ ] `Blsct.GenSubAddrId(account, index)` returns valid handle
- [ ] `Blsct.DeriveSubAddress(viewKey, spendKey, id)` produces correct 96-byte sub-address
- [ ] `Blsct.EncodeAddress(subAddr, "tnv")` produces correct `tnv1…` string matching daemon output
- [ ] `Blsct.DecodeAddress("tnv1…", "tnv")` round-trips correctly
- [ ] Native library loads on Linux x64 (Docker target platform)

### Foundation
- [ ] NBitcoin: Navio testnet network parses BLSCT `tnv` addresses correctly
- [ ] NBitcoin: RPC method remapping works (`getbalance` → `getblsctbalance`, etc.)
- [ ] NBitcoin: `CreateWalletAsync` passes `blsct=true` for Navio
- [ ] NBitcoin: `BlsctAddressDeriver.Derive()` produces correct `tnv1…` addresses matching daemon output (verify against `keyman_tests.cpp` expected values)
- [ ] NBitcoin: `BlsctDerivationStrategy` round-trips correctly through `ToString()` / `Parse()`

### NBXplorer ↔ Daemon
- [ ] NBXplorer: creates BLSCT wallet on Navio daemon (`createwallet` with `blsct=true`)
- [ ] NBXplorer: BLSCT RPC overrides are active for Navio connections
- [ ] NBXplorer: `getblsctbalance` returns correct balance via remapped `getbalance`
- [ ] NBXplorer: `listblsctunspent` returns BLSCT UTXOs via remapped `listunspent`
- [ ] NBXplorer: `listblscttransactions` returns transaction history
- [ ] NBXplorer: indexes Navio testnet blocks and tracks BLSCT transactions
- [ ] NBXplorer: descriptor import is correctly skipped for Navio (BLSCT seed-based)
- [ ] NBXplorer: `GenerateAddressesCore` with `BlsctDerivationStrategy` produces correct `tnv1…` addresses
- [ ] NBXplorer: `ImportAddressAsync` (Legacy fallback) is skipped for Navio

### BTCPayServer
- [ ] BTCPayServer: Navio appears as available cryptocurrency
- [ ] BTCPayServer: can create store with Navio payment method
- [ ] BTCPayServer: generates valid BLSCT `tnv` testnet addresses
- [ ] BTCPayServer: receives testnet BLSCT payment and confirms invoice
- [ ] BTCPayServer: webhook fires on payment confirmation
- [ ] BTCPayServer: refund flow works with BLSCT transactions

### Transaction Flow
- [ ] `sendtoblsctaddress` works via remapped `sendtoaddress`
- [ ] `fundblsctrawtransaction` + `signblsctrawtransaction` produce valid BLSCT transactions
- [ ] `sendrawtransaction` broadcasts BLSCT transactions successfully

### Docker
- [ ] Docker: `naviod` container starts and syncs testnet
- [ ] Docker: NBXplorer connects to naviod and indexes chain
- [ ] Docker: `navio-cli.sh` wrapper works correctly

## PR Submission Order

> **Draft PRs open within mxaddict forks.** See [PRs.md](PRs.md) for links. These are tracking PRs in the `mxaddict/*` repos — upstream PRs to `MetacoSA/NBitcoin`, `dgarage/NBXplorer`, etc. have **not** been opened yet.

PRs depend on each other. Submit upstream in this order once code is ready for review:

1. **libblsct-bindings** → `mxaddict/libblsct-bindings:csharp-support` → PR to `nav-io/libblsct-bindings:main` — C# P/Invoke layer for address derivation (independent, no upstream deps)
2. **NBitcoin** → `mxaddict/NBitcoin:navio-support` → PR to `MetacoSA/NBitcoin:master` — Navio network definition + RPC method remapping + BLSCT RPC operations + `CreateWalletOptions.Blsct` + `BlsctDerivationStrategy` (depends on #1 being merged/published as NuGet)
3. **NBXplorer** → `mxaddict/NBXplorer:navio-support` → PR to `dgarage/NBXplorer:master` — Chain registration + BLSCT RPC overrides + skip descriptor import + BLSCT UTXO scanning + `GenerateBlsctAddressesCore` + BLSCT tx creation + RPC whitelist (depends on #2)
4. **BTCPayServer** → `mxaddict/btcpayserver:navio-support` → PR to `btcpayserver/btcpayserver:master` — Altcoins plugin + audit key wallet setup (depends on #3)
5. **btcpayserver-docker** → `mxaddict/btcpayserver-docker:navio-support` → PR to `btcpayserver/btcpayserver-docker:master` — Docker deployment + `navio-cli.sh` + libblsct.so volume mount (independent, can go in parallel with #2-4)

## Future Work (Mainnet)

- Finalize mainnet chainparams (`fBLSCT` activation, genesis block)
- Register a BIP44 coin type via [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)
- Update NBitcoin mainnet genesis hex
- Add Navio to CoinGecko/exchange rate providers for `DefaultRateRules`
- Set up block explorer and update `blockExplorerLink`
- Build and publish Docker image for `navio/naviod`
- Add Lightning support if applicable

## References

- [NBitcoin.Altcoins pattern](https://github.com/MetacoSA/NBitcoin/tree/master/NBitcoin.Altcoins) — Litecoin.cs, Groestlcoin.cs
- [NBXplorer chain registration](https://github.com/dgarage/NBXplorer/tree/master/NBXplorer.Client) — NBXplorerNetworkProvider.*.cs
- [BTCPayServer AltcoinsPlugin](https://github.com/btcpayserver/btcpayserver/tree/master/BTCPayServer/Plugins/Altcoins) — AltcoinsPlugin.Groestlcoin.cs
- [BTCPayServer Docker fragments](https://github.com/btcpayserver/btcpayserver-docker/tree/master/docker-compose-generator/docker-fragments)
- [Navio Chain Parameters](navio-core/src/kernel/chainparams.cpp)
- [Navio RPC Interface](navio-core/doc/JSON-RPC-interface.md)
- [Navio BLSCT Wallet RPCs](navio-core/src/blsct/wallet/rpc.cpp) — all BLSCT wallet RPC implementations
