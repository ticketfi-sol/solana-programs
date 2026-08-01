<div align="center">

# 🔒 TicketFi — Staking Program

**High-precision token locking, dynamic rewards & yield vaults**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](../../LICENSE)
[![Built with Anchor](https://img.shields.io/badge/Built%20with-Anchor-9945FF)](https://www.anchor-lang.com/)
[![Network: Solana](https://img.shields.io/badge/Network-Solana-14F195)](https://solana.com)
[![Module: Staking](https://img.shields.io/badge/Module-Staking-ff69b4)]()

[🌐 Website](https://ticketfi.com) · [📄 Whitepaper](../../WHITEPAPER.md) · [🐦 Twitter/X](https://x.com/ticketfi)

</div>

---

An on-chain, high-yield staking protocol built with the **Anchor Framework** on Solana. The staking program allows users to lock their **TICKETFI** tokens to earn protocol-backed yields and a direct share of the platform's raffle buybacks.

## 📈 Staking Tiers & APY Structure

Users can choose from four lock tiers. Longer locking periods yield higher APY rates and a significantly larger share of the protocol buyback rewards.

| Tier | Lock Duration | Fixed APY Rate | Buyback Reward Weight | Reward Pool Index |
| :--- | :------------ | :------------- | :-------------------- | :---------------- |
| **0** | 7 Days | **10%** | **10** | `acc_reward_per_token[0]` |
| **1** | 30 Days | **15%** | **15** | `acc_reward_per_token[1]` |
| **2** | 90 Days | **25%** | **25** | `acc_reward_per_token[2]` |
| **3** | 180 Days | **50%** | **50** | `acc_reward_per_token[3]` |

---

## 🔄 Dual-Reward Mechanism

The protocol distributes rewards from two separate sources, ensuring sustainable tokenomics and real yield generation:

### 1. Fixed APY Rewards
* Funded directly by the protocol admin into the dedicated `apy_vault` via the `fund_apy_vault` instruction.
* Accrues continuously and linearly based on the staked amount, the elapsed lock time, and the tier's APY rate.
* APY formula:
  $$\text{Pending APY} = \frac{\text{Staked Amount} \times \text{APY Rate} \times \text{Time Elapsed}}{\text{100} \times \text{Seconds Per Year}}$$

### 2. Protocol Buyback Rewards (Platform Fee Sharing)
* **Buyback Source**: 50% of the raffle program platform fees (~14.28% of all ticket sales) are used to buy back TICKETFI tokens from the open market.
* **Distribution**: These bought-back tokens are deposited into the staking pool via `deposit_rewards`.
* **Dynamic Weight Allocation**: The deposited tokens are split dynamically among the active tiers based on their weights (**10 / 15 / 25 / 50**). Only tiers with active stakers receive a share, preventing rewards from being distributed to empty pools.
* **Per-Token Accumulator**: Rewards are tracked with a high-precision multiplier (`PRECISION = 1e12`) to eliminate rounding errors:
  $$\text{Accumulator Increment} = \frac{\text{Tier Reward Share} \times 10^{12}}{\text{Total Staked in Tier}}$$

---

## 🛠️ Instruction Reference

| Instruction | Access | Description |
| :--- | :--- | :--- |
| `initialize_pool` | Admin | One-time initialization of the staking pool configuration, linking the TICKETFI mint and treasury vaults. |
| `initialize_vaults` | Admin | Initializes the PDA token accounts: `staking_vault` (stores staked/buyback tokens) and `apy_vault` (stores APY reward tokens). |
| `stake` | Public | Locks a specified amount of TICKETFI tokens for a selected tier duration (7, 30, 90, or 180 days). Initializes the user's position state PDA. |
| `unstake` | Public | Unlocks and returns the user's staked tokens *after* the lock duration has expired. Any unclaimed rewards are safely cached in the state account. |
| `claim_rewards` | Public | Calculates and transfers all pending APY and Buyback rewards to the user's token wallet. If the user has already unstaked, this closes the account and refunds lamports (rent exemption). |
| `deposit_rewards` | Public/Bot | Deposits bought-back TICKETFI tokens from the platform buyback wallet and updates the dynamic weighted reward accumulators. |
| `fund_apy_vault` | Admin | Deposits TICKETFI tokens into the APY reward vault to fund staking yields. |

---

## 🧮 High-Precision & Security Architecture

* **Linear APY Accrual**: The system calculates rewards on every interaction (e.g., claiming, unstaking) based on the Solana `Clock` timestamp.
* **User Stake Counter PDA**: Safe derivation of individual staking positions using a counter PDA: `["user_stake", user_pubkey, stake_index_le_bytes]`. Allows multiple staking positions per wallet.
* **Auto-Cleanup**: When a user unstakes and subsequently claims all remaining rewards, the program closes the state account, returning the rent-exemption lamports to the user's wallet.
* **Checked Math**: All operations use checked math (`checked_add`, `checked_mul`, etc.) to prevent overflow/underflow attacks.
