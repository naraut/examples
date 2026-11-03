The complete optimization design is a closed-loop calibration pipeline where Python drives the Bayesian Optimization (BO) loop and your Java simulator acts as a deterministic black-box evaluator. The historical market data is used to extract target distributions and statistics, which become the "ground truth" that the simulator must match.

🏗️ Complete Architecture Overview

```
Historical HKEX Data (MBO/Tick DB)
        ↓
[Feature Extraction] → Target Distributions & Statistics
        ↓
Python BO Loop (scikit-optimize Optimizer)
        ↓ ask() → parameter vector (JSON)
[Java Simulator Subprocess] (fresh JVM per evaluation)
        ↓ Run full-day simulation → Compute K-S distance
        ↓ stdout: objective (float)
Python BO Loop (tell() → update surrogate)
        ↓
Optimal Parameter Set
```

📊 What You Need from Historical Market Data

You do not need raw prices. You need empirical distributions and summary statistics extracted from the historical data to serve as calibration targets.

1. Order Arrival Distributions (Hawkes Parameters)
From the historical MBO/trade stream, extract inter-arrival times for order submissions. Fit a Hawkes process to capture self-excitation (clustering) . The target is the branching ratio (degree of endogeneity) and the decay kernel parameters. The simulator's synthetic agents must produce arrival patterns with the same clustering characteristics.

2. Trade Size Distribution
Extract the empirical distribution of trade sizes (or order sizes) from historical trades . The K-S statistic between the simulated and historical trade size distributions is a direct calibration target.

3. Spread Distribution
Extract the empirical distribution of bid-ask spreads at each timestamp. The simulator's market maker agents must produce spreads with a matching distribution.

4. Market Impact Law (Square-Root Exponent)
If you have historical metaorders (or can reconstruct them), fit the square-root law I/σ ∝ (Q/V)^0.5 and extract the exponent and pre-factor . The simulator's impact must match this concave relationship.

5. Stylized Facts (Return Kurtosis, Volatility Clustering)
Compute the return kurtosis and the autocorrelation of absolute returns. These serve as validation targets (not direct optimization targets) to confirm the calibrated simulator produces realistic price dynamics.

🐍 Python Side: The BO Loop Implementation

Use scikit-optimize's Optimizer class with ask-and-tell interface to control the loop manually .

```python
from skopt import Optimizer
from skopt.space import Real, Integer
import subprocess, json

# 1. Define parameter space (from ABIDES calibration example)
space = [
    Integer(2500, 7500, name='num_noise'),
    Real(1e-16, 2e-12, name='lambda_noise'),
    Integer(0, 200, name='num_value'),
    Real(1e-16, 2e-12, name='lambda_value'),
    Real(0.01, 1.0, name='mm_risk_aversion'),
    # ... other parameters
]

# 2. Initialize optimizer
opt = Optimizer(dimensions=space, base_estimator='GP', acq_func='EI', random_state=42)

# 3. BO loop
for iteration in range(300):  # Budget: 300 evaluations for high-dim [citation:8]
    next_params = opt.ask()  # Suggest parameter vector
    params_json = json.dumps(dict(zip([d.name for d in space], next_params)))
    
    # Call Java simulator as subprocess
    result = subprocess.run(
        ['java', '-jar', 'hkex-sim.jar', '--params', params_json],
        capture_output=True, text=True, timeout=600
    )
    objective = float(result.stdout.strip())  # K-S distance
    
    opt.tell(next_params, objective)  # Update surrogate
```

Key BO settings from research: For high-dimensional calibration (18 parameters), use TuRBO (Trust-Region BO) with 300 evaluations . For fewer parameters (< 6), standard BO with 100 evaluations and a mixture of EI/PI/LCB acquisition works .

☕ Java Simulator: The Objective Evaluator

The Java simulator must:

1. Parse the JSON parameter vector from --params.
2. Instantiate all agents together (noise, value, momentum, market maker) with the given parameters. Each agent gets its own seeded Random for determinism.
3. Run a full-day simulation through the matching engine (continuous + auctions).
4. Compute the K-S distance between the simulated output and the historical targets.
5. Print only the objective value to stdout.

