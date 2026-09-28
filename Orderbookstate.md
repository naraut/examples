Core Conclusion

MBO data needs to be converted into order events — it cannot be used directly. The best way to join the two event streams is a timestamp-priority queue: historical MBO events (after conversion to order events) and algo order events both enter the same ordered queue, consumed by the matching engine in time order.

---

1. Converting MBO Data to Order Events

MBO data is already order-level, but needs standardization:

Conversion rules:

· Add Order message → NEW_ORDER event (with orderId, price, qty, side, timestamp)
· Modify Order message → MODIFY_ORDER event (quantity reduction preserves priority, quantity increase loses priority)
· Delete Order message → CANCEL_ORDER event
· Trade message → TRADE event (used to consume queue_ahead)

Key point: HKEX's SF data provides MBO during continuous trading, but does not publish individual order information during auction sessions. Auctions must use aggregated data or the IEP/IEV stream.

---

2. Strategy for Joining the Two Event Streams

Recommended Approach: Unified Event Queue (Timestamp Priority)

```
Historical MBO stream ──→ [Converter] ──→ Order events ──┐
                                                         ├──→ [Priority Queue] ──→ Matching Engine
Algo order stream ───────────────────────────────────────┘
```

Merge rules:

1. Primary sort key: Event timestamp (nanosecond precision)
2. Secondary sort key: Event type priority (historical events before simultaneous algo events, ensuring the algo sees the "already occurred" market state)
3. Final tiebreaker: Sequence number (ensures determinism)

Code Sketch

```java
public class EventJoiner {
    private final PriorityQueue<SimEvent> eventQueue;
    
    public void onHistoricalEvent(SimEvent event) {
        eventQueue.offer(event);
    }
    
    public void onAlgoOrder(Order order) {
        SimEvent event = SimEvent.fromAlgoOrder(order);
        eventQueue.offer(event);
    }
    
    public void processNext() {
        SimEvent event = eventQueue.poll();
        matchingEngine.process(event);
        // Matching engine may produce fills; algo updates state accordingly
    }
}
```

---

3. Building Order Book State

The matching engine builds order book state incrementally by replaying historical order events:

Build process:

1. Start with an empty order book
2. Consume historical MBO events in time order
3. For each Add event, insert the order at the back of the FIFO queue for its price level
4. For each Trade event, consume quantity from the corresponding price level (first consume queue_ahead, then algo orders)
5. For each Cancel event, update queue position according to the cancellation policy

Key constraint: The order book must be cleared at simulation start to ensure no orders leak from a previous run.

---

4. Algo Order Injection and Matching

When an algo order arrives:

1. Validation: Check session phase, price limits, and whether the order type is allowed
2. Inject into queue: Add the algo order event to the priority queue with the current simulation timestamp
3. Matching: When that event is consumed, the matching engine:
   · If market order or marketable limit order, attempts to match against the opposite side of the book
   · If passive limit order, inserts into the book (behind historical orders, receiving a time-priority disadvantage)
4. Generate fills: If matching succeeds, produces Trade events published to the algo via the event bus

Key behavior: An algo order's initial queue_ahead = the total historical order quantity already resting at its price level at that moment. When subsequent historical Trade events consume that level, queue_ahead is reduced first.

---

5. Simplification Suggestion

For a first version, use a serial processing model:

```
for each historical event in time order:
    apply to book
    update queue_ahead for algo orders
    if algo order fully filled:
        notify algo with fill
    if algo should react to market update:
        let algo generate orders
        apply algo orders to book immediately (or queue for next tick)
```

Note: Algo orders do not feed back into the historical event stream. This is the "no market impact" assumption, acceptable for small orders.


Market Impact Assessment

Yes, for market impact assessment you need synthetic agents. Replaying historical MBO data alone cannot measure impact because your algo's orders do not feed back into the market — the historical event stream proceeds as if your algo never existed.

Why Market Replay Cannot Measure Impact

Market replay with historical MBO data operates under the "no market impact" assumption: your algo's orders are injected into the book, but they do not affect the flow of subsequent historical events . If your algo places a large order that would realistically move prices and trigger other participants to react, replay will ignore that entirely.

