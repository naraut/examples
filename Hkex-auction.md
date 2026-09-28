HKEX does not publish order-level (MBO) data during auction sessions, but it does publish a set of key market indicator data that is sufficient to support simulation needs during auctions.

📊 Data Actually Published During Auctions

According to HKEX's OMD-C interface specification, the following information is published through dedicated channels during auctions:

Data Item Message Type Purpose
IEP (Indicative Equilibrium Price) Indicative Equilibrium Price (41) Core anchor for simulating matching
IEV (Indicative Equilibrium Volume) Published alongside IEP Determines executable size
Order Imbalance Order Imbalance (56) Reflects supply/demand imbalance direction
Reference Price Reference Price (43) Basis for CAS price limits
Aggregate Order Book Aggregate Order Book Update (53) 10-level aggregated depth

🎯 Using IEP/IEV/Imbalance to Simulate Auction Fills

Since MBO data is missing during auctions, you need to infer the synthetic order book state from the IEP stream.

Core idea: IEP and IEV actually tell you "if matching occurred right now, what the trade price and volume would be." Order Imbalance tells you the direction of imbalance. Your simulator does not need to know every individual order — it only needs to maintain a synthetic order book consistent with IEP/IEV/Imbalance.

Specific Simulation Methods

1. Use IEP as the Price Anchor

When IEP updates, treat the IEP as the current theoretical auction trade price. If your algo order's price is better than the IEP (buy ≥ IEP, sell ≤ IEP), there is a possibility of execution.

2. Use IEV to Constrain Executable Size

IEV is the total number of shares matchable at the IEP. Your algo order's executable upper limit is constrained by IEV. If the algo's order quantity exceeds IEV, only partial execution is possible.

3. Use Imbalance to Determine Direction

The sign of Order Imbalance tells you which side is in surplus. If buy surplus, your sell orders are more likely to execute; vice versa.

4. Building the Synthetic Order Book

```java
public class AuctionSyntheticBook {
    private long iep;           // Updated from IEP messages
    private long iev;           // Updated from IEP messages
    private long imbalance;     // Updated from Order Imbalance messages
    private boolean buySurplus; // Imbalance direction
    
    public boolean isMarketable(Order algoOrder) {
        if (algoOrder.side == BUY) {
            return algoOrder.price >= iep && buySurplus == false;
        } else {
            return algoOrder.price <= iep && buySurplus == true;
        }
    }
    
    public long estimateFill(Order algoOrder) {
        if (!isMarketable(algoOrder)) return 0;
        // Simplified: assume algo order participates in IEV proportionally
        return Math.min(algoOrder.qty, iev);
    }
}
```

⚠️ Key Limitations

This method cannot precisely reproduce queue position. You can only answer "is the algo order within the executable price range" — you cannot answer "what is the algo order's exact position in the queue." IEV is an aggregated result and does not distinguish who is ahead.

For HKEX auction matching rules, orders are sorted by type (At-auction Orders first) → price → time. Your synthetic order book can use estimateFill to give a probabilistic fill estimate, rather than a deterministic result.

💡 Practical Recommendations

For CAS (Closing Auction Session) simulation:

· Use Reference Price as the basis for price limits (±5% in Stage 1, then tightened)
· Use the IEP stream to drive matching timing
· Accept uncertainty in fill volume; annotate confidence intervals in results

For POS (Pre-Opening Session) simulation:

· Similarly use IEP/IEV streams
· Note POS's random matching window (9:20–9:22)

Key realization: Auction-period simulation is inherently less precise than continuous trading periods. If your strategy is highly dependent on precise fill probability during auctions, it is advisable to focus testing on continuous trading periods, using auction results only as reference.