The K-S Distance Calculation (Java):

```java
// Two-sample K-S statistic between simulated and historical distributions
public double ksStatistic(double[] simulated, double[] historical) {
    // Sort both arrays, compute empirical CDFs, find max absolute difference
    // Return the K-S statistic (lower = more similar)
}
```

The paper uses a generalized K-S test with Bonferroni correction to handle high-dimensional time series distributions . For simplicity, you can compute the K-S statistic separately for each target (arrival times, trade sizes, spreads) and combine them with weights.

🔄 The Complete Step-by-Step Procedure

Step 1: Extract Historical Targets
From your HKEX tick DB, compute:

· Inter-arrival time distribution (empirical CDF)
· Trade size distribution (empirical CDF)
· Spread distribution (empirical CDF)
· Impact exponent (fit from reconstructed metaorders)
  Store these as reference arrays/files accessible to the Java simulator.

Step 2: Define Parameter Ranges
Based on the ABIDES calibration example , define realistic ranges:

· num_noise: [2500, 7500]
· lambda_noise: [1e-16, 2e-12]
· num_value: [0, 200]
· lambda_value: [1e-16, 2e-12]
· mm_risk_aversion: [0.01, 1.0]

Step 3: Run the BO Loop (Python)
For each iteration:

· ask() → parameter vector
· Launch Java subprocess with JSON params
· Parse objective from stdout
· tell() → update GP surrogate

Step 4: Validate on Hold-Out Data
After BO converges, run the calibrated simulator on a different historical period and verify that the K-S distances remain low and the stylized facts hold.

⚠️ Critical Design Decisions

Fresh JVM per evaluation: This guarantees no state leakage between BO iterations. The ~200ms JVM startup overhead is acceptable for 300 evaluations (~60 seconds total).

Determinism: The Java simulator must use a fixed master seed so that the same parameter vector always produces the same objective. Without this, the BO surrogate receives noisy data and fails.

All agents together: Do not calibrate agents in isolation. The K-S distance is computed on the aggregate market output produced by all agents interacting through the matching engine .

Budget: For 18 parameters, use 300 evaluations . For 2-6 parameters, 100 evaluations is sufficient.

📐 What the Historical Data Provides (Summary)

Data Need Extracted Target Used By
MBO order arrivals Hawkes parameters (branching ratio, decay) Java simulator's agent arrival rates
Trade records Trade size distribution (CDF) K-S distance objective
Quote snapshots Spread distribution (CDF) K-S distance objective
Reconstructed metaorders Square-root exponent (~0.5) Impact validation target
Price returns Kurtosis, volatility clustering Post-calibration validation

This design gives you a programmatic, reproducible calibration pipeline that runs hundreds of simulations to find the parameter set that makes your HKEX simulator's statistical output match the real market.


The complete optimization design is a closed-loop calibration pipeline where Python drives the Bayesian Optimization (BO) loop and your Java simulator acts as a deterministic black-box evaluator. The historical market data is used to extract target distributions and statistics, which become the "ground truth" that the simulator must match.

🏗️ Complete Architecture Overview

```
Historical HKEX Data (MBO/Tick DB)
        ↓
[Feature Extraction] → Target Distributions & Statistics
        ↓
Python BO Loop (scikit-optimize Optimizer)
        ↓ ask() → parameter vector (JSON)
[Java Simulator Subprocess] (fresh JVM per evaluation)
        ↓ Run full-day simulation → Compute K-S distance
        ↓ stdout: objective (float)
Python BO Loop (tell() → update surrogate)
        ↓
Optimal Parameter Set
```

📊 What You Need from Historical Market Data

You do not need raw prices. You need empirical distributions and summary statistics extracted from the historical data to serve as calibration targets.

1. Order Arrival Distributions (Hawkes Parameters)
From the historical MBO/trade stream, extract inter-arrival times for order submissions. Fit a Hawkes process to capture self-excitation (clustering) . The target is the branching ratio (degree of endogeneity) and the decay kernel parameters. The simulator's synthetic agents must produce arrival patterns with the same clustering characteristics.

