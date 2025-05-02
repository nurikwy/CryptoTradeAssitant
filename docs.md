# System Design & Research Documentation

_All system design, interaction patterns, data models, research findings and mockups for Crypto Trade Assistant are consolidated in this single document._

---

## 1. Architecture Overview

- **Components**  
  - **Telegram Bot (Aiogram)**  
    - Receives and handles inline button callbacks  
    - Sends rich messages with inline keyboards  
  - **Trading Engine**  
    - Async wrapper over Solana JSON-RPC (with Helius fallback)  
    - Executes market and limit orders  
  - **RugCheck Service**  
    - Fetches on-chain token metadata  
    - Displays raw on-chain metrics for user review  
  - **Copy-Trading Manager**  
    - Subscribes to master-account events  
    - Replicates trades to followers in real time  
  - **Wallet Monitor**  
    - Subscribes via WebSocket to balance changes  
    - Pushes real-time alerts via bot messages  

- **Async Event Flow**  
  - All I/O handled via `asyncio`  
  - Rate-limited RPC calls managed by worker pools  
  - Internal pub/sub via Redis or PostgreSQL LISTEN/NOTIFY  

---

## 2. Telegram Interaction Patterns

_100% inline buttons; slash commands only as hidden fallback._

- **Onboarding**  
  - **Start** button → wallet-link prompt → display referral summary card  

- **Main Menu**  
  - Inline buttons: **Trade** | **Copy-Trading** | **RugCheck** | **Wallet Monitor** | **Analytics** | **Referral**  

- **Trading Flow**  
  1. Tap **Trade** → submenu: **Buy**, **Sell**, **Limit**  
  2. Inline prompts:  
     - Token selector (search + pagination)  
     - Amount input (slider or numeric)  
     - (For limit) Price input  
     - **Confirm** → execute → success/failure alert  

- **RugCheck Flow**  
  1. Tap **RugCheck** → token selector  
  2. Show on-chain metrics card (mint date, liquidity, holder count)  
  3. **Refresh** to re-fetch data  

- **Copy-Trading Flow**  
  1. Tap **Copy-Trading** → paginated list of masters  
  2. Select master → risk slider appears  
  3. **Follow** / **Unfollow**  

- **Wallet Monitor & Analytics**  
  - **Wallet Monitor** → balance card + **Refresh**  
  - **Analytics** → P&L & volume summary, date navigation  

- **Referral & Rewards**  
  - **Referral** → show code + **Share**  

- **Error Handling**  
  - Invalid input → inline “Try Again” alert  
  - RPC failures → retry → “Service Unavailable” card  

---

## 3. Data Models

The database schema uses SQLAlchemy models as follows:

### User (`users`)

| Column             | Type         | Notes                                                            |
| ------------------ | ------------ | ---------------------------------------------------------------- |
| `id`               | Integer      | Primary key                                                     |
| `telegram_id`      | BigInteger   | Unique, indexed                                                |
| `solana_wallet`    | String(44)   | Unique, indexed (Base58 address)                               |
| `_private_key`     | Text         | Encrypted private key                                          |
| `referral_code`    | String(8)    | Unique, indexed                                                |
| `total_volume`     | Float        | Default 0.0                                                    |
| `last_buy_amount`  | Float        | Nullable                                                       |
| `referral_id`      | Integer      | FK → `users.id`, nullable                                      |
| `created_at`       | DateTime     | Default `utcnow()`                                             |
| `last_activity`    | DateTime     | Default `utcnow()`                                             |

**Relationships:**  
- `referred_users` → other `User` rows (cascade delete)  
- `referrer` ← one `User`  
- `user_settings` → `UserSettings`

---

### Trade (`trades`)

| Column             | Type         | Notes                                     |
| ------------------ | ------------ | ----------------------------------------- |
| `id`               | Integer      | Primary key                              |
| `user_id`          | Integer      | FK → `users.id`, indexed                 |
| `token_address`    | String(44)   | Indexed                                  |
| `amount`           | Float        |                                          |
| `price_usd`        | Float        |                                          |
| `amount_sol`       | Float        | Nullable                                 |
| `created_at`       | DateTime     | Default `utcnow()`, indexed              |
| `transaction_type` | SmallInteger | 0 = BUY, 1 = SELL                        |
| `status`           | String(20)   | Default `'pending'`, indexed             |
| `gas_fee`          | Float        | Nullable                                 |
| `transaction_hash` | String(88)   | Unique, nullable                         |
| `extra_data`       | JSONB        | Default `{}`                             |

