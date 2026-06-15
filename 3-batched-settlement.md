# Variant 3 — Batched Settlement

- **On Etherlink:** real liquidity, in aggregate (pool).
- **Movement:** messages per operation, tokens in batches (our relayer); on release/refund liquidity returns from Etherlink to Base.
- **User waits:** yes — until the liquidity batch comes back.
- **Trust:** on us. We hold the keys that control the funds (buffer on Base, pool on Etherlink) and run the relayer. Both the safety of the money and the operation of the scheme depend on us.
- **Capital:** ours — minimal (real liquidity is moved, not our float).
- **Cost:** low–medium — transfers in batches, fees are amortized.

## Table of contents

1. **Architecture**
   - Contracts on Base: deposit, payout buffer.
   - Contract on Etherlink: allocation state, condition checks, liquidity pool.
   - Who moves messages (per operation) and tokens (in batches): our relayer or a bridge — to discuss.
   - Where the state, money, and messages live (source of truth for allocations — Etherlink).
2. **Per-operation flow** — deposit, allocate, release, refund, withdraw step by step; separately — what exactly the user waits for and how long.
3. **Moving funds between networks** — real liquidity in batches; what to move it with; batching policy (on schedule / on threshold).
4. **Trust model and security** — trust on us (relayer + buffer).
5. **Liquidity and solvency (the hardest section)** — accounting across two networks, buffer size on Base, payout queue when the buffer is short.
6. **Failures and edge cases** — buffer ran short, batch stuck, queue grows, accounting desync between networks.
7. **Implementation complexity** — contracts + solvency accounting + queue + batching.
8. **Cost** — fewer transfers (batches) → cheaper to move; the price is delay for the user; our capital is minimal.
9. **Solo timeline** — weeks.
10. **Audit scope and cost** — focus on solvency accounting and the queue.
11. **Pros / cons**.
12. **Open questions**.

---

## 1. Architecture

All logic, allocation state, and liquidity are on Etherlink; Base only accepts deposits and makes payouts from the buffer. Messages and money move separately: messages per operation (cheap), tokens rarely and in batches.

### EscrowBase (Base)

- accepting deposits;
- buffer for payouts — a liquidity reserve from which recipients are paid;
- payout to the recipient on a command from Etherlink (with a check that the command comes from our EscrowEtherlink — section 4);
- if the buffer ran short — the payout goes into the queue until the next batch rebalance.

### EscrowEtherlink (Etherlink)

- allocation state — source of truth (to whom, how much, condition, timelock);
- preimage + Merkle check;
- pool of real liquidity — this is the TVL on Etherlink;
- on the result of the check, sends a command to Base to pay out from the buffer.

### Relayer and batching

- messages per operation between Base and Etherlink (our relayer or third-party messaging — open);
- tokens in batches: the surplus from Base goes to the Etherlink pool on a threshold/schedule, the Base buffer is topped up from the pool when it drops. Tokens are not moved back on each withdraw — the buffer pays.

### What lives where

- **allocation state** — on Etherlink (source of truth). Different from Variant 2, where the source of truth is on Base.
- **money** — real liquidity in the pool on Etherlink (TVL) + payout buffer on Base.
- **messages** — per operation, separate from tokens.
- **user** — works only with Base.

## 2. Per-operation flow

The user works only with Base. On each operation a message is sent to Etherlink; tokens are not moved at that point.

- **deposit** — the user deposits USDC/USDT into EscrowBase; a deposit message is sent to Etherlink. The liquidity surplus from Base goes to the Etherlink pool not immediately, but in a batch on schedule/threshold.
- **allocate** — a message to Etherlink, EscrowEtherlink records the allocation (to whom, how much, condition, timelock). Real liquidity is in the Etherlink pool.
- **release / withdraw** — the recipient initiates on Base, a message to Etherlink, EscrowEtherlink checks the preimage and Merkle and sends a command to pay out from the Base buffer. If the buffer is there — payout, the user waits one round of messages (Base → Etherlink → Base). If the buffer ran short — the payout goes into the queue until the next batch rebalance.
- **refund** — after the timelock EscrowEtherlink confirms the refund, a command to pay out to the depositor from the buffer; the same queue logic.

Bottom line: the user always waits for a round of messages; they only wait for the batch to come back when the buffer is short. Tokens are not moved back on each withdraw.

