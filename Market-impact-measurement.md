The main measurements for market impact fall into two broad categories: the shape of the impact function (how price change scales with order size) and the temporal decomposition of impact (how much is temporary versus permanent, and how it decays).

📐 The Impact Law (Size-Dependence)

The most widely used empirical regularity is the square-root law. It states that the expected price change I(Q) for a metaorder of size Q scales with the square root of participation:

I(Q) = \sigma \cdot Y \cdot \left(\frac{Q}{V}\right)^{\beta}, \quad \beta \approx 0.5

where \sigma is daily volatility, V is daily volume, and Y is a dimensionless amplitude . The key metric here is the exponent \beta. If your simulator produces an exponent far from 0.5, its impact mechanics are unrealistic. Recent benchmarks on generative LOB models found exponents ranging from 0.63 to 1.21, slightly above the traditional 0.5 . The amplitude Y is the second metric—it tells you how large the impact is for a given participation rate.

⏱️ Temporal Decomposition (Temporary vs. Permanent)

Impact is typically split into two components over time:

Temporary Impact is the price concession you pay during execution. It is driven by the mechanical consumption of liquidity and typically decays after you stop trading. ABIDES-based execution models explicitly aggregate spread, temporary impact, and transient slippage into the realized trading cost . The key metric is the decay rate—how quickly the price reverts once execution ends.

Permanent Impact is the lasting price change that does not revert. It reflects information leakage or a genuine shift in the market's valuation. In the Almgren-Chriss framework, permanent impact is the limit of the impact function as t \to \infty . The key metric is the ratio of permanent to peak impact—for example, one theoretical model predicts permanent impact is about 2/3 of the peak .

📊 Explanatory Variables for Impact Models

When building a regression model for impact, the literature identifies four standard explanatory variables :

1. Daily participation rate r_D = \nu / V_D (metaorder size divided by daily volume)
2. Trading rate r = \nu / V (size divided by volume traded during the metaorder)
3. Volatility during the metaorder \sigma(\omega)
4. Duration T(\omega) (a proxy for information leakage)

💡 What to Measure in Your Simulator

For your HKEX simulator, the minimal set of measurements is:

Measurement What It Tells You How to Compute
Impact exponent \beta Does your simulator produce realistic concavity? Regress \log(I) on \log(Q/V) across runs
Temporary decay rate How fast does the price revert after your algo stops? Measure mid-price relaxation in the post-execution window
Permanent/peak ratio How much impact "sticks"? Compare price at t=\infty to peak impact
Amplitude Y Is the magnitude of impact realistic? Fit the square-root law to your baseline data

The critical methodological point from the ABIDES research is that you must measure these against seed-replay counterfactuals—paired baseline and treatment runs with identical seeds—to isolate causal impact from noise .
