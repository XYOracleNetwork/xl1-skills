# Headless dApp Verification

Read this pattern when you need to prove that a dApp's chain interactions work end-to-end without launching a browser or driving the wallet extension. This is the verification mode of choice for agentic development, CI smoke tests, regression scripts, and any context where browser automation is overkill.

**Builds on:**
- [Node Gateway](../xl1-knowledge/gateway-node.md) — `GatewayBuilder` and the seed-phrase signer
- [XL1 Identity & Wallets](../xl1-knowledge/identity.md) — canonical `generateXyoBaseWalletFromPhrase` + `DEFAULT_WALLET_PATH` derivation
- [Gateway](../xl1-knowledge/gateway.md) — viewer API, transaction methods, capability detection

---

## The Ethos

A dApp's UX may live in the browser — wallet popups, React components, gateway providers — but its **on-chain behaviour does not depend on any of that**. The chain doesn't care whether a transaction was signed by the Chrome extension or by a Node process holding the same seed. Both paths submit the same `TransactionBoundWitness` to the same gateway RPC.

That symmetry is the verification surface:

> Anything a connected browser user can do, a headless Node script holding the same seed can do too.

Build the dApp the way users will use it (browser wallet, React, providers). Verify it the way agents and CI can drive it (Node + `GatewayBuilder.build(signer)` + a seed phrase from `.env`). When both paths produce the same on-chain outcome, you have high-confidence verification without the operational cost of a headless browser.

---

## When to Use

- **Agent-driven development** — an agent building a dApp can run the same end-to-end flow it expects users to run, without needing browser automation or a manually-installed extension.
- **CI smoke tests** — assert that a deployment can submit and read back at least one transaction per release.
- **Regression scripts** — replay a known-good user flow against a new build before promoting it.
- **Multi-account scenarios** — verify atomic exchanges, commit-reveal pairs, or other multi-party flows by deriving multiple accounts from the same seed (or different seeds) and orchestrating them deterministically from one process.

The pattern complements browser testing — it does not replace user-facing UX validation. A passing headless run says "the chain side works"; it does not say "the UI works".

---

## Prompt Shape

Agents recognise this pattern from prompts of the form:

> Given the `.env` file in this empty repository which holds a seed phrase for a wallet with Sequence XL1 in accounts 0 and 1, build me a [dApp] …

The cues:

- An `.env` file with a seed phrase is the signal that headless verification is expected.
- A note about which **accounts** (indices) hold funds tells the agent how to derive signers.
- The network (mainnet / sequence / local) tells the agent which `DefaultNetworks` entry to point at.

The agent should: (1) build the dApp normally, with browser wallet wiring intact; (2) write a Node verification script that exercises the full happy path through `GatewayBuilder.build(signer)`; (3) run that script to prove the dApp works before reporting completion.

---

## Setup

### 1. Configure environment

```sh
# .env
XL1_NETWORK=sequence
XL1_SEED_PHRASE="word1 word2 ... word12"
```

Treat the seed phrase like any other secret — never commit, never log, never echo. Load with `dotenv/config` at the very top of the script.

### 2. Derive signers from the seed

```ts
import 'dotenv/config'

import {
  DEFAULT_WALLET_PATH, DefaultNetworks, GatewayBuilder,
  generateXyoBaseWalletFromPhrase, NetworkDataLakeUrls,
} from '@xyo-network/xl1-sdk'
import { type XyoGatewayRunner } from '@xyo-network/xl1-protocol-lib'

const id = process.env.XL1_NETWORK ?? 'sequence'
const network = DefaultNetworks.find((n) => n.id === id)
if (!network) throw new Error(`Unknown network "${id}"`)

const baseWallet = await generateXyoBaseWalletFromPhrase(process.env.XL1_SEED_PHRASE!)

// Account 0 — primary actor
const player1 = await baseWallet.derivePath(DEFAULT_WALLET_PATH)
// Account 1 — counterparty (m/44'/60'/0'/0/1)
const player2 = await baseWallet.derivePath("m/44'/60'/0'/0/1")
```

These addresses match what MetaMask and the XYO browser extension show for accounts 1 and 2 on the same seed. That alignment is the whole point — the headless run is provably the same identity a browser user would hold.

### 3. Build a runner per signer