**Relationship:**  
- `user` ← one `User`

---

### ReferralRecord (`referral_records`)

| Column       | Type     | Notes                                         |
| ------------ | -------- | --------------------------------------------- |
| `id`         | Integer  | Primary key                                  |
| `user_id`    | Integer  | FK → `users.id`, indexed                     |
| `trade_id`   | Integer  | FK → `trades.id`, indexed, nullable          |
| `amount_sol` | Float    |                                              |
| `is_sent`    | Boolean  | Default `False`                              |
| `created_at` | DateTime | Default `utcnow()`, indexed                  |

---

### CopyTrade (`copy_trades`)

| Column                | Type      | Notes                                                  |
| --------------------- | --------- | ------------------------------------------------------ |
| `id`                  | Integer   | Primary key                                           |
| `user_id`             | Integer   | FK → `users.id`                                       |
| `name`                | String    | Nullable                                              |
| `wallet_address`      | String(44)|                                                  |
| `is_active`           | Boolean   | Default `True`                                        |
| **Copy Settings**     |           |                                                        |
| `copy_percentage`     | Float     | Default 100.0                                         |
| `min_amount`          | Float     | Default 0.0                                           |
| `max_amount`          | Float     | Nullable                                              |
| `total_amount`        | Float     | Nullable                                              |
| `max_copies_per_token`| Integer   | Nullable                                              |
| `copy_sells`          | Boolean   | Default `True`                                        |
| `retry_count`         | Integer   | Default 1                                             |
| **Transaction Settings** |        |                                                        |
| `buy_gas_fee`         | Integer   | Default 100000                                        |
| `sell_gas_fee`        | Integer   | Default 100000                                        |
| `buy_slippage`        | Float     | Default 5.0                                           |
| `sell_slippage`       | Float     | Default 1.0                                           |
| `anti_mev`            | Boolean   | Default `False`                                       |
| `created_at`          | DateTime  | Default `utcnow()`                                    |
| `updated_at`          | DateTime  | onupdate `utcnow()`, nullable                         |

**Relationships:**  
- `user` ← one `User`  
- `transactions` → `CopyTradeTransaction`

---

### ExcludedToken (`excluded_tokens`)

| Column          | Type       | Notes                             |
| --------------- | ---------- | --------------------------------- |
| `id`            | Integer    | Primary key                      |
| `user_id`       | Integer    | FK → `users.id`                  |
| `token_address` | String(44) |                                  |
| `created_at`    | DateTime   | Default `utcnow()`               |

**Relationship:**  
- `user` ← one `User`

---

### CopyTradeTransaction (`copy_trade_transactions`)

| Column               | Type       | Notes                                   |
| -------------------- | ---------- | --------------------------------------- |
| `id`                 | Integer    | Primary key                            |
| `copy_trade_id`      | Integer    | FK → `copy_trades.id`                  |
| `token_address`      | String(44) |                                         |
| `original_signature` | String     |                                         |
| `copied_signature`   | String     | Nullable                                |
| `transaction_type`   | String     | `'BUY'` / `'SELL'`                      |
| `status`             | String     | e.g. `'SUCCESS'` / `'FAILED'`           |
| `error_message`      | String     | Nullable                                |
| `amount_sol`         | Float      |                                         |
| `created_at`         | DateTime   | Default `utcnow()`                      |

---

### Setting (`settings`)

| Column         | Type    | Notes                       |
| -------------- | ------- | --------------------------- |
| `id`           | Integer | Primary key                |
| `name`         | String  |                             |
| `slug`         | String  | Unique                     |
| `default_value`| JSONB   |                             |

**Relationship:**  
- `user_settings` → `UserSettings`

---

### UserSettings (`user_settings`)

| Column      | Type     | Notes                                 |
| ----------- | -------- | ------------------------------------- |
| `id`        | Integer  | Primary key                          |
| `user_id`   | Integer  | FK → `users.id`, indexed             |
| `setting_id`| Integer  | FK → `settings.id`, indexed          |
| `value`     | JSONB    |                                       |

