# BTCPayServer Integration Plan for Navio

## Overview

Add Navio (a Bitcoin Core fork with BLSCT) as a supported cryptocurrency in BTCPayServer. Navio uses Bitcoin-compatible RPC and UTXO model, so it follows the **NBXplorer path** (same as Litecoin, Groestlcoin, Dogecoin, etc.). This requires changes across 4 repositories, each submitted as a separate PR.

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

BTCPayServer's on-chain wallet infrastructure (address generation, UTXO tracking, transaction signing) is shared across all NBXplorer-based coins. No custom payment handlers needed.

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

| RPC Method | File | Line |
|------------|------|------|
| `getblockchaininfo` | src/rpc/blockchain.cpp | 1236 |
| `getblock` | src/rpc/blockchain.cpp | 662 |
| `getbestblockhash` | src/rpc/blockchain.cpp | 242 |
| `getrawtransaction` | src/rpc/rawtransaction.cpp | 291 |
| `getnetworkinfo` | src/rpc/net.cpp | 613 |
| `sendrawtransaction` | src/rpc/mempool.cpp | 37 |
| `getmempoolinfo` | src/rpc/mempool.cpp | 682 |
| `sendtoaddress` | src/wallet/rpc/spend.cpp | 217 |
| `createwallet` | src/wallet/rpc/wallet.cpp | 348 |
| `getnewaddress` | src/wallet/rpc/addresses.cpp | 22 |
| `listunspent` | src/wallet/rpc/coins.cpp | 536 |
| `fundrawtransaction` | src/wallet/rpc/spend.cpp | 726 |
| `signrawtransactionwithwallet` | src/wallet/rpc/spend.cpp | 843 |
| `getwalletinfo` | src/wallet/rpc/wallet.cpp | 44 |

Wallet features confirmed: descriptor wallets, HD key derivation, multisig, external signer.

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
├── BTCPAY.md              ← This integration plan
├── NBitcoin/              ← mxaddict/NBitcoin (fork of MetacoSA/NBitcoin)
├── NBXplorer/             ← mxaddict/NBXplorer (fork of dgarage/NBXplorer)
├── btcpayserver/          ← mxaddict/btcpayserver (fork of btcpayserver/btcpayserver)
├── btcpayserver-docker/   ← mxaddict/btcpayserver-docker (fork of btcpayserver/btcpayserver-docker)
└── navio-core/            ← nav-io/navio-core (chain params reference)
```

The `navio-core` submodule is read-only reference. When testnet parameters change or mainnet is ready, update the submodule pointer, read the new values from `src/kernel/chainparams.cpp`, and propagate them to the 4 fork submodules.

After implementation, push each fork's `navio-support` branch and open draft PRs — see **PR Submission Order** section below.

---

### Step 1: NBitcoin — Add Navio Network Definition

**Repo:** `mxaddict/NBitcoin` (fork of `MetacoSA/NBitcoin`)
**Branch:** `navio-support`

#### 1a. Create `NBitcoin.Altcoins/Navio.cs`

This file defines the Navio network for NBitcoin. Follow the exact pattern of `Litecoin.cs` or `Groestlcoin.cs`.

```csharp
using NBitcoin;
using NBitcoin.DataEncoders;
using NBitcoin.Protocol;
using System;
using System.Linq;

namespace NBitcoin.Altcoins
{
    public class Navio : NetworkSetBase
    {
        public static Navio Instance { get; } = new Navio();
        public override string CryptoCode => "NAV";

        private Navio()
        {
        }

        protected override void PostInit()
        {
            RegisterDefaultCookiePath("Navio",
                new FolderName() { TestnetFolder = "testnet5" });
        }

        // Navio testnet uses BLSCT genesis (custom block format).
        // The consensus factory stays Bitcoin-compatible for standard
        // P2PKH/P2SH/P2WPKH/P2WSH transactions that BTCPayServer uses.

