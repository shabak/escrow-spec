# Variant 1 — Canonical Bridge

- **On Etherlink:** the user's deposit 1:1 — the user's real money.
- **Movement:** canonical bridge on every operation (LayerZero OFT / Circle CCTP / other — to be chosen later).
- **User waits:** yes — bridge on every operation.
- **Trust:** on the bridge/protocol, not on us.[^bridges]
- **Capital:** our own is not needed.
- **Cost:** expensive — a transfer fee on every operation + gas on both chains.

## Table of contents

1. **Architecture** — contracts on Base and Etherlink, an external bridge moving tokens and messages; where the state, money, and messages are (source of truth for allocations — Etherlink).
2. **Per-operation flow** — deposit, allocate, release, refund, withdraw step by step.
3. **Moving funds between chains (the main section of this variant)** — what is available on Etherlink: LayerZero, CCTP, deBridge, etc.; comparison by price, latency, trust, USDC/USDT support.
4. **Trust model and security** — what it rests on (the bridge), the attack surface.
5. **Liquidity** — no capital of our own is needed; but the user waits for the bridge on every operation.
6. **Failures and edge cases** — a stuck message or transfer, a desync "tokens left — no message", reorg, bridge unavailability.
7. **Implementation complexity** — contracts + bridge integration.
8. **Cost** — a fee per transfer + gas on both chains.
9. **Solo timeline** — weeks.
10. **Audit scope and cost** — what to audit, the price range.
11. **Pros / cons**.
12. **Open questions**.

---

## 1. Architecture

Two contracts, one in each chain, and an external bridge between them. In this variant there is no off-chain service of our own: the tokens and messages are moved by the bridge itself.

### EscrowBase (Base)

The entry point for the user: all of their operations run on Base.

- **deposit / withdraw** — accepting a deposit and paying out funds to the recipient. The deposit balance accounting is stored here (this is the deposit state on Base). The deposit stays on Base: tokens are not sent through the bridge and no message is passed to Etherlink.
- **allocate** — locks the user's funds, hands them to the bridge for sending to Etherlink, and forms a message with the allocation parameters (to whom, how much, the condition).
- **withdraw (by the recipient)** — the recipient initiates a withdrawal on Base and passes the preimage. The contract[^q-contract] sends it to Etherlink for verification[^q-etherlink] and, once it receives confirmation, pays the returned funds to the recipient on Base.
- **bridge adapter** — the part through which the contract passes and receives tokens and messages. The exact interface depends on the bridge (section 3): for LayerZero this is an OFTAdapter with lock/unlock, for a burn/mint bridge — its send/receive.

### EscrowEtherlink (Etherlink)

This contract implements the allocation logic, and the funds reside in it while an allocation is active.

- **allocation state** — the state of allocations: to whom, how much, the condition (preimage hash + Merkle root), the timelock on refund. This is the source of truth for allocations, and it is stored only on Etherlink.
- **receiving funds** — the bridge delivers tokens here; the token form depends on the bridge (section 3). These funds are the TVL on Etherlink.
- **release / refund** — on release the contract verifies the preimage and Merkle proof, then initiates the return of funds to Base and a command to pay out to the recipient. On refund it returns the deposit to the depositor after the timelock expires.

### Bridge (external protocol)

On every operation (except deposit) the bridge moves both the tokens and the messages:

- **tokens** — the user's real money is transferred between Base and Etherlink (this is what makes the variant expensive);
- **messages**[^msg-passing] — the allocation parameters and the release/refund commands.

This is an external protocol (LayerZero, CCTP, deBridge, or a native bridge), not our service, so we are forced to trust it (see the footnote at the top and section 4). Which exact bridge, whether it supports Etherlink and the required tokens (USDC/USDT) — we check in section 3.

### Where things are

- **deposit state** — on Base, in EscrowBase.
- **allocation state** — on Etherlink, in EscrowEtherlink; the source of truth for allocations.
- **money** — while an allocation is active, on Etherlink (TVL).
- **messages** — on allocate, release, refund, withdraw through the bridge; on deposit none are sent.
- **user** — works only with Base.

