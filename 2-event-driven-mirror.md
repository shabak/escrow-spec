# Variant 2 — Event-Driven Mirror

- **On Etherlink:** our mirror capital; the user's money stays on Base.
- **Movement:** the Go relayer catches an event from Base and moves the same amount of our capital into the Etherlink escrow; on release/refund it returns it.
- **User waits:** no — the payout on Base is immediate.
- **Trust:** on us. We hold the keys controlling the funds (our capital on Etherlink) and run the relayer. The safety of the money and the operation of the scheme depend on us.
- **Capital:** ours, at maximum — a full mirror on Etherlink.
- **Cost:** very cheap — Etherlink gas + relayer infrastructure; no bridge fee.

## Table of contents

1. **Architecture**
   - Contracts on Base: deposit, allocate, release, refund, withdraw — all operations and payouts on Base; emits events.
   - Contract on Etherlink: a mirror of allocations backed by our capital.
   - Off-chain: the Go relayer — listens to Base, moves our capital on Etherlink.
   - Where state, money, and messages live (the user's money on Base; the source of truth for allocations is Base).
2. **Per-operation flow** — deposit, allocate, release, refund, withdraw step by step.
3. **Moving funds between chains** — our capital is moved by our relayer; no third-party bridge is needed for the user's money; how our capital gets onto Etherlink in the first place.
4. **Trust model and security** — trust is on us: keys on the funds, our relayer, what happens on compromise.
5. **Liquidity** — our capital on Etherlink is locked at maximum; how much = the sum of active allocations (at the limit — max supply).
6. **Failures and edge cases** — the relayer is down, mirror desync, a duplicate event, a reorg on Base, a shortage of our capital.
7. **Implementation complexity** — the contracts are simple, the main work is in the relayer.
8. **Cost** — Base gas + Etherlink gas (fractions of a cent) + relayer infrastructure; no bridge fee. Locked capital is a separate axis (section 5).
9. **Solo timeline** — weeks.
10. **Audit scope and cost** — contracts + off-chain relayer + keys.
11. **Pros / cons**.
12. **Open questions**.

---

## 1. Architecture

Two contracts, one in each chain, and our off-chain service between them. No third-party bridge is needed for the user's money: the user's funds stay on Base and are paid out on Base. On Etherlink there is our capital, synchronized by our relayer based on Base events. The contracts of the two chains do not call each other directly — only the relayer holds the connection.

### EscrowBase (Base)

The entry point for the user: all operations and payouts on Base. Unlike Variant 1, release, refund and withdraw are also on Base; there is no return path through a bridge.

- **deposit / withdraw** — accepting a deposit and disbursing funds. The user's money is always on Base; balance accounting (deposit state) is here.
- **allocate** — locks the user's funds for an allocation (to whom, how much, condition, refund timelock) and emits an event. The money does not leave Base.
- **release / refund** — close an allocation and pay out: release — to the recipient, refund — to the depositor after the timelock. Each emits an event.
- **payout rights** — the right to debit the locked funds is held by this contract and our management (section 4).

EscrowBase is a full single-chain escrow: all logic and the user's money are here, and it already works without Etherlink. Events are needed only for the mirror.

### EscrowEtherlink (Etherlink)

A mirror of allocations backed by our capital. There is no user money here — here is our capital, and its amount under active allocations is the TVL on Etherlink.

- **allocation state** — a copy of the allocations (to whom, how much, condition, timelock). This is a mirror, not the source of truth: the source of truth is EscrowBase, and the state arrives here through the relayer with a delay.
- **our capital under allocations** — for each allocation the relayer moves the same amount of our capital here. Real funds, but ours, not the same ones the user deposited.
- **release / refund** — when an allocation is closed on Base, the relayer closes the copy here and frees our capital.

The user's money stays on Base — only our capital is on Etherlink. If it is held in the same USDC/USDT (to mirror allocations 1:1 by amount), it will be LZUSDC/LZUSDT: there is no native USDC/USDT from Circle or Tether on Etherlink, only their LayerZero wrappers. The capital can also be held in another asset. How the capital gets onto Etherlink is section 3.

### Go relayer (our off-chain service)

The connection between the chains. Works only with our capital.

- listens to EscrowBase events (allocate, release, refund);
- on allocate moves the same amount of our capital into EscrowEtherlink, on release/refund frees it;
- keeps the mirror in line with the allocations on Base.

The relayer manages our capital, so its keys and availability are part of our trust zone (section 4). If the relayer is stopped, payouts to the user on Base are not blocked — only the mirror diverges (section 6).

### What lives where

- **deposit state** — on Base.
- **allocation state** — the source of truth on Base; on Etherlink a copy (mirror) maintained by the relayer. Different from Variant 1, where the source of truth was on Etherlink.
- **the user's money** — always on Base; it is not on Etherlink.
- **our capital** — on Etherlink, under active allocations; this is the Etherlink TVL. Real funds, but ours.
- **messages** — there are no direct messages between the contracts; the connection is the EscrowBase events that the relayer reads. The mirror is not affected on deposit and withdraw.
- **the user** — works only with Base.

## 2. Per-operation flow

The user does all operations on Base, and payouts arrive there too; their state is final in the same transaction. The relayer asynchronously reflects events on Etherlink with our capital; the mirror catches up to Base with a delay that is not visible to the user.

- **deposit** — the user deposits USDC/USDT into EscrowBase, the balance grows. Etherlink is not affected (there is no allocation yet).
- **allocate** — EscrowBase records an allocation from the user's balance and emits an event. The relayer moves the same amount of our capital into the escrow on Etherlink — this is TVL. The user's money stays on Base.
- **release** — EscrowBase immediately pays out to the recipient on Base, from the user's funds; emits an event. The relayer returns our capital from the escrow to the pool.
- **refund** — after the timelock EscrowBase returns the funds to the user on Base; the relayer frees our capital in the same way. For the mirror, release and refund are the same: the allocation is closed, the capital is returned.
- **withdraw** — the user takes their free (unallocated) balance on Base. Etherlink is not affected.

Bottom line: the user does not wait for cross-chain on any operation. The mirror changes only on allocate, release, refund — and always after the relayer has processed the event.

## 3. Moving funds between chains

Only our capital is moved between chains, and it is moved by our relayer. The user's money stays on Base. No third-party bridge is needed for the user's money — it does not leave Base.

Our mirror capital needs to be brought onto Etherlink once (a one-time top-up, not on every operation). The ready-made Etherlink bridge on LayerZero is suitable for this: LayerZero V2 is deployed on Base and Etherlink [^v2s3-lz], the bridge works on a lock-mint scheme [^v2s3-evm-bridge], and USDC/USDT on Etherlink are LayerZero representation tokens [^v2s3-tokens]. CCTP is not applicable: Etherlink is not supported in it [^v2s3-cctp].

After that the relayer only synchronizes our capital within Etherlink: on allocate it moves the amount into the escrow, on release/refund it returns it. There is no transfer of the user's money between chains anywhere. This is what distinguishes Variant 2 from Variant 1, where the user's money goes through the bridge on every operation.

## 4. Trust model and security

There is no third-party bridge for the user's money, all trust is on us. The scheme is custodial.

**What we control:** the payout rights from EscrowBase on Base (the user's money is there) and the keys of our capital on Etherlink. The user's money is not transferred across the network, but the right to hand it out on Base is ours, so we are responsible for its safety.

**Attack surface:** EscrowBase, EscrowEtherlink, the Go relayer, the keys on the funds (the payout rights from EscrowBase + the capital key on Etherlink — the most sensitive). In Variant 1 the largest part of the attack surface is the external bridge, not ours; here there is no bridge, the whole surface is ours.

**Risks:**
- Compromise of our keys — direct loss of funds: the payout key takes the user's money off Base, the capital key takes our capital on Etherlink.
- The relayer works incorrectly or is stopped: the mirror is not updated or diverges from Base. Payouts to the user on Base do not depend on the relayer (they come from EscrowBase), but the mirror will diverge.
- Mirror desync: between the event on Base and the relayer's action the states diverge (section 6).

**What there is not:** the risks of a third-party bridge (bridge compromise, theft in flight, a forged message). In exchange, all the risk is on us: we trust our own keys, the relayer and the code.

**Measures:** key separation and multisig on critical rights; limiting the relayer's rights separately from the payout rights; monitoring of the relayer, mirror divergence and key movements.

## 5. Liquidity

Variant 2 keeps a full mirror of active allocations on Etherlink: for each open allocation our capital is held in the same amount. In total, our capital equal to the sum of all active allocations is needed; the upper bound is the total deposit cap (max supply). This is locked capital, and its cost (cost of capital) is the main price of Variant 2.

Where the funds are: the user's money on Base; our capital on Etherlink under allocations (this is TVL). On release/refund our capital is freed back into the pool.

The amount locked at any moment equals the sum of active allocations; it depends on the number and size of open allocations, with the peak at the full deposit cap.

For comparison: Variant 1 — no capital of our own is needed, but the user waits for the bridge; Variant 3 — minimal capital, but the user waits for the batch return; Variant 2 — the user does not wait, but we pay with locked capital.

## 6. Failures and edge cases

The user's money is always on Base, payouts go from Base in the same transaction — the relayer and our capital do not take part in the payout. Therefore almost all failures hit our mirror and capital, not the availability of funds to the user. This is different from Variant 1, where a failure of the cross-chain part delayed access to the money.

- **The relayer is down or lagging.** The mirror is not updated, the TVL on Etherlink does not match the allocations on Base; payouts to the user do not suffer. Protection: a processing status per event, a queue of unprocessed events, monitoring of the lag.
- **Mirror desync (Base ahead of Etherlink).** A normal intermediate state under asynchrony, which drags on under failures. The source of truth is Base, a divergence is always treated as "Etherlink has not caught up." Protection: a sync status per allocation, a one-directional reconciliation (Etherlink is brought to Base).
- **Double processing of an event.** Without protection the relayer moves or closes capital twice. Protection: idempotency by a stable event identifier (allocation id + type, or block + log index).
- **Reorg on Base.** The relayer reacted to an event, the block was rolled back. Protection: react only to final Base blocks[^v2s6-finality]; on a late rollback — reconcile the mirror with the final state of Base.
- **Not enough of our capital on Etherlink.** The allocation cannot be fully reflected, the TVL is understated; on Base the allocation and the user's funds are fine. Protection: monitoring of free capital and topping up in advance; the event waits in the queue until the top-up.

All these failures are not visible to the user and appear only on our side, so monitoring is needed: relayer lag, mirror divergence, the remaining capital.

## 7. Implementation complexity

There is no third-party bridge, the contracts are simple; the main complexity is in our off-chain relayer and the infrastructure around the capital.

- **EscrowBase** — medium/simple. All the business logic (deposit/allocate/release/refund/withdraw, payouts, events). An ordinary single-chain escrow, without cross-chain calls; events are simply emitted.
- **EscrowEtherlink** — simple. A mirror backed by our capital: stores copies of allocations, locks and frees capital on the relayer's commands.
- **Go relayer** — high. The main work: reliable event processing, idempotency, mirror synchronization, reacting only to final blocks, handling failures. It manages real money and works asynchronously between the chains.
- **Keys and capital** — medium/high. Custodial infrastructure (storing the keys to the funds), monitoring, top-ups. A constant operational load, not a one-time development.

The hardest part is the relayer (idempotency, failure handling) and storing the keys for the capital, not the contracts. In Variant 1 the complexity was in the contracts and the bridge; here it is shifted off-chain.

## 8. Cost

Here there is only the operational cost — what each operation costs. The cost of locked capital (the main price of Variant 2) is in section 5. We pay for gas; there is no bridge fee, because the user's money does not go through a bridge.

**Per operation:**

| Operation | Relayer transaction on Etherlink | Cost |
|---|---|---|
| deposit | no | Base gas only |
| allocate | yes | Base gas + Etherlink gas |
| release | yes | Base gas + Etherlink gas |
| refund | yes | Base gas + Etherlink gas |
| withdraw | no | Base gas only |

There is no bridge fee on any operation. deposit and withdraw — Base gas only. allocate, release and refund add a cheap relayer transaction on Etherlink (fractions of a cent).

**Cost by item:**

| Item | Cost |
|---|---|
| Gas on Base | 1–3 cents per operation |
| Gas on Etherlink (relayer transaction) | fractions of a cent |
| Relayer infrastructure | a constant cost, not per operation |

- gas on Base — about \$0.007 per ERC-20 transfer, typically \$0.03; plus a variable part for writing data to Ethereum [^v2s8-base-gas];
- gas on Etherlink — about 0.0006 XTZ at 1 gwei, in external reviews "\$0.001 or less" [^v2s8-etl-gas];
- relayer infrastructure — a server and monitoring; depends on hosting, counted over time, not per operation.

**Conclusion:** by operational cost this is the cheapest of the three — there is no bridge fee at all. The real price of Variant 2 is not here but in the locked capital (section 5).

## 9. Solo timeline

An estimate for one full-time developer, not a commitment. The audit is not included.

| Part | Weeks |
|---|---|
| Contracts on Base | 0.5–0.7 |
| Mirror contract on Etherlink | 0.2–0.3 |
| Go relayer (the main part) | 0.5–0.8 |
| Custodial infrastructure | 0.3–0.5 |

**Total: ~2 weeks.** The contracts are simpler than in Variant 1 (no bridge integration), but the Go relayer and the custodial wrapping are added — the main volume and risk are there (idempotency, failure handling, key storage).

## 10. Audit scope and cost

**What to audit:** the EscrowBase and EscrowEtherlink contracts; the synchronization logic and idempotency; key and capital management; the relayer's rights. The most sensitive are the custodial part (the keys on the funds) and mirror synchronization.

**There is little smart-contract audit here.** A significant part of the logic and all control over the capital is outside the contracts. Separately, an audit of the off-chain relayer, the key infrastructure and operational security is needed; this is not covered by an ordinary contract audit and is counted separately.

**The price of the contract part is a market estimate, not a quote:** an auditor's day rate is ~\$500–1200/day [^v2s10-rate]; a simple contract ~\$5–20k, a medium-complexity DeFi ~\$40–100k [^v2s10-sherlock][^v2s10-mor]; re-audit +\$5–20k per round; well-known auditors (MixBytes and others) price per project [^v2s10-mixbytes]. The contracts in Variant 2 are simpler than in Variant 1 (there is no cross-chain logic in them) — the reference is closer to the lower part of the ranges. A full security assessment is wider than the contract part.

## 11. Pros / cons

**Pros:**
- The user does not wait for cross-chain — payouts from Base are immediate.
- Operationally cheap — no bridge fee.
- No dependence on a third-party bridge and its risks.
- Cross-chain failures do not hit the availability of funds to the user (their money is on Base).

**Cons:**
- Trust is fully on us, custodial (keys on the funds).
- Our capital is needed at maximum (locked, with its cost).
- The keys and the relayer are a critical point of failure.
- The TVL on Etherlink is our money, not the "same" money the user deposited; and the source of truth for allocations is Base, not Etherlink. May not satisfy the requirement of the task or the fund — an open question.

## 12. Open questions

- How our capital initially gets onto Etherlink and who tops it up.
- In which asset to hold our capital on Etherlink: LZUSDC/LZUSDT for a 1:1 mirror of USDC/USDT, or another asset.
- How much capital to hold (a function of the sum of active allocations) and at what threshold to top up.
- Whether the fund will accept that the TVL is our capital and the source of truth for allocations is Base, not Etherlink. The key question of the variant's acceptability.
- What happens on compromise of our keys or a relayer failure; what rights the relayer has and how they are limited.
- Idempotency of event processing; the reaction to a reorg on Base (at what finality).
- The acceptable mirror lag; how and how often to reconcile the mirror with Base.

[^v2s3-lz]: LayerZero deployments — Base (EID 30184): https://docs.layerzero.network/v2/deployments/chains/base ; Etherlink (EID 30292): https://docs.layerzero.network/v2/deployments/chains/etherlink

[^v2s3-evm-bridge]: Etherlink — Bridging between EVM networks (lock-mint on LayerZero, representation tokens, fee is gas only, time "a few minutes"): https://docs.etherlink.com/bridging/bridging-evm/

[^v2s3-tokens]: Etherlink — Tokens (USDC `0x796Ea11Fa2dD751eD01b53C372fFDB4AAa8f00F9`, USDT `0x2C03058C8AFC06713be23e58D2febC8337dbfE6A`): https://docs.etherlink.com/building-on-etherlink/tokens/ ; LZUSDC: https://www.coingecko.com/en/coins/layerzero-bridged-usdc-etherlink ; LZUSDT: https://www.coinbase.com/price/layerzero-bridged-usdt-etherlink

[^v2s3-cctp]: Circle — Cross-Chain Transfer Protocol (supported networks include Base, Etherlink is absent): https://www.circle.com/cross-chain-transfer-protocol

[^v2s8-base-gas]: Gas on Base: https://coincodex.com/article/59940/fees-for-transfering-usdc-on-the-base-network ; https://sqmagazine.co.uk/ethereum-gas-fees-statistics/

[^v2s8-etl-gas]: Gas on Etherlink: https://docs.etherlink.com/building-on-etherlink/estimating-fees/ ; https://thirdweb.com/etherlink

[^v2s10-rate]: FailSafe — comparison of auditors, day rate \$500–1200/day: https://getfailsafe.com/how-to-guide-to-compare-cheap-smart-contract-auditors-in-july-2025

[^v2s10-sherlock]: Sherlock — Smart Contract Audit Pricing 2026: https://sherlock.xyz/post/smart-contract-audit-pricing-a-market-reference-for-2026

[^v2s10-mor]: Mor Software — Smart Contract Audit Cost: https://morsoftware.com/blog/smart-contract-audit-cost

[^v2s10-mixbytes]: MixBytes — Audit (priced per project, minimum two auditors): https://mixbytes.io/audit

[^v2s6-finality]: Base finality (OP Stack L2): a transaction lands in a batch on Ethereum in ~2 minutes, and no L2 reorgs after that have been observed; full Ethereum-level finality is ~13 minutes (2 epochs, 64 L1 blocks). The relayer reacts to a Base event after this threshold. Source: https://docs.base.org/base-chain/network-information/transaction-finality