2. Trade Size Distribution
Extract the empirical distribution of trade sizes (or order sizes) from historical trades . The K-S statistic between the simulated and historical trade size distributions is a direct calibration target.

3. Spread Distribution
Extract the empirical distribution of bid-ask spreads at each timestamp. The simulator's market maker agents must produce spreads with a matching distribution.

4. Market Impact Law (Square-Root Exponent)
If you have historical metaorders (or can reconstruct them), fit the square-root law I/σ ∝ (Q/V)^0.5 and extract the exponent and pre-factor . The simulator's impact must match this concave relationship.

5. Stylized Facts (Return Kurtosis, Volatility Clustering)
Compute the return kurtosis and the autocorrelation of absolute returns. These serve as validation targets (not direct optimization targets) to confirm the calibrated simulator produces realistic price dynamics.

🐍 Python Side: The BO Loop Implementation

Use scikit-optimize's Optimizer class with ask-and-tell interface to control the loop manually .

```python
from skopt import Optimizer
from skopt.space import Real, Integer
import subprocess, json

# 1. Define parameter space (from ABIDES calibration example)
space = [
    Integer(2500, 7500, name='num_noise'),
    Real(1e-16, 2e-12, name='lambda_noise'),
    Integer(0, 200, name='num_value'),
    Real(1e-16, 2e-12, name='lambda_value'),
    Real(0.01, 1.0, name='mm_risk_aversion'),
    # ... other parameters
]

# 2. Initialize optimizer
opt = Optimizer(dimensions=space, base_estimator='GP', acq_func='EI', random_state=42)

# 3. BO loop
for iteration in range(300):  # Budget: 300 evaluations for high-dim [citation:8]
    next_params = opt.ask()  # Suggest parameter vector
    params_json = json.dumps(dict(zip([d.name for d in space], next_params)))
    
    # Call Java simulator as subprocess
    result = subprocess.run(
        ['java', '-jar', 'hkex-sim.jar', '--params', params_json],
        capture_output=True, text=True, timeout=600
    )
    objective = float(result.stdout.strip())  # K-S distance
    
    opt.tell(next_params, objective)  # Update surrogate
```

Key BO settings from research: For high-dimensional calibration (18 parameters), use TuRBO (Trust-Region BO) with 300 evaluations . For fewer parameters (< 6), standard BO with 100 evaluations and a mixture of EI/PI/LCB acquisition works .

☕ Java Simulator: The Objective Evaluator

The Java simulator must:

1. Parse the JSON parameter vector from --params.
2. Instantiate all agents together (noise, value, momentum, market maker) with the given parameters. Each agent gets its own seeded Random for determinism.
3. Run a full-day simulation through the matching engine (continuous + auctions).
4. Compute the K-S distance between the simulated output and the historical targets.
5. Print only the objective value to stdout.

The K-S Distance Calculation (Java):

```java
// Two-sample K-S statistic between simulated and historical distributions
public double ksStatistic(double[] simulated, double[] historical) {
    // Sort both arrays, compute empirical CDFs, find max absolute difference
    // Return the K-S statistic (lower = more similar)
}
```

The paper uses a generalized K-S test with Bonferroni correction to handle high-dimensional time series distributions . For simplicity, you can compute the K-S statistic separately for each target (arrival times, trade sizes, spreads) and combine them with weights.

🔄 The Complete Step-by-Step Procedure

Step 1: Extract Historical Targets
From your HKEX tick DB, compute:

· Inter-arrival time distribution (empirical CDF)
· Trade size distribution (empirical CDF)
· Spread distribution (empirical CDF)
· Impact exponent (fit from reconstructed metaorders)
  Store these as reference arrays/files accessible to the Java simulator.

Step 2: Define Parameter Ranges
Based on the ABIDES calibration example , define realistic ranges:

· num_noise: [2500, 7500]
· lambda_noise: [1e-16, 2e-12]
· num_value: [0, 200]
· lambda_value: [1e-16, 2e-12]
· mm_risk_aversion: [0.01, 1.0]