## 3. Moving funds between networks

Two separate flows.

**Messages — per operation.** Cheap. What to move them with — our relayer or third-party messaging via the LayerZero endpoint of Etherlink (open). LayerZero V2 is deployed on Base and Etherlink [^v3s3-lz].

**Tokens — in batches.** Real liquidity is moved rarely and in large amounts. With what — the ready-made Etherlink bridge on LayerZero (lock-mint, representation tokens LZUSDC/LZUSDT) [^v3s3-evm-bridge][^v3s3-tokens] or another (open). CCTP is not applicable: Etherlink is not supported in it [^v3s3-cctp]. Batching policy: the surplus from Base goes to the Etherlink pool on a threshold, the Base buffer is topped up from the pool on a drop; on schedule and/or on threshold.

Expensive token transfers are rare and are amortized over many operations; on each operation — only a cheap message. This is what distinguishes Variant 3 from Variant 1, where the user's tokens go through the bridge on each operation.

## 4. Trust model and security

Trust on us (custodial); we do not trust a third-party bridge on the hot path.

**What we control:** the keys of the pool on Etherlink and the buffer on Base, our relayer, the batch rebalance mechanism.

**Attack surface:** EscrowBase (buffer), EscrowEtherlink (state and pool), relayer, keys, rebalance logic.

**Risks:**
- compromise of the buffer or pool keys — direct loss of funds;
- a false payout command (a compromised relayer or EscrowEtherlink) — siphoning funds out of the buffer;
- desync of solvency accounting — overpayment or double payout.

**Protection:** EscrowBase executes a payout command only if it came from our EscrowEtherlink (source check, as in Variant 1); limits on the buffer and on a single payout; multisig on the pool and rebalance; idempotency of payout commands. The exact composition is a design decision, not fixed.

In Variant 1 trust is on an external bridge; in Variant 2 — also on us, but there it is mirror capital; here it is real liquidity and solvency accounting.

## 5. Liquidity and solvency

The hardest section and the main subject of the audit. Base pays from the buffer on a command from Etherlink, so strict solvency accounting across both networks is needed.

**Accounting across two networks.** Four quantities: how much is in the Etherlink pool, how much is in the Base buffer, how much is promised for payout, how much is in flight (batch rebalance). The rule: obligations are no greater than the real funds. Accounting is kept consistently across both networks; the source of truth for allocations is Etherlink.

**Buffer on Base.** The size is a trade-off: a small buffer — the user waits for the rebalance more often; a large one — more liquidity is locked on Base. It is filled from the pool in a batch.

**Payout queue.** If a payout command came in but the buffer is low — the payout goes into the queue until the next batch rebalance. The promise enters the accounting from the moment of approval; the order of discharge (for example, FIFO) is a design decision.

**Capital.** Our capital is minimal: real liquidity is moved, not our float. But the buffer is a temporarily unavailable (locked) part of the liquidity.

**Double counting.** Funds "in flight" between the pool and the buffer must not be counted twice; the rebalance changes the accounting consistently. If this is not built in from the very start, the scheme is either insolvent or stuck.

## 6. Failures and edge cases

- **Buffer ran short.** The queue grows, the user waits for the rebalance. Protection: buffer monitoring, top-up on a threshold, a clear queue order, a cap on waiting time.
- **Batch (rebalance) stuck or did not arrive.** Liquidity is stuck between the pool and the buffer, accounting has diverged. Protection: track "in flight" as a separate state, resend, reconcile after delivery.
- **Desync of solvency accounting.** Base and Etherlink have diverged on how much is where. Protection: source of truth is Etherlink, regular pool↔buffer reconciliation, blocking new payouts when a discrepancy is detected.
- **Operation message stuck.** The operation is not completed on one of the sides. Protection: status by identifier, resend, timeouts.
- **Reorg.** Reaction to a rolled-back block. Protection: wait for finality before a payout and before a rebalance[^v3s6-finality].
- **Double payout on a command.** Protection: idempotency of payout commands by identifier.

The main threats are buffer insolvency and double payout; both are closed by strict solvency accounting (section 5).

## 7. Implementation complexity

The hardest of the three variants — it adds cross-network solvency accounting, which is not present in Variants 1 and 2.

