<div align="center">

# TicketFi — Smart Contracts

**On-chain raffle & staking protocol built on Solana**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Built with Anchor](https://img.shields.io/badge/Built%20with-Anchor-9945FF)](https://www.anchor-lang.com/)
[![Network: Solana](https://img.shields.io/badge/Network-Solana-14F195)](https://solana.com)
[![Status: In Development](https://img.shields.io/badge/Status-In%20Development-orange)]()
[![Audit: Pending](https://img.shields.io/badge/Audit-Pending-yellow)]()

[🌐 Website](https://ticketfi.com) · [📄 Whitepaper](./WHITEPAPER.md) · [💬 Discord](#) · [🐦 Twitter/X](#)

</div>

---

## 📋 Overview

TicketFi is a transparent, provably-fair raffle protocol on Solana. Users buy tickets for on-chain raffle rooms, winners are selected using **Switchboard VRF** (Verifiable Random Function), and a portion of every room's fees automatically funds a **TicketFi token buyback**, creating sustainable tokenomics.

### Core Features

- 🎟️ **Raffle Rooms** — Fixed-price ticket rooms with configurable max tickets and prize pools
- 🎲 **Provably Fair Draws** — Winner selection via Switchboard On-Demand VRF — fully verifiable on-chain
- 💰 **Automatic Fee Distribution** — 71.43% to winner · 28.57% split between treasury, buyback & jackpot
- 🔒 **Token Staking** — Lock TICKETFI tokens for 7/30/90/180 days to earn protocol rewards
- 🔄 **Buyback Mechanism** — Platform fees used to buy back TICKETFI from open market
- ⏸️ **Emergency Pause** — Admin can pause new ticket purchases only (draws & settlements unaffected)

---

## 📦 Programs

| Program | Program ID | Description |
|---------|-----------|-------------|
| `ticketfi-raffle` | `Coming soon — post audit` | Raffle rooms, ticket purchases, VRF draw logic |
| `ticketfi-staking` | `Coming soon — post audit` | Token staking with tiered lock periods & rewards |

> 📋 **Note:** Program IDs will be published after the security audit is complete and programs are deployed to Solana Mainnet.

---

## 🏗️ Architecture

### Raffle Program — Account Structure

```
PlatformConfig (singleton PDA)
├── admin: Pubkey
├── treasury_wallet: Pubkey
├── buyback_wallet: Pubkey
├── jackpot_wallet: Pubkey
├── room_counter: u64
├── paused: bool
└── pending_admin: Pubkey

RaffleRoom (one per room)
├── ticket_price: u64           (lamports)
├── max_tickets: u32
├── prize_amount: u64
├── tickets_sold: u32
├── status: u8                  (0=active · 1=finished · 2=drawing)
├── winner: Option<Pubkey>
├── randomness_account: Pubkey  ← Switchboard VRF account
├── commit_slot: u64
└── ends_at: i64

Ticket (PDA per ticket: ["ticket", room, index])
├── raffle_room: Pubkey
├── buyer: Pubkey
├── ticket_index: u32
└── bought_at: i64
```

### Staking Program — Lock Tiers

| Tier | Lock Period | Reward Pool Index |
|------|------------|-------------------|
| 0    | 7 days     | `acc_reward[0]`   |
| 1    | 30 days    | `acc_reward[1]`   |
| 2    | 90 days    | `acc_reward[2]`   |
| 3    | 180 days   | `acc_reward[3]`   |

---

## 🔄 Raffle Flow

```
1. Admin  →  initialize_platform()
             Sets treasury, buyback, jackpot wallets
             ↓
2. Admin  →  initialize_raffle_room(price, max_tickets, prize, ends_at)
             Creates room with vault PDA
             ↓
3. Users  →  buy_ticket()
             SOL transferred → Vault PDA
             Ticket PDA minted with deterministic index
             ↓  (when room is full OR timer expired)
4. Admin  →  commit_draw()
             Registers Switchboard VRF account
             Sets status = 2 (drawing)
             Records commit_slot
             ↓  (after VRF resolves on-chain)
5. Admin  →  settle_draw()
             Reads VRF result from Switchboard
             Derives winning_index = random_u32 % tickets_sold
             Verifies winning Ticket PDA
             ↓
6. Funds distributed from Vault PDA:
   ├── 71.43%  →  Winner wallet
   ├── ~14.28% →  Buyback wallet (→ TICKETFI buy pressure)
   ├── ~13.99% →  Treasury
   └──  ~0.29% →  Jackpot accumulator
```

---

## 💸 Fee Distribution

Every raffle room distributes funds as follows:

```
Total Collected = tickets_sold × ticket_price

Winner Payout  = Total × 71.43%
Platform Fee   = Total × 28.57%
  ├── Buyback    ~14.28% of total  (50% of platform fee → TICKETFI open market buyback)
  ├── Treasury   ~13.99% of total  (49% of platform fee → operational costs)
  └── Jackpot     ~0.29% of total  ( 1% of platform fee → accumulated prize pool)
```

> All distributions happen atomically in a single on-chain transaction. No off-chain intermediaries.

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Smart Contracts | [Anchor Framework](https://www.anchor-lang.com/) (Rust) |
| Blockchain | [Solana](https://solana.com) |
| Randomness (VRF) | [Switchboard On-Demand](https://switchboard.xyz/) |
| Token Standard | SPL Token / Token-2022 |
| Frontend | Next.js + TypeScript *(separate repo)* |

---

## 🚀 Local Development

> 🔒 **Source code will be made public after the security audit is complete.**

The contracts are built with the [Anchor Framework](https://www.anchor-lang.com/) on Solana. Once the code is public, you'll be able to build and test locally using standard Anchor tooling:

```bash
# Prerequisites
# - Rust + Cargo
# - Solana CLI
# - Anchor AVM

anchor build
anchor test
```

---

## 🧪 Instructions Reference

### ticketfi-raffle

| Instruction | Access | Description |
|-------------|--------|-------------|
| `initialize_platform` | Admin | One-time platform setup with wallet addresses |
| `initialize_raffle_room` | Admin | Create a new raffle room with parameters |
| `buy_ticket` | Public | Purchase one ticket — SOL locked in vault |
| `commit_draw` | Admin | Trigger VRF request when room is full or expired |
| `settle_draw` | Admin | Resolve VRF result, pick winner, distribute funds |
| `handle_expired_room` | Admin | On-chain expiry logic: resets room (0/1 buyer) or blocks (2+ buyers) |
| `propose_admin` | Admin | Initiate two-step admin ownership transfer |
| `accept_admin` | Pending Admin | Complete two-step admin ownership transfer |
| `set_paused` | Admin | Pauses/unpauses ticket purchases (blocks buy_ticket only — draws/resets are unaffected) |
| `update_platform_wallets` | Admin | Update treasury, buyback, and jackpot wallets |

### ticketfi-staking

| Instruction | Access | Description |
|-------------|--------|-------------|
| `initialize_pool` | Admin | One-time staking pool setup |
| `initialize_vaults` | Admin | Initialize SPL token vault accounts |
| `stake` | Public | Lock TICKETFI tokens for selected tier duration |
| `unstake` | Public | Withdraw staked tokens after lock expires |
| `claim_rewards` | Public | Retrieve accrued Buyback & APY rewards |
| `deposit_rewards` | Public | Deposit platform fees / buyback rewards to distribute to stakers |
| `fund_apy_vault` | Admin | Deposit TICKETFI tokens into APY pool to fund staking yields |

---

## 🔐 Security Design

- **PDA-controlled vaults** — No admin can unilaterally withdraw user funds
- **Switchboard VRF** — Randomness is cryptographically verifiable on-chain; no party can predict or manipulate the winner
- **Deterministic ticket PDAs** — `["ticket", room_pubkey, ticket_index]` — auditable by anyone
- **Emergency pause** — Admin can halt ticket purchases in critical situations
- **Checked arithmetic** — All math uses `checked_*` operations to prevent overflow

### Error Codes

| Code | Description |
|------|-------------|
| `RaffleNotActive` | Room is not in active state |
| `RaffleNotFull` | Draw attempted before room is full or expired |
| `AlreadyFinished` | Action attempted on a completed room |
| `InsufficientFunds` | Vault balance too low for prize payout |
| `WinnerMismatch` | Winning ticket account does not match VRF result |
| `PlatformPaused` | Platform temporarily paused by admin |
| `RaffleExpired` | Ticket purchase attempted after room expiry |
| `InvalidTreasury` | Wrong treasury wallet passed to instruction |
| `InvalidBuybackWallet` | Wrong buyback wallet passed to instruction |
| `InvalidJackpotWallet` | Wrong jackpot wallet passed to instruction |

---

## 📁 Repository Structure

* 🎟️ [**Raffle Program (`ticketfi-raffle`)**](./programs/ticketfi-raffle/README.md) — Detailed guide on raffle assets (SOL, BTC, RWA), refund scenarios (A, B, and C), and instruction API.
* 🔒 [**Staking Program (`ticketfi-staking`)**](./programs/ticketfi_staking/README.md) — Detailed guide on lock tiers (7/30/90/180 days), dynamic buyback weights, and yield configurations.

```
contracts/
├── programs/
│   ├── ticketfi-raffle/          # Raffle & rooms program (see README inside)
│   │   └── src/
│   │       ├── lib.rs            # All instructions
│   │       ├── state.rs          # Account structures
│   │       ├── error.rs          # Custom error codes
│   │       └── constants.rs      # Program constants
│   └── ticketfi_staking/         # Staking program (see README inside)
│       └── src/
│           ├── lib.rs            # All instructions & state
│           ├── error.rs          # Custom error codes
│           └── constants.rs      # Staking constants
├── tests/                        # Integration tests (TypeScript)
├── scripts/                      # Deployment & admin scripts
├── Anchor.toml                   # Anchor workspace config
└── Cargo.toml                    # Rust workspace config
```

---

## 📊 Roadmap

- [x] Raffle room architecture & ticket purchase logic
- [x] Switchboard VRF integration (commit/settle two-phase draw)
- [x] Automatic fee distribution with buyback wallet
- [x] Tiered staking program (7/30/90/180 days)
- [ ] Staking reward distribution finalization
- [ ] Jackpot accumulator logic
- [ ] Security audit
- [ ] Devnet public deployment
- [ ] Mainnet launch

---

## 🤝 Contributing

The contracts are in active development. Public contributions will be welcomed after the initial security audit. In the meantime, feel free to open Issues or Discussions for questions.

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

<div align="center">

Built with ❤️ on Solana &nbsp;·&nbsp; [ticketfi.com](https://ticketfi.com)

</div>
