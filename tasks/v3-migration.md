# Viem v3 Migration Plan

## Architecture Target

```
import { Account, Chain, Client, Key, Transport } from 'viem'
import { publicActions } from 'viem'

const client = Client.create({
  chain: Chain.mainnet,
  transport: Transport.http(),
})
  .extend(publicActions())

const blockNumber = await client.getBlockNumber()
// or standalone:
const blockNumber = await getBlockNumber(client)
```

**Key principles:**
- Module-driven namespaces: `Client.create()`, `Transport.http()`, `Chain.from()`, `Account.fromPrivateKey()`
- `declare namespace` for params/returns: `getBlockNumber.Options`, `getBlockNumber.ReturnType`
- Proxy Ox types (Address, Hex, Hash, etc.) from viem — don't re-export from abitype
- Align with Ox type names (Transaction, TransactionReceipt, TxEnvelope*)
- Chain ID is `bigint`
- Delete all `@deprecated` exports
- Delete utils that are thin Ox wrappers
- L2 extensions stay at their entrypoints (`viem/op-stack`, etc.) with same module pattern internally

## Directory Structure Target

```
src/
  core/
    internal/          # shared internal utils (uid, types, promise, errors)
    actions/           # all action implementations
      getBlockNumber.ts
      getBalance.ts
      sendTransaction.ts
      ...
    Account.ts         # Account namespace module
    Chain.ts           # Chain namespace module
    Client.ts          # Client namespace module
    Errors.ts          # Errors namespace module
    Key.ts             # Key namespace module
    Transport.ts       # Transport namespace module
  chains/              # chain definitions (Chain.mainnet re-exported)
  op-stack/            # L2 extension (same module pattern)
  zksync/              # L2 extension
  celo/                # L2 extension
  linea/               # L2 extension
  tempo/               # L2 extension
  index.ts             # main entrypoint
```

---

## Migration Phases (leaf → root)

### Phase 0: Scaffolding & Cleanup
- [ ] **0.1** Create `src/core/` directory structure (`core/`, `core/internal/`, `core/actions/`)
- [ ] **0.2** Delete all `@deprecated` exports from `src/index.ts`, `src/chains/index.ts`, `src/zksync/index.ts`, `src/actions/index.ts`
- [ ] **0.3** Remove `types/utils.ts` TODO (`NoInfer` comment)

### Phase 1: Internal Utilities (leaves — no deps)
- [ ] **1.1** `core/internal/types.ts` — move type utils from `types/utils.ts` (Compute, ExactPartial, OneOf, Prettify, etc.)
- [ ] **1.2** `core/internal/uid.ts` — move from `utils/uid.ts`
- [ ] **1.3** `core/internal/promise.ts` — move `withCache` and promise utils
- [ ] **1.4** `core/internal/errors.ts` — version helper, URL helper
- [ ] **1.5** `core/internal/lru.ts` — move LruMap from `utils/lru.ts`

### Phase 2: Errors Module
- [ ] **2.1** `core/Errors.ts` — new BaseError (aligned with viem-next2 pattern, wraps ox/Errors)
- [ ] **2.2** Migrate error classes from `errors/*.ts` into namespaced submodules or keep flat under `core/errors/`

### Phase 3: Type Alignment with Ox (leaves of types/)
- [ ] **3.1** Replace `types/misc.ts` — proxy `Hex`, `Hash`, `ByteArray`, `Signature` from Ox
- [ ] **3.2** Replace `types/fee.ts` — align with Ox fee types
- [ ] **3.3** Replace `types/authorization.ts` — align with Ox
- [ ] **3.4** Replace `types/withdrawal.ts` — align with Ox
- [ ] **3.5** Replace `types/log.ts` — align with Ox
- [ ] **3.6** Replace `types/transaction.ts` — align with Ox Transaction/TransactionReceipt/TxEnvelope* naming
- [ ] **3.7** Replace `types/block.ts` — align with Ox Block types
- [ ] **3.8** Replace `types/account.ts` — align with new Account module
- [ ] **3.9** Replace `types/chain.ts` — `Chain.id` → `bigint`
- [ ] **3.10** Replace `types/transport.ts` — align with Transport module
- [ ] **3.11** Replace `types/eip1193.ts` — align with Ox RpcSchema
- [ ] **3.12** Replace `types/rpc.ts` — align with Ox Rpc types

### Phase 4: Delete/Proxy Utils to Ox
Utils that are thin wrappers around Ox and can be deleted (consumers import from Ox directly or we proxy):

- [ ] **4.1** `utils/address/` → proxy to `ox/Address` (getAddress, isAddress, isAddressEqual, checksumAddress)
- [ ] **4.2** `utils/data/` → proxy to `ox/Hex` + `ox/Bytes` (isHex, isBytes, pad, slice, concat, size, trim)
- [ ] **4.3** `utils/encoding/` → proxy to `ox/Hex`, `ox/Bytes`, `ox/Rlp` (toHex, toBytes, fromHex, fromBytes, fromRlp, toRlp)
- [ ] **4.4** `utils/hash/` → proxy to `ox/Hash`, `ox/AbiItem` (keccak256, sha256, ripemd160, toFunctionSelector, toEventSelector)
- [ ] **4.5** `utils/signature/` → proxy to `ox/Signature`, `ox/PersonalMessage`, `ox/TypedData` (parseSignature, serializeSignature, hashMessage, hashTypedData, recoverAddress, etc.)
- [ ] **4.6** `utils/unit/` → proxy to `ox/Value` (formatUnits, parseUnits, formatEther, parseEther, formatGwei, parseGwei)
- [ ] **4.7** `utils/blob/` → proxy to `ox/Blobs`
- [ ] **4.8** `utils/ens/` → proxy to `ox/Ens`
- [ ] **4.9** `utils/authorization/` → proxy to `ox/Authorization`