        protected override NetworkBuilder CreateTestnet()
        {
            return new NetworkBuilder()
                .SetConsensus(new Consensus()
                {
                    SubsidyHalvingInterval = 210000,
                    MajorityEnforceBlockUpgrade = 51,
                    MajorityRejectBlockOutdated = 75,
                    MajorityWindow = 100,
                    PowTargetTimespan = TimeSpan.FromSeconds(14 * 24 * 60 * 60),
                    PowTargetSpacing = TimeSpan.FromSeconds(10 * 60),
                    PowAllowMinDifficultyBlocks = true,
                    PowNoRetargeting = true,
                    CoinbaseMaturity = 100,
                    ConsensusFactory = BitcoinConsensusFactory.Instance,
                    SupportSegwit = true,
                    SupportTaproot = true,
                })
                .SetBase58Bytes(Base58Type.PUBKEY_ADDRESS, new byte[] { 111 })
                .SetBase58Bytes(Base58Type.SCRIPT_ADDRESS, new byte[] { 196 })
                .SetBase58Bytes(Base58Type.SECRET_KEY, new byte[] { 239 })
                .SetBase58Bytes(Base58Type.EXT_PUBLIC_KEY, new byte[] { 0x04, 0x35, 0x87, 0xCF })
                .SetBase58Bytes(Base58Type.EXT_SECRET_KEY, new byte[] { 0x04, 0x35, 0x83, 0x94 })
                .SetBech32(Bech32Type.WITNESS_PUBKEY_ADDRESS, Encoders.Bech32("tb"))
                .SetBech32(Bech32Type.WITNESS_SCRIPT_ADDRESS, Encoders.Bech32("tb"))
                .SetMagic(0xdf0e3cb9)  // bytes: b9 3c 0e df → LE uint32
                .SetPort(33570)
                .SetRPCPort(33577)
                .SetMaxP2PVersion(70016)
                .SetName("nav-test")
                .SetNetworkStringParser(new BitcoinStringParser())
                .SetGenesis("GENESIS_HEX_HERE");
                // To get genesis hex: navio-cli -testnet getblock \
                //   57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b 0
        }

        // Mainnet stub — not ready yet (fBLSCT=false on mainnet, chainparams incomplete)
        protected override NetworkBuilder CreateMainnet()
        {
            return new NetworkBuilder()
                .SetConsensus(new Consensus()
                {
                    SubsidyHalvingInterval = 210000,
                    MajorityEnforceBlockUpgrade = 750,
                    MajorityRejectBlockOutdated = 950,
                    MajorityWindow = 1000,
                    PowTargetTimespan = TimeSpan.FromSeconds(14 * 24 * 60 * 60),
                    PowTargetSpacing = TimeSpan.FromSeconds(10 * 60),
                    PowAllowMinDifficultyBlocks = false,
                    PowNoRetargeting = false,
                    CoinbaseMaturity = 100,
                    ConsensusFactory = BitcoinConsensusFactory.Instance,
                    SupportSegwit = true,
                    SupportTaproot = true,
                })
                .SetBase58Bytes(Base58Type.PUBKEY_ADDRESS, new byte[] { 0 })
                .SetBase58Bytes(Base58Type.SCRIPT_ADDRESS, new byte[] { 5 })
                .SetBase58Bytes(Base58Type.SECRET_KEY, new byte[] { 128 })
                .SetBase58Bytes(Base58Type.EXT_PUBLIC_KEY, new byte[] { 0x04, 0x88, 0xB2, 0x1E })
                .SetBase58Bytes(Base58Type.EXT_SECRET_KEY, new byte[] { 0x04, 0x88, 0xAD, 0xE4 })
                .SetBech32(Bech32Type.WITNESS_PUBKEY_ADDRESS, Encoders.Bech32("bc"))
                .SetBech32(Bech32Type.WITNESS_SCRIPT_ADDRESS, Encoders.Bech32("bc"))
                .SetMagic(0xacb1d2db)  // bytes: db d2 b1 ac → LE uint32
                .SetPort(8333)
                .SetRPCPort(48484)
                .SetMaxP2PVersion(70016)
                .SetName("nav-main")
                .SetNetworkStringParser(new BitcoinStringParser())
                .SetGenesis("GENESIS_HEX_HERE");
                // To get genesis hex: navio-cli getblock \
                //   00000000e563d370b42d83c98b811fb1bda076dd6c2b01dac9c1e21104c25277 0
        }