Step 3: Run the BO Loop (Python)
For each iteration:

· ask() → parameter vector
· Launch Java subprocess with JSON params
· Parse objective from stdout
· tell() → update GP surrogate

Step 4: Validate on Hold-Out Data
After BO converges, run the calibrated simulator on a different historical period and verify that the K-S distances remain low and the stylized facts hold.

⚠️ Critical Design Decisions

Fresh JVM per evaluation: This guarantees no state leakage between BO iterations. The ~200ms JVM startup overhead is acceptable for 300 evaluations (~60 seconds total).

Determinism: The Java simulator must use a fixed master seed so that the same parameter vector always produces the same objective. Without this, the BO surrogate receives noisy data and fails.

All agents together: Do not calibrate agents in isolation. The K-S distance is computed on the aggregate market output produced by all agents interacting through the matching engine .

Budget: For 18 parameters, use 300 evaluations . For 2-6 parameters, 100 evaluations is sufficient.

📐 What the Historical Data Provides (Summary)

Data Need Extracted Target Used By
MBO order arrivals Hawkes parameters (branching ratio, decay) Java simulator's agent arrival rates
Trade records Trade size distribution (CDF) K-S distance objective
Quote snapshots Spread distribution (CDF) K-S distance objective
Reconstructed metaorders Square-root exponent (~0.5) Impact validation target
Price returns Kurtosis, volatility clustering Post-calibration validation

This design gives you a programmatic, reproducible calibration pipeline that runs hundreds of simulations to find the parameter set that makes your HKEX simulator's statistical output match the real market.

The Bayesian Optimization (BO) loop is the "brain" of the calibration pipeline. Its job is to intelligently decide which parameter vector to try next so that your Java simulator's output matches historical HKEX data with as few expensive full-day simulations as possible.

🏗️ Core Architecture: Python Owns the Loop

You use scikit-optimize's Optimizer class with the ask-and-tell interface. This is specifically designed for situations where the objective function is not a simple Python call, but an external process you must launch yourself . The loop follows a strict pattern: ask() → run Java subprocess → parse objective → tell() → repeat .

🐍 Python Loop Design (Step-by-Step)

1. Define the Parameter Space
Map your tunable agent parameters to skopt dimensions. Based on the ABIDES calibration research, the parameter ranges are concrete :

```python
from skopt.space import Real, Integer

space = [
    Integer(2500, 7500, name='num_noise'),
    Real(1e-16, 2e-12, name='lambda_noise'),
    Integer(0, 200, name='num_value'),
    Real(1e-16, 2e-12, name='lambda_value'),
    Real(0.01, 1.0, name='mm_risk_aversion'),
    # ... add up to 18 parameters for TuRBO
]
```

2. Initialize the Optimizer
For few parameters (< 6), use standard BO with a Gaussian Process. For many parameters (up to 18), you need TuRBO (Trust-Region BO), which is not built into skopt. You would either implement the trust-region logic yourself (tracking success/failure counters and expanding/shrinking a local search radius) or switch to BoTorch which has TuRBO built-in .

```python
from skopt import Optimizer

opt = Optimizer(
    dimensions=space,
    base_estimator='GP',
    acq_func='EI',  # Expected Improvement (default is a mix, but EI is standard)
    random_state=42  # CRITICAL: Fixed seed for reproducibility
)
```

3. The Core Loop
The budget is determined by the research: 100 evaluations for 2–6 parameters, and 300 evaluations for 18 parameters .

```python
import subprocess, json

for iteration in range(300):  # Budget = 300 for high-dimensional
    # A. Ask for next parameter vector
    next_params = opt.ask()
    
    # Convert to JSON for the Java subprocess
    params_json = json.dumps(dict(zip([d.name for d in space], next_params)))
    
    # B. Launch Java simulator (fresh JVM per evaluation)
    result = subprocess.run(
        ['java', '-jar', 'hkex-sim.jar', '--params', params_json],
        capture_output=True, text=True, timeout=600
    )
    
    # C. Parse objective (ONLY the float from stdout)
    objective = float(result.stdout.strip())
    
    # D. Tell the optimizer the result
    opt.tell(next_params, objective)
    
    print(f"Iter {iteration}: objective={objective:.6f}")
```

