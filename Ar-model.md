ABIDES does not natively include an autoregressive (AR) model trained on historical data as a core participant type, because its design philosophy prioritizes mechanistic, agent-based realism over statistical curve-fitting. The framework's creators explicitly note that models relying purely on historical data face a fundamental limitation: they "encounter limitations stemming from the inability to repeat history while introducing an experimental change" . An AR model would simply replay a statistical shadow of the past, not a market that can react to your algorithmic order.

🎯 What ABIDES Uses Instead of a Pure AR Model

The closest native component is the Background Agent, which uses historical data in a very specific, constrained way.

According to the ABIDES paper, the Background Agent queries a "data oracle" for a noisy observation of the most recent historical trade and mixes it with a prior belief to form a valuation . It then trades rationally toward that belief. This is not an autoregressive model of returns; it is a belief-updating mechanism that anchors the simulated price to history while allowing random noise and agent interaction to shape the path.

The paper describes this as inspired by Zero-Intelligence (ZI) traders but with a key distinction: the central value belief is a mixture of prior belief and noisy historical observation, and the agent implements a basic arbitrage strategy between the simulated price and its belief . The goal is to produce a "near approximation of historical trades" while still allowing agents to react to experimental changes.

🔬 The Research Gap ABIDES Was Designed to Address

The ABIDES authors are explicit about why they didn't build a pure historical-replay or AR-driven model. They state that "models that rely on historical data encounter limitations stemming from the inability to repeat history while introducing an experimental change and allowing subsequent events to be altered by that change" . If you use an AR model to generate order flow, your algorithmic order will not affect that flow—the model will continue generating its pre-trained statistical pattern regardless of your intervention.

ABIDES was built precisely to solve this: to create a counterfactual environment where your order does change the subsequent market path, because other agents react to the new prices and liquidity conditions.

💡 How You Can Add an AR Model to Your HKEX Simulator

If you still want an autoregressive participant, you can implement it yourself. The research literature has explored this direction via hybrid models. A 2023 paper introduces a "Neural Stochastic Agent-Based Limit Order Book Simulation" that pre-trains a neural point process model on historical LOB data to act as a background trader, then embeds it in ABIDES alongside strategic agents .

For your simulator, a simple AR participant would work like this:

1. Pre-train an AR model on your historical HKEX MBO/trade data to predict the next order's price/side/quantity based on recent history.
2. Wrap it as an Agent that, on wakeup, queries recent market state, feeds it to the AR model, and submits the predicted order.
3. Accept the limitation: This participant will generate realistic-looking background order flow, but it will not react to your algo's orders in a strategic way. It is a statistical shadow, not a thinking counterparty.

For your purposes—testing execution algos—this is often fine. The AR participant provides realistic texture to the order book. But it cannot measure strategic impact. That requires the reactive agents (Stoikov market makers, momentum traders, LLM agents) that ABIDES is designed to support.
