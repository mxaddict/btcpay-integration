# Navio BTCPayServer — End-to-End Test Plan

Five phases, each with hard prereqs from the previous. Run in order. Fail fast.

| Phase | Name | Deps |
|-------|------|------|
| 0 | Environment Setup | None |
| 1 | Unit Tests | .NET SDK |
| 2 | libblsct.so Integration | libblsct.so |
| 3 | Live Daemon Tests | naviod running |
| 4 | Docker Stack | Docker + libblsct.so volume |
| 5 | Payment Flow | Phase 4 stack + funded testnet wallet |

---

## Phase 0: Environment Setup

### Required Tools

| Tool | Min Version | Notes |
|------|-------------|-------|
| .NET SDK | 8.0 | `dotnet --version` |
| Docker | 24+ | `docker --version` |
| Docker Compose | v2 | `docker compose version` |
| `naviod` binary | latest testnet | for Phase 3 |
| `libblsct.so` | from libblsct-bindings | for Phases 2, 4 |

### Build libblsct.so

```bash
cd libblsct-bindings
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
# Output: build/src/libblsct.so (or .dylib on macOS)
export LIBBLSCT_SO_PATH="$(pwd)/build/src/libblsct.so"
```

### Environment Variables

| Variable | Used In | Example |
|----------|---------|---------|
| `LIBBLSCT_SO_PATH` | Phases 2, 3, 4 | `/path/to/libblsct.so` |
| `NBXPLORER_NAVTESTNET_RPCURL` | Phase 3 | `http://127.0.0.1:33577/` |
| `NBXPLORER_NAVTESTNET_RPCUSER` | Phase 3 | `navuser` |
| `NBXPLORER_NAVTESTNET_RPCPASSWORD` | Phase 3 | `navpass` |

### Verify Submodules Initialized

```bash
git submodule update --init --recursive
```

---

## Phase 1: Unit Tests

No daemon. No `libblsct.so`. Must all pass before anything else.

### 1a — NBitcoin Unit Tests

```bash
cd NBitcoin
dotnet test NBitcoin.Tests/NBitcoin.Tests.csproj \
  --filter "FullyQualifiedName~NavioTests" \
  --logger "console;verbosity=normal"
```

**Expected passing tests:**

| Test | Asserts |
|------|---------|
| `NavioNetwork_Testnet_HasCorrectGenesisHash` | `57b37639…9019e2b` |
| `NavioNetwork_Testnet_HasCorrectPorts` | P2P=33570, RPC=33577 |
| `NavioNetwork_Testnet_HasCorrectBlsctBech32HRP` | `"tnv"` encoder exists |
| `NavioNetwork_Testnet_HasCorrectBase58Prefixes` | `0x6f`, `0xc4` |
| `NavioNetwork_RegisteredInAltNetworkSets` | `AltNetworkSets.Navio` not null |
| `ConfigureBLSCTOverrides_SetsRPCMethodOverridesDictionary` | dict not null |
| `ConfigureBLSCTOverrides_RemapsAllEightMethods` | all 8 entries correct |
| `ConfigureBLSCTOverrides_DoesNotRemapCreatewallet` | no `createwallet` key |
| `ConfigureBLSCTOverrides_DoesNotRemapGetnewaddress` | no `getnewaddress` key |
| `ConfigureBLSCTOverrides_ExactlyEightMappings` | count == 8 |
| `CreateWalletOptions_BlsctProperty_DefaultsToNull` | default null |
| `CreateWalletOptions_BlsctTrue_IncludedInSerialization` | JSON has `"blsct": true` |
| `CreateWalletOptions_BlsctNull_OmittedFromSerialization` | no `"blsct"` key |
| `RPCOperations_BlsctEnums_HaveCorrectStringValues` | 15 theory cases |

### 1b — NBXplorer Unit Tests

```bash
cd NBXplorer
dotnet test NBXplorer.Tests/NBXplorer.Tests.csproj \
  --filter "FullyQualifiedName~NavioTests" \
  --logger "console;verbosity=normal"
```

**Expected passing tests:**

