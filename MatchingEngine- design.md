HKEX Exchange Simulator — High-Level Technical Design

1. Overview

This document describes the architecture, algorithms, and data structures for a high-fidelity HKEX exchange simulator. The system is designed for low latency and deterministic behavior, mirroring the architectural philosophy of CoinTossX: the matching engine is a standalone node, with market data dissemination and persistence running as separate nodes communicating via a low-latency messaging layer.

1.1 Design Goals

· Rule fidelity: Accurately model HKEX trading sessions, order types, and price validation.
· Low latency: Single-threaded matching core with zero-GC event processing.
· Determinism: Reproducible results for any given input sequence and seed.
· Separation of concerns: Matching, dissemination, and persistence are independent nodes.
· Extensibility: Rule parameters configurable per instrument and per session.

1.2 Out of Scope

· High availability / disaster recovery
· Real network protocol (FIX/Binary) serialization
· HKEX certification or conformance testing
· Pre-trade risk checks (position/credit limits)

---

2. System Architecture

2.1 Node Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                        SIMULATION CLOCK                          │
│                  (drives all nodes via events)                   │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  MATCHING NODE  │  │  MARKET DATA    │  │  PERSISTENCE    │
│                 │  │  NODE           │  │  NODE           │
│  ┌───────────┐  │  │                 │  │                 │
│  │ Session   │  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ State     │  │  │  │ Conflator │  │  │  │ Event Log │  │
│  │ Machine   │  │  │  └───────────┘  │  │  └───────────┘  │
│  └───────────┘  │  │                 │  │                 │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ Order     │  │  │  │ Snapshot  │  │  │  │ Tick DB   │  │
│  │ Books     │  │  │  │ Builder   │  │  │  └───────────┘  │
│  └───────────┘  │  │  └───────────┘  │  │                 │
│  ┌───────────┐  │  │                 │  │                 │
│  │ Matching  │  │  │                 │  │                 │
│  │ Engines   │  │  │                 │  │                 │
│  └───────────┘  │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
                    ┌─────────────────┐
                    │  EVENT BUS      │
                    │  (Ring Buffer)  │
                    │  LMAX Disruptor │
                    └─────────────────┘
```

2.2 Node Responsibilities

Node Responsibility Latency Target
Matching Node Order validation, session transitions, matching, trade generation < 10 µs per event
Market Data Node Conflate book updates, build IEP snapshots, publish to subscribers < 50 µs per update
Persistence Node Append events to log, write tick DB, snapshot state Async, non-blocking

2.3 Inter-Node Communication

All nodes communicate via a shared ring buffer (LMAX Disruptor pattern). Events are immutable value objects, pre-allocated to avoid GC pressure. The matching node publishes events; the market data and persistence nodes consume them independently.

```java
// Event types published by matching node
enum EventType {
    ORDER_ACCEPTED, ORDER_REJECTED, ORDER_MODIFIED, ORDER_CANCELLED,
    TRADE_EXECUTED, BOOK_UPDATE, IEP_UPDATE, SESSION_CHANGE
}
```

---

3. Session-Aware State Machine

3.1 Session Phases

The simulator models three trading sessions, each with distinct sub-phases:

Pre-Opening Session (POS)

· Order Input Period (09:00–09:15): ±15% price limit
· No-Cancellation Period (09:15–09:20): price limit tightens to BBO at 09:15
· Random Matching Period (09:20–09:22): random start within window

Continuous Trading Session (CTS)

· Morning (09:30–12:00)
· Afternoon (13:00–16:00)

Closing Auction Session (CAS)

· Reference Price Fixing (16:00–16:01): no order input
· Order Input Period (16:01–16:06): ±5% of reference price
· No-Cancellation Period (16:06–16:08): price limit tightens to BBO
· Random Closing Period (16:08–16:10): random close within window
· Order Matching: final IEP match

3.2 State Machine

```java
public enum SessionPhase {
    PRE_OPENING_INPUT,
    PRE_OPENING_NO_CANCEL,
    PRE_OPENING_RANDOM_MATCH,
    CONTINUOUS,
    CAS_REFERENCE_FIXING,
    CAS_ORDER_INPUT,
    CAS_NO_CANCEL,
    CAS_RANDOM_CLOSE,
    CAS_MATCHING,
    CLOSED
}