        protected override NetworkBuilder CreateRegtest()
        {
            // Regtest has fBLSCT=false, not useful for Navio testing.
            // Return null network so it's not registered.
            return new NetworkBuilder()
                .SetConsensus(new Consensus()
                {
                    ConsensusFactory = BitcoinConsensusFactory.Instance,
                    SupportSegwit = true,
                    SupportTaproot = true,
                })
                .SetName("nav-reg")
                .SetGenesis("GENESIS_HEX_HERE");
        }
    }
}
```

**Important — Getting genesis hex:**

The `SetGenesis()` call requires the raw serialized genesis block in hex. Obtain it from a running Navio node:

```bash
# Testnet genesis
./navio-cli -testnet getblock \
  57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b 0

# Mainnet genesis (for later)
./navio-cli getblock \
  00000000e563d370b42d83c98b811fb1bda076dd6c2b01dac9c1e21104c25277 0
```

Replace `"GENESIS_HEX_HERE"` with the output.

#### 1b. Register in `NBitcoin.Altcoins/AltcoinNetworkSets.cs`

Add to the static properties and the `GetAll()` method:

```csharp
public static Navio Navio { get; } = Navio.Instance;
```

And in `GetAll()`:
```csharp
yield return Navio;
```

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

## Testing Checklist

- [ ] NBitcoin: Navio testnet network parses addresses correctly
- [ ] NBXplorer: indexes Navio testnet blocks and tracks transactions
- [ ] BTCPayServer: Navio appears as available cryptocurrency
- [ ] BTCPayServer: can create store with Navio payment method
- [ ] BTCPayServer: generates valid `tb1` testnet addresses
- [ ] BTCPayServer: receives testnet payment and confirms invoice
- [ ] BTCPayServer: webhook fires on payment confirmation
- [ ] BTCPayServer: refund flow works
- [ ] Docker: `naviod` container starts and syncs testnet
- [ ] Docker: NBXplorer connects to naviod and indexes chain

## PR Submission Order

PRs depend on each other. Submit in this order from each `mxaddict` fork:

1. **NBitcoin** → `mxaddict/NBitcoin:navio-support` → PR to `MetacoSA/NBitcoin:master` — Navio network definition
2. **NBXplorer** → `mxaddict/NBXplorer:navio-support` → PR to `dgarage/NBXplorer:master` — Chain registration (depends on #1 being merged/released as NuGet)
3. **BTCPayServer** → `mxaddict/btcpayserver:navio-support` → PR to `btcpayserver/btcpayserver:master` — Altcoins plugin (depends on #2)
4. **btcpayserver-docker** → `mxaddict/btcpayserver-docker:navio-support` → PR to `btcpayserver/btcpayserver-docker:master` — Docker deployment (independent, can go in parallel)

Submit each PR as a **draft** (WIP) using `gh` from inside the fork directory:

```bash
git push -u origin navio-support
gh pr create --draft --repo <upstream-owner>/<repo> \
  --title "WIP: Add Navio (NAV) testnet support" \
  --body "$(cat <<'EOF'
## Summary
- Add Navio testnet as a supported cryptocurrency
- Navio is a Bitcoin Core fork with BLSCT consensus
- Testnet only (mainnet chainparams not yet finalized)

> **Work in progress** — not ready for merge yet.
EOF
)"
```

**Do NOT mark PRs as ready for review.** The maintainer (mxaddict) will do that manually.

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
- [Navio Chain Parameters](src/kernel/chainparams.cpp)
- [Navio RPC Interface](doc/JSON-RPC-interface.md)
