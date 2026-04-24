# Gateway

**Key npm packages:**
- `@xyo-network/xl1-rpc` — RPC type definitions, Zod schemas, engine handlers
- `@xyo-network/xl1-providers` — Browser, Node, and Neutral provider implementations

Note: The gateway API server itself is part of the `xyo-chain` runtime repo (not published as a standalone npm package). The packages above cover the client-side RPC and provider interfaces needed for dApp development.

---

## XL1 Gateway

The gateway is a JSON-RPC 2.0 API server that exposes XL1 chain data and operations.

### Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/rpc` | POST | JSON-RPC 2.0 — all viewer and runner methods |
| `/chain` | Various | Archivist middleware for finalized chain data (datalake) |
| `/startupz` | GET | Startup health probe |
| `/readyz` | GET | Readiness health probe |
| `/livez` | GET | Liveness health probe |

---

## Networks

XL1 has three networks. The gateway name (`'mainnet'`, `'sequence'`, `'local'`) is what you pass to `WalletGatewayProvider` (wallet-only) or `GatewayProvider` (hybrid, with `InPageGatewaysProvider` for read-only fallback). The SDK's `DefaultNetworks` maps these to the correct URLs automatically.

| Network | Gateway Name | Gateway RPC | Datalake | Explorer |
|---------|-------------|-------------|----------|----------|
| **Mainnet** | `'mainnet'` | `https://api.chain.xyo.network/rpc` | `https://api.archivist.xyo.network/dataLake` | `https://explore.xyo.network` |
| **Sequence** (beta) | `'sequence'` | `https://beta.api.chain.xyo.network/rpc` | `https://beta.api.archivist.xyo.network/dataLake` | `https://beta.explore.xyo.network` |
| **Local** | `'local'` | `http://localhost:8080/rpc` | `http://localhost:8080/dataLake` | `http://localhost:3000` |

**When to use each:**
- **Mainnet** — production deployments. Real XL1 tokens, real transactions.
- **Sequence** — testing and staging. Use this for development against a live network without affecting production. This is the default for beta/staging deployments.
- **Local** — local development with a locally running gateway (`xl1 start api`). No network dependency.

For dApp development, start with **Sequence** (beta) to test against a live chain, then switch to **Mainnet** for production.

---

## RPC Method Namespaces

Methods follow the pattern `<namespace>_<methodName>`. The **Wire Name** column shows the JSON-RPC method string (transport layer). The **TypeScript API** column shows the typed accessor on `gateway.connection.viewer` — this is what application code should use. See [Gateway Usage](../xl1-patterns/gateway-usage.md) for full usage examples.

**`connection.viewer` is optional** (`XyoViewer | undefined`). The in-page gateway populates it once it finishes resolving, but a wallet-only or runner-only gateway may not have a viewer. Always guard access with `?.` or an explicit null check.

### Block Queries (`blockViewer_*`) — `connection.viewer.block`

| Wire Name | TypeScript API | Parameters | Returns |
|-----------|---------------|-----------|---------|
| `blockViewer_blocksByHash` | `.block.blocksByHash(...)` | `(hash, limit?)` | `SignedHydratedBlockWithHashMeta[]` |
| `blockViewer_blocksByNumber` | `.block.blocksByNumber(...)` | `(block, limit?)` | `SignedHydratedBlockWithHashMeta[]` |
| `blockViewer_blockByHash` | `.block.blockByHash(...)` | `(hash)` | `SignedHydratedBlockWithHashMeta \| null` |
| `blockViewer_blockByNumber` | `.block.blockByNumber(...)` | `(block)` | `SignedHydratedBlockWithHashMeta \| null` |
| `blockViewer_currentBlock` | `.block.currentBlock()` | `()` | `SignedHydratedBlockWithHashMeta` |
| `blockViewer_currentBlockNumber` | `.block.currentBlockNumber()` | `()` | `XL1BlockNumber` |
| `blockViewer_currentBlockHash` | `.block.currentBlockHash()` | `()` | `Hash` |
| `blockViewer_chainId` | `.block.chainId(...)` | `(blockNumber?)` | `ChainId` |
| `blockViewer_payloadsByHash` | `.block.payloadsByHash(...)` | `(hashes)` | `WithHashMeta<Payload>[]` |

### Transaction Queries (`transactionViewer_*`) — `connection.viewer.transaction`

| Wire Name | TypeScript API | Parameters | Returns |
|-----------|---------------|-----------|---------|
| `transactionViewer_byHash` | `.transaction.byHash(...)` | `(txHash)` | `SignedHydratedTransactionWithHashMeta \| null` |
| `transactionViewer_byBlockHashAndIndex` | `.transaction.byBlockHashAndIndex(...)` | `(blockHash, index)` | `SignedHydratedTransactionWithHashMeta \| null` |
| `transactionViewer_byBlockNumberAndIndex` | `.transaction.byBlockNumberAndIndex(...)` | `(blockNumber, index)` | `SignedHydratedTransactionWithHashMeta \| null` |

### Account Balances (`accountBalanceViewer_*`) — `connection.viewer.account.balance`

| Wire Name | TypeScript API | Parameters | Returns |
|-----------|---------------|-----------|---------|
| `accountBalanceViewer_accountBalance` | `.account.balance.accountBalance(...)` | `(address, config?)` | `AttoXL1` |
| `accountBalanceViewer_accountBalances` | `.account.balance.accountBalances(...)` | `(addresses, config?)` | `Record<Address, AttoXL1>` |
| `accountBalanceViewer_accountBalanceHistory` | `.account.balance.accountBalanceHistory(...)` | `(address, config?)` | `AccountBalanceHistoryItem[]` |

