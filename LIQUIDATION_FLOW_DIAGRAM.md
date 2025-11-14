# Phoenix Liquidation Flow - Visual Diagram

## High-Level Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL LIQUIDATION PROTOCOL                 │
│              (e.g., Lending Platform, Margin Protocol)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ 1. Detects undercollateralization
                             │ 2. Prepares liquidation order
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Phoenix::Swap Instruction (IOC Order)               │
│  Instruction Tag: 0 (Swap) or 1 (SwapWithFreeFunds)             │
│  OrderPacket: ImmediateOrCancel {                                │
│    side: Ask (sell collateral) or Bid (buy debt token)          │
│    price_in_ticks: None (market) or Some(max_price)             │
│    num_base_lots: liquidation_amount                            │
│    min_base_lots_to_fill: minimum_acceptable                    │
│  }                                                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    process_instruction()                         │
│                    src/lib.rs:93                                 │
│  • Decode instruction tag                                        │
│  • Create EventRecorder                                          │
│  • Route to handler                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    process_swap()                                │
│          src/program/processor/new_order.rs:140                 │
│  • Load NewOrderContext (validate accounts)                     │
│  • Decode OrderPacket                                           │
│  • Validate: Must be IOC/FOK (take-only)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  process_new_order()                             │
│          src/program/processor/new_order.rs:360                 │
│  • Get market header (lot sizes)                                │
│  • Load market state                                            │
│  • Call matching engine                                         │
│  • Calculate token amounts                                      │
│  • Perform token transfers                                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              load_with_dispatch_mut()                            │
│              src/program/dispatch_market.rs:33                  │
│  • Dispatch to correct FIFOMarket variant                       │
│  • Based on market size params (bids, asks, seats)              │
│  • Returns MarketWrapperMut                                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              place_order_inner()                                 │
│              src/state/markets/fifo.rs:759                      │
│  • Validate market initialized                                  │
│  • Validate order parameters                                    │
│  • Check expiration                                             │
│  • Create InflightOrder                                         │
│  • Call match_order()                                           │
│  • Calculate fees                                               │
│  • Create MatchingEngineResponse                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    match_order()                                 │
│              src/state/markets/fifo.rs                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Get opposite side book (bids if selling, asks if      │  │
│  │    buying)                                                │  │
│  │                                                           │  │
│  │ 2. Iterate through orderbook (FIFO):                     │  │
│  │    ├─ Get best price level                               │  │
│  │    ├─ Check if order can match                           │  │
│  │    ├─ Check price limit                                  │  │
│  │    ├─ Check expiration                                   │  │
│  │    ├─ Check self-trade behavior                          │  │
│  │    │                                                      │  │
│  │    ├─ For each matching order:                           │  │
│  │    │  ├─ Calculate fill size                             │  │
│  │    │  ├─ Calculate quote lots (price × base_lots)        │  │
│  │    │  ├─ Apply taker fee                                 │  │
│  │    │  ├─ Update maker trader state                       │  │
│  │    │  │  • Unlock funds                                  │  │
│  │    │  │  • Add to free balance                           │  │
│  │    │  ├─ Update taker trader state                       │  │
│  │    │  │  • Lock funds                                    │  │
│  │    │  │  • Deduct from free balance                      │  │
│  │    │  ├─ Record Fill event                               │  │
│  │    │  └─ Remove/reduce resting order                     │  │
│  │    │                                                      │  │
│  │    └─ Continue until:                                    │  │
│  │       • Order fully filled                               │  │
│  │       • Budget exhausted                                 │  │
│  │       • No more matching orders                          │  │
│  │       • Price limit exceeded                             │  │
│  │       • Match limit reached                              │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Token Transfers (Atomic)                            │
│              src/program/token_utils.rs                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ maybe_invoke_withdraw()                                   │  │
│  │  • Transfers from vault → trader account                 │  │
│  │  • Uses PDA signature: [b"vault", market, mint, bump]    │  │
│  │  • SPL Token::transfer instruction                       │  │
│  │                                                           │  │
│  │ maybe_invoke_deposit()                                    │  │
│  │  • Transfers from trader account → vault                 │  │
│  │  • SPL Token::transfer instruction                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Event Recording                                     │
│              src/program/event_recorder.rs                      │
│  • Record Fill events (price, size, trader)                    │
│  • Record FillSummary (total fills, fees)                      │
│  • Increment market sequence number                            │
│  • Log via CPI to log authority                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Return MatchingEngineResponse                       │
│  {                                                               │
│    num_quote_lots_in: QuoteLots,    // Quote paid               │
│    num_base_lots_in: BaseLots,      // Base paid                │
│    num_quote_lots_out: QuoteLots,   // Quote received           │
│    num_base_lots_out: BaseLots,     // Base received            │
│    ...                                                           │
│  }                                                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              EXTERNAL LIQUIDATION PROTOCOL                       │
│  • Receives response                                            │
│  • Verifies fill amounts                                        │
│  • Updates position state                                       │
│  • Distributes liquidation bonus                                │
└─────────────────────────────────────────────────────────────────┘
```

## Detailed Matching Engine Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    match_order() - Detailed Flow                 │
└─────────────────────────────────────────────────────────────────┘

Input: InflightOrder (liquidation order)
       ├─ side: Ask (selling collateral)
       ├─ price_in_ticks: None (market order) or Some(max_price)
       ├─ base_lot_budget: amount_to_liquidate
       ├─ quote_lot_budget: None or max_quote_to_spend
       └─ self_trade_behavior: CancelProvide

       ↓

┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Get Opposite Book                                      │
│  • If side == Ask → Get bids book                               │
│  • If side == Bid → Get asks book                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Iterate Through Orderbook (FIFO)                       │
│                                                                  │
│  For each price level (best price first):                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • Get best price level from book                        │  │
│  │  • Check if price satisfies limit (if set)               │  │
│  │  • Check if order is expired                             │  │
│  │  • Check if order has size > 0                           │  │
│  │                                                           │  │
│  │  For each resting order at this price:                   │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  • Check self-trade behavior                        │  │  │
│  │  │    - If same trader: CancelProvide → cancel maker  │  │  │
│  │  │    - If same trader: DecrementTake → reduce maker  │  │  │
│  │  │    - If same trader: Abort → skip                  │  │  │
│  │  │                                                      │  │  │
│  │  │  • Calculate fill size:                             │  │  │
│  │  │    fill_size = min(                                 │  │  │
│  │  │      inflight_order.remaining_base_lots,            │  │  │
│  │  │      resting_order.num_base_lots                    │  │  │
│  │  │    )                                                 │  │  │
│  │  │                                                      │  │  │
│  │  │  • Calculate quote lots:                            │  │  │
│  │  │    quote_lots = (price_in_ticks × fill_size ×       │  │  │
│  │  │                  tick_size) / base_lots_per_unit    │  │  │
│  │  │                                                      │  │  │
│  │  │  • Calculate taker fee:                             │  │  │
│  │  │    fee = quote_lots × taker_fee_bps / 10000         │  │  │
│  │  │                                                      │  │  │
│  │  │  • Update maker trader state:                       │  │  │
│  │  │    - Unlock quote_lots (from locked)                │  │  │
│  │  │    - Add quote_lots to free balance                 │  │  │
│  │  │    - Receive base_lots (add to free balance)        │  │  │
│  │  │                                                      │  │  │
│  │  │  • Update taker trader state:                       │  │  │
│  │  │    - Lock quote_lots (for payment)                  │  │  │
│  │  │    - Deduct quote_lots from free balance            │  │  │
│  │  │    - Pay base_lots (deduct from free balance)       │  │  │
│  │  │                                                      │  │  │
│  │  │  • Record Fill event:                               │  │  │
│  │  │    MarketEvent::Fill {                              │  │  │
│  │  │      maker_order_id,                                │  │  │
│  │  │      maker,                                         │  │  │
│  │  │      taker,                                         │  │  │
│  │  │      price_in_ticks,                                │  │  │
│  │  │      base_lots_filled,                              │  │  │
│  │  │      quote_lots_filled,                             │  │  │
│  │  │      fee_in_quote_lots,                             │  │  │
│  │  │    }                                                 │  │  │
│  │  │                                                      │  │  │
│  │  │  • Update resting order:                            │  │  │
│  │  │    - If fully filled: remove from book              │  │  │
│  │  │    - If partially filled: reduce size               │  │  │
│  │  │                                                      │  │  │
│  │  │  • Update inflight_order:                           │  │  │
│  │  │    - Deduct fill_size from remaining                │  │  │
│  │  │    - Add quote_lots to matched                      │  │  │
│  │  │    - Add fee to total fees                          │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  • Check if order fully filled or budget exhausted      │  │
│  │  • Check if match_limit reached                         │  │
│  │  • Continue to next price level if needed               │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Return Result                                          │
│  • Remaining resting order (if any size left)                   │
│  • Updated inflight_order with matched amounts                  │
└─────────────────────────────────────────────────────────────────┘
```