## 2. Per-operation flow

### deposit

1. The user calls `deposit()` on Base and transfers USDT/USDC.
2. EscrowBase records the deposit balance (deposit state on Base).

Result: the bridge is not involved, the funds are on Base, the deposit is accounted for, the user does not wait.

### allocate

1. The depositor calls `allocate()` on Base: recipient, amount, condition (preimage hash + Merkle root), refund timelock.
2. EscrowBase locks the amount, hands it to the bridge for sending to Etherlink, and forms a message with the allocation parameters.
3. EscrowEtherlink receives the message, verifies that it is from our EscrowBase, and records the allocation; the funds end up on Etherlink and bound to the allocation — this is the TVL.

Result: the bridge moves tokens + a message Base → Etherlink, the allocation is recorded on Etherlink, the user waits for one path.

### release / withdraw

release and withdraw here are a single action[^release-split]: the recipient reveals the preimage and immediately collects the funds.

1. The recipient calls `withdraw()` on Base and passes the preimage.
2. EscrowBase sends the preimage to Etherlink for verification (a message Base → Etherlink).
3. EscrowEtherlink verifies the preimage and Merkle proof against the recorded allocation. If correct — it removes the funds from the allocation and hands them to the bridge for return to Base together with a payout command.
4. EscrowBase receives the funds and pays them out to the recipient on Base.

Result: the bridge moves a message Base → Etherlink, then tokens + a command Etherlink → Base; the recipient received the funds on Base, the allocation is closed, the user waits for the round-trip path.

### refund

1. After the refund timelock expires, refund is started from Base.
2. EscrowBase sends a refund request to Etherlink.
3. EscrowEtherlink verifies that the timelock has expired and the condition was not met; it removes the funds from the allocation and hands them to the bridge for return to Base.
4. EscrowBase returns the funds to the depositor.

Result: the bridge moves a request Base → Etherlink, then tokens Etherlink → Base; the depositor received the refund on Base, the allocation is closed, the user waits for the round-trip path.

## 3. Moving funds between chains

On Etherlink both USDC and USDT exist not as native tokens but as tokens issued by the LayerZero bridge: LZUSDC (`0x796E…00F9`) and LZUSDT (`0x2C03…fE6A`) [^s3-tokens]. This sets almost the entire choice.

### LayerZero — applicable

LayerZero V2 is deployed on both Base (EID 30184) and Etherlink (EID 30292) [^s3-lz]. The Etherlink bridge between EVM chains is itself built on LayerZero with a lock-mint scheme: the token is locked on the source chain, its representation is issued on Etherlink, and on withdrawal it is burned [^s3-evm-bridge]. Two paths: use the ready-made Etherlink bridge, or build your own OFT/OFTAdapter channel on the same endpoints. (OFT is a LayerZero token that is burned on one chain and issued on another; OFTAdapter locks an ordinary ERC-20 on the main chain if the token has no OFT contract of its own, as is the case for USDT.) LayerZero V2 can carry both tokens and operation data in a single message. Security here is provided by LayerZero (its message verification and delivery); there is no service of our own to run.

### CCTP — not suitable

Circle CCTP moves native USDC via burn/mint. Etherlink is not in the list of CCTP-supported chains, and there is no native USDC on Etherlink [^s3-cctp]. So for the Base ↔ Etherlink pair CCTP is currently inapplicable. It would fit only if Circle adds Etherlink.

### Other bridges

- **deBridge:** Base is supported, Etherlink — not confirmed (there is no full list in the documentation) [^s3-debridge].
- **Stargate:** in the documentation Etherlink is listed as an option, but it works on top of LayerZero — the same trust model [^s3-bridging]. Routes and prices not checked.
- **Etherlink native bridge:** connects Tezos L1 and Etherlink, not Base. Not fit for Base ↔ Etherlink [^s3-tezos-bridge].

