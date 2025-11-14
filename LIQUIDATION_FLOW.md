# Phoenix Liquidation Function Call Flow

## Overview

Phoenix is an on-chain orderbook DEX that operates without a crank. While Phoenix itself doesn't have built-in liquidation logic, **liquidation protocols** (like lending platforms) use Phoenix's order execution mechanisms to liquidate undercollateralized positions.

## Key Insight

Liquidations in Phoenix are executed through **IOC (Immediate or Cancel) swap orders** that immediately match against the orderbook. The liquidation flow uses the same matching engine as regular trades.

---

## Function Call Flow for Liquidations

### 1. Entry Point: `process_instruction()` 
**File:** `src/lib.rs:93`

```
process_instruction(program_id, accounts, instruction_data)
  ↓
  Decodes instruction tag → PhoenixInstruction::Swap (0) or SwapWithFreeFunds (1)
  ↓
  Creates EventRecorder for logging
  ↓
  Routes to appropriate handler
```

### 2. Swap Handler: `process_swap()` or `process_swap_with_free_funds()`
**File:** `src/program/processor/new_order.rs:140` or `182`

**For Regular Swap (Liquidation Protocol):**
```rust
process_swap(program_id, market_context, accounts, data, record_event_fn)
  ↓
  Loads NewOrderContext (validates accounts, loads vaults)
  ↓
  Decodes OrderPacket from instruction data
  ↓
  Validates: Order must be IOC or FOK (take-only)
  ↓
  Calls process_new_order()
```

**For Swap with Free Funds (Seat Holder):**
```rust
process_swap_with_free_funds(...)
  ↓
  Similar flow but uses only deposited funds (no token transfers)
  ↓
  Requires seat on market
```

### 3. Core Order Processing: `process_new_order()`
**File:** `src/program/processor/new_order.rs:360`

```rust
process_new_order(new_order_context, market_context, order_packet, record_event_fn, order_ids)
  ↓
  Gets market header (quote_lot_size, base_lot_size)
  ↓
  Checks if order should fail silently on insufficient funds
  ↓
  Loads market state (FIFOMarket) via load_with_dispatch_mut()
  ↓
  Calls market_wrapper.inner.place_order() → MATCHING ENGINE
  ↓
  Gets MatchingEngineResponse with:
    - num_quote_lots_in/out
    - num_base_lots_in/out
    - num_quote_lots_posted
    - num_base_lots_posted
  ↓
  Calculates token amounts to withdraw/deposit
  ↓
  Performs token transfers via maybe_invoke_withdraw() and maybe_invoke_deposit()
```

### 4. Market Dispatch: `load_with_dispatch_mut()`
**File:** `src/program/dispatch_market.rs:33`

```rust
load_with_dispatch_mut(market_size_params, bytes)
  ↓
  Dispatches to correct FIFOMarket variant based on size params
  (e.g., FIFOMarket<Pubkey, 512, 512, 128>)
  ↓
  Returns MarketWrapperMut
```

### 5. Matching Engine: `place_order()` → `place_order_inner()`
**File:** `src/state/markets/fifo.rs:437` → `759`

```rust
place_order_inner(trader_id, order_packet, record_event_fn, get_clock_fn)
  ↓
  Validates market is initialized
  ↓
  Validates order sequence number hasn't overflowed
  ↓
  Validates price (bids > 0, asks >= 1 tick)
  ↓
  Gets or registers trader index
  ↓
  Validates order size (base_lots or quote_lots must be > 0)
  ↓
  Checks if order is expired (last_valid_slot/timestamp)
  ↓
  For IOC orders (LIQUIDATIONS):
    Creates InflightOrder with:
      - side (Bid/Ask)
      - price_in_ticks (or None for market orders)
      - base_lot_budget
      - quote_lot_budget (adjusted for fees)
      - self_trade_behavior
      - match_limit
  ↓
  Calls match_order() → CORE MATCHING LOGIC
  ↓
  Calculates matched amounts and fees
  ↓
  Creates MatchingEngineResponse
  ↓
  Records FillSummary event
  ↓
  Returns (order_id, MatchingEngineResponse)
```