| Test | Asserts |
|------|---------|
| `BlsctDerivationStrategy_Parse_ValidString_ReturnsStrategy` | keys match input |
| `BlsctDerivationStrategy_Parse_MissingPrefix_ReturnsNull` | null (no "blsct:") |
| `BlsctDerivationStrategy_Parse_WrongViewKeyLength_ReturnsNull` | null (62 chars) |
| `BlsctDerivationStrategy_Parse_WrongSpendKeyLength_ReturnsNull` | null (94 chars) |
| `BlsctDerivationStrategy_Parse_NonHexViewKey_ReturnsNull` | null ('Z' chars) |
| `BlsctDerivationStrategy_Parse_NullInput_ReturnsNull` | null |
| `BlsctDerivationStrategy_ToString_RoundTrips` | exact string preserved |
| `BlsctDerivationStrategy_ToString_Parse_RoundTrip` | keys survive round-trip |
| `BlsctDerivationStrategyFactory_Parse_BlsctString_ReturnsBlsctStrategy` | correct type |
| `BlsctDerivationStrategyFactory_Parse_BlsctString_KeysMatchInput` | keys correct |
| `BlsctDerivationStrategyFactory_Parse_NonBlsctString_FallsBackToBase` | not BlsctStrategy |
| `NavioNBXplorerNetwork_Testnet_HasCorrectCryptoCode` | `"NAV"` |
| `NavioNBXplorerNetwork_Testnet_HasCorrectMinRPCVersion` | 220000 |
| `NavioNBXplorerNetwork_Testnet_CreateStrategyFactory_ReturnsBlsctFactory` | correct type |
| `NavioNBXplorerNetwork_Mainnet_CoinTypeIs0` | 0 |
| `NavioNBXplorerNetwork_Testnet_CoinTypeIs1` | 1 |

### 1c — BTCPayServer Unit Tests

```bash
cd btcpayserver
dotnet test BTCPayServer.Tests/BTCPayServer.Tests.csproj \
  --filter "FullyQualifiedName~NavioPluginTests" \
  --logger "console;verbosity=normal"
```

**Expected passing tests:**

| Test | Asserts |
|------|---------|
| `NavioBTCPayNetwork_IsBLSCT_IsTrue` | `IsBLSCT == true` |
| `BitcoinBTCPayNetwork_IsBLSCT_IsFalse` | `IsBLSCT == false` |
| `DerivationSchemeViewModel_IsBLSCT_DefaultsFalse` | default false |
| `ParseDerivationStrategy_RawAuditKey_160HexChars_ConvertsToBlsctFormat` | `blsct:VIEW:SPEND` |
| `ParseDerivationStrategy_AlreadyBlsctFormat_PassesThrough` | unchanged |
| `ParseDerivationStrategy_WrongLength_IsHandled` | no crash |
| `SetupWallet_ForNavio_SetsIsBLSCT_True` | view model flag set |
| `ImportWallet_ForNavio_SetsIsBLSCT_True` | view model flag set |

### Phase 1 Pass Criteria

All unit tests green. Zero failures, zero errors. Skips only for tests explicitly
marked `[Fact(Skip = "...")]`.

---

## Phase 2: libblsct.so Integration Tests

Prereq: `LIBBLSCT_SO_PATH` set and file exists.

```bash
test -f "$LIBBLSCT_SO_PATH" && echo "OK" || echo "MISSING — set LIBBLSCT_SO_PATH"
```

### 2a — libblsct-bindings C# P/Invoke Tests

```bash
cd libblsct-bindings/ffi/csharp
dotnet test tests/ \
  --filter "FullyQualifiedName~BlsctIntegrationTests" \
  --logger "console;verbosity=normal"
```

**Expected passing tests:**

| Test | Asserts |
|------|---------|
| `GenSubAddrId_ReturnsNonZeroHandle` | handle != IntPtr.Zero |
| `DeriveSubAddress_ValidKeys_ReturnsNonZeroHandle` | handle != IntPtr.Zero |
| `EncodeAddress_Testnet_StartsWithTnv1` | prefix `tnv1` |
| `EncodeAddress_Mainnet_StartsWithNav1` | prefix `nav1` |
| `DecodeAddress_RoundTrips` | decoded bytes == original |
| `FullPipeline_MatchesFixture` | skip until vectors generated (see 2c) |