### Price and latency

The Etherlink bridge does not take a fee for itself: the source chain's gas is paid; the time on the EVM side is "a few minutes", depending on finalization [^s3-evm-bridge]. The order of magnitude of the price of one transfer in dollars is not checked — it needs to be measured on test transfers.

### Conclusion

Only LayerZero is applicable: the ready-made Etherlink bridge or your own OFT/OFTAdapter channel.

Not decided: which of the two; the OFT/OFTAdapter addresses for USDC and USDT; whether operation data is carried together with the transfer; the price and latency of one transfer.

## 4. Trust model and security

In Variant 1 the security of funds in flight is no higher than the security of the chosen bridge.

**What we trust the bridge with:** the safekeeping of the funds while they are with the bridge (at the moment of transfer they are not with us); the delivery of the message without tampering, exactly once, and only if it was actually sent by our contract. The contents of the messages are public, though: the secret is protected by preimage+Merkle on Etherlink, not by the secrecy of the channel.

**Attack surface:** EscrowBase; EscrowEtherlink; the code at the seam with the bridge endpoint; the bridge itself with its agents and verifiers (the largest part, and not ours); the contracts' admin keys, if any (pause, upgrade, changing the trusted sender) [^s4-admin]. The first three and the keys are our zone, the bridge is external.

**If the bridge is compromised** it is possible to not deliver or to steal the funds in flight and to deliver a forged message in the name of our contract (to force a release of funds or to record a false allocation). preimage+Merkle limit tampering with the allocation's content, but they do not protect against a forged message about the movement of funds if the contract trusts it.