### 6. Core Matching: `match_order()`
**File:** `src/state/markets/fifo.rs` (internal method)

```rust
match_order(inflight_order, trader_index, record_event_fn, current_slot, current_timestamp)
  ↓
  Gets opposite side book (bids if selling, asks if buying)
  ↓
  Iterates through orderbook levels (FIFO):
    - Gets best price level
    - Checks if order can match at this price
    - Checks if price limit is satisfied
    - Checks if order is expired
    - Checks self-trade behavior
  ↓
  For each matching order:
    - Calculates fill size (min of inflight_order remaining and resting order size)
    - Calculates quote lots (price * base_lots)
    - Applies taker fee
    - Updates trader states:
      * Maker: unlocks funds, adds to free balance
      * Taker: locks funds, deducts from free balance
    - Records Fill event
    - Removes or reduces resting order
  ↓
  Continues until:
    - Order fully filled
    - Budget exhausted (base_lots or quote_lots)
    - No more matching orders
    - Price limit exceeded
    - Match limit reached
  ↓
  Returns resting_order (if any remaining size) and updates inflight_order
```

### 7. Token Transfers: `maybe_invoke_withdraw()` and `maybe_invoke_deposit()`
**File:** `src/program/token_utils.rs:57` and `87`

**Withdraw (for filled orders):**
```rust
maybe_invoke_withdraw(market_key, mint_key, bump, amount, token_program, account, vault)
  ↓
  If amount > 0:
    invoke_signed() with SPL Token transfer instruction
    Transfers from vault → trader account
    Uses PDA signature: [b"vault", market_key, mint_key, bump]
```

**Deposit (for new orders):**
```rust
maybe_invoke_deposit(amount, token_program, account, vault, trader)
  ↓
  If amount > 0:
    invoke() with SPL Token transfer instruction
    Transfers from trader account → vault
```

### 8. Event Recording: `EventRecorder`
**File:** `src/program/event_recorder.rs`

```rust
EventRecorder records:
  - Fill events (price, size, trader)
  - FillSummary events (total fills, fees)
  - Market sequence number increments
  ↓
  Events are logged via CPI to log authority
  ↓
  Can be queried from transaction logs
```

---

## Liquidation-Specific Flow

### Typical Liquidation Scenario:

1. **Lending Protocol Detects Undercollateralization**
   - Monitors positions off-chain or via oracle
   - Determines liquidation amount and target market

2. **Lending Protocol Calls Phoenix Swap**
   ```
   Instruction: PhoenixInstruction::Swap (0)
   OrderPacket: ImmediateOrCancel {
     side: Side::Ask (selling collateral) or Side::Bid (buying debt token)
     price_in_ticks: None (market order) or Some(max_price)
     num_base_lots: liquidation_amount_in_base_lots
     num_quote_lots: 0
     min_base_lots_to_fill: minimum_acceptable_fill
     min_quote_lots_to_fill: 0
     self_trade_behavior: CancelProvide
     match_limit: None
     use_only_deposited_funds: false
   }
   ```

3. **Phoenix Executes Swap**
   - Matches against best available orders
   - Fills immediately (IOC) or fails if insufficient liquidity
   - Transfers tokens atomically
   - Charges taker fee

4. **Liquidation Complete**
   - Collateral sold / debt token bought
   - Proceeds sent to liquidator
   - Position closed or reduced

---

## Key Data Structures

### OrderPacket (for Liquidations)
```rust
OrderPacket::ImmediateOrCancel {
  side: Side,                    // Bid (buy) or Ask (sell)
  price_in_ticks: Option<Ticks>, // None = market order, Some = limit
  num_base_lots: BaseLots,       // Size in base lots
  num_quote_lots: QuoteLots,     // Size in quote lots (mutually exclusive with base)
  min_base_lots_to_fill: BaseLots, // Minimum fill requirement
  min_quote_lots_to_fill: QuoteLots,
  self_trade_behavior: SelfTradeBehavior,
  match_limit: Option<u64>,
  use_only_deposited_funds: bool,
  last_valid_slot: Option<u64>,
  last_valid_unix_timestamp_in_seconds: Option<u64>,
}
```