A 2025 paper on execution simulators states this limitation explicitly: "Since our simulator does not model market impact, the algorithm's measured cost carries no component that scales with participation rate or order size" . This is precisely the problem for impact assessment.

The Agent-Based Alternative

Agent-based Interactive Discrete Event Simulation (ABIDES) is the standard framework for impact-aware simulation . In this model:

· Multiple autonomous agents generate order flow based on their own decision rules (market makers, momentum traders, arbitrageurs, noise traders) .
· Your algo is just another agent — its orders are processed by the matching engine alongside all others.
· The market state evolves endogenously — if your algo consumes liquidity, other agents react by adjusting their quotes, and prices move accordingly .

This endogeneity is what makes impact measurable. The same order that filled instantly in replay will now "walk the book" and push prices against you.

The Practical Path

Phase 1 (now): Use MBO replay for rule-accurate fill simulation and queue position modeling. This tells you whether you would have filled, not how much your order would have moved the market.

Phase 2 (impact assessment): Implement synthetic agents. CoinTossX already supports agent-based modeling with point-process test cases . You would:

1. Calibrate agent parameters from historical data (order arrival rates, cancellation rates, price placement distributions) .
2. Replace the historical MBO stream with agent-generated order flow.
3. Let your algo interact with these agents through the matching engine.

The agent-based approach is more complex to build and calibrate, but it is the only way to answer the question: "If I send this order, what will happen to the price?" Market replay can never answer that question because it assumes your order does not exist in the causal chain.



MBO Replay mode and algo order fills

Core Conclusion

In pure MBO replay mode, you cannot precisely determine "who consumed this liquidity" at the moment of a trade. Historical Trade events only tell you "X quantity was consumed at this price," but MBO data does not always contain the complete pairing information between the aggressor and the passive side. This is an inherent ambiguity of MBO replay.

Why This Ambiguity Exists

HKEX's SF data provides order-level information (price, quantity, type) during continuous trading, but the Trade Tick itself does not guarantee it contains complete counterparty order information. A trade in the data feed may appear only as:

· A Trade event (price, quantity, time)
· Simultaneously or shortly after, a Delete/Modify event at the corresponding price level

But the pairing relationship between these two is not always explicitly given. Your matching engine must infer which orders were consumed based on price, quantity, and time window.

Practical Approach: Aggregate Consumption by Price Level

In replay mode, the most practical approach is to not distinguish which specific order was consumed, but instead perform aggregate deduction by price level:

Step 1: When a Trade event arrives, locate the price level

```java
PriceLevel level = (tradeSide == SELL) ? book.bids.get(tradePrice) 
                                        : book.asks.get(tradePrice);
```

Step 2: Consume orders at that level in FIFO order

```java
long remaining = tradeQuantity;
while (remaining > 0 && !level.orders.isEmpty()) {
    Order resting = level.orders.peek();
    long fill = Math.min(remaining, resting.remainingQty);
    resting.remainingQty -= fill;
    level.totalQuantity -= fill;
    remaining -= fill;
    if (resting.remainingQty == 0) level.orders.poll();
}
```

This is the answer to "who gets consumed": in time-priority order, starting from the front of the queue. This is the most consistent inference with HKEX price-time priority rules.

How Your Algo Orders Participate in Consumption

Once your Algo order is injected into the book, it queues behind all existing historical orders. When a historical Trade event consumes that price level:

· Historical orders at the front of the queue are consumed first
· Only after all historical orders are consumed does your Algo order begin to be consumed

This is the natural manifestation of the queue_ahead mechanism. Your Algo order's initial queue_ahead = the total existing quantity at that price level when it joined.

Regarding the "Correlation" Question

You asked "historical trades won't change, but we need to consume liquidity for new Algo orders" — these two are not in conflict. Historical Trade events consume orders that actually exist in the book. If your Algo order has already joined the queue, then when a historical Trade event arrives, it will also be included in the consumption order (if it has reached the front of the queue).

The key constraint is: the total consumption of historical Trade events cannot exceed the actual liquidity at that price level. If your Algo order increases the total quantity at that level after joining, historical Trade events may "over-consume" — in which case your Algo order will bear that portion of consumption, equivalent to being "partially filled" within historical trades.

