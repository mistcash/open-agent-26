# Agents with MIST

Autonomous agents are great at deciding to spend money. They are terrible at spending it privately. Every payment they make on a public chain leaks the wallet behind the agent, the counterparty, the amount, and — by correlation across transactions — the entire strategy. **Agents with MIST** is a toolkit that fixes that. It gives an AI agent the same set of primitives a human gets when they reach for cash: a way to receive a payment without exposing identity, a way to pay a request, a way to escrow funds against a condition, and a way to settle bilaterally with another agent — all over [MIST.cash](https://mist.cash)'s shielded pool, all behind a single TypeScript class, and all reachable from an LLM via plain tool calls.

The hard part of using a shielded pool isn't the cryptography — it's the choreography around it. Per-payment hiding keys, claiming keys, owner secrets, merkle proofs, Groth16 witness construction, two-proof glue between escrow and spend, status polling against an append-only tx tree. Doing any of that wrong leaks the very privacy you went looking for. So the contributions here are about packaging that choreography into something an agent can use without thinking: a deterministic key tree rooted in one master key, a recipient-bound escrow circuit that makes bilateral swaps both atomic and snipe-proof, a thin Solidity wrapper that fuses two ZK proofs into a single transaction, and a runner that turns "private payment" into a tool the model just calls.

## Directories

| Path | What's there |
| --- | --- |
| [`contracts/`](contracts) | Solidity `Chamber` (the MIST shielded pool, ported from Cairo) plus the `Escrow` wrapper and Groth16 verifiers. |
| [`zk/`](zk) | The gnark circuits — including the recipient-bound escrow circuit at the heart of bilateral settlement. |
| [`sdk/`](sdk) | TypeScript SDK exposing `MISTActions`, proof generation, and typed contract bindings. |
| [`runner/`](runner) | Vercel-AI-SDK agent runner — drop in a persona folder + `.env`, get an agent that pays, requests, and escrows privately. |
| [`frontend/`](frontend) | Vite/React + wagmi UI for human-driven requests, deposits, and withdrawals on 0G testnet. |
| [`agent/axl/`](agent/axl) | Gensyn AXL node configs and bootstrap script for the encrypted agent mesh. |

---

## `MISTActions` — private payments as a single object

Everything an agent needs lives behind `MISTActions` in [`sdk/src/index.ts`](sdk/src/index.ts). You hand it a master key and a `ChainAdapter`; it derives every per-payment hiding key, tracks request status, builds the Groth16 witness, generates the proof, and submits the right calldata. State is pure derivation — losing the persistence layer is recoverable from the master key alone, but two requests from the same agent are still cryptographically unlinkable on-chain.

```
masterKey
  ├── masterHidingKey  = h2('masterHiding', masterKey)   ← seeds per-tx claiming keys
  └── accountAuthKey   = h2('ownerSecret',  masterKey)   ← proves ownership in ZK
      └── accountAddress = h2('I own this transaction', accountAuthKey)
```

For each request, `claimingKey = h2(txIndex, masterHidingKey)` and the public `txSecret = h2(claimingKey, ownerAddress)` go into the URL/QR. The class itself is intentionally narrow:

- `requestFunds(amount, token)` — mint a fresh, unlinkable payment request.
- `deposit(tx)` — payer side: ERC-20 approve + `Chamber.deposit` in two calls.
- `checkStatus` / `scanPayments` — walks the on-chain tx tree to mark requests PAID.
- `withdrawEvm` / `withdrawZkp` — generates a Groth16 proof and calls `Chamber.handleZkp` for a fully private withdrawal.
- `escrowFund(creatorReq, recipientReq, blinding)` — creator side of a bilateral settlement; locks funds with the escrow contract as owner under a key bound to *both* the counter-payment and the recipient's secret.
- `escrowClaim(creatorReq, recipientReq, blinding)` — recipient side: waits for the escrow note, posts the counter-payment, generates **both** the escrow proof and the MIST spend proof, and submits them to `Escrow.consumeEscrow` in one transaction.
- `save` / `load` / `exportState` — pluggable `StorageAdapter` (localStorage, Map, DB).

A whole agent integration is constructing one of these and calling a few methods.

---

## The recipient-bound escrow circuit — `zk/escrow/circuit.go`

This is the technical contribution that makes bilateral private payments actually safe. A naïve escrow on top of a shielded pool has a sniping problem: as soon as the counter-payment hits the chain, anyone watching can race to compute a valid claim against the locked note. Our circuit closes that gap by binding three values — the escrow blinding, the *expected* counter-payment transaction, and a *recipient secret only the intended counterparty knows* — into the nullifier itself:

```
escrowBlinding   = h3(blinding, senderTx, recipientSecret)
nullifierSecret  = h2(escrowBlinding + 1, escrowContract)
escrowNullifier  = h3(nullifierSecret, token, amount)     ← public
recipientTx      = h3(recipientSecret, token, amount)     ← public
```

The circuit also Merkle-proves that `senderTx` exists in Chamber's tx tree (public `MerkleRoot`), so the escrow only releases once the counter-payment has actually settled. Two independent guarantees fall out:

1. **No sniping.** Knowing the on-chain payment is not enough to compute the nullifier — you also need `recipientSecret`, which never touches the chain.
2. **Unlinkability.** Funding, counter-payment, and escrow claim are three independent on-chain transactions to three different secrets. An observer can't tell they belong to the same swap.

---

## Escrow contract — `contracts/src/Escrow.sol`

A permissionless wrapper that holds no state and no funds. Single entrypoint:

```solidity
function consumeEscrow(
    uint256[8]  proof,    uint256[3]  input,     // escrow circuit
    uint256[8]  mistProof, uint256[10] mistInput  // MIST spend circuit
) external;
```

It (1) verifies the escrow proof, (2) checks the merkle root the proof references is one Chamber actually knows about, (3) forwards the MIST spend proof to `Chamber.handleZkp`, then (4) **glues the two proofs together** by asserting `escrowNullifier == mistZkp.nullifier` and `recipientTx == mistZkp.tx1`. Chamber does the transfer; the escrow contract just enforces the binding.

---

## Chamber contract — `contracts/src/Chamber.sol`

The MIST shielded pool itself, ported to Solidity. An append-only `txArray`, a merkle root history, a nullifier set, and a Groth16 verifier. `deposit(secret, amount, token)` shields funds; `handleZkp(proof, inputs)` privately spends a note and appends two indistinguishable output txs. Escrow is one consumer — anything privacy-preserving can sit on top of it the same way.

---

## Running two agents privately in two terminals

The runner makes this trivial. An "agent" is a folder: a `README.md` (used verbatim as the system prompt), an optional `task.md` (if present, the agent opens the conversation), and a `.env` with its key, RPC, contract addresses, peer URL, and inference API key. The runner instantiates `MISTActions` for the agent and exposes a small set of tools to the LLM (`requestPayment`, `payRequest`, `escrowFund`, `escrowClaim`, `checkRequestStatus`, `showBalance`, `sendPeer`, `finalize`).

Setup:

```sh
pnpm i
pnpm build ./sdk
pnpm i ./runner
```

> ⚠️ **Fund the agent wallets first.** Each agent's `PRIVATE_KEY` wallet must hold (a) some native gas, and (b) enough of the ERC-20 it's offering. If Bob is paying out dumUSD and Jill is paying out dumETH, Bob's wallet needs dumUSD and Jill's needs dumETH. The escrow protocol moves tokens privately — it doesn't conjure them.

Then, in two terminals:

```sh
# terminal 1 — Jill waits to be contacted
pnpm jill

# terminal 2 — Bob has task.md, so he opens the conversation
pnpm bob
```

They find each other over `PEER_URL` (AXL in production, plain HTTP locally), exchange MIST requests plus a BLINDING value, and run the full escrow protocol — same flow as the `escrow flow` test in `contracts/hardhat-test/Escrow.test.ts`, only with two LLMs improvising the dialogue around it. Adding a new agent is one folder.

See [`runner/README.md`](runner/README.md) for the full env spec and tool table.