**Source verification (mandatory).** EscrowEtherlink executes a message only if the callback came from the bridge endpoint and the source is our EscrowBase (by chainId and the sender's address); symmetrically EscrowBase verifies messages from Etherlink. Without this, any sender through the same bridge could impersonate our contract. But this verification works on top of trust in the bridge: it does not protect against identifier forgery by a compromised bridge.

**The risk depends on the bridge (section 3):** closer to trustless/non-custodial — the risk is lower (delivery on proofs, the funds are not with a single operator); closer to custodial — a compromise of the operator or signers means a compromise of the funds and messages.

## 5. Liquidity

No capital of our own is needed: on every operation the user's real deposit is transferred, not our funds. allocate transfers the deposit Base → Etherlink, withdraw and refund return it Etherlink → Base.

The price for this is delay: the user waits for the bridge on allocate and on withdraw/refund. The duration of one bridge crossing is not checked, it depends on the bridge.

Where the funds are: before allocate — on Base; while an allocation is active — on Etherlink (this is the TVL); after withdraw or refund — back on Base.

The difference from Variants 2 and 3: there, instead of waiting for the bridge, we lock our own capital. Here it is the opposite: no capital of our own is needed, but the user waits for the bridge every time.

## 6. Failures and edge cases

Any operation except deposit runs in two chains, with time passing between steps. Hence failures that do not happen when everything occurs in a single chain.

- **A message is stuck or not delivered.** On Base the step is done, on Etherlink not yet (or vice versa). Protection: operation status by identifier; manual re-sending (depends on the bridge [^s6-bridge]); a timeout on the source side, noticeably larger than the usual delivery time.
- **Desync: tokens left, no message** (or a return by timeout, while the message arrived later). This threatens a balance mismatch and double accounting. Protection: a single source of truth — EscrowEtherlink; keep funds locked until confirmation or timeout, but do not release on both at once; the return timeout is larger than the delivery window; reject a late callback by identifier; periodic reconciliation. The bridge usually does not provide atomicity between chains [^s6-atomicity].
- **Reorg** (a block rollback after a reaction). The bridge may have accepted a message by a transaction that is gone after the rollback. Protection: react to finalized blocks, the proof — on final data; the depth depends on Base and Etherlink [^s6-finality].
- **The bridge is unavailable or paused.** All operations except deposit stop. Protection: locked funds are recoverable by a local timeout, without the bridge's involvement; after recovery — re-sending the messages. The bridge usually has an operator with the right to pause — an external point of failure [^s6-trust].
- **Not enough native gas on Etherlink for the return path.** The withdrawal does not leave for Base, which for the user is a stuck withdrawal. Protection: build the cost of return delivery into the operation, or keep a replenishable gas reserve on Etherlink (depends on the bridge [^s6-gas]); a reserve with a price margin; monitoring; the operation stays in a "waiting for delivery" state, to be completed later. There must be no dead-end states.
- **Double execution of one message** (re-delivery, re-sending, reorg, a replay by an attacker). The worst case: a withdrawal or a credit beyond what was put in. Protection: idempotency — the message has a unique id, the first callback marks it as used, a repeat is ignored; source verification; an operation state model (created / waiting / completed / expired / returned). preimage+Merkle do not replace replay protection: one correct preimage cannot be applied twice.

## 7. Implementation complexity

What needs to be done and the relative complexity:

- **EscrowBase** (deposit/withdraw, allocate, calling the bridge adapter, accounting, access control) — medium. The state on Base is not final on its own: it changes after a message from Etherlink.
- **EscrowEtherlink** (allocation state, preimage+Merkle, receiving funds, release/refund, timelocks) — medium. The logic is local, but the data format and the order of checks match Base.
- **Bridge integration** (sending and receiving through the endpoint, sender verification, field encoding) — high. The main part; weak sender verification means an outsider can call release or refund.
- **Token handling** (lock/unlock or burn/mint) — high. The transfer of funds and the delivery of the message are two different events, diverging in time and in outcome.
- **Allocations, preimage, and Merkle** — medium. The algorithms are local; the hash format and the Merkle leaves match on both chains.
- **Timelocks and refund** — medium. refund is again the return path with all its complexities.
- **Handling cross-chain failures** (retries, idempotency, partial failures) — high. The hardest part.

The hardest spots: reliable message delivery with failure handling; the return path and paying its gas on Etherlink. The gas model of the return path and the delivery guarantees depend on the bridge (section 3).

## 8. Cost

On every operation except deposit we pay for the transfer between chains. The cost of an operation with a transfer = bridge fee + Base gas + Etherlink gas. If the transfer is needed there and back, all of this doubles: the return path is a second full payment, not a small addition.

By operation:

| Operation | Bridge transfers | Cost |
|---|---|---|
| deposit | 0 | only Base gas |
| allocate | 1 | 1× bridge fee + gas |
| withdraw | 2 | 2× bridge fee + gas |
| refund | 2 | 2× bridge fee + gas |

deposit is almost free. allocate pays for the bridge once. withdraw and refund — twice, these are the most expensive operations.

**Cost of one transfer** (a guide, the exact figure needs to be measured):

| Item | Cost |
|---|---|
| Bridge fee (LayerZero) | a few cents, up to \$1 in the bad case |
| Gas on Base | 1–3 cents |
| Gas on Etherlink | fractions of a cent |

- bridge fee — LayerZero's base charge of about \$0.005 plus message verification and delivery (DVN, executor); a few cents per message, up to \$1 in the bad case [^s8-lz];
- gas on Base — about \$0.007 for an ERC-20 transfer, typically \$0.03; plus a variable part for writing data to Ethereum [^s8-base];
- gas on Etherlink — about 0.0006 XTZ at 1 gwei, "\$0.001 or less" in external reviews [^s8-etl].

The main item is the bridge fee, not gas. **Conclusion:** of the three variants this one is the most expensive in operational cost: we pay for the bridge on every operation with a transfer, and this cannot be reduced — the transfer is built into the scheme.

## 9. Solo timeline

An estimate for one full-time developer, not a commitment. It depends on the choice of bridge (section 3) and on how the open questions are closed.

| Part | Weeks |
|---|---|
| EscrowBase | 0.3–0.7 |
| EscrowEtherlink | 0.3–0.7 |
| Bridge integration | 0.7–1 |
| preimage+Merkle, timelocks/refund | 0.3–0.7 |
| Handling cross-chain failures | 0.7–1 |

The parts overlap (integration and failure handling are done together), so the sum of the rows and the total need not match.

**Total: ~2.5–4 weeks.** Variant 1 leans on a ready-made bridge — no relayer of our own to write, which lowers the timeline. The audit is not included in the estimate.

## 10. Audit scope and cost

**What to audit:** EscrowBase; EscrowEtherlink and the consistency of the two contracts' states; the bridge integration (sending, receiving, **sender verification**, idempotency); token handling (lock/unlock or burn/mint, double unlock, non-standard tokens); preimage+Merkle; timelocks and refund; handling of cross-chain failures. The most sensitive part is the cross-chain logic and sender verification: an error there means a forged message unlocks or issues tokens.

**Price — a market estimate, not a quote:**
- One auditor's day rate is ~\$500–1200 per day on the market [^s10-rate].
- By project complexity: a simple contract ~\$5–20k; a mid-complexity DeFi ~\$40–100k; multi-contract cross-chain systems — from \$50k, for large ones \$150–300k+ [^s10-sherlock][^s10-mor]. A re-audit after fixes — usually +\$5–20k per round, and at least one is needed almost always.
- Surcharges: cross-chain and bridges ~25–30%, non-Solidity ~25–45%, urgency ~20–40% [^s10-sherlock][^s10-mor].
- Well-known auditors (MixBytes and others) do not publish fixed rates, they estimate per project [^s10-mixbytes].

**For Variant 1** (two contracts + a cross-chain integration) the guide is **~\$50–150k+** for the initial audit plus \$5–20k per re-audit round. The exact figure is given by an auditor against the code scope.

## 11. Pros / cons

**Pros:**
- No capital of our own is needed — the user's deposit is transferred, not our mirror capital.
- Trust is moved to an external bridge, not onto us; we do not hold the user's funds.
- The user's money really and verifiably passes through Etherlink — these are the same funds, the path is visible in the blockchain.
- Less code of our own — no relayer and operator of our own.

**Cons:**
- Expensive: a bridge fee and gas on every operation with a transfer.
- The user waits for the bridge on every operation.
- A dependency on a third-party bridge: its failure, pause, or vulnerability directly breaks the operation.
- Native gas on Etherlink is needed for the return path.
- Complex handling of cross-chain failures: status, retries, returning funds on failure.

## 12. Open questions

- Within LayerZero: use the ready-made Etherlink bridge or build your own OFT/OFTAdapter channel.
- Whether operation data can be carried together with the stablecoin transfer; the OFT/OFTAdapter addresses for USDC and USDT.
- Who pays the gas for the return path Etherlink → Base: the user, us (we sponsor), or a deduction from the amount.[^gas-payer]
- Who may start a refund: only the depositor or any address.
- release — a separate operation or part of withdraw (by default: a single action).
- What to do with stuck and undelivered messages and with partial delivery.
- Whether a native gas reserve on Etherlink is needed and who replenishes it.
- The price and latency of one transfer (to measure).
- The bridge's limits on the operation amount and on turnover; whether the delivery delay is acceptable.
- What to show the user while the transfer is in flight; how to reconcile amounts Base ↔ Etherlink.

[^bridges]: Bridges differ by trust — a spectrum from trustless/non-custodial to custodial.

    - Across — trust-minimized: an optimistic oracle (UMA), disputes are resolved in a decentralized way.
    - Relay.link — faster, rests on a selected set of intermediaries with collateral: security here is not mathematical but economic — it is unprofitable for the intermediary to cheat, but it may fail to execute; this is between trustless and custodial.

    There are also fully custodial bridges, where the funds are held by a central operator. The exact bridge for this variant we will choose in section 3.

[^q-contract]: **Why is it the contract that sends?** In Variant 1 there is no trusted operator — trust is only on the bridge. For Etherlink to accept a message as genuine, it must come from EscrowBase through the canonical bridge: the bridge confirms the sender, and EscrowEtherlink verifies that it is exactly our contract. An off-chain process or the user themselves could not do this — anyone could forge an allocation or a withdrawal. And only the contract can, in the same transaction, lock the funds on Base, so the message is backed by real money. Entrusting the sending to an off-chain service is already Variants 2/3, where the trust is on us.

[^q-etherlink]: **Why verification on Etherlink and not on Base?** The allocation record and its condition (preimage hash + Merkle root) exist only on Etherlink — they are not on Base, there is nothing to verify against. The money is also on Etherlink, so the command to return it is formed there too. The allocation state could be duplicated on Base, but then Etherlink is not needed for the logic — and by the task both the state and the TVL must pass through Etherlink.

[^msg-passing]: **How does a message get from Base to Etherlink and how does the contract receive it?** Contracts in different chains do not call each other directly, so this is three steps.

    1. Sending (on Base): EscrowBase calls the bridge endpoint contract on Base and passes it the message bytes and the recipient's address on Etherlink. For Base this is an ordinary local transaction.
    2. Delivery (outside the contracts): the bridge's external agents watch Base, verify the message, and deliver it to Etherlink together with a proof. These are the bridge's agents, not ours (in LayerZero — DVNs confirm, the executor delivers).
    3. Receiving (on Etherlink): the bridge endpoint calls a pre-declared callback function on EscrowEtherlink (in LayerZero — `_lzReceive`). The contract does not poll on its own — it is called. EscrowEtherlink verifies that the call came from the endpoint and that the sender is our EscrowBase (by chainId and address), and only then processes the message.

    External agents are still here, but they are the bridge's agents, and their work is verified on-chain by the endpoint itself — so we do not need an operator of our own. In Variants 2/3 we run these agents, and the trust is on us.

[^release-split]: This can be split into two actions — a separate release, then withdraw. This is needed if release for us is a manual permission to give the funds to the recipient (granted by the depositor or an arbiter), and withdraw is a separate moment when the recipient collects them: different actions, often different people. But if the condition is only the preimage, then revealing the secret is itself the release, and a separate step is not needed. The split would add one more message path through the bridge — more expensive and slower for the user.

[^gas-payer]: The gas for the transaction itself on Base is paid by whoever sends it. The contested part is the cross-chain charge: the bridge fee plus gas for execution on the other side, especially on the return path Etherlink → Base. Three options.

    1. The user pays: when calling on Base they attach an amount on top, it covers the bridge and gas on that side, including the return path. Free for us, but the user needs native gas and a variable fee on top of the operation.
    2. We pay (we sponsor): we keep and replenish a native gas reserve, the user signs an ordinary transaction. Simple for the user, but it is our ongoing expense and a dependency on us — it partly dilutes "in Variant 1 there is no operator".
    3. Deducted from the amount: we subtract the fee from the transferred funds or take a protocol fee on allocate. Self-funded, the user needs no native gas, but the recipient receives a bit less.

    A subtlety: the return path starts on Etherlink, and the delivery to Base can only be paid for with native gas on Etherlink. So it must be decided who replenishes the gas on the Etherlink side — otherwise the withdrawal gets stuck.

[^s3-tokens]: Etherlink — Tokens (addresses USDC `0x796Ea11Fa2dD751eD01b53C372fFDB4AAa8f00F9`, USDT `0x2C03058C8AFC06713be23e58D2febC8337dbfE6A`): https://docs.etherlink.com/building-on-etherlink/tokens/ ; LZUSDC: https://www.coingecko.com/en/coins/layerzero-bridged-usdc-etherlink ; LZUSDT: https://www.coinbase.com/price/layerzero-bridged-usdt-etherlink

[^s3-lz]: LayerZero deployments — Base (EID 30184): https://docs.layerzero.network/v2/deployments/chains/base ; Etherlink (EID 30292): https://docs.layerzero.network/v2/deployments/chains/etherlink

[^s3-evm-bridge]: Etherlink — Bridging between EVM networks (lock-mint on LayerZero, representation tokens, fee is only gas, time "a few minutes"): https://docs.etherlink.com/bridging/bridging-evm/

[^s3-cctp]: Circle — Cross-Chain Transfer Protocol (supported chains include Base, Etherlink absent): https://www.circle.com/cross-chain-transfer-protocol

[^s3-debridge]: deBridge Support (Base supported; Etherlink not named in the available list): https://debridge.com/support/

[^s3-bridging]: Etherlink — Bridging overview (Stargate as a third-party option, on top of LayerZero): https://docs.etherlink.com/bridging/

[^s3-tezos-bridge]: Etherlink — Bridging Tezos ↔ Etherlink (XTZ and FA tokens, ~16 c): https://docs.etherlink.com/bridging/bridging-tezos/

[^s4-admin]: The presence and composition of the contracts' admin functions (pause, upgradeability via proxy, changing the trusted sender/endpoint) is a design decision, not fixed at the time of writing.

[^s6-bridge]: The availability of re-delivery of a stuck message, the way to re-send it, and its cost depend on the chosen bridge; clarified in section 3.

[^s6-atomicity]: A canonical bridge in the general case does not guarantee atomicity of changes between two chains: the steps are applied one after another. The exact guarantees depend on the chosen bridge (section 3).

[^s6-finality]: Chain finality. **Base** (OP Stack L2): in about 2 minutes the transaction makes it into a batch on Ethereum, and no L2 reorgs after that have ever been recorded; full Ethereum-level finality — in ~13 minutes (2 epochs, 64 L1 blocks). **Etherlink** (Tezos Smart Rollup): deterministic finality in 2 Tezos blocks (a few seconds), soft confirmations < 500 ms. In practice: react to a Base event after it makes it into a batch (≈2 min) or after full finality (~13 min), on Etherlink — after 2 blocks. Sources: https://docs.base.org/base-chain/network-information/transaction-finality , https://docs.etherlink.com/network/architecture/

[^s6-trust]: The bridge's trust model — the management roles, the right to pause, the emergency stop — depends on the chosen bridge; clarified in sections 3 and 4.

[^s6-gas]: The ways to pay for return delivery (payment at the moment of the operation or a replenishable gas reserve on Etherlink) depend on the chosen bridge; clarified in section 3.

[^s8-lz]: LayerZero — fees and the OFT standard: https://defillama.com/protocol/layerzero ; https://docs.layerzero.network/v2/developers/evm/configuration/gas-fees ; https://layerzero.network/blog/explaining-the-oft-standard

[^s8-base]: Gas on Base: https://coincodex.com/article/59940/fees-for-transfering-usdc-on-the-base-network ; https://sqmagazine.co.uk/ethereum-gas-fees-statistics/

[^s8-etl]: Gas on Etherlink: https://docs.etherlink.com/building-on-etherlink/estimating-fees/ ; https://thirdweb.com/etherlink

[^s10-rate]: FailSafe — auditor comparison, day rate \$500–1200/day: https://getfailsafe.com/how-to-guide-to-compare-cheap-smart-contract-auditors-in-july-2025

[^s10-sherlock]: Sherlock — Smart Contract Audit Pricing 2026 (ranges by complexity, surcharges, re-audit): https://sherlock.xyz/post/smart-contract-audit-pricing-a-market-reference-for-2026

[^s10-mor]: Mor Software — Smart Contract Audit Cost (cross-chain up to \$150k+, surcharge for niche expertise): https://morsoftware.com/blog/smart-contract-audit-cost

[^s10-mixbytes]: MixBytes — Audit (price per project, a minimum of two auditors per part of the code): https://mixbytes.io/audit
