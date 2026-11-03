Calibrating your HKEX simulator against historical market data is the step that turns a collection of agents and matching logic into a tool that produces trustworthy, realistic results. The core principle is to compare the statistical output of your simulator against the stylized facts and impact mechanics of real HKEX data, then tune your parameters until the gap is minimized.

🎯 What to Calibrate

You are not trying to match a specific historical price path tick-for-tick. Instead, you are calibrating the statistical properties of the simulated market so that it behaves like the real one.

1. The Impact Law (Square-Root Law)
The most critical calibration target for an execution simulator is the relationship between order size and price impact. The empirical benchmark is the square-root law, where impact scales with the square root of the order's participation rate (Q/V). Recent research on U.S. equities confirms the exponent is approximately 0.5, with a prefactor c around 0.34 (bias-corrected). You should run your simulator with varying order sizes and regress the simulated impact to check if your exponent is near 0.5. If it is not, your agents' reaction to order flow (or the matching engine's liquidity consumption) is miscalibrated.

2. Order Flow Arrival Dynamics (Hawkes Processes)
Real order flow is not a simple Poisson process; it is self-exciting. A trade or a limit order arrival increases the probability of further arrivals in the near future. The standard model for this is a Hawkes process. You calibrate a Hawkes model on your historical HKEX MBO data (estimating the baseline intensity, excitation kernel, and decay rate) and then use those parameters to drive the arrival times of your synthetic agents. This ensures your simulator produces realistic clustering of order flow and volatility.

3. Agent-Specific Parameters (ABIDES-Style Calibration)
If you are using an ABIDES-like agent population (noise traders, value agents, market makers), their parameters must be fitted to historical data. Research on ABIDES calibration uses Bayesian Optimization to find parameters that minimize the difference between simulated and real data. The parameters typically tuned include:

· Agent counts (e.g., num_noise, num_value)
· Arrival rates (lambda_a)
· Order size distributions (min/max size)
· Risk aversion / market maker spreads.

The objective function is often the Kolmogorov-Smirnov (K-S) statistic between the simulated and real distributions of a target variable (e.g., inter-arrival times or spreads).

🛠️ How to Execute the Calibration

Step 1: Extract Historical Benchmarks from Your HKEX Data
From your MBO/tick database, compute the target statistics:

· Impact: The exponent and prefactor of the impact law from historical executions (if you have them) or reconstructed metaorders from the tape.
· Arrivals: The Hawkes parameters (baseline intensity, branching ratio, decay) for order arrivals.
· Stylized Facts: The distribution of bid-ask spreads, order sizes, and return kurtosis.

Step 2: Run the Simulator with a Parameter Vector
Set your agent parameters (arrival rates, impact sensitivities, counts). Run the simulator over a representative period (e.g., one full trading day) with a fixed seed.

Step 3: Compute the Objective Function (The Gap)
Compare the simulator's output statistics to the historical benchmarks. Use a metric like:

· The K-S statistic for distributional fit.
· The absolute error in the impact exponent.
· The difference in autocorrelation of order flow.

Step 4: Optimize the Parameters
Use an optimization algorithm to find the parameter set that minimizes the objective. For high-dimensional, noisy objectives like agent-based simulations, Bayesian Optimization (BO) or its variants (e.g., TuRBO) are the standard choice. The algorithm intelligently selects the next parameter set to try based on previous results, typically converging in a few hundred simulation runs rather than a brute-force grid search.

Step 5: Validate on a Hold-Out Period
Once the parameters are calibrated, run the simulator on a different historical period (e.g., a different week or month) and verify that the stylized facts and impact law still hold. This guards against overfitting to the calibration window.

⚠️ Key Practical Considerations for HKEX

· Volume Time vs. Physical Time: Impact is not uniform throughout the day. Real markets exhibit a U-shaped volume curve. When calibrating, you should map physical time to volume time (or at least use intraday volume profiles) so that your simulator's impact is correctly scaled for the time of day.
· Auction Sessions: The square-root law and Hawkes processes are calibrated on continuous trading data. Auction periods (POS/CAS) have different mechanics (IEP/IEV) and should be calibrated separately, if at all, using the indicative data available.
· The "No-Algo" Baseline: Calibration must be performed on the baseline simulator (without your execution algorithm injecting orders). You are calibrating the market environment, not your strategy. Your algo's performance is then measured against this calibrated environment.




The short answer is: Yes, the square-root law is highly relevant for HKEX, but with an important caveat regarding the market's microstructure. There is no direct study in the search results that explicitly fits the square-root law to HKEX metaorder data, but the theoretical and empirical evidence strongly suggests it applies, while HKEX's order-driven nature introduces nuances you must account for.

📐 Why It Should Apply to HKEX

The square-root law (I(Q) \propto \sqrt{Q/V}) has been shown to be remarkably universal. A 2025 study using complete data from the Tokyo Stock Exchange (TSE) found the exponent was exactly 1/2 for all liquid stocks, strongly supporting the "universality hypothesis". The TSE is an order-driven market, much like HKEX. Earlier research on HKEX itself confirms the necessary precondition: price impact is increasing and concave in trade size, which is the qualitative signature of the square-root relationship.

⚠️ HKEX-Specific Nuances

While the law applies, HKEX's pure order-driven structure means you must calibrate the pre-factor (\alpha in \Delta P = \alpha \sigma \sqrt{Q/V}) carefully. Unlike quote-driven markets, HKEX has no designated market makers for equities; liquidity is provided solely by limit orders. This can make the market more sensitive to large metaorders, as there is no obligation for a specialist to absorb the flow.

Historical HKEX research from 1995 found that price impact follows a U-shaped pattern over the trading day (high at open and close, lower midday), driven by variations in informed trading intensity and liquidity. Your calibration for HKEX must account for this intraday seasonality; the "average" square-root relationship will not hold uniformly throughout the day.

💡 Practical Calibration for Your Simulator

1. Target the Exponent: Use the square-root law (I \propto \sqrt{Q/V}) as your benchmark. If your simulated impact exponent deviates significantly from 0.5, your agents' reaction to order flow is likely miscalibrated.
2. Calibrate the Pre-factor: Fit the pre-factor \alpha using your historical HKEX MBO data. Do not assume it is the same as US or Japanese markets.
3. Account for Intraday Seasonality: Segment your calibration by time of day (e.g., open, midday, close) because HKEX's U-shaped impact pattern means the relationship's magnitude changes throughout the session.

In summary, the square-root law is your theoretical anchor for impact calibration on HKEX. You should expect it to hold qualitatively, but you must fit the parameters to HKEX's specific order-driven microstructure and intraday liquidity patterns.



Yes, calibration must be done with all agents in the mix together. You cannot calibrate each agent type in isolation, because the market's statistical properties emerge from the interaction between agents, not from any single agent's behavior.

🎯 Why Joint Calibration Is Necessary

The search results are explicit about this. A study on calibrating ABIDES states that agent-based models require calibration because "the behavior of the model is not analytically tractable" — the output is a complex function of all agents' parameters interacting through the matching engine . If you calibrate noise traders alone to match historical spread, then add market makers, the spread will change. The calibrated parameters are no longer valid.

Research on generative LOB models confirms this: the model is evaluated as a whole system, not agent by agent. The stylized facts (spread, volatility, impact law) are properties of the aggregate market, produced by the interaction of all participants . Calibrating one agent to match a target while holding others fixed will produce a parameter set that fails when the others are present.

📊 The Standard Calibration Approach

The dominant method in the literature is simulation-based calibration: you define a vector of all agent parameters, run the full simulation, compute the distance between simulated and historical statistics, and use an optimizer to find the parameter vector that minimizes this distance .

A representative study uses Bayesian Optimization to calibrate ABIDES parameters, treating the entire agent population as a single system . The objective function is typically a weighted combination of distributional distances (e.g., K-S statistics) across multiple targets: spread, order arrival rates, trade size distribution, and volatility .

⚠️ The Identifiability Problem

There is a known challenge: different agent parameter combinations can produce similar aggregate statistics. For example, a market with many noise traders and a tight-spread market maker might look statistically similar to a market with fewer noise traders and a wider-spread maker. The calibration may converge to a parameter set that matches the targets but does not uniquely identify the "true" agent composition .

The practical mitigation is to calibrate against multiple targets simultaneously (spread, volatility, arrival rates, impact exponent) and to validate on a hold-out period. If the calibrated parameters generalize to unseen data, the identifiability issue is less concerning for your purposes.

💡 Practical Workflow for Your Simulator

1. Define the parameter vector: All agent counts, arrival rates, order size distributions, and market maker risk aversion.
2. Run the full simulation with all agents present.
3. Compute the objective: The weighted distance between simulated and historical stylized facts (spread distribution, arrival rate, impact exponent).
4. Optimize: Use Bayesian Optimization to find the parameter vector that minimizes the objective.
5. Validate: Run the calibrated simulator on a held-out period and verify the stylized facts still hold.

One caveat: If you have a very large number of agent types (noise, value, momentum, Stoikov, AR), the parameter space becomes high-dimensional, and calibration becomes computationally expensive. In practice, you fix the parameters of "background" agents (noise traders) to reasonable values from literature, and calibrate only the parameters of the agents that most affect your target metrics (market makers, momentum traders). This reduces dimensionality while preserving joint calibration of the agents that matter.