- **EscrowBase** (buffer, payout queue, payout on command) — medium/high.
- **EscrowEtherlink** (allocations, preimage+Merkle, timelocks, pool) — high.
- **Solvency accounting across two networks** — the hardest part.
- **Payout queue when the buffer is short** — high.
- **Batching and pool↔buffer rebalance** (policy, "in flight" accounting) — high.
- **Messaging per operation** — medium.
- **Failure handling** — high.

The hardest part is solvency accounting, the queue, and the batch rebalance.

## 8. Cost

Three kinds of cost: build (one-time), maintenance per year (recurring), and operational (per operation). The audit is separate (section 10).

**Build cost (one-time).** Solo development at \$60/h, 40 h/week (\$2,400/week); the audit is not included. From the timeline (section 9): ~4.5–6 weeks = ~180–240 h → **~\$10,800–14,400**.

**Maintenance per year.** Infrastructure + support + key custody + cost of capital; the highest support load of the three, because solvency accounting and the batch rebalance need watching:
- infrastructure (relayer, monitoring, batch rebalance): ~\$1–2k/year;
- support at \$60/h: ~10–16 h/month → ~\$7.2–11.5k/year;
- key custody (self-custody multisig on the buffer and the pool): little hard cost;
- cost of capital — only on the buffer, which is small: buffer × ~10%/year. Example: at \$20k buffer → ~\$2k/year.
- **~\$8–13.5k/year fixed, plus a small cost of capital on the buffer.**

**Operational (per operation).** Operationally low–medium — per operation we pay only for cheap messages and gas; the expensive token movement goes in batches, rarely.

**Per operation:**

| Operation | Messages | Token transfer | Cost |
|---|---|---|---|
| deposit | 1 | no | gas + message |
| allocate | 1 | no | gas + message |
| release / withdraw | 2 (there and a command back) | no | gas + 2 messages |
| refund | 2 | no | gas + 2 messages |

There is no token transfer on any operation — it goes in batches separately, once per period. Per operation we pay only for messages and gas, all cheap.

**Cost by line item:**

| Line item | Cost |
|---|---|
| Message (per operation) | a few cents or less |
| Gas on Base | 1–3 cents |
| Gas on Etherlink | fractions of a cent |
| Batch token transfer | expensive, but rare — amortized over all operations of the batch |

- message — a few cents or less; depends on the method (our relayer or third-party messaging), the exact figure for the Base ↔ Etherlink pair needs to be measured;
- gas on Base — about \$0.007 per ERC-20 transfer, typically \$0.03; plus a variable part for writing data to Ethereum [^v3s8-base-gas];
- gas on Etherlink — about 0.0006 XTZ at 1 gwei, in external reviews "\$0.001 or less" [^v3s8-etl-gas];
- batch token transfer — the bridge fee for a large move, once per period; split over all operations within the batch.