Utils that stay (viem-specific logic, not in Ox):
- `utils/chain/` (chain utils), `utils/formatters/` (block/tx formatters), `utils/filters/`, `utils/errors/`, `utils/abi/`, `utils/transaction/` (serialize/parse), `utils/rpc/`, `utils/promise/`, `utils/buildRequest.ts`, `utils/ccip.ts`, `utils/getAction.ts`, `utils/observe.ts`, `utils/poll.ts`, `utils/nonceManager.ts`, `utils/stateOverride.ts`, `utils/typedData.ts`, `utils/stringify.ts`

### Phase 5: Key Module (leaf — only depends on Ox)
- [ ] **5.1** `core/Key.ts` — secp256k1 key type + `Key.from()`, `Key.fromSecp256k1()` (port from viem-next2)

### Phase 6: Account Module (depends on Key, Ox)
- [ ] **6.1** `core/Account.ts` — `Account.from()`, `Account.fromJsonRpc()`, `Account.fromLocal()`, `Account.fromPrivateKey()`, `Account.random()`
- [ ] **6.2** Migrate `accounts/toAccount.ts`, `accounts/privateKeyToAccount.ts`, `accounts/mnemonicToAccount.ts`, `accounts/hdKeyToAccount.ts` into Account module methods

### Phase 7: Chain Module (depends on internal types)
- [ ] **7.1** `core/Chain.ts` — `Chain` type with `id: bigint`, `Chain.from()`, re-export chain definitions
- [ ] **7.2** Migrate all `chains/definitions/*.ts` — update `id` from `number` to `bigint`

### Phase 8: Transport Module (depends on Chain, Errors, Ox)
- [ ] **8.1** `core/Transport.ts` — `Transport.http()`, `Transport.webSocket()`, `Transport.custom()`, `Transport.fallback()` (port from viem-next2 pattern)

### Phase 9: Client Module (depends on Account, Chain, Transport, Errors)
- [ ] **9.1** `core/Client.ts` — `Client.create()` with `.extend()` pattern
- [ ] **9.2** Remove `createPublicClient`, `createWalletClient`, `createTestClient` — replace with `Client.create().extend(publicActions())`

### Phase 10: Actions (depends on Client, Chain, types)
- [ ] **10.1** Move all `actions/public/*.ts` → `core/actions/` with `declare namespace` pattern for params/returns
- [ ] **10.2** Move all `actions/wallet/*.ts` → `core/actions/`
- [ ] **10.3** Move all `actions/test/*.ts` → `core/actions/`
- [ ] **10.4** Move all `actions/ens/*.ts` → `core/actions/`
- [ ] **10.5** Create `publicActions()` extension (decorator) — exported from `viem`
- [ ] **10.6** Create `walletActions()` extension (decorator)
- [ ] **10.7** Create `testActions()` extension (decorator)
- [ ] **10.8** Export all actions from top-level `viem` (standalone functions)

### Phase 11: Entrypoint & Barrel
- [ ] **11.1** Rewrite `src/index.ts` — export `Account`, `Chain`, `Client`, `Key`, `Transport`, `Errors` as namespace modules + all standalone actions + `publicActions`, `walletActions`
- [ ] **11.2** Update `package.json` exports map

### Phase 12: L2 Extensions
- [ ] **12.1** Migrate `op-stack/` to module pattern
- [ ] **12.2** Migrate `zksync/` to module pattern (delete deprecated exports)
- [ ] **12.3** Migrate `celo/` to module pattern
- [ ] **12.4** Migrate `linea/` to module pattern
- [ ] **12.5** Migrate `tempo/` to module pattern (address Tempo v3 TODOs)

### Phase 13: Account Abstraction
- [ ] **13.1** Migrate `account-abstraction/` to module pattern

### Phase 14: Cleanup
- [ ] **14.1** Delete old `src/clients/`, `src/accounts/`, `src/types/`, `src/errors/`, `src/constants/` (or keep as re-export shims temporarily)
- [ ] **14.2** Delete old `src/utils/` files that were proxied to Ox
- [ ] **14.3** Update all tests
- [ ] **14.4** Update all imports across codebase

---

## API Cheat Sheet (v2 → v3)

| v2 | v3 |
|---|---|
| `createPublicClient({...})` | `Client.create({...}).extend(publicActions())` |
| `createWalletClient({...})` | `Client.create({...}).extend(walletActions())` |
| `http('https://...')` | `Transport.http('https://...')` |
| `webSocket('wss://...')` | `Transport.webSocket('wss://...')` |
| `privateKeyToAccount('0x...')` | `Account.fromPrivateKey('0x...')` |
| `mnemonicToAccount('...')` | `Account.fromMnemonic('...')` |
| `defineChain({...})` | `Chain.from({...})` |
| `mainnet` (from `viem/chains`) | `Chain.mainnet` (or still from `viem/chains`) |
| `GetBlockNumberParameters` | `getBlockNumber.Options` |
| `GetBlockNumberReturnType` | `getBlockNumber.ReturnType` |
| `chain.id` (number) | `chain.id` (bigint) |
| `type Transaction` | `type Transaction` (aligned w/ Ox) |
| `type TransactionReceipt` | `type TransactionReceipt` (aligned w/ Ox) |
| `getAddress()` | proxy to `ox/Address` |
| `keccak256()` | proxy to `ox/Hash` |
| `toHex()` | proxy to `ox/Hex` |
| `formatEther()` | proxy to `ox/Value` |
