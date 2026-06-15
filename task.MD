# escrow

## Task statement

We have a minimal escrow contract:
- deposit / withdrawal
- allocate / release / refund

It needs to be reworked so that the business logic stays as is and availability on the
Base network is preserved, while the movement of USDC/USDT goes through Etherlink.

Users will make deposits on Base and withdraw allocated funds from there as well, while
the money that takes part in the escrow needs to cycle through Etherlink.

Users interact only with Base. The TVL passes through Etherlink.
State is partially on both chains: deposit state on Base, allocation state on Etherlink.

### Contracts

- **EscrowBase.sol** (Base): deposit, allocate + OFTAdapter (lock/unlock USDT)
- **EscrowEtherlink.sol** (Etherlink): allocation state, verify preimage + OFT (mint/burn)

### Flow

1. Depositor makes a deposit on Base
   - USDT is stored locally in EscrowBase
   - cross-chain is not needed

2. Depositor calls allocate() on Base
   - OFTAdapter.lock(USDT) — USDT is locked on Base
   - [LZ OFT] -->
   - OFT.mint(wrappedUSDT) — wrappedUSDT appears on Etherlink
   - [LZ message] -->
   - EscrowEtherlink._lzReceive() — receives the allocation parameters, records state

3. Receiver obtains the preimage off-chain

4. Receiver calls withdraw() on Base
   - [LZ message] -->
   - EscrowEtherlink._lzReceive() — verifies the preimage + Merkle proof
   - OFT.burn(wrappedUSDT) — wrappedUSDT is burned on Etherlink
   - [LZ OFT] -->
   - OFTAdapter.unlock(USDT) — USDT is unlocked on Base
   - USDT arrives at the receiver's address on Base

### What to evaluate

- complexity
- cost and timeline
- risks

The cost of bridge / debridge or mint/burn depends on how the stablecoin is moved. The
scheme uses LayerZero as the means of transferring USDT/USDC, but this may not be the
best option.
