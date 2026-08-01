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
                               │     Raffle Ticket Sales      │
                               │  (140% of target valuation)  │
                               └──────────────┬───────────────┘
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      │ (100% Prize Value)                            │ (40% Platform Fee)
                      │ (71.43% of total pool)                        │ (28.57% of total pool)
                      ▼                                               ▼
             ┌──────────────────────┐                     ┌───────────────────────┐
             │    Winner Payout     │                     │   Atomic Fee Split    │
             │ (SOL/BTC/RWA Purc..) │                     └───────────┬───────────┘
             └──────────────────────┘                                 │
                                              ┌───────────────────────┼───────────────────────┐
                                              │ (50% of fee)          │ (49% of fee)          │ (1% of fee)
                                              │ (20% Prize Value)     │ (19.6% Prize Value)   │ (0.4% Prize Value)
                                              │ (14.28% of total)     │ (13.99% of total)     │ (0.29% of total)
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
- **On-chain PDA Vaults:** All ticket funds (SOL or USDC) are deposited into Program Derived Address (PDA) vaults. No administrator, developer, or third party can access or withdraw user funds during an active room.
- **Provably Fair Draws & Winner Payouts:** Randomness is requested on-chain from the **Switchboard On-Demand VRF Oracle** to ensure unmanipulable drawings. Upon drawing settlement, **71.43%** of the ticket sales pool is instantly distributed to the winner's wallet (for cryptocurrency rooms) or sent to the Treasury to buy and ship the physical RWA prize (for asset rooms).
- **Fees to Buy Pressure Loop:** 50% of the platform fees (representing **~14.28%** of every room's total ticket sales) are automatically swapped for `$TICKETFI` on the open market and routed to the staking pool.

---

## 4. Raffle Room Lifecycle

TicketFi raffle rooms are governed entirely by autonomous smart contracts supporting both native cryptocurrencies and SPL tokens.

### Step 1: Initialization
The protocol administrator initializes a raffle room with defined parameters:
- **Room Type:** Specifies if the room is a Native SOL room (0) or a USDC/SPL-token room (1).
- **Ticket Price:** Constant cost in SOL or USDC per ticket entry.
- **Maximum Tickets:** The hard cap of entries for the room.
- **Prize Pool:** The guaranteed prize amount (for crypto rooms) or the target RWA physical item (for RWA rooms).
- **Room Expiration (`ends_at`):** A UNIX timestamp marking the drawing deadline (typically 24 hours).

### Step 2: Buying Entries
Users purchase entries by calling `buy_ticket` (for SOL) or `buy_ticket_spl` (for USDC). The transaction:
1. Transfers SOL or USDC from the buyer to the room’s non-custodial PDA vault.
2. Mints a deterministic `Ticket` account on-chain containing the buyer’s pubkey and ticket index.
3. Tracks unique buyers via a `BuyerEntry` PDA to ensure correct state validation at expiry.

### Step 3: Expiry & On-Chain Resolution (Three Scenarios)
When the room's deadline expires, the smart contract evaluates the state:

#### **Scenario A: Insufficient Participation (0 or 1 Unique Buyer)**
If the room expires with **0 or 1 unique buyer**, a drawing is mathematically illogical (a single buyer would win their own funds minus fees). 
- **Action:** The room is reset.
- **On-Chain Refund:** If there is exactly 1 buyer, the smart contract (`handle_expired_room`) atomically refunds **100% of their ticket purchase price (SOL or USDC)** from the vault.
- **State Reset:** Tickets sold are set to 0, unique buyer count is set to 0, the room cycle increments (invalidating old ticket accounts), and the timer is extended by another 24 hours to restart the raffle.

#### **Scenario B: Active Participation (2+ Unique Buyers, Not Full)**
If the room expires with **2 or more unique buyers** but is not fully sold out:
- **Action:** The room **will not refund**. Instead, the draw is committed and resolved on-chain using Switchboard VRF to select a winner among active participants.
- **Dynamic Pool Drawing:** The winner is selected using the modulo of the actual tickets sold:
  $$\text{Winner Index} = \text{VRF Random Number} \pmod{\text{Tickets Sold}}$$
  This prevents drawings from getting stuck on partially-filled rooms.

#### **Scenario C: 100% Sold Out (Full Room)**
If the room reaches its `max_tickets` sold prior to expiration:
- **Action:** The drawing is immediately committed, and the Switchboard VRF resolves the winner from the full ticket array:
  $$\text{Winner Index} = \text{VRF Random Number} \pmod{\text{Maximum Tickets}}$$

---

## 5. Tokenomics & Fee Distribution

To maintain transparent and sustainable mathematical models, TicketFi raffle rooms operate on a **100% + 40% Target Valuation Model**:
* **100%** represents the target prize value (the amount paid directly to the winning wallet or utilized by the treasury to purchase/fulfill the physical RWA prize).
* **40%** represents the platform fee added on top of the target prize value.
* This brings the **Total Target Room Capacity** (100% sold tickets) to **140%** of the prize value.

Consequently, when we look at the **total pool of collected ticket sales (140% of target value)**, the exact percentage distributions represent:
* **71.43%** of the total collected pool goes directly to the winner/RWA purchase ($100 / 140 \approx 71.43\%$).
* **28.57%** of the total collected pool goes to the platform fee split ($40 / 140 \approx 28.57\%$).

The **28.57% Platform Fee** is distributed atomically on-chain as follows:
* **50% of the fee** (~14.28% of the total collected pool, which corresponds to exactly **20%** of the target prize value) goes to the **Buyback Loop** to buy back `$TICKETFI` and route it to stakers.
* **49% of the fee** (~13.99% of the total collected pool, which corresponds to exactly **19.6%** of the target prize value) goes to the **Treasury** for operations.
* **1% of the fee** (~0.29% of the total collected pool, which corresponds to exactly **0.4%** of the target prize value) goes to the **Jackpot Accumulator**.

Every successfully resolved raffle room distributes the collected ticket revenues (SOL or USDC) vault balance atomically:

### SOL-Denominated Crypto Rooms (SOL & BTC Prizes)
For native cryptocurrency prize rooms, the winner's prize is paid out directly on-chain:

| Stakeholder | % of Total Pool | % of Platform Fee (28.57%) | Sourced From | Purpose |
|-------------|----------------|----------------------------|--------------|---------|
| **Winner** | **71.43%** | - | Total Pool | Paid out instantly to the winning wallet on-chain. |
| **Buyback Loop** | **~14.28%** | **50.00%** | Platform Fee | Transferred to Buyback wallet to purchase `$TICKETFI` and fund staking rewards. |
| **Treasury** | **~13.99%** | **49.00%** | Platform Fee | Dedicated to protocol development, marketing, and hosting costs. |
| **Jackpot** | **~0.29%** | **1.00%** | Platform Fee | Accumulated in the Jackpot vault for special events. |

### USDC/SPL-Denominated Rooms (Real-World Assets)
For physical Real-World Assets (RWA), the winner receives the physical item, and the ticket sales revenue is split to fund the acquisition and shipping:

| Stakeholder | % of Total Pool | % of Platform Fee (28.57%) | Sourced From | Purpose |
|-------------|----------------|----------------------------|--------------|---------|
| **RWA Purchase (Treasury)** | **71.43%** | - | Total Pool | Transferred to Treasury to purchase and ship the physical asset (e.g. watch, Pokémon cards, coins) to the winner. |
| **Buyback Loop** | **~14.28%** | **50.00%** | Platform Fee | Transferred to Buyback token vault to purchase `$TICKETFI` and fund staking rewards. |
| **Treasury (Ops)** | **~13.99%** | **49.00%** | Platform Fee | Operational and platform maintenance costs. |
| **Jackpot** | **~0.29%** | **1.00%** | Platform Fee | Transferred to Jackpot token vault. |

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

TicketFi resolves this via **Dynamic Weight Redistribution**. Platform fees are split only among tiers that have **active stakers** (weights 10 / 15 / 25 / 50). If a tier is empty, its reward weight is dynamically redistributed to active tiers, ensuring that 100% of deposited platform rewards are distributed to the community without locking tokens inside empty arrays.

---

## 7. Security Architecture

TicketFi prioritizes smart contract safety and decentralization:

1. **Non-Escrow Vaults:** All raffle funds are locked in PDAs derived from the room ID. No admin key can withdraw SOL or USDC from an active room.
2. **Two-Step Admin Transfer:** Transitioning ownership of the protocol requires a two-step handshake (`propose_admin` and `accept_admin`) to prevent permanent admin lockout through typographical errors.
3. **Emergency Pause Circuit Breaker:** The admin can trigger `set_paused` in case of market volatility or issues. This **only blocks new ticket purchases**, ensuring that ongoing draws, settlements, and refunds can still be executed to protect user funds.
4. **Verifiable Randomness:** We utilize Switchboard On-Demand VRF. The contract enforces strict owner checks (`randomness_account_data.owner == &SWITCHBOARD_PROGRAM_ID`) to prevent oracle manipulation.

---

## 8. Real-World Asset (RWA) Integration & Ecosystem Support

TicketFi is designed from the ground up to support high-value physical goods alongside cryptocurrencies, using USDC/SPL ticket rooms to target three primary asset classes:

* **⌚ Luxury Watches:** High-end watches (e.g., Rolex, Audemars Piguet) tokenized or backed physically. TicketFi's Switchboard VRF ensures that multi-thousand dollar luxury items are distributed with absolute mathematical fairness and proof of drawing transparency.
* **📦 Pokémon Card Packs:** Graded, high-value vintage booster packs (e.g., Base Set, Skyridge) and platinum packs. Winners can redeem their cards physically or trade the corresponding asset-backed tokens.
* **🪙 Collectible Coins:** Physical collectible coins (silver/gold) and bullion packs.

For all RWA rooms, the 71.43% portion of ticket sales is automatically routed to the Treasury wallet to cover the cost of the physical item purchase, grading, vaulting, and secure insured shipping to the winner's global address.

---

## 9. Conclusion

TicketFi bridges the gap between decentralized finance (DeFi), Real-World Assets (RWA), and online gaming. By shifting raffle mechanics entirely on-chain, eliminating custodian risks, and directing platform fee revenues into a market buyback loop for `$TICKETFI` stakers, TicketFi creates a secure and economically sustainable playground on Solana.