## Account Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Account Structure                             │
└─────────────────────────────────────────────────────────────────┘

Accounts Array (for Swap instruction):
┌──────────────────────────────────────────────────────────────┐
│ [0] phoenix_program      - Phoenix program ID                 │
│ [1] log_authority        - Phoenix log authority PDA         │
│ [2] market (writable)    - Market state account              │
│ [3] trader (signer)      - Liquidator's account              │
│ [4] base_account (w)     - Trader's base token account       │
│ [5] quote_account (w)    - Trader's quote token account      │
│ [6] base_vault (w)       - Market's base vault PDA           │
│ [7] quote_vault (w)      - Market's quote vault PDA          │
│ [8] token_program        - SPL Token program                 │
└──────────────────────────────────────────────────────────────┘

Token Flow (Example: Selling Collateral):
┌──────────────────────────────────────────────────────────────┐
│  Before:                                                      │
│  • base_vault: 1000 base tokens                              │
│  • quote_vault: 500 quote tokens                             │
│  • base_account: 100 base tokens (collateral)                │
│  • quote_account: 0 quote tokens                             │
│                                                               │
│  Liquidation Order: Sell 100 base tokens                     │
│                                                               │
│  Matching:                                                    │
│  • Matches against best bids in orderbook                    │
│  • Fills 100 base tokens at average price                    │
│  • Receives quote tokens (minus fees)                        │
│                                                               │
│  After:                                                       │
│  • base_vault: 1100 base tokens (+100 from trader)           │
│  • quote_vault: 400 quote tokens (-100 to trader, +fees)     │
│  • base_account: 0 base tokens (sold)                        │
│  • quote_account: ~100 quote tokens (received)               │
└──────────────────────────────────────────────────────────────┘
```

## State Updates

```
┌─────────────────────────────────────────────────────────────────┐
│                    Market State Updates                          │
└─────────────────────────────────────────────────────────────────┘