public class SessionStateMachine {
    private SessionPhase currentPhase;
    private LocalTime phaseStart;
    private LocalTime phaseEnd;
    private final Map<SessionPhase, PhaseRules> rules;
    private final Random rng; // seeded for reproducibility

    public void onTimerEvent(LocalTime now) {
        if (now.isAfter(phaseEnd)) {
            transitionTo(nextPhase());
        }
        // Handle random closing / random matching
        if (currentPhase == CAS_RANDOM_CLOSE) {
            if (now.isAfter(scheduledRandomClose)) {
                transitionTo(CAS_MATCHING);
            }
        }
    }

    public boolean acceptsOrder(OrderType type) {
        return rules.get(currentPhase).allowedOrderTypes().contains(type);
    }

    public boolean acceptsCancel() {
        return rules.get(currentPhase).cancellationAllowed();
    }
}
```

3.3 Phase Rules

Each phase has a PhaseRules object defining:

· Allowed order types: Limit, Enhanced Limit, Special Limit, At-Auction, At-Auction Limit
· Cancellation allowed: boolean
· Modification allowed: boolean
· Price limit policy: None, Fixed%, BBO-based, Reference-based
· Matching behavior: Immediate, Auction, None

3.4 Random Closing / Random Matching

```java
// Generate once per session, applied globally to all securities
private LocalTime scheduleRandomClose(LocalTime windowStart, LocalTime windowEnd) {
    long windowSeconds = Duration.between(windowStart, windowEnd).getSeconds();
    long offset = rng.nextLong(windowSeconds + 1);
    return windowStart.plusSeconds(offset);
}
```

Critical: The random close time is global — all CAS securities close at the same randomly chosen moment.

---

4. Order Book Data Structures

4.1 Price-Level Aggregation

```java
public class PriceLevel {
    long price;              // integer ticks
    long totalQuantity;      // aggregate resting quantity
    long orderCount;         // for statistics
    Queue<Order> orders;     // FIFO queue for price-time priority
}
```

4.2 Order Book Per Instrument

```java
public class OrderBook {
    // Descending for bids, ascending for asks
    TreeMap<Long, PriceLevel> bids;
    TreeMap<Long, PriceLevel> asks;
    
    // Cumulative quantity caches (updated incrementally)
    long[] cumulativeBidQty;  // indexed by price level
    long[] cumulativeAskQty;
    
    // Reference prices
    long nominalPrice;        // for 9-times rule
    long previousClose;       // for opening quotation
    long casReferencePrice;   // for CAS price limits
}
```

4.3 Why TreeMap?

· O(log n) insertion and lookup by price
· Native ordering (descending for bids via Comparator.reverseOrder())
· Easy iteration over "top N price levels" for Enhanced/Special Limit orders

4.4 Integer Pricing

All prices are stored as integer ticks (e.g., HKD 0.001 = tick 1). This eliminates floating-point rounding errors and ensures deterministic equality checks.

```java
// Spread table lookup
public long tickSize(long price) {
    return spreadTable.getTickSize(price);
}