4. Handling the Parallel Option (Optional)
If you want to reduce wall-clock time, skopt supports asking for multiple points simultaneously using the "Constant Liar" strategy (e.g., cl_min for conservative exploration). You would launch 4 Java processes in parallel, then call opt.tell(points, [obj1, obj2, obj3, obj4]). However, this trades sample efficiency for speed—for a first implementation, keep it sequential .

⚠️ Critical Design Decisions & Pitfalls

The Java Simulator Must Be Deterministic
The BO loop relies on the objective being a deterministic function of the parameters. Your Java simulator must use a fixed master seed (passed as a parameter). If the same parameter vector produces different objectives on different runs, the GP surrogate receives noisy data and calibration will fail.

Fresh JVM per Evaluation
Launching a new Java process for each of the 300 evaluations guarantees no state leakage (no cached agent state, no previous order book). The ~200ms JVM startup overhead is negligible compared to a full-day simulation.

Stdout Must Be Clean
The Java simulator must print only the objective value to stdout. Any debug logs must go to stderr. If Python parses an empty string or a log line, the loop crashes.

Budget & Acquisition Function from Research
The ACM study explicitly recommends: for high-dimensional calibration (18 params), use TuRBO with 300 evaluations. For standard BO (fewer params), use a mixture of EI/PI/LCB as the acquisition function . skopt's default acq_func="EI" is a reasonable starting point for low dimensions.

Pickling for Resumption
Because 300 full-day simulations may take hours, skopt allows you to pickle the Optimizer instance. If the process is interrupted, you can reload it and continue from where you left off without losing the surrogate model's learned state .

💡 What the Python Loop Does NOT Do

· It does not build agents or run the matching engine. That is exclusively the Java simulator's job.
· It does not compute the K-S distance. The Java simulator computes the distance and returns the scalar.
· It does not read historical data. The Java simulator loads the pre-computed target distributions (from Step 1) and compares its own output against them.

One-Sentence Summary: The Python BO loop is a deterministic optimization driver that asks skopt for a parameter vector, launches a fresh Java JVM to evaluate it, parses the returned distance, and tells skopt the result—repeating until the budget is exhausted.

The Java Objective Evaluator is a stateless, deterministic function that takes a parameter vector, runs one full-day simulation, compares the simulated market output against pre-computed historical target distributions, and returns a single scalar (the objective value). It does not optimize; it only evaluates.

🎯 Core Responsibility

Given a parameter vector  \theta  (agent counts, arrival rates, market maker risk aversion, etc.):

1. Load the pre-computed historical target distributions (from Step 1).
2. Build the agent population using  \theta  and a fixed master seed.
3. Run the matching engine through the full trading day.
4. Compute the K-S statistic between simulated and historical distributions for each target.
5. Combine the K-S statistics into a single weighted objective.
6. Print only the objective value to stdout.

🏗️ Class Design

```java
public class ObjectiveEvaluator {
    public static void main(String[] args) {
        // 1. Parse params JSON from args
        // 2. Load historical targets from serialized files
        // 3. Build agents with params + fixed seed
        // 4. Run simulation
        // 5. Compute K-S distances
        // 6. Print objective to stdout
    }
}

public class SimulationResult {
    long[] interArrivalNanos;   // from simulated order arrivals
    long[] tradeSizes;          // from simulated trades
    long[] spreadsInTicks;      // from simulated book snapshots
    double impactExponent;      // fitted from simulated metaorders
}

public class HistoricalTargets {
    long[] interArrivalNanos;
    long[] tradeSizes;
    long[] spreadsInTicks;
    double impactExponent;
}

public class KsCalculator {
    // Two-sample K-S statistic
    public double statistic(double[] x, double[] y);
}

public class ObjectiveCombiner {
    // Weighted sum of K-S distances
    public double combine(double ksArrival, double ksTradeSize, 
                          double ksSpread, double impactError);
}
```

