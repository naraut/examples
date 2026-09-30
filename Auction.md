Yes, you can get IEP (Indicative Equilibrium Price) and IEV (Indicative Equilibrium Volume) from Refinitiv Tick History to drive auction matching. The "surplus" data (Order Imbalance) is also available.

📊 IEP/IEV Availability in Tick History

HKEX publishes IEP and IEV as specific message types during auction sessions. The Indicative Equilibrium Price (41) message contains both values: the Price field holds the IEP, and the AggregateQuantity field holds the IEV .

Refinitiv Tick History captures these messages as part of its Market Depth content. The Tick History Market Depth report explicitly supports RawMarketByOrder and other views that include these message types . Since Tick History records "unmanipulated recorded trade and quote messages from our real-time feed" , it captures the IEP/IEV stream as published by HKEX.

📈 Surplus (Order Imbalance) Data

The Order Imbalance (56) message type is part of HKEX's OMD-C feed . This gives you the surplus direction and magnitude during auctions. Tick History captures this as part of its market data recording.

🎯 How to Use IEP/IEV for Auction Matching

The IEP/IEV stream tells you the theoretical outcome if matching occurred at this moment. Your simulator uses this as the anchor:

1. IEP as the Trade Price Anchor
When IEP updates, it becomes the current theoretical auction price. Your algo order is "marketable" if:

· Buy: algo.price >= IEP
· Sell: algo.price <= IEP

2. IEV as the Volume Constraint
IEV is the total executable volume at the IEP. Your algo's fill cannot exceed what the IEV implies for its side .

3. Surplus as the Directional Indicator
Order Imbalance tells you which side is in surplus. If buy surplus, sell orders have priority; if sell surplus, buy orders have priority.

⚠️ Critical Limitation

IEP/IEV is aggregated. It does not tell you:

· Where your specific order sits in the auction queue
· Whether your order would be among the IEV shares selected for execution

Your fill estimate is therefore probabilistic, not deterministic. The best you can do is:

· If marketable, estimate fill as min(algo.qty, IEV) adjusted by your assumed queue position
· Report confidence intervals around the estimate

💡 Practical Recommendation

Use IEP/IEV from Tick History to drive your auction matching logic. Accept that auction fills are inherently less precise than continuous trading fills. For strategies highly sensitive to auction execution, treat auction results as directional validation rather than precise P&L prediction.