### 2b — NBXplorer BLSCT Derivation Tests

```bash
cd NBXplorer
dotnet test NBXplorer.Tests/NBXplorer.Tests.csproj \
  --filter "FullyQualifiedName~BlsctDerivationTests" \
  --logger "console;verbosity=normal"
```

**Expected passing tests:**

| Test | Asserts |
|------|---------|
| `DeriveBlsctAddress_IsDeterministic` | same inputs → same output |
| `DeriveBlsctAddress_Testnet_StartsWithTnv1` | prefix `tnv1` |
| `DeriveBlsctAddress_Mainnet_StartsWithNav1` | prefix `nav1` |
| `DeriveBlsctAddress_DifferentAccount_ProducesDifferentAddress` | account 0 ≠ account 1 |
| `DeriveBlsctAddress_DifferentIndex_ProducesDifferentAddress` | index 0 ≠ index 1 |
| `DeriveBlsctAddress_ChangeAccount_NegativeOne_Differs` | receive ≠ change |
| `DeriveBlsctAddress_MatchesFixture` | skip until vectors generated (see 2c) |

### 2c — Generate Test Vectors

Run once after `libblsct.so` is built. Generates `blsct_vectors.json`.

```bash
cd NBXplorer
dotnet run --project NBXplorer.Tests/NavioVectorGenerator \
  --so-path "$LIBBLSCT_SO_PATH" \
  --output NBXplorer.Tests/TestData/blsct_vectors.json

# Copy to libblsct-bindings test data
cp NBXplorer/NBXplorer.Tests/TestData/blsct_vectors.json \
   libblsct-bindings/ffi/csharp/tests/TestData/blsct_vectors.json
```

Input parameters for vectors (derived from `external_api_tests.cpp` seeds):

| viewKey | spendKey | account | index | hrp |
|---------|----------|---------|-------|-----|
| `gen_scalar(11)` (32 bytes) | `gen_scalar(12)` (48 bytes) | 0 | 0 | `tnv` |
| same | same | 0 | 1 | `tnv` |
| same | same | -1 | 0 | `tnv` |
| same | same | 0 | 0 | `nav` |

After generating, commit both JSON files:

```bash
git -C NBXplorer add NBXplorer.Tests/TestData/blsct_vectors.json
git -C libblsct-bindings add ffi/csharp/tests/TestData/blsct_vectors.json
```

Then re-run Phase 2a and 2b — `MatchesFixture` tests must now pass (remove
`[Fact(Skip = ...)]` marker first).

### Phase 2 Pass Criteria

- All non-fixture tests green
- Fixture tests green after vector generation
- Both `blsct_vectors.json` files committed

---

## Phase 3: Live Daemon Tests

Prereq: `naviod` running on testnet with known RPC creds.

### 3a — Start naviod

```bash
naviod -testnet \
  -rpcuser=navuser \
  -rpcpassword=navpass \
  -rpcport=33577 \
  -rpcallowip=127.0.0.1 \
  -daemon

# Verify running
navio-cli -testnet getblockchaininfo | grep '"chain": "test"'
```

### 3b — Verify Daemon Basics

```bash
# Correct genesis hash
navio-cli -testnet getblockhash 0
# Expected: 57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b

# Correct chain
navio-cli -testnet getblockchaininfo | grep chain
# Expected: "chain": "test"
```

### 3c — Run NBXplorer Daemon Tests

```bash
export NBXPLORER_NAVTESTNET_RPCURL="http://127.0.0.1:33577/"
export NBXPLORER_NAVTESTNET_RPCUSER="navuser"
export NBXPLORER_NAVTESTNET_RPCPASSWORD="navpass"

cd NBXplorer
dotnet test NBXplorer.Tests/NBXplorer.Tests.csproj \
  --filter "FullyQualifiedName~NavioDaemonTests" \
  --logger "console;verbosity=normal"
```

**Expected passing tests:**

| Test | Asserts |
|------|---------|
| `EnsureWalletCreated_ForNavio_CreatesBlsctWallet` | `getwalletinfo["blsct"] == true` |
| `GetBalance_RoutesTo_Getblsctbalance` | no `RPCException` |
| `Listunspent_RoutesTo_Listblsctunspent` | result is `JArray`, no exception |
| `NavioChain_IndexesBlocks` | `ChainHeight > 0` within 30s |