🔍 K-S Statistic Implementation

The two-sample K-S statistic  D_{n,m}  is the maximum absolute difference between the empirical CDFs of two samples . You can implement it directly without third-party libraries:

```java
public class KsCalculator {
    public double statistic(double[] x, double[] y) {
        // Sort both arrays (assumes inputs are unsorted)
        double[] sx = x.clone();
        double[] sy = y.clone();
        Arrays.sort(sx);
        Arrays.sort(sy);
        
        int i = 0, j = 0;
        double maxDiff = 0.0;
        double cdfX = 0.0, cdfY = 0.0;
        
        while (i < sx.length && j < sy.length) {
            if (sx[i] < sy[j]) {
                i++;
            } else if (sx[i] > sy[j]) {
                j++;
            } else {
                i++;
                j++;
            }
            cdfX = (double) i / sx.length;
            cdfY = (double) j / sy.length;
            maxDiff = Math.max(maxDiff, Math.abs(cdfX - cdfY));
        }
        // Handle remaining elements
        while (i < sx.length) {
            i++;
            cdfX = (double) i / sx.length;
            maxDiff = Math.max(maxDiff, Math.abs(cdfX - cdfY));
        }
        while (j < sy.length) {
            j++;
            cdfY = (double) j / sy.length;
            maxDiff = Math.max(maxDiff, Math.abs(cdfX - cdfY));
        }
        
        return maxDiff;
    }
}
```

Note: Apache Commons Math provides KolmogorovSmirnovTest with a kolmogorovSmirnovStatistic(x, y) method . If you allow Apache Commons Math as a dependency, you can use it directly. If not, the implementation above is sufficient.

📊 Computing Each Target

Target 1: Inter-Arrival Times
During simulation, record the timestamp of each synthetic agent's Add Order submission. Compute inter-arrival times per agent (or aggregate across all agents). Return as long[].

Target 2: Trade Sizes
During simulation, record the quantity of each Trade event. Return as long[].

Target 3: Spreads
During simulation, sample the best bid/ask at each book update. Compute spread = ask - bid in ticks. Return as long[].

Target 4: Impact Exponent
Reconstruct metaorders from the simulated trade stream (same heuristic as Step 1). Fit the square-root law via OLS on log-transformed data. Return the exponent as double.

⚖️ Combining the Objective

The objective is a weighted sum of the K-S statistics and the impact error:

```java
public double combine(double ksArrival, double ksTradeSize, 
                      double ksSpread, double impactError) {
    double w1 = 0.25;  // arrival weight
    double w2 = 0.25;  // trade size weight
    double w3 = 0.25;  // spread weight
    double w4 = 0.25;  // impact weight
    return w1 * ksArrival + w2 * ksTradeSize + w3 * ksSpread + w4 * impactError;
}
```

Weight selection: The weights should reflect which targets matter most for your use case. If you are testing execution algorithms, the impact exponent and spread are more important. If you are testing price discovery, arrival dynamics and trade sizes matter more. Start with equal weights and adjust based on validation results.

⚠️ Critical Constraints

Determinism: The evaluator must use a fixed master seed (passed as an argument or hardcoded) so that the same parameter vector always produces the same objective. All randomness flows from new Random(masterSeed + agentId).

Statelessness: Each invocation is a fresh JVM. No cached state, no previous order book, no persistent files written.

Clean stdout: Print only the objective value to stdout. All logs go to stderr.

Fast fail: If a simulation produces invalid results (e.g., zero trades, infinite spread), return a large penalty value (e.g., Double.MAX_VALUE) rather than crashing. The BO loop will learn to avoid that region.

Normalization: K-S statistics are already in [0, 1]. The impact error should also be normalized (e.g., |exponent - 0.5| / 0.5) so it does not dominate.

💡 One-Sentence Summary

The Java Objective Evaluator is a stateless batch job that reads a parameter JSON, runs one full-day simulation with a fixed seed, computes K-S distances against pre-loaded historical distributions, prints a single weighted objective to stdout, and exits.
