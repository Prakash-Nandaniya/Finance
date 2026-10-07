# Exchange & Trading System — Full Working Notes

---

## 1. Tick Data vs Bar Data

**Exchanges record tick data natively — bars are derived, not native.**

A tick is generated every time a trade actually executes (one match between a buy and sell order). Each tick carries:
- Timestamp (microsecond precision on modern exchanges)
- Price
- Quantity
- Buy/sell side
- Order IDs (internal, not public)

**Bars (1-min, 1-sec, etc.) are built by aggregating the tick stream:**
```
Open  = price of FIRST tick in the window
High  = MAX price among ticks in the window
Low   = MIN price among ticks in the window
Close = price of LAST tick in the window
Volume = SUM of quantities traded in the window
```
This is a groupby-aggregate on the tick stream, bucketed by a chosen window (time, volume, or dollar amount — see Week 1 roadmap: time bars vs volume bars vs dollar bars).

**"Live" moving bars** = the same bar being continuously recalculated as new ticks arrive within its still-open window. At 10:00:00 a new bar opens; every tick between 10:00:00–10:00:59 updates its O/H/L/C; at 10:01:00 it locks and a new bar starts. There's no special "live mode" — it's the same OHLC logic, just visibly mid-update.

**Tick frequency is irregular, not fixed.** Ticks happen whenever a trade executes — multiple per second for liquid stocks (Reliance, HDFC Bank), minutes apart for illiquid ones. This irregularity is exactly why sampling by FIXED TIME (bars) can produce statistically flawed, non-stationary data compared to sampling by activity (volume/dollar bars) — some time-bars contain hundreds of ticks, others almost none.

---

## 2. Order Book Structure

Two sorted lists per stock, maintained continuously:

```
BUY side (Bids) — sorted HIGHEST price first     SELL side (Asks) — sorted LOWEST price first
₹100.05 x 200                                      ₹100.10 x 150
₹100.00 x 500                                      ₹100.15 x 300
₹99.95  x 1000                                     ₹100.20 x 80
```

- **Best Bid** = highest buy price. **Best Ask** = lowest sell price.
- **Bid-ask spread** = Best Ask − Best Bid.

### Price-Time Priority (standard matching algorithm, NSE/BSE/most exchanges)
1. **Price priority first** — better-priced orders matched first.
2. **Time priority second (tie-break)** — among orders at the SAME price, earliest arrival (timestamp) matched first. Strict FIFO queue within each price level.

### What execution price is used when a trade happens

**Rule: the trade executes at the price of whichever order was ALREADY RESTING in the book — never at the incoming/aggressive order's own limit price.**