### Finalization (`finalizationViewer_*`) — `connection.viewer.finalization`

| Wire Name | TypeScript API | Parameters | Returns |
|-----------|---------------|-----------|---------|
| `finalizationViewer_head` | `.finalization.head()` | `()` | `SignedHydratedBlockWithHashMeta` |
| `finalizationViewer_headNumber` | `.finalization.headNumber()` | `()` | `XL1BlockNumber` |
| `finalizationViewer_headHash` | `.finalization.headHash()` | `()` | `Hash` |
| `finalizationViewer_chainId` | `.finalization.chainId()` | `()` | `ChainId` |

### Mempool (`mempoolViewer_*` / `mempoolRunner_*`) — `connection.viewer.mempool`

| Wire Name | TypeScript API | Parameters | Returns |
|-----------|---------------|-----------|---------|
| `mempoolViewer_pendingBlocks` | `.mempool.pendingBlocks(...)` | `(options?)` | `SignedHydratedBlockWithHashMeta[]` |
| `mempoolViewer_pendingTransactions` | `.mempool.pendingTransactions(...)` | `(options?)` | `SignedHydratedTransactionWithHashMeta[]` |
| `mempoolRunner_submitBlocks` | _(via MempoolRunner)_ | `(blocks)` | `Hash[]` |
| `mempoolRunner_submitTransactions` | _(via MempoolRunner)_ | `(txs)` | `Hash[]` |

### Staking (`stakeViewer_*`) — `connection.viewer.stake`

| Wire Name | TypeScript API | Parameters | Returns |
|-----------|---------------|-----------|---------|
| `stakeViewer_stakeById` | `.stake.stakeById(...)` | `(id)` | `Position` |
| `stakeViewer_stakesByStaker` | `.stake.stakesByStaker(...)` | `(staker)` | `Position[]` |
| `stakeViewer_stakesByStaked` | `.stake.stakesByStaked(...)` | `(staked)` | `Position[]` |
| `stakeViewer_activeStakes` | `.stake.activeStakes()` | `()` | `Position[]` |

### Additional Sub-Viewers

The `XyoViewer` interface also exposes these sub-viewers, which do not have direct RPC wire equivalents but are available on `connection.viewer`:

| Sub-viewer | Type | Purpose |
|------------|------|---------|
| `.networkStake` | `NetworkStakeViewer` | Network-level staking queries |
| `.step` | `StepViewer` | Step/epoch queries |
| `.time` | `TimeSyncViewer` | Time synchronization queries |

### Transaction Operations (`xyoRunner_*` / `xyoSigner_*`)

| Wire Name | Parameters | Returns |
|-----------|-----------|---------|
| `xyoRunner_broadcastTransaction` | `(tx)` | `Hash` |
| `xyoSigner_address` | `()` | `Address` |
| `xyoSigner_signTransaction` | `(tx)` | `SignedHydratedTransactionWithHashMeta` |

These are internal to the wallet/runner transport. Application code uses `gateway.addPayloadsToChain()`, `gateway.send()`, etc. — see [Gateway Usage — Submitting Transactions](../xl1-patterns/gateway-usage.md).

---

## Connection Properties

The gateway object (`XyoGateway` or `XyoGatewayRunner`) exposes chain access through `gateway.connection`:

| Property | Type | Description |
|----------|------|-------------|
| `.viewer` | `XyoViewer \| undefined` | Read-only chain state (sub-viewers for blocks, transactions, balances, etc.) |
| `.storage` | `DataLakeViewer \| undefined` | Read-only datalake attached to this connection. May not point to the dApp's desired endpoint. |
| `.runner` | `XyoRunner \| undefined` | Low-level runner (internal — use gateway methods instead) |
| `.network` | `XyoNetwork \| undefined` | Network metadata |

**`connection.storage` is not the recommended datalake path.** It is a read-only `DataLakeViewer` populated from the connection's configuration — it cannot write, and it may not point to the endpoint the dApp wants to use. For datalake access, create standalone `RestDataLakeRunner` / `RestDataLakeViewer` clients. See [Gateway Usage — Accessing the Datalake](../xl1-patterns/gateway-usage.md) and [Datalakes](datalakes.md).

---

## Transports

| Transport | Use Case |
|-----------|----------|
| `HttpRpcTransport` | Network — connect to a remote gateway over HTTP |
| `PostMessageRpcTransport` | Browser — cross-window communication (wallet ↔ dApp) |
| `MemoryRpcTransport` | Testing — in-memory JSON-RPC engine |

---

## Providers

`@xyo-network/xl1-providers` offers environment-specific provider bundles:

- **Browser provider** — for web dApps, uses PostMessage transport
- **Node provider** — for backend services, uses HTTP transport
- **Neutral provider** — platform-agnostic

Use `buildProviderLocator()` to wire up the standard provider dependency tree.

---

## Running the Gateway

### Via CLI
```bash
xl1 start api                    # Start API server only
xl1 start api producer validator # Start multiple actors
```

### Via pnpm (from xyo-chain repo)
```bash
pnpm run-api
```

### Configuration

100% environment-driven via `XL1_*` variables:

| Variable | Purpose |
|----------|---------|
| `XL1_CHAIN__ID` | Staking contract address on backing EVM |
| `XL1_EVM__CHAIN_ID` | EVM chain (Sepolia, Mainnet, Ganache) |
| `XL1_ACTORS__API_*` | API server configuration |
| `XL1_STORAGE__ROOT` | LMDB data directory |
| `XL1_STORAGE__MONGO__*` | MongoDB connection settings |