**Relationships:**  
- `user` ← one `User`  
- `setting` ← one `Setting`

---

### LimitOrder (`limit_orders`)

| Column               | Type       | Notes                                          |
| -------------------- | ---------- | ---------------------------------------------- |
| `id`                 | Integer    | Primary key                                   |
| `user_id`            | Integer    | FK → `users.id`, indexed                      |
| `token_address`      | String(44) |                                                |
| `order_type`         | String(4)  | `'buy'` / `'sell'`                             |
| `amount_sol`         | Float      | Nullable (for token sells)                     |
| `amount_tokens`      | Float      | Nullable (for SOL buys)                        |
| `trigger_price_usd`  | Float      |                                                |
| `trigger_price_percent`| Float    |                                                |
| `slippage`           | Float      | Default 1.0                                   |
| `status`             | String(20) | Default `'active'`, indexed                   |
| `transaction_hash`   | String(88) | Nullable                                      |
| `created_at`         | DateTime   | Default `utcnow()`                             |
| `executed_at`        | DateTime   | Nullable                                       |

**Relationship:**  
- `user` ← one `User`

---

## 4. Solana Integration Research

- **RPC Providers**  
  - **Helius RPC:** ~80 ms avg latency, batched calls up to 100/s  
  - **Public JSON-RPC:** ~120 ms avg, fallback  

- **DEX Patterns**  
  - WebSocket orderbook for real-time depth  
  - Limit orders via Serum RPC; confirm via on-chain events  

- **Recommendations**  
  - Primary: Helius; fallback: JSON-RPC  
  - Client-side rate limiter + circuit breaker  

---

## 5. Security Assessment & RugCheck Service

- **On-chain Signals Only**  
  - Token mint date & supply history  
  - Liquidity pool age & depth  
  - Holder concentration metrics  

_Bot displays these metrics in a formatted card. Users interpret on-chain data manually; no off-chain checks or automated heuristics._

---

## 6. Copy-Trading Algorithm

- **Subscription Flow**  
  1. User taps **Copy-Trading** → selects master  
  2. Manager subscribes to master’s on-chain events  
  3. New master order → get subscriber settings → place matched orders  

## 7. Polkadot Integration Research

- **Consensus & RPC**  
  - Polkadot nodes expose a JSON-RPC interface (e.g. `chain_getBlock`, `author_submitExtrinsic`) over HTTP and WebSocket  
  - We will use the Python Substrate Interface library to simplify RPC calls and event subscriptions

- **Key Management & Signing**  
  - Integrate Parity Signer (Polkadot Vault) for offline key storage and QR-based transaction signing  
  - Implement a QR code workflow: encode unsigned extrinsic → sign in Vault → decode signed payload → submit via RPC

- **Cross-Consensus Messaging (XCM)**  
  - XCM v2 serves as the standard format for sending instructions and assets between Relay Chain and parachains  
  - Use HRMP channels for development/testing and mature to XCMP for production messaging

- **Asset Bridges & Token Standards**  
  - Leverage ORML’s `xtokens` pallet to perform XC-20 style cross-chain transfers via XCM  
  - Support lock-mint and burn-release workflows to move assets between Solana and Polkadot

- **DEX Patterns on Polkadot**  
  - Integrate AMM functionality from parachains like HydraDX and Polkadex via Substrate swap pallets  
  - Support orderbook models (e.g. Polkaswap) by subscribing to on-chain events for limit/market order confirmations

### Plan of Action

1. **Extend Wallet Flow**  
   - Add Substrate keypair generation and encryption to the existing Telegram wallet-link process  
2. **Build RPC Client**  
   - Implement an async WebSocket client using the Python Substrate Interface to handle Polkadot JSON-RPC  
3. **Develop XCM Module**  
   - Create an async wrapper for `teleport_assets` and `reserve_transfer_assets` extrinsics via ORML `xtokens`  
4. **Implement DEX Adapter**  
   - Integrate Substrate swap and orderbook pallets on Plaza for AMM and limit/market orders  
5. **End-to-End Testing**  
   - On Rococo testnet: execute cross-chain trade from Solana → XCM bridge → Plaza and confirm via Telegram notifications  