Example: resting sell at ₹105. New buy order arrives willing to pay up to ₹110.
→ Trade executes at ₹105 (seller's resting price). Buyer gets "price improvement" — pays less than their max.

If multiple orders rest at the same price (e.g., two buyers both at ₹110), the one with the earlier timestamp fills first — genuine FIFO queue, matched by exact arrival time.

### Market orders (no price specified — "execute now, whatever price")

- Market order vs resting limit order → executes at the resting limit order's price (same rule as above).
- Market order bigger than what's available at best price → "sweeps" through multiple price levels, generating **multiple separate ticks**, one per price level consumed. This is literally what market impact / slippage means.
- Market order vs Market order (no limit orders in book at all) → falls back to **Last Traded Price (LTP)** as the reference execution price, since neither side specifies a price. Rare in liquid stocks (limit orders almost always present).
- Market order with NOTHING on the opposite side → rejected (or the unfilled portion is cancelled if partially matched).

### What trader details does the order book hold

**Publicly visible:** price, quantity, (sometimes) number of orders at that level. **No identity information at all** — this anonymity prevents others from trading differently based on knowing who's behind an order.

**Internally tracked by the exchange (not public):** Trading Member ID (broker code), Client ID, Order ID, timestamp, and in India specifically — **UCC (Unique Client Code)**, mandated by SEBI so every order is traceable to a specific investor for regulatory/surveillance purposes, even though invisible in real time.

Demat account details are NOT part of the matching engine's live processing — they come into play downstream, during settlement (see Section 5).

---

## 3. Continuous Trading vs Call Auctions — When Matching Runs

### Continuous trading (most of the trading day, liquid stocks)
**Event-driven, not interval-based.** The matching engine processes each order the instant it arrives — no polling loop, no fixed check interval. This is exactly why nanosecond/microsecond speed matters (see Section 4) — if matching only ran every fixed interval, being faster within that interval would be pointless.

Orders are processed strictly sequentially per stock (often one dedicated core/thread per symbol in modern engines, to avoid race conditions) — "continuous" means no artificial batching between orders, not literal parallel processing of the same book.

### Call auctions (batch-based — the genuine exceptions)

Used at:
- **Market open** (NSE pre-open: 9:00–9:15 AM) — orders accumulate without matching, then one batch match at a single "equilibrium price."
- **Market close** — closing auction, same logic, for a stable closing price (used in index calculation, settlement).
- **Re-opening after a circuit-breaker halt** — avoids an arbitrary re-opening price from whoever's order lands first.
- **Illiquid stocks (NSE)** — run periodic call auctions throughout the day instead of continuous matching, since there isn't enough natural order flow for continuous matching to be meaningful.

### Call auction algorithm (equilibrium price), step by step

1. Collect all buy/sell orders during the window.
2. Build **cumulative demand** (buy orders at-or-above each price — since a buyer's limit is their max willing price) and **cumulative supply** (sell orders at-or-below each price — a seller's limit is their min acceptable price).
3. At each candidate price, matchable quantity = MIN(cumulative demand, cumulative supply).
4. **The equilibrium price = whichever price MAXIMIZES matchable quantity.**
5. All qualifying buy orders (≥ equilibrium price) and sell orders (≤ equilibrium price) execute AT that single price — including price improvement for anyone willing to pay more / accept less.
6. Leftover imbalance exactly at the equilibrium price is resolved via standard price-time priority (FIFO) among orders at that price.

**Tick generation in a call auction:** internally, each matched buy-sell pair still generates its own distinct trade record (needed for settlement/clearing — different buyers/sellers need separate trade confirmations). Externally, on market data feeds, this usually appears as ONE consolidated "opening price" print with total matched quantity — not a flurry of individual ticks like continuous trading shows.

---

## 4. Hardware/Software Architecture & Latency

### Is it hardware or software?

**Both, layered:**
- **Core matching engine logic** — predominantly software (highly optimized C++) on dedicated high-performance servers. Some cutting-edge venues are moving matching logic itself into FPGA, but this isn't yet the norm.
- **Surrounding infrastructure** (order parsing, validation, risk checks, network ingestion) — increasingly **FPGA-accelerated** (Field-Programmable Gate Array — reconfigurable hardware circuits), since these are fixed, repeatable logical checks well-suited to hardware implementation, avoiding OS-level overhead (interrupts, context switches, syscalls) that a general CPU running software incurs.

### Clock cycles / frequency

No single unified "exchange clock" — depends on the component:
- **CPU-based components:** ~3–4 GHz, sequential instruction execution — a single match operation takes many clock cycles (memory access, comparisons, data structure updates).
- **FPGA-based components:** lower raw clock frequency (100s of MHz to low GHz) but achieve lower END-TO-END latency because operations run as a custom hardware PIPELINE — many steps happen in parallel across the chip rather than sequentially.

### Realistic latency figures (tick arriving → trade executed)
- FPGA-based systems: sub-microsecond, some under 500 nanoseconds tick-to-trade
- Direct Memory Access (DMA) systems: 1–10 microseconds
- Kernel-bypass software systems: 10–100 microseconds
- Co-located round-trip to matching engine: low single-digit to tens of microseconds
- Retail broker connections (not co-located): 150–300+ microseconds one-way

### Why physical distance matters (co-location)

Signals travel through fiber at roughly 2/3 light speed. At nanosecond/microsecond timescales, even a few meters of extra cable is measurable latency. **Co-location** = trading firms pay to place servers physically inside/adjacent to the exchange's own data center, cutting distance to meters of fiber.

### Full pipeline, hardware/software boundary

```
Your order → network packet → fiber to exchange data center
  → NIC (often FPGA-accelerated)
  → Parsing/validation layer (often FPGA — fixed, repeatable logic)
  → Matching engine core (mostly software, C++, on optimized servers)
  → Trade confirmation + tick generation
  → Broadcast to subscribers (multicast)
```

### Broadcast — multicast, and to whom

Exchange broadcasts ticks ONCE via **multicast** networking; network switches replicate delivery to all subscribers simultaneously (more efficient than point-to-point to thousands of recipients).

**Recipients, by latency tier:**
1. Co-located HFT/institutional firms — direct raw feed, single-digit microseconds
2. Data vendors (Bloomberg, Refinitiv, exchange's own direct products) — redistribute further
3. Brokers — relay to retail platforms, with added latency
4. Retail platforms (Zerodha etc.) — can lag tens to hundreds of milliseconds behind the "true" real-time tick, after several additional hops

This latency gradient is exactly why retail can't compete with HFT on pure speed, and is the structural basis for latency-arbitrage strategies (Section 4a).

---

## 4a. Latency-Based HFT Strategies ("Front-Running"-Adjacent)

Three distinct things people often lump together:

**1. Speed / latency arbitrage (legal).** If a price moves on Exchange A, that information takes time to physically reach Exchange B. A firm co-located at both can trade on B's stale price before B catches up. Profiting from being faster at reacting to PUBLIC information — not from seeing anyone's private order.

**2. Direct feed vs consolidated feed arbitrage (legal, US-specific concept — SIP vs direct feeds).** Consolidated public feeds (aggregating multiple exchanges) have extra processing delay vs each exchange's own direct raw feed. Firms paying for direct feeds see true real-time prices before SIP-based (most retail) systems do.

**3. Order anticipation (legal but controversial) vs true front-running (illegal).**
   - **True front-running (illegal):** someone with PRIVATE access to a client's pending order trades ahead of it using that private knowledge. Clear regulatory violation (SEBI prohibits this in India).
   - **Order anticipation / "sniffing" (legal gray area):** HFT firms detect PUBLIC patterns — e.g., a large institutional order being split into many small pieces across venues — and use superior speed to race ahead to OTHER venues before the remaining pieces of that order arrive, buying liquidity first and selling back at a worse price for the institution. No private information accessed, just fast pattern inference from legally-visible order flow. This is the core subject of Michael Lewis's *Flash Boys*.

**Countermeasures built against this:**
- **IEX's speed bump** — deliberately adds ~350 microseconds artificial delay on incoming orders to neutralize pure speed advantage.
- **Batch auctions** (see Section 3) — collecting orders over a window and matching simultaneously removes the value of being microseconds faster.
- **SEBI colocation fairness rules** — monitor for unfair co-location advantages and actual illegal front-running by brokers.

---

## 5. Post-Trade Settlement — Exchange → Clearing Corp → Depository → Your Account

### The four entities involved

1. **Exchange (NSE/BSE)** — matching engine, price discovery, generates the trade.
2. **Clearing Corporation (NSE Clearing Ltd / ICCL for BSE)** — calculates net obligations, acts as Central Counterparty (CCP).
3. **Depository (CDSL or NSDL)** — the authoritative electronic record of who owns what shares.
4. **Depository Participant (DP)** — your broker, your interface to the depository (you don't interact with CDSL/NSDL directly).

### Step-by-step flow

**Step 1 — Trade execution.** Matching engine executes, tick generated, tagged internally with Trading Member ID + Client ID (UCC).

**Step 2 — Exchange reports to Clearing Corporation.** Full trade ledger sent at session end (and intraday for some segments).

**Step 3 — Netting.** Clearing Corp nets each client's day's trades (e.g., bought 500 + sold 300 of the same stock → net obligation is just 200) — massively reduces actual transfer volume across the market.

**Step 3a — CCP guarantee.** Clearing Corp interposes itself between every buyer and seller (buyer to every seller, seller to every buyer) — same CCP concept as in futures clearing. You have no counterparty risk to the specific person on the other side of your trade; if they fail to deliver, the Clearing Corp still guarantees your side.

**Step 4 — Settlement (T+1 in India).**
- Shares debited from seller's demat (via instruction to CDSL/NSDL)
- Money collected from buyer, routed to Clearing Corp
- Shares credited to buyer's demat
- Payment released to seller (via their broker)
- This simultaneous/conditional exchange of money and shares = **DvP (Delivery versus Payment)**, ensuring neither side can be cheated.

**Step 5 — Depository (CDSL/NSDL) updates its ledger.** CDSL/NSDL don't decide anything — they execute the Clearing Corp's instructions, updating the authoritative "who owns what" database. Two depositories exist for historical/competitive reasons (NSDL 1996, CDSL 1999, BSE-promoted) — functionally identical, doesn't matter which one your broker uses.

Because your demat holding is NOT exchange-tagged (just "100 shares of Reliance," no NSE/BSE label), you can sell shares bought via NSE on BSE or vice versa — same underlying holding.

**Step 6 — Reflected in your account.** Your broker's app (Kite etc.) queries CDSL/NSDL's records and displays them — the broker's app is a front-end layer, not an independent source of truth. Typically shows in "Holdings" the day after settlement completes (T+1), before that it's in a pending/positions view.

### Full pipeline diagram

```
You place order
   ↓
Exchange matching engine (executes, generates tick) — T+0
   ↓
Exchange reports trade to Clearing Corporation
   ↓
Clearing Corp nets your day's trades, calculates final obligation
   ↓
Clearing Corp acts as CCP — guarantees both sides
   ↓
T+1: Clearing Corp instructs CDSL/NSDL — debit seller, credit buyer
   ↓
Money flows: buyer → Clearing Corp → seller (via brokers), simultaneously (DvP)
   ↓
CDSL/NSDL updates official ownership record
   ↓
Your broker's app queries CDSL/NSDL, displays updated holdings
```

### Why the system is split into separate layers

Each layer is architecturally optimized for a different requirement:
- **Exchange** → optimized for SPEED (nanosecond/microsecond price discovery)
- **Clearing Corp** → optimized for RISK MANAGEMENT (netting, counterparty guarantee, preventing cascading default)
- **Depository** → optimized for ACCURACY (authoritative ownership ledger, no nanosecond urgency needed)

This is a deliberate separation-of-concerns design, the same pattern you'd recognize from software architecture — keeping the latency-critical matching engine lean and fast, while pushing the more complex, less time-critical account/ownership bookkeeping to a separate downstream system.

---

## Quick-Reference Summary Table

| Stage | What happens | Who/what does it | Speed |
|---|---|---|---|
| Order submission | Order sent from broker to exchange | You → Broker → Exchange gateway | Network/fiber dependent |
| Matching | Price-time priority match against order book | Exchange matching engine (software, FPGA-assisted ingestion) | Nanoseconds–microseconds |
| Tick generation | Trade recorded, broadcast | Exchange, multicast feed | Immediate, event-driven |
| Reporting | Trade ledger sent downstream | Exchange → Clearing Corp | End of session (some intraday) |
| Netting & guarantee | Obligations calculated, CCP role assumed | Clearing Corporation (NSE Clearing/ICCL) | Same day |
| Settlement | Shares/money actually move | Clearing Corp instructs CDSL/NSDL, DvP | T+1 |
| Record update | Ownership ledger updated | CDSL/NSDL | T+1 |
| Reflection to you | Holdings displayed | Your broker's app (queries depository) | T+1 (next day) |

---

*Compiled from conversation — covers tick/bar data, order book mechanics, price-time priority, market vs limit order matching, continuous trading vs call auctions, matching engine hardware/software architecture, latency-based HFT strategies, and the full post-trade settlement pipeline (Exchange → Clearing Corp → Depository → Broker → You).*