// Price conversion
public long toTicks(BigDecimal price) {
    return price.divide(new BigDecimal(tickSize(price))).longValueExact();
}
```

---

5. Matching Algorithms

5.1 Continuous Trading — Price-Time Priority

Algorithm: Best price wins; ties broken by time priority.

```java
public List<Trade> matchContinuous(Order incoming) {
    List<Trade> trades = new ArrayList<>();
    TreeMap<Long, PriceLevel> opposite = (incoming.side == BUY) ? asks : bids;
    
    while (incoming.remainingQty > 0 && !opposite.isEmpty()) {
        Map.Entry<Long, PriceLevel> best = opposite.firstEntry();
        long bestPrice = best.getKey();
        
        // Price check for limit orders
        if (!crosses(incoming, bestPrice)) break;
        
        PriceLevel level = best.getValue();
        while (incoming.remainingQty > 0 && !level.orders.isEmpty()) {
            Order resting = level.orders.peek();
            long fillQty = Math.min(incoming.remainingQty, resting.remainingQty);
            
            trades.add(new Trade(incoming, resting, bestPrice, fillQty));
            incoming.remainingQty -= fillQty;
            resting.remainingQty -= fillQty;
            level.totalQuantity -= fillQty;
            
            if (resting.remainingQty == 0) level.orders.poll();
        }
        
        if (level.orders.isEmpty()) opposite.pollFirstEntry();
    }
    
    return trades;
}
```

5.2 Enhanced Limit Order

Matches up to 10 price queues from the best available price. Any residual rests in the book as a Limit Order.

```java
public List<Trade> matchEnhancedLimit(Order incoming) {
    List<Trade> trades = new ArrayList<>();
    TreeMap<Long, PriceLevel> opposite = (incoming.side == BUY) ? asks : bids;
    
    int queuesMatched = 0;
    while (incoming.remainingQty > 0 && !opposite.isEmpty() && queuesMatched < 10) {
        Map.Entry<Long, PriceLevel> best = opposite.firstEntry();
        // ... match same as continuous
        queuesMatched++;
    }
    
    // Residual rests in book
    if (incoming.remainingQty > 0) {
        addToBook(incoming);
    }
    return trades;
}
```

5.3 Special Limit Order

Identical to Enhanced Limit, but residual is cancelled immediately rather than resting.

```java
public List<Trade> matchSpecialLimit(Order incoming) {
    List<Trade> trades = matchEnhancedLimitInternal(incoming, 10);
    // Residual is discarded
    return trades;
}
```

5.4 Auction Matching — IEP Calculation

Algorithm: Find the price that maximizes executable volume, then apply tie-breaking rules.

```java
public IepResult calculateIep(OrderBook book) {
    // Collect all unique candidate prices (union of bid and ask prices)
    TreeSet<Long> candidates = new TreeSet<>();
    candidates.addAll(book.bids.keySet());
    candidates.addAll(book.asks.keySet());
    
    long bestIep = -1;
    long bestIev = -1;
    long bestImbalance = Long.MAX_VALUE;
    boolean bestIsBuySurplus = false;
    
    for (long price : candidates) {
        long cumBid = cumulativeBidAt(book, price);  // bids >= price
        long cumAsk = cumulativeAskAt(book, price);  // asks <= price
        
        long execVol = Math.min(cumBid, cumAsk);
        long imbalance = Math.abs(cumBid - cumAsk);
        
        if (execVol == 0) continue;
        
        if (execVol > bestIev) {
            bestIep = price; bestIev = execVol; bestImbalance = imbalance;
            bestIsBuySurplus = cumBid > cumAsk;
        } else if (execVol == bestIev) {
            if (imbalance < bestImbalance) {
                bestIep = price; bestImbalance = imbalance;
                bestIsBuySurplus = cumBid > cumAsk;
            } else if (imbalance == bestImbalance) {
                // Directional bias
                if (bestIsBuySurplus && price > bestIep) bestIep = price;
                else if (!bestIsBuySurplus && price < bestIep) bestIep = price;
                else {
                    // Reference price distance
                    long distNew = Math.abs(price - book.nominalPrice);
                    long distBest = Math.abs(bestIep - book.nominalPrice);
                    if (distNew < distBest || (distNew == distBest && price > bestIep)) {
                        bestIep = price;
                    }
                }
            }
        }
    }
    
    return new IepResult(bestIep, bestIev);
}
```

5.5 Auction Matching Execution

Once IEP is determined, match all orders at that price:

```java
public List<Trade> matchAuction(OrderBook book, long iep) {
    List<Trade> trades = new ArrayList<>();
    
    // All buy orders with price >= IEP are eligible
    // All sell orders with price <= IEP are eligible
    // At-Auction Orders (no price) have priority over At-Auction Limit Orders
    
    // Match eligible bids against eligible asks at IEP
    // Priority: At-Auction Orders > At-Auction Limit Orders (price, then time)
    
    return trades;
}
```

5.6 Fallback Rules

Scenario Behavior
IEP exists Match at IEP. Closing price = IEP.
No IEP, reference price exists Match at reference price.
No IEP, no reference price No matching. All auction orders cancelled.

---

6. Order Entry & Validation Pipeline

6.1 Validation Pipeline

```java
public ValidationResult validate(Order order) {
    // 1. Session check
    if (!sessionState.acceptsOrder(order.type)) {
        return reject("INVALID_ORDER_TYPE_FOR_SESSION");
    }
    
    // 2. Price validation (session-specific)
    PriceValidation pv = validatePrice(order);
    if (!pv.valid) return reject(pv.reason);
    
    // 3. Quantity validation
    if (order.qty <= 0 || order.qty % lotSize != 0) {
        return reject("INCORRECT_QTY");
    }
    
    // 4. Order chaining check
    if (order.chainId > MAX_CHAIN) {
        return reject("ORDER_EXCEED_LIMIT");
    }
    
    // 5. SMP check (see Section 7)
    SmpResult smp = checkSelfMatch(order);
    if (smp.shouldCancel) return smp.result;
    
    return accept();
}
```

6.2 Price Validation Rules

9-Times Rule (universal):

```java
long deviation = Math.abs(order.price - book.nominalPrice) / tickSize;
if (deviation >= 9) return reject("PRICE_EXCEEDS_PRICE_BAND");
```

Opening Quotation (first order of the day):

```java
long maxUp = Math.max(24 * tickSize, previousClose * 0.05);
long maxDown = Math.min(24 * tickSize, previousClose * 0.05);
if (order.side == BUY && order.price < previousClose - maxDown) reject(...);
if (order.side == SELL && order.price > previousClose + maxUp) reject(...);
```

CAS Two-Stage Limits:

```java
if (phase == CAS_ORDER_INPUT) {
    // ±5% of reference price
    long limit = casReferencePrice * 0.05;
    if (Math.abs(order.price - casReferencePrice) > limit) reject(...);
} else if (phase == CAS_NO_CANCEL || phase == CAS_RANDOM_CLOSE) {
    // Bounded by highest bid / lowest ask at end of Order Input
    if (order.price < casLowestAsk || order.price > casHighestBid) reject(...);
}
```

6.3 Rejection Reason Codes

Code Meaning
3 Order Exceed Limit
6 Duplicate order
13 Incorrect Qty
16 Price exceeds current price band
19 Reference price is not available
20 Notional value exceeds threshold

---

7. Self-Match Prevention (SMP)

```java
public SmpResult checkSelfMatch(Order incoming) {
    TreeMap<Long, PriceLevel> opposite = (incoming.side == BUY) ? asks : bids;
    
    for (PriceLevel level : opposite.values()) {
        for (Order resting : level.orders) {
            if (resting.smpId.equals(incoming.smpId)) {
                if (smpMode == AGGRESSIVE) {
                    return SmpResult.cancelIncoming();
                } else {
                    return SmpResult.cancelResting(resting);
                }
            }
        }
    }
    return SmpResult.noMatch();
}
```

Rules:

· SMP ID is inherited on amendment.
· Aggressive SMP: cancel incoming order.
· Passive SMP: cancel resting order.

---

8. Reference Price Calculation

8.1 CAS Reference Price

```java
public long calculateCasReferencePrice(OrderBook book, List<Long> snapshots) {
    // Take 5 nominal prices at 15-second intervals from 15:59:00
    // (or 11:59:00 for half-day trading)
    // Return the median (3rd value)
    snapshots.sort(Long::compare);
    return snapshots.get(2);
}
```

8.2 Nominal Price

The nominal price is the last traded price, or if no trade has occurred, the best bid/ask midpoint or the previous close. This is the anchor for the 9-times rule.

---

9. Volatility Control Mechanism (VCM)

9.1 Trigger Detection

```java
public boolean checkVcmTrigger(OrderBook book, long potentialTradePrice) {
    long referencePrice = book.price5MinAgo;
    long deviation = Math.abs(potentialTradePrice - referencePrice);
    double pct = (double) deviation / referencePrice;
    
    if (pct >= vcmThreshold) {
        return true;
    }
    return false;
}
```

9.2 Cooling-Off Period

· Duration: 5 minutes
· Trading continues but restricted to a pre-defined price band (±10% of reference price)
· Suppression windows: first 15 min of AM/PM sessions, last 20 min of PM session

9.3 State

```java
public class VcmState {
    boolean inCoolingOff;
    LocalTime coolingOffStart;
    LocalTime coolingOffEnd;
    long vcmReferencePrice;
    long vcmUpperLimit;
    long vcmLowerLimit;
}
```

---

10. Event Flow & Order Lifecycle

10.1 Order Lifecycle

```
NEW → VALIDATED → ACCEPTED → [MATCHED | RESTING] → [FILLED | CANCELLED | EXPIRED]
                    ↓
                REJECTED