FIFOMarket State:
┌──────────────────────────────────────────────────────────────┐
│  • bids: RedBlackTree<FIFOOrderId, FIFORestingOrder>         │
│    - Updated: Orders filled/removed, new orders added        │
│                                                               │
│  • asks: RedBlackTree<FIFOOrderId, FIFORestingOrder>         │
│    - Updated: Orders filled/removed, new orders added        │
│                                                               │
│  • traders: RedBlackTree<Pubkey, TraderState>                │
│    - Updated: Free balances, locked balances                 │
│                                                               │
│  • order_sequence_number: u64                                 │
│    - Incremented for each new order                          │
│                                                               │
│  • unclaimed_quote_lot_fees: QuoteLots                       │
│    - Increased by taker fees from fills                      │
└──────────────────────────────────────────────────────────────┘

TraderState Updates:
┌──────────────────────────────────────────────────────────────┐
│  For Maker (resting order owner):                            │
│  • quote_lots_free: += quote_lots_received                   │
│  • quote_lots_locked: -= quote_lots_unlocked                 │
│  • base_lots_free: += base_lots_received                     │
│                                                               │
│  For Taker (liquidation order):                              │
│  • quote_lots_free: -= quote_lots_paid                       │
│  • quote_lots_locked: += quote_lots_locked                   │
│  • base_lots_free: -= base_lots_paid                         │
└──────────────────────────────────────────────────────────────┘
```