### MatchingEngineResponse
```rust
MatchingEngineResponse {
  num_quote_lots_in: QuoteLots,    // Quote paid
  num_base_lots_in: BaseLots,      // Base paid
  num_quote_lots_out: QuoteLots,   // Quote received
  num_base_lots_out: BaseLots,     // Base received
  num_quote_lots_posted: QuoteLots, // Quote locked in order
  num_base_lots_posted: BaseLots,   // Base locked in order
  num_free_quote_lots_used: QuoteLots,
  num_free_base_lots_used: BaseLots,
}
```

---

## Account Structure for Liquidations

### Required Accounts (Swap Instruction):
1. `phoenix_program` - Phoenix program ID
2. `log_authority` - Phoenix log authority PDA
3. `market` (writable) - Market state account
4. `trader` (signer) - Liquidator's account
5. `base_account` (writable) - Trader's base token account
6. `quote_account` (writable) - Trader's quote token account
7. `base_vault` (writable) - Market's base vault PDA
8. `quote_vault` (writable) - Market's quote vault PDA
9. `token_program` - SPL Token program

---

## Important Notes for Liquidations

1. **Atomic Execution**: All matching and token transfers happen atomically in a single transaction
2. **No Crank Required**: Phoenix matches orders on-chain without external keepers
3. **Fee Structure**: Taker fees are charged on quote lots transacted (in basis points)
4. **Slippage Protection**: Use `min_base_lots_to_fill` or `min_quote_lots_to_fill` to ensure minimum fill
5. **Price Limits**: Set `price_in_ticks` to limit maximum price paid (important for liquidations)
6. **Self-Trade Protection**: `SelfTradeBehavior::CancelProvide` cancels maker orders if same trader
7. **Expiration**: Orders can expire by slot or unix timestamp
8. **Market Status**: Market must be in active status for cross trades

---

## Example Liquidation Transaction Flow

```
1. Lending Protocol → Phoenix::Swap
   ├─ Decode OrderPacket (IOC sell order)
   ├─ Load market state
   ├─ Match against orderbook
   │  ├─ Find best bids
   │  ├─ Fill at best prices (FIFO)
   │  ├─ Apply fees
   │  └─ Update trader states
   ├─ Withdraw base tokens (collateral) from vault → liquidator
   ├─ Deposit quote tokens (payment) to vault
   ├─ Record Fill events
   └─ Return MatchingEngineResponse

2. Lending Protocol receives response
   ├─ Verifies fill amounts
   ├─ Updates position state
   └─ Distributes liquidation bonus
```

---

## Files Involved in Liquidation Flow

1. **Entry Point**: `src/lib.rs` - `process_instruction()`
2. **Order Processing**: `src/program/processor/new_order.rs` - `process_swap()`, `process_new_order()`
3. **Market Dispatch**: `src/program/dispatch_market.rs` - `load_with_dispatch_mut()`
4. **Matching Engine**: `src/state/markets/fifo.rs` - `place_order_inner()`, `match_order()`
5. **Token Utils**: `src/program/token_utils.rs` - `maybe_invoke_withdraw()`, `maybe_invoke_deposit()`
6. **Order Schema**: `src/state/order_schema/order_packet.rs` - `OrderPacket` definition
7. **Matching Response**: `src/state/matching_engine_response.rs` - `MatchingEngineResponse`
8. **Events**: `src/program/events.rs` - Event definitions
9. **Event Recorder**: `src/program/event_recorder.rs` - Event logging

---

## Summary

Liquidations in Phoenix work by:
1. **External protocol** (lending platform) calls Phoenix's `Swap` instruction
2. Phoenix **matches the order** against the orderbook using FIFO matching
3. **Token transfers** happen atomically via SPL Token program
4. **Events are recorded** for off-chain indexing
5. **Response is returned** with fill details

The entire process is **atomic** - either the liquidation fully executes or it fails, ensuring no partial fills that could leave positions in bad states.