```

10.2 Session Transition Events

On each session transition:

1. Pre-opening → Continuous:
   · Unmatched At-Auction Limit Orders within 9-times limit convert to Limit Orders.
   · Unmatched At-Auction Orders are cancelled.
2. Continuous → CAS:
   · Unmatched Limit Orders (within CAS price limits) carry forward as At-Auction Limit Orders.
   · Orders outside CAS price limits are cancelled.
3. CAS → Closed:
   · All unmatched orders are cancelled.

10.3 Event Publishing

The matching node publishes events to the ring buffer for consumption by market data and persistence nodes.

```java
public void onOrderAccepted(Order order) {
    ringBuffer.publish(EventType.ORDER_ACCEPTED, order);
    // ... continue matching
}

public void onTradeExecuted(Trade trade) {
    ringBuffer.publish(EventType.TRADE_EXECUTED, trade);
}
```

---

11. Market Data Node

11.1 Responsibilities

· Consume events from the ring buffer
· Build conflated book snapshots (10 levels)
· Calculate and publish IEP/IEV updates during auctions
· Publish session status changes

11.2 Conflation Strategy

The market data node aggregates book updates at a configurable interval (e.g., every 10 ms) to simulate HKEX's conflated feed behavior.

```java
public class Conflator {
    private final OrderBook book;
    private long lastPublishTime;
    private final long publishIntervalNanos;
    