Confirmation from the 2026 Paper

Danait et al.'s paper explicitly states: when reconstructing the order book from aggregate data, the attribution rules for cancellations and trades are unobservable, and different attribution assumptions lead to significantly different passive fill outcomes. For MBO data, although order identities are known, the pairing of Trade events with passive orders still needs to be inferred through FIFO order. This inference is exactly what your matching engine does.


Matching engine constraints in consuming liquidity fot fills

Key Constraints

The consumption of a historical Trade event cannot exceed the actual historical liquidity at the corresponding price level, and consumption must strictly follow price-time priority (FIFO) order.

Constraint 1: Consumption Cap = Historical Executable Quantity at That Level

When a historical Trade event arrives, the liquidity it can consume is limited to orders at that price level that have already been confirmed to exist by the MBO data.

When your Algo order is injected into the book, it queues behind all historical orders at that price level. Therefore, when a historical Trade event consumes from the queue:

· Historical orders at the front of the queue are consumed first
· Only after all historical orders at the front are consumed does it reach your Algo order

This means a single historical Trade's consumption cannot exceed "the total historical order quantity at the front of the queue when it arrived". If it exceeds that, something is wrong with the data (e.g., you set the Algo order's initial queue_ahead too small, or the Trade pairing logic is incorrect).

Constraint 2: FIFO Order Cannot Be Violated

HKEX continuous trading rules explicitly state: price priority, then time priority for same price — earlier-entered orders must be fully executed before later orders at the same price can be processed.

Your consumption logic must strictly enforce this:

```java
// Start from the front of the queue, consume one by one; "skipping" earlier orders is not allowed
while (remaining > 0 && !level.orders.isEmpty()) {
    Order head = level.orders.peek();  // Must be the front of the queue
    long fill = Math.min(remaining, head.remainingQty);
    // ... consume
    remaining -= fill;
}
```

You cannot randomly pick orders from the middle or back of the queue to consume.

Constraint 3: Algo Order's queue_ahead Must Accurately Reflect Queue State at Join Time

When your Algo order is injected into the book, its queue_ahead must equal the total existing historical order quantity at that price level at the moment of injection.

This value is naturally obtained from MBO replay: after processing all historical events (Add/Modify/Delete) up to the point where the Algo order is injected, that level's totalQuantity is the queue_ahead.

Constraint 4: Trade Events Cannot "Overdraw" Future Liquidity

A Trade event can only consume liquidity that currently exists in the order book. It cannot consume historical orders that have not yet arrived, nor orders already consumed by previous Trade events.

This is the key to preventing "double consumption": after each historical Trade consumes, the corresponding level's totalQuantity must decrease, and subsequent Trades can only consume the remaining decreased amount.

Constraint 5: Consumption Direction Must Match the Trade's Aggressor Side

Trade events are directional. A SELLER Trade (aggressor is the seller) consumes bid liquidity; a BUYER Trade (aggressor is the buyer) consumes ask liquidity.

```java
PriceLevel level = (trade.side == SELLER) ? book.bids.get(trade.price) 
                                          : book.asks.get(trade.price);
```

Reversing the direction will completely corrupt the order book state.

Constraint 6: Cannot Consume Already-Cancelled Orders

If a historical order has already been removed by a Delete event, subsequent Trade events cannot consume it again. This means your consumption logic must maintain the order queue in real time: Delete events must immediately remove the corresponding order from the queue, not wait until a Trade occurs.

Final Answer on "Correlation"

The correlation between historical Trade events and your Algo orders is one-directional:

· Historical Trades consume historical orders: This is the core replay logic and cannot be changed.
· Historical Trades also consume your Algo orders: Only when your Algo order has reached the front of the queue and historical orders have been exhausted.
· Your Algo orders cannot "create" historical Trades: Replay data is fixed; your orders will not trigger new historical Trade events.

This is why your Algo order's initial queue_ahead must be conservatively estimated — it represents your queue position in the real market. If this value is set too small, your Algo will appear easier to fill than it actually is.
