<div align="center">

# 🎟️ TicketFi — Raffle Program

**On-chain raffle rooms, ticket purchases & Switchboard VRF draws**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](../../LICENSE)
[![Built with Anchor](https://img.shields.io/badge/Built%20with-Anchor-9945FF)](https://www.anchor-lang.com/)
[![Network: Solana](https://img.shields.io/badge/Network-Solana-14F195)](https://solana.com)
[![Module: Raffle](https://img.shields.io/badge/Module-Raffle-blueviolet)]()

[🌐 Website](https://ticketfi.com) · [📄 Whitepaper](../../WHITEPAPER.md) · [🐦 Twitter/X](https://x.com/ticketfi)

</div>

---

The core engine of the TicketFi platform. This smart contract is responsible for initializing raffle rooms, managing ticket purchases, enforcing the Switchboard VRF drawing protocol, and executing the platform's transparent fee-sharing and refund mechanisms on Solana.

## 🌎 Raffle Assets (SOL, BTC & Real World Assets)

TicketFi supports both cryptocurrency prizes and tokenized/physical **Real World Assets (RWA)**:
* **SOL & BTC Raffles**: Ticket pricing and prize payouts denominated in Native SOL or wrapped tokens.
* **Real World Assets (RWA) Raffles**: High-value physical goods such as:
  * 📦 **Pokémon Card Packs** (e.g., vintage packs, graded boosters)
  * ⌚ **Luxury Watches** (e.g., tokenized/physical watches via Courtyard.io)
  * 🪙 **Collectible Coins** (e.g., silver/gold coins)
* **Payment Currencies**: Supports both Native SOL and USDC (`buy_ticket_spl`) ticket purchases.

---

## 🔄 Room Expiry & Refund Scenarios (Platform FAQ Rules)

Every raffle room is bound by a maximum ticket capacity (`max_tickets`) and an expiration deadline (`ends_at`). The program executes three deterministic pathways depending on the room status at expiration:

### Case A: 100% Sold Out (Full Room)
* **Trigger**: All `max_tickets` are sold before the expiration time.
* **Drawing**: The room status changes to `drawing`. Switchboard VRF is requested on-chain to generate cryptographically secure randomness.
* **Settlement**: The winner is determined (`random_u32 % tickets_sold`) and receives **71.43%** of the collected pool. The platform fee of **28.57%** is split (Buyback, Treasury, Jackpot).
* **Reopening**: The room resets `tickets_sold = 0`, `unique_buyer_count = 0`, increments the `room_cycle`, and extends the deadline, reopening the room for the next round.

### Case B: Expired but Partially Sold ($\ge 2$ Unique Buyers)
* **Trigger**: The expiration time (`ends_at`) passes, the room is *not* full, but **2 or more unique wallets** have bought tickets.
* **Drawing**: The room **will not refund**. Instead, a draw is triggered.
* **Settlement**: Switchboard VRF selects a winner from the tickets that *were* sold:
  $$\text{winning\_index} = \text{random\_value} \pmod{\text{tickets\_sold}}$$
  The winner receives **71.43%** of the accumulated pool, and the **28.57%** fee split is distributed.
* **Reopening**: The room cycles, resets, and reopens.

### Case C: Expired with Insufficient Participation ($\le 1$ Unique Buyer)
* **Trigger**: The expiration time passes, and **0 or 1 unique buyers** have purchased tickets.
* **Refund**: The `handle_expired_room` instruction is called:
  * If there is **1 buyer**, **100% of their ticket price is immediately refunded** back to their wallet on-chain.
  * If there are **0 buyers**, no refund is processed.
* **Reset & Reopen**: To keep the platform active, the room is reset (`tickets_sold = 0`, `unique_buyer_count = 0`), the `room_cycle` is incremented (invalidating old buyer entry PDAs so they can participate in the next round), and the expiration timer is extended by `DEFAULT_ROOM_EXTENSION_SECONDS` (24 hours).

---

## 🛠️ Instruction Reference

| Instruction | Access | Description |
| :--- | :--- | :--- |
| `initialize_platform` | Admin | Sets up global treasury, buyback, and jackpot wallets. |
| `initialize_raffle_room`| Admin | Creates a new raffle room (SOL or USDC) with price, max tickets, and expiration deadline. |
| `buy_ticket` | Public | Purchases a ticket for a SOL room. Transfers SOL to the vault PDA. Mints a deterministic Ticket account. |
| `buy_ticket_spl` | Public | Purchases a ticket for a USDC room. Transfers USDC tokens to the vault token account. |
| `commit_draw` | Admin/Bot | Commits the drawing process for full or expired ($\ge 2$ buyers) rooms, locking in the Switchboard VRF account. |
| `settle_draw` | Admin/Bot | Resolves the Switchboard VRF output, verifies the winner, distributes the prize pool, and updates room status. |
| `handle_expired_room` | Admin/Bot | Resets an expired room with $\le 1$ buyer, triggers refunds, increments room cycle, and extends deadline by 24h. |
| `admin_rescue_spl` | Admin | Rescue utility for admin to withdraw any stuck SPL tokens from vault accounts. |

---

## 🎲 Provably Fair Randomness (Switchboard VRF)

All draws utilize **Switchboard On-Demand VRF (Verifiable Random Function)**:
1. **On-Chain Request**: When drawing is committed, the program records the current slot.
2. **SGX Enclave Execution**: Switchboard off-chain oracles execute the random number generation inside a secure Intel SGX enclave.
3. **Cryptographic Verification**: The generated randomness is submitted back to the Solana network, where its signature is verified on-chain.
4. **Unmanipulable Winner Selection**: Neither the admin, players, nor the oracles can predict or alter the outcome, guaranteeing absolute fairness.