**Conclusion:** cheaper than Variant 1 (there tokens go through the bridge on each operation), more expensive than Variant 2 (there there are no transfers of the user's money between networks at all). The rarer and larger the batches, the cheaper per operation, but the more liquidity is locked or in flight. Our capital is minimal; the buffer is a separate locked part (section 5).

## 9. Timeline

An estimate for a single full-time developer, not a commitment. The audit is not included.

| Part | Weeks |
|---|---|
| Contracts on Base (buffer, queue) | 0.5–0.7 |
| Contract on Etherlink | 0.5–0.7 |
| Solvency accounting across two networks | 0.7–1 |
| Batching and rebalance | 0.5–0.7 |
| Messaging per operation | 0.3–0.5 |
| Failure handling | 0.5–0.7 |
| Tests, testnet, monitoring, relayer infra | 1.5–2 |

**Total: ~4.5–6 weeks.** Variant 3 is the hardest of the three, the main weight and risk are in solvency accounting and the batch rebalance. This is an estimate, not a measurement.

## 10. Audit scope and cost

**What to audit:** EscrowBase (buffer, queue, payout on command, source check, idempotency), EscrowEtherlink (allocations, preimage+Merkle, timelocks, pool), solvency accounting across two networks, the batching/rebalance mechanism, messaging, keys. The most sensitive and most expensive to audit are solvency accounting and the queue: an error means insolvency or a double payout. This is the hardest variant to audit.

**Cost — a market estimate, not a quote:** day rate ~\$500–1200/day [^v3s10-rate]; a simple contract ~\$5–20k, a medium-complexity DeFi ~\$40–100k, multi-contract cross-chain systems from \$50k and up to \$150–300k+ [^v3s10-sherlock][^v3s10-mor]; re-audit +\$5–20k per round; a cross-chain surcharge ~25–30%; well-known auditors (MixBytes and others) price per project [^v3s10-mixbytes]. Because of the complex cross-network solvency accounting, the reference point is closer to the upper part plus the cross-chain surcharge. Separately — an audit of the off-chain relayer, batching, and key infrastructure.

## 11. Pros / cons

**Pros:**
- Both the allocation state and the real liquidity are genuinely on Etherlink — closest to the literal requirement of the task.
- Our capital is minimal — real liquidity is moved, not our float.
- Operationally cheaper than Variant 1 — tokens in batches, not on each operation.
- Messages are cheap.

**Cons:**
- The hardest variant: solvency accounting across two networks + the queue — the main risk both in development and in the audit.
- The user waits: a round of messages always, the batch coming back when the buffer is short.
- Trust on us; the risk of buffer insolvency.
- Complex failure handling; longer and more expensive to audit.

## 12. Open questions

- Batching policy: on schedule, on threshold, or both.
- The size of the buffer on Base and who/what fills it.
- What happens when the buffer is short: how the queue is arranged and how long the user waits at most.
- What to move tokens with in batches and what to move messages with per operation (our relayer, third-party messaging, a bridge).
- How we keep solvency accounting across two networks and how often we reconcile pool↔buffer.
- Idempotency of payout commands; reaction to a reorg (by which finality).
- Who initiates the rebalance and how "in flight" is counted.
- What we show the user while a payout is in the queue.

[^v3s3-lz]: LayerZero deployments — Base (EID 30184): https://docs.layerzero.network/v2/deployments/chains/base ; Etherlink (EID 30292): https://docs.layerzero.network/v2/deployments/chains/etherlink

[^v3s3-evm-bridge]: Etherlink — Bridging between EVM networks (lock-mint on LayerZero, representation tokens): https://docs.etherlink.com/bridging/bridging-evm/

[^v3s3-tokens]: Etherlink — Tokens (USDC `0x796Ea11Fa2dD751eD01b53C372fFDB4AAa8f00F9`, USDT `0x2C03058C8AFC06713be23e58D2febC8337dbfE6A`): https://docs.etherlink.com/building-on-etherlink/tokens/ ; LZUSDC: https://www.coingecko.com/en/coins/layerzero-bridged-usdc-etherlink ; LZUSDT: https://www.coinbase.com/price/layerzero-bridged-usdt-etherlink

[^v3s3-cctp]: Circle — Cross-Chain Transfer Protocol (supported networks include Base, Etherlink is absent): https://www.circle.com/cross-chain-transfer-protocol

[^v3s8-base-gas]: Gas on Base: https://coincodex.com/article/59940/fees-for-transfering-usdc-on-the-base-network ; https://sqmagazine.co.uk/ethereum-gas-fees-statistics/

[^v3s8-etl-gas]: Gas on Etherlink: https://docs.etherlink.com/building-on-etherlink/estimating-fees/ ; https://thirdweb.com/etherlink

[^v3s10-rate]: FailSafe — auditor comparison, day rate \$500–1200/day: https://getfailsafe.com/how-to-guide-to-compare-cheap-smart-contract-auditors-in-july-2025

[^v3s10-sherlock]: Sherlock — Smart Contract Audit Pricing 2026: https://sherlock.xyz/post/smart-contract-audit-pricing-a-market-reference-for-2026

[^v3s10-mor]: Mor Software — Smart Contract Audit Cost (cross-chain up to \$150k+, surcharge for niche expertise): https://morsoftware.com/blog/smart-contract-audit-cost

[^v3s10-mixbytes]: MixBytes — Audit (priced per project, minimum two auditors): https://mixbytes.io/audit

[^v3s6-finality]: Network finality. Base (OP Stack L2): in a batch on Ethereum in ~2 min (no L2 reorgs after that), full finality — ~13 min (2 epochs, 64 L1 blocks). Etherlink (Tezos Smart Rollup): deterministic finality in 2 Tezos blocks (seconds), soft confirmations < 500 ms. Make the payout on Base and the rebalance after these thresholds. Sources: https://docs.base.org/base-chain/network-information/transaction-finality , https://docs.etherlink.com/network/architecture/