Each signer needs its own `GatewayBuilder.build(signer)` call. Cache them with the lazy-promise pattern from [Node Gateway — Caching](../xl1-knowledge/gateway-node.md#caching) if the script reuses them across phases.

```ts
const runner1: XyoGatewayRunner = await new GatewayBuilder()
  .name(`${id}-player1`)
  .rpcUrl(`${network.url}/rpc`)
  .dataLakeEndpoint(NetworkDataLakeUrls[id])
  .build(player1)

const runner2: XyoGatewayRunner = await new GatewayBuilder()
  .name(`${id}-player2`)
  .rpcUrl(`${network.url}/rpc`)
  .dataLakeEndpoint(NetworkDataLakeUrls[id])
  .build(player2)
```

Use distinct `.name()` values so logs and traces can tell the actors apart.

---

## Verification Script Shape

A headless verification script is a deterministic happy-path replay of one user flow. Keep it linear and explicit — assertions over abstractions.

```ts
// 1. Pre-flight: confirm both accounts have balance
const balance1 = await runner1.connection.viewer?.account.balance.accountBalance(await player1.address())
if (!balance1 || balance1 === 0n) throw new Error('player1 has no XL1 — fund the account first')

// 2. Submit the dApp's actual on-chain action through the same code path the UI uses.
//    Import the dApp's domain functions — do not rebuild logic in the script.
const [txHash] = await submitMove(runner1, { game: 'rps', choice: 'rock', salt })

// 3. Wait for inclusion
const confirmed = await runner1.confirmSubmittedTransaction(txHash)

// 4. Read back through the viewer to verify shape
const tx = await runner1.connection.viewer?.transaction.byHash(txHash)
if (!tx) throw new Error('transaction not found after confirmation')

// 5. If the flow is multi-party, drive the counterparty through runner2 and assert outcome
```

**Import the dApp's own functions.** A verification script that re-implements payload construction or transaction submission proves nothing — it only proves the script works. The script is valuable because it exercises *the same code* the UI calls. Domain functions (`submitMove`, `revealMove`, `settleGame`, etc.) should accept a runner as a parameter so they work in both contexts.

---

## Cross-Environment Identity Guarantee

Because the script derives via `generateXyoBaseWalletFromPhrase` + `DEFAULT_WALLET_PATH` (and sibling indices), the signing identity is bit-for-bit the identity a browser user would have after importing the same seed into the XYO Chrome wallet or MetaMask. Implications:

- A developer can fund the seed in MetaMask, then a CI script using the same seed can submit transactions from those funded accounts. No address mismatch, no separate funding step.
- An agent can set up the seed once in `.env`, exercise the dApp headlessly, then hand the seed to a human reviewer who imports it into the browser wallet and continues from the same state.
- Multi-account flows derived in the script (`m/44'/60'/0'/0/1`, `…/0/2`, …) match accounts 2, 3, … in MetaMask on the same seed.

If addresses do not line up, the script bypassed the canonical helpers — the failure is in the script, not the chain. See [Identity & Wallets — Anti-Patterns](../xl1-knowledge/identity.md#anti-patterns).

---

## Anti-Patterns

| Anti-Pattern | Why it fails | Do this instead |
|---|---|---|
| Re-implementing transaction logic inside the verification script | Verifies the script, not the dApp — false confidence | Import the dApp's domain functions; pass the runner in |
| `Account.create({ mnemonic })` for the headless signer | Produces an address that won't match MetaMask / XYO extension on the same seed | Use `generateXyoBaseWalletFromPhrase` + `derivePath` |
| Generating a fresh random wallet at script start | Identity changes every run; impossible to fund or reproduce | Load seed from `.env` and derive deterministically |
| Logging or committing the seed phrase | Catastrophic if the repo or CI logs are exposed | Treat the seed like any other secret; load via `dotenv/config`; never `console.log` |
| Building one runner and pretending it represents both parties | Multi-party flows (commit-reveal, atomic exchange) need distinct signers to be meaningful | Derive each party from a different index; build a runner per signer |
| Skipping the read-back step after submission | Confirms the chain accepted the tx, not that the data is queryable as the UI expects | Always round-trip via `connection.viewer` to assert the shape the UI will read |
| Pointing the script at `mainnet` for routine verification | Real funds, real chain pressure | Default to `sequence` in `.env`; require an explicit override for mainnet runs |

---

## Cross-References

- [Node Gateway](../xl1-knowledge/gateway-node.md) — `GatewayBuilder` API, caching, advanced locator escape hatch
- [XL1 Identity & Wallets](../xl1-knowledge/identity.md) — canonical seed-phrase derivation and the cross-environment guarantee
- [Gateway — Submitting Transactions](../xl1-knowledge/gateway.md#submitting-transactions) — `addPayloadsToChain`, `send`, `confirmSubmittedTransaction`
- [Gateway — Reading Chain State](../xl1-knowledge/gateway.md#reading-chain-state) — viewer sub-viewers used for read-back assertions
- [dApp Definition of Done](dapp-checklist.md) — broader pre-ship checklist this verification step plugs into