    public void onBookUpdate(BookUpdate update) {
        long now = System.nanoTime();
        if (now - lastPublishTime >= publishIntervalNanos) {
            publishSnapshot(book);
            lastPublishTime = now;
        }
    }
}
```

11.3 IEP Updates During Auctions

During POS and CAS, the market data node publishes IEP/IEV updates on every order arrival that changes the equilibrium.

```java
public void onOrderArrival(Order order) {
    IepResult iep = calculateIep(book);
    if (iep.changed()) {
        publishIepUpdate(iep);
    }
}
```

---

12. Persistence Node

12.1 Responsibilities

· Append all events to a write-ahead log (WAL)
· Write tick data to a time-series database
· Periodically snapshot book state

12.2 Event Log

```java
public class EventLog {
    private final WritableByteChannel channel;
    
    public void append(Event event) {
        // Serialize event to bytes
        // Write to channel (async)
    }
}
```

12.3 Tick DB Schema

Column Type Description
timestamp BIGINT Nanoseconds since epoch
event_type SMALLINT ORDER_ADD, ORDER_MODIFY, TRADE, etc.
order_id BIGINT Unique order identifier
price BIGINT Integer ticks
quantity BIGINT Order quantity
side TINYINT BUY/SELL
session_phase SMALLINT Current session phase

12.4 Snapshot Strategy

· Snapshot book state every N events or every T seconds.
· On restart, replay from last snapshot + WAL.

---

13. Simulation Clock

13.1 Event-Driven Time

The simulation clock advances via a priority queue of TimerEvents. Wall-clock time is not used during backtesting.

```java
public class SimulationClock {
    private final PriorityQueue<TimerEvent> eventQueue;
    private long currentTimeNanos;
    
