# TicketFi Whitepaper

**A Non-Custodial, Decentralized Raffle & Staking Protocol on Solana**

*Document Version: 1.0.0 (Release Candidate)*  
*Target Network: Solana (Mainnet)*  
*Official Website:* [ticketfi.com](https://ticketfi.com)

---

## 1. Executive Summary

TicketFi is a decentralized, non-custodial gaming and utility protocol built on the Solana blockchain. It addresses the lack of transparency, trust, and structural sustainability in traditional online raffles. 

By combining on-chain smart contracts, **Verifiable Random Functions (VRF)** via Switchboard, and a sustainable **buyback-and-stake tokenomics loop**, TicketFi aligns the incentives of casual gamers, large-scale players, and long-term token holders. 

Every raffle room generates atomic fee splits that feed buy pressure for the native `$TICKETFI` token, rewarding stakers with automated buyback distributions and competitive APY yields.

---

## 2. The Problem

The global online tombola and raffle market is growing rapidly but suffers from three systemic issues:

1. **Lack of Transparency & Trust (Opaque Randomness):** Traditional raffle websites rely on off-chain scripts, random number generators (RNG) controlled by admins, or physical draws that can easily be manipulated. Users have no mathematical proof that the winner was selected fairly.
2. **High Middleman Fees with Zero Token Utility:** Legacy platforms charge up to 30-50% in fees. These fees enrich the central platform owners, leaving players with no secondary value or governance power.
3. **Unsustainable Token Models in Web3 Gaming:** Most GameFi projects rely on inflationary minting mechanics to reward users. When demand drops, the token price collapses due to constant sell pressure.

---

## 3. The TicketFi Solution

TicketFi introduces a fully transparent on-chain raffle architecture combined with a deflationary reward flywheel.

```
                  ┌──────────────────────────────┐
                  │      Raffle Ticket Sales     │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                     Atomic Fee Split (28.57%)
                                 │
         ┌───────────────────────┼───────────────────────┐
         │ (50% of fee)          │ (49% of fee)          │ (1% of fee)
         ▼                       ▼                       ▼
 ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
 │ $TICKETFI Buy │       │   Treasury    │       │    Jackpot    │
 │  Market Order │       │  (Operations) │       │  Accumulator  │
 └───────┬───────┘       └───────────────┘       └───────────────┘
         │
         ▼
 ┌───────────────┐
 │ Staking Pool  │
 │ (Redistribute)│
 └───────────────┘
```

### Protocol Pillars:
- **On-chain PDA Vaults:** All ticket funds (SOL) are deposited into Program Derived Address (PDA) vaults. No administrator, developer, or third party can access or withdraw user funds during an active room.
- **Provably Fair Draws:** Randomness is requested on-chain from the **Switchboard On-Demand VRF Oracle**. Winners are determined cryptographically through verified randomness, preventing any form of frontrunning or admin manipulation.
- **Fees to Buy Pressure Loop:** A significant portion of the fees collected from every raffle room is converted directly into `$TICKETFI` market buys, creating persistent demand.

---

## 4. Raffle Room Lifecycle

TicketFi raffle rooms are governed entirely by autonomous smart contracts.

### Step 1: Initialization
The protocol administrator initializes a raffle room with defined parameters:
- **Ticket Price:** Constant cost in SOL per ticket entry.
- **Maximum Tickets:** The hard cap of entries for the room.
- **Prize Pool:** The guaranteed prize amount to be won.
- **Room Expiration (`ends_at`):** A UNIX timestamp marking the drawing deadline (typically 24 hours).

### Step 2: Buying Entries
Users purchase entries by calling `buy_ticket`. The transaction:
1. Transfers SOL from the buyer to the room’s non-custodial PDA vault.
2. Mints a deterministic `Ticket` account on-chain containing the buyer’s pubkey and ticket index.
3. Tracks unique buyers via a `BuyerEntry` PDA to ensure correct state validation at expiry.

### Step 3: Expiry & On-Chain Resolution (Two Scenarios)
When the room's deadline expires, the smart contract evaluates the state:

#### **Scenario A: Low Participation (0 or 1 Unique Buyer)**
If the room expires with **0 or 1 unique buyer**, a drawing is mathematically illogical (a single buyer would win their own funds minus fees). 
- **Action:** The room is reset.
- **On-Chain Refund:** If there is exactly 1 buyer, the smart contract atomically refunds **100% of their SOL** from the vault.
- **State Reset:** Tickets sold are set to 0, the room cycle increments (invalidating old ticket accounts), and the timer is extended by another 24 hours.

#### **Scenario B: Active Participation (2+ Unique Buyers)**
If the room expires with 2 or more unique buyers, it qualifies for a drawing, regardless of whether all tickets were sold.
- **Action:** The draw is committed and resolved on-chain using Switchboard VRF.
- **Dynamic Pool Drawing:** The winner is selected using the modulo of the actual tickets sold:
  $$\text{Winner Index} = \text{VRF Random Number} \pmod{\text{Tickets Sold}}$$
  This prevents drawings from getting stuck on partially-filled rooms.

---

## 5. Tokenomics & Fee Distribution

Every successfully resolved raffle room distributes the collected SOL vault balance atomically:

| Stakeholder | % of Total SOL | % of Platform Fee (28.57%) | Sourced From | Purpose |
|-------------|----------------|----------------------------|--------------|---------|
| **Winner** | **71.43%** | - | Total SOL | Paid out instantly to the winning wallet. |
| **Buyback Loop** | **~14.28%** | **50.00%** | Platform Fee | Transferred to Buyback wallet to purchase `$TICKETFI` and fund staking rewards. |
| **Treasury** | **~13.99%** | **49.00%** | Platform Fee | Dedicated to protocol development, marketing, and hosting costs. |
| **Jackpot** | **~0.29%** | **1.00%** | Platform Fee | Accumulated in the Jackpot vault for special events. |

---

## 6. Staking Program & Utility

The `$TICKETFI` token is built to capture the value generated by the raffle platform. Users who lock their tokens in the staking contract are rewarded with both guaranteed staking yields (APY) and platform buyback rewards.

### Tiered Lockups & APY
Stakers can choose from four lockup durations, each offering distinct yield rates:

| Tier | Lock Duration | Staking Power Multiplier | Guaranteed APY |
|------|---------------|--------------------------|----------------|
| **Tier 0** | 7 Days | 1.0x | **10%** |
| **Tier 1** | 30 Days | 1.5x | **15%** |
| **Tier 2** | 90 Days | 2.5x | **25%** |
| **Tier 3** | 180 Days | 5.0x | **50%** |

### Dynamic Staking Weight Redistribution
Traditional staking protocols often lock up rewards inside empty tiers if no users participate in a particular bracket. 

TicketFi resolves this via **Dynamic Weight Redistribution**. Platform fees are split only among tiers that have **active stakers**. If a tier is empty, its reward weight is dynamically redistributed to active tiers, ensuring that 100% of deposited platform rewards are distributed to the community without locking tokens inside empty arrays.

---

## 7. Security Architecture

TicketFi prioritizes smart contract safety and decentralization:

1. **Non-Escrow Vaults:** All raffle funds are locked in PDAs derived from the room ID. No admin key can withdraw SOL from an active room.
2. **Two-Step Admin Transfer:** Transitioning ownership of the protocol requires a two-step handshake (`propose_admin` and `accept_admin`) to prevent permanent admin lockout through typographical errors.
3. **Emergency Pause Circuit Breaker:** The admin can trigger `set_paused` in case of market volatility or issues. This **only blocks new ticket purchases**, ensuring that ongoing draws, settlements, and refunds can still be executed to protect user funds.
4. **Verifiable Randomness:** We utilize Switchboard On-Demand VRF. The contract enforces strict owner checks (`randomness_account_data.owner == &SWITCHBOARD_PROGRAM_ID`) to prevent oracle manipulation.

---

## 8. Conclusion

TicketFi bridges the gap between decentralized finance (DeFi) and online gaming. By shifting raffle mechanics entirely on-chain, eliminating custodian risks, and directing platform fee revenues into a market buyback loop for `$TICKETFI` stakers, TicketFi creates a secure and economically sustainable playground on Solana.
