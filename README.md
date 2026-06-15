# escrow

A simple escrow contract (`deposit` / `withdraw`, `allocate` / `release` / `refund`)
reworked so that the business logic and all user interaction stay on **Base**, while
the funds (USDC/USDT — the TVL) are routed through **Etherlink**.

Users interact only with Base: they deposit on Base and withdraw allocated funds on
Base. The escrow funds must cycle through Etherlink. State is split — deposit state on
Base, allocation state on Etherlink.

Full task statement: [task.MD](task.MD).

## Approaches under evaluation

Three ways to route the stablecoins. Each is specified in its own document.

| Variant | What goes to Etherlink | How it moves | User waits to withdraw? | Trust | Our capital | Running cost |
|---|---|---|---|---|---|---|
| **[1. Canonical Bridge](1-canonical-bridge.md)** | user's deposit, 1:1 | canonical bridge, every operation | yes, on each op | bridge / protocol | none | very high — bridge fee per op |
| **[2. Event-Driven Mirror](2-event-driven-mirror.md)** | our own mirror capital | our Go relayer, per event | no — paid from Base instantly | us | maximum | very low — Etherlink gas + relayer |
| **[3. Batched Settlement](3-batched-settlement.md)** | real liquidity, aggregated | our relayer, in batches | yes — waits for batch return | us | minimum | low–medium — batched, leans cheap |

- **Canonical Bridge** — moves the real user deposit across on every operation via a canonical bridge (LayerZero OFT / Circle CCTP / other — to be chosen).
- **Event-Driven Mirror** — keeps user funds on Base and mirrors the amount on Etherlink with our own capital, so the user never waits for a cross-chain round trip.
- **Batched Settlement** — moves real liquidity in batches, trading user-facing latency for far less locked capital.

Each document covers: idea, architecture, per-operation flow, stablecoin transfer
mechanism, trust & security, liquidity/solvency, failure modes, complexity, cost,
solo timeline, audit scope & cost, pros/cons, and open questions.

## Cost of Build & Maintenance

| Variant | Build (one-time) | Maintenance / year |
|---|---|---|
| **1. Canonical Bridge** | ~\$6–9.6k | ~\$2–4k |
| **2. Event-Driven Mirror** | ~\$4.8k | ~\$5–9k + ~10% of locked capital |
| **3. Batched Settlement** | ~\$7.2–9.6k | ~\$8–13.5k + ~10% of the buffer |

Build is one-time solo development at \$60/h, 40 h/week. The hours per variant come from
the solo timeline (section 9 of each spec). The audit is not included; it is a separate
one-time cost (section 10 of each spec).

Maintenance per year is the sum of four items:

- Infrastructure (server, monitoring): ~\$0.5–1k for Variant 1, ~\$1–2k for Variants 2
  and 3.
- Support: hours per month × \$60. ~2–4 h/month for Variant 1, ~6–10 for Variant 2,
  ~10–16 for Variant 3.
- Key custody: self-custody multisig; little direct cost, the time is included in the
  support hours. None for Variant 1.
- Cost of capital: locked capital × ~10%/year. None for Variant 1; the full mirror for
  Variant 2; the buffer for Variant 3. The 10% is an assumed annual rate (opportunity or
  borrowing cost), not a measured figure.

The per-operation running cost is the Running cost column in the table above. Per-variant
detail is in section 8 of each spec.

## Recommended path

Of the three approaches, **Variant 2 is the cheapest to run and the simplest to
build**. Variant 3 is more expensive and has many hard parts — cross-chain solvency
accounting, a payout buffer, a payout queue, and batch rebalancing — so the sensible
place to start is Variant 2.

- **MVP** — Variant 2 as specified: all logic and payouts on Base, with our mirror
  capital on Etherlink.
- **Evolution** — keep the source of truth and payouts on Base, but instead of mirror
  capital, move each allocation's real funds to Etherlink on `allocate` and pull them
  back to Base on `release` / `refund` / `withdraw`. Real TVL then actually cycles
  through Etherlink.

The evolution stays simple: with each allocation owning its own funds, payouts follow
the money instead of drawing on a shared buffer, so no buffer, queue, or solvency
accounting is needed — markedly simpler than full Variant 3. The cost is that the user
waits for the funds to return on withdrawal, and real user funds cross the bridge (the
same bridge risk as Variant 1). Removing the wait would require a buffer, which brings
back the queue and solvency accounting — i.e. full Variant 3.

I have implemented Variant 2 twice — at Clip Finance and Unigox — so this is a design
proven in practice, not a theoretical one. Across five years of working with different
bridges and architectures, I have not seen a solution cheaper to run than Variant 2. In
a real decision the running cost is what matters most; user wait time and the custody
model (custodial / non-custodial) matter far less. The numbers in Variant 2 are the
unavoidable cost of the task itself: any scheme has to pay gas for ERC-20 transfers, and
Variant 2 adds no bridge fee on top — that is the operational floor for this task.