    public void schedule(long delayNanos, Runnable action) {
        eventQueue.add(new TimerEvent(currentTimeNanos + delayNanos, action));
    }
    
    public void advance() {
        TimerEvent next = eventQueue.poll();
        currentTimeNanos = next.timestamp;
        next.action.run();
    }
}
```

13.2 Determinism

· All randomness uses a seeded Random instance.
· Event ordering is deterministic: timestamp, then sequence number.
· No wall-clock dependencies in matching logic.

---

14. Performance Considerations

14.1 Zero-GC Design

· Pre-allocate all event objects in the ring buffer.
· Reuse Order objects via object pooling.
· Avoid boxing/unboxing; use primitive long for prices and quantities.

14.2 Cache Efficiency

· Keep hot data (best bid/ask, cumulative quantities) in contiguous arrays.
· Use TreeMap only for price-level lookup; keep FIFO queues as intrusive linked lists.

14.3 Single-Threaded Matching

The matching node is single-threaded to guarantee determinism. Parallelism is achieved by:

· Partitioning by instrument (each instrument has its own matching thread).
· Running market data and persistence nodes on separate threads.

14.4 Latency Budget

Operation Target
Order validation < 1 µs
Continuous match < 5 µs
IEP calculation < 20 µs
Event publish < 100 ns
Book snapshot < 50 µs

---

15. Testing Strategy

15.1 Unit Tests

· IEP calculation with known order books
· Price validation edge cases (9-times rule, opening quotation)
· Session transition logic

15.2 Integration Tests

· Full order lifecycle: submit → match → trade → cancel
· Session transitions with order carry-forward
· SMP scenarios

15.3 Scenario Replay

· Replay historical MBO data through the simulator
· Compare simulated IEP against historical IEP
· Compare simulated fills against known outcomes (where available)

15.4 Parity Audit

· Run the same strategy through the simulator and a naive backtester
· Quantify the difference in fill rates, slippage, and P&L
· The simulator should show less profitable results if it's more realistic

---

16. Configuration

16.1 Per-Instrument Configuration

```yaml
instrument:
  code: "0700"
  lotSize: 100
  spreadTable: "HKEX_STANDARD"
  vcmEligible: true
  casEligible: true
  vcmThreshold: 0.10
```

16.2 Session Configuration

```yaml
sessions:
  preOpening:
    orderInputStart: "09:00:00"
    noCancelStart: "09:15:00"
    randomMatchStart: "09:20:00"
    randomMatchEnd: "09:22:00"
  continuous:
    morningStart: "09:30:00"
    morningEnd: "12:00:00"
    afternoonStart: "13:00:00"
    afternoonEnd: "16:00:00"
  cas:
    referenceFixingStart: "16:00:00"
    orderInputStart: "16:01:00"
    noCancelStart: "16:06:00"
    randomCloseStart: "16:08:00"
    randomCloseEnd: "16:10:00"
```

---

17. Summary

This design provides a low-latency, deterministic, session-aware matching engine that accurately models HKEX trading rules. The separation of matching, market data, and persistence into independent nodes ensures scalability and maintainability, while the single-threaded matching core guarantees reproducibility.

The key implementation priorities are:

1. Session state machine with sub-phase rules
2. IEP calculation with correct tie-breaking
3. Order type handlers (Limit, Enhanced, Special, At-Auction)
4. Price validation (9-times rule, opening quotation, CAS limits)
5. Event-driven architecture with zero-GC ring buffer

This foundation enables high-fidelity backtesting, strategy validation, and scenario replay for HKEX-listed instruments.