### 3d — Mine BLSCT Blocks and Verify UTXOs

```bash
# Get a BLSCT receive address from the wallet
BLSCT_ADDR=$(navio-cli -testnet getnewaddress "" blsct)
echo "BLSCT address: $BLSCT_ADDR"
# Expected: starts with tnv1

# Mine 10 blocks to that address
navio-cli -testnet generatetoblsctaddress 10 "$BLSCT_ADDR"

# Wait for 1 confirmation (already mined), check UTXOs
navio-cli -testnet listblsctunspent
# Expected: non-empty array with confirmed outputs

# Check balance
navio-cli -testnet getblsctbalance
# Expected: non-zero decimal
```

### Phase 3 Pass Criteria

- All `NavioDaemonTests` pass
- `getnewaddress` returns `tnv1...` address
- `listblsctunspent` returns funded UTXOs after mining
- `getblsctbalance` returns correct non-zero balance

---

## Phase 4: Docker Stack

Prereq: `libblsct.so` built, Docker running.

### 4a — Configure and Start Stack

```bash
cd btcpayserver-docker

# Set libblsct.so path for volume mount
export LIBBLSCT_SO_PATH="/path/to/libblsct.so"

# Start with Navio testnet fragment
BTCPAYGEN_CRYPTO1="nav" \
BTCPAYGEN_NETWORK="testnet" \
NAVIO_LIBBLSCT_PATH="$LIBBLSCT_SO_PATH" \
docker compose -f Generated/docker-compose.generated.yml up -d
```

### 4b — Verify naviod Container

```bash
# Container running
docker compose ps naviod
# Expected: Up

# Chain syncing
docker compose exec naviod navio-cli -testnet getblockchaininfo | grep blocks
# Expected: "blocks": <N> increasing

# Correct genesis
docker compose exec naviod navio-cli -testnet getblockhash 0
# Expected: 57b37639169f354fd61978f8e88db8d7da085c1c6ac4e625c5d018b0d9019e2b
```

### 4c — Verify navio-cli.sh Wrapper

```bash
./navio-cli.sh -testnet getblockchaininfo
# Expected: JSON with "chain": "test"

./navio-cli.sh -testnet getnetworkinfo | grep subversion
# Expected: Navio version string
```

### 4d — Verify NBXplorer Connects

```bash
# Check NBXplorer logs for Navio
docker compose logs nbxplorer | grep -i "nav\|navio"
# Expected: connection established, indexing started

# Check NAV chain status via NBXplorer API
curl http://localhost:24445/v1/cryptos/NAV/status | python3 -m json.tool
# Expected: { "cryptoCode": "NAV", "isFullySynced": false (or true), "chainHeight": N }
```

### 4e — Verify BTCPayServer Has Navio

```bash
# Check BTCPay logs for Navio registration
docker compose logs btcpayserver | grep -i "navio\|NAV"
# Expected: Navio registered as supported network

# Open BTCPayServer in browser
open http://localhost:3000
# Navigate: Server Settings → Currencies — verify "NAV" listed
```

### Phase 4 Pass Criteria

- `naviod` container up and syncing blocks
- `navio-cli.sh` wrapper returns valid JSON
- NBXplorer `/v1/cryptos/NAV/status` returns valid response with `chainHeight > 0`
- BTCPayServer UI shows Navio as available currency

---

## Phase 5: Full Payment Flow

Prereq: Phase 4 stack running. Need a second funded testnet wallet to send from
(separate `naviod` instance or CLI wallet with testnet BLSCT coins).

### 5a — Create Store with Navio Payment Method

1. Log into BTCPayServer at `http://localhost:3000`
2. Create new store: **Stores → Create Store**
3. Go to: **Store Settings → Onchain → Setup NAV wallet**
4. Get audit key from CLI:
   ```bash
   navio-cli -testnet getblsctauditkey
   # Returns: <160-char hex> or blsct:VIEW:SPEND format
   ```
5. Paste audit key into BTCPayServer wallet setup form
6. Submit — verify stored as `blsct:VIEW_HEX:SPEND_HEX`

**Assert:** Store wallet shows Navio, no error on save.

### 5b — Generate Invoice and Get Address

1. Go to: **Stores → Create Invoice**
2. Set amount (e.g., 0.001 NAV), currency NAV
3. Create invoice
4. View payment address

**Assert:** Address starts with `tnv1` (BLSCT testnet bech32m).

### 5c — Send Payment

From a funded external testnet wallet:

```bash
# On external wallet node
navio-cli -testnet sendtoblsctaddress "<tnv1_address_from_invoice>" 0.001
# Returns: txid
```

**Assert:** Transaction broadcasts without error.

### 5d — Confirm Payment

Wait for 1 confirmation (BLSCT has no unconfirmed UTXO visibility):

```bash
# On external node: mine a block
navio-cli -testnet generatetoblsctaddress 1 "<any_tnv1_addr>"
```

Refresh BTCPayServer invoice page.

**Assert:** Invoice status changes to **Paid**.

### 5e — Verify Webhook

If webhook configured:

```bash
# Check webhook delivery in BTCPayServer
# Store Settings → Webhooks → Deliveries
# Expected: POST with { "type": "InvoiceSettled", ... }
```

Or check logs:

```bash
docker compose logs btcpayserver | grep -i "webhook\|invoicesettled"
```

**Assert:** Webhook delivered with `InvoiceSettled` event.

### 5f — Refund Flow

1. In BTCPayServer UI: open paid invoice → **Issue Refund**
2. Enter refund destination BLSCT address
3. Submit refund

**Assert:**

```bash
# On naviod: verify sendtoblsctaddress was called
docker compose logs naviod | grep sendtoblsctaddress
```

Refund transaction appears in naviod mempool:

```bash
navio-cli -testnet getmempoolinfo
# Expected: "size" > 0
```

### Known Limitation: Transaction History

`listblscttransactions` is a stub in navio-core — returns empty array. BTCPayServer
transaction history tab for Navio will be empty even after confirmed payments.
This is a navio-core limitation, not a BTCPayServer bug. Document in the PR.

```
src/blsct/wallet/rpc.cpp:1102-1112  ← commented-out implementation
```

### Phase 5 Pass Criteria

- [ ] Invoice generates valid `tnv1` address
- [ ] Payment sends without RPC error
- [ ] Invoice marks **Paid** after 1 confirmation
- [ ] Webhook delivers `InvoiceSettled` event
- [ ] Refund submits without error
- [ ] Transaction history empty (expected — document as known limitation)

---

## Test Execution Matrix

| Test Suite | Needs `libblsct.so` | Needs Daemon | Run in CI |
|------------|---------------------|--------------|-----------|
| NBitcoin `NavioTests` | No | No | Yes |
| NBXplorer `NavioTests` (unit) | No | No | Yes |
| NBXplorer `BlsctDerivationTests` | Yes | No | Conditional (`LIBBLSCT_SO_PATH` set) |
| NBXplorer `NavioDaemonTests` | Yes | Yes | No (manual) |
| BTCPayServer `NavioPluginTests` | No | No | Yes |
| libblsct-bindings `BlsctTests` (existing) | No | No | Yes |
| libblsct-bindings `BlsctIntegrationTests` | Yes | No | Conditional (`LIBBLSCT_SO_PATH` set) |

CI runs all "No" rows unconditionally. "Conditional" rows run only when
`LIBBLSCT_SO_PATH` is set. "Manual" rows are pre-PR validation against live testnet.

---

## PR Readiness Checklist

All phases must be green before opening upstream PRs.

- [ ] Phase 1: all unit tests pass
- [ ] Phase 2: all `libblsct.so` tests pass + vectors committed
- [ ] Phase 3: all daemon tests pass + UTXOs visible after mining
- [ ] Phase 4: Docker stack starts cleanly, NBXplorer syncs NAV chain
- [ ] Phase 5: full payment flow confirmed on testnet
- [ ] Known limitation documented in each PR: `listblscttransactions` stub

**Upstream PR order:** libblsct-bindings → NBitcoin → NBXplorer → BTCPayServer.
btcpayserver-docker can go in parallel with NBitcoin–BTCPayServer.
See [PRs.md](PRs.md) for draft PR links.
