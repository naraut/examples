Core Conclusion

An ABIDES-alternative for HKEX simulation should be a lightweight, single-process, discrete-event simulation that focuses on market impact assessment through synthetic agents, while reusing the matching engine and session state machine from the MBO replay simulator. The key design trade-off is simplicity over ABIDES' full agent-based generality.

---

1. Scope and Positioning

What This Alternative Is

A simplified agent-based simulation (ABM) that generates endogenous order flow to measure how your algo's orders affect market prices and liquidity. Unlike ABIDES, which aims for high-fidelity multi-agent research with configurable network latencies and NASDAQ-like protocols , this design prioritizes HKEX-specific rule fidelity and practical impact assessment over research generality.

What This Alternative Is Not

· Not a full ABIDES replacement with 300+ agents and configurable network topology
· Not a multi-asset, multi-venue simulation
· Not designed for reinforcement learning training at scale

Why an Alternative Is Needed

ABIDES is Python-based, single-threaded, and designed for AI research rather than production rule simulation . It lacks HKEX session semantics, auction rules, and VCM. Adapting ABIDES to HKEX would require rewriting its exchange agent and session logic. A purpose-built alternative can share the matching engine with the MBO replay simulator, ensuring rule consistency between the two modes.

---

2. Architecture

2.1 Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    SIMULATION KERNEL                        │
│              (Discrete Event Scheduler)                     │
│                                                             │
│  - Priority queue of events (timestamp-ordered)             │
│  - Advances global virtual time (GVT)                       │
│  - Dispatches events to agents and exchange                 │
│  - Single-threaded, deterministic with seeded RNG           │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  SYNTHETIC    │    │   EXCHANGE    │    │   ALGO UNDER  │
│  AGENTS       │    │   AGENT       │    │   TEST        │
│               │    │               │    │               │
│ - MM Agent    │    │ - Order Book  │    │ - Your Strategy│
│ - Noise Agent │    │ - Matching    │    │ - Order Entry │
│ - Momentum    │    │ - Session Mgr │    │ - Fill Handler│
│ - Value Agent │    │ - VCM         │    │               │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                    ┌───────────────────┐
                    │  EVENT BUS        │
                    │  (In-Memory Queue)│
                    └───────────────────┘
```

2.2 Key Differences from ABIDES

Aspect ABIDES This Alternative
Language Python Java (shared with matching engine)
Network latency Configurable pairwise latencies  Simplified: uniform or per-agent-type latency
Agent count Up to tens of thousands  Hundreds (sufficient for impact assessment)
Exchange rules NASDAQ-like (ITCH/OUCH)  HKEX-specific (session phases, IEP, VCM)
Matching Price-time priority Shared with MBO replay simulator
Purpose AI research, RL training Market impact assessment for execution algos

---

3. Synthetic Agent Types

3.1 Agent Taxonomy

Based on ABIDES' agent framework , implement the following agent types:

Market Maker Agent

· Places a ladder of buy and sell limit orders around a reference price
· Reference price derived from an Ornstein-Uhlenbeck process (fundamental value)
· Half-spread and depth are configurable parameters
· Reacts to inventory changes by skewing quotes

Noise Trader Agent

· Arrives via Poisson process with rate λ
· Submits random limit or market orders
· Uses "zero intelligence" (ZI) strategy: random side, price depth, and volume
· Extended ZI: updates beliefs based on transaction prices

Momentum Trader Agent

· Observes recent price trends
· Buys when price is rising, sells when falling
· Arrival rate calibrated to produce realistic autocorrelation in order flow

Value Trader Agent

· Maintains an estimate of fundamental value
· Submits limit orders at prices reflecting perceived mispricing
· Updates fundamental estimate from order flow (reflexive mechanism) 

3.2 Agent Parameters (Calibratable)

```java
public class AgentConfig {
    // Arrival
    double arrivalRate;           // Poisson lambda
    double computationDelayNanos; // processing time per decision
    
    // Order placement
    double orderSizeMean;         // average quantity
    double orderSizeStdDev;
    double limitOrderProbability; // vs market order
    
    // Pricing
    double halfSpread;            // for MM agents
    double priceImprovementProb;  // probability of crossing spread
    
    // Fundamental value (OU process)
    double fundamentalMean;
    double fundamentalVolatility;
    double meanReversionSpeed;
}
```

---

4. Market Impact Mechanism

4.1 Why Synthetic Agents Produce Impact

In MBO replay, your algo's orders do not affect historical flow. In this alternative, your algo's orders consume liquidity that synthetic agents would otherwise have provided. When your algo buys aggressively, it clears the bid side, and:

· Market maker agents observe inventory depletion and widen spreads
· Momentum agents observe price movement and join the trend
· Value agents observe deviation from fundamental and provide counter-liquidity

This endogenous reaction is what produces measurable market impact.

4.2 Impact Measurement

Run two simulations with identical seeds and agent configurations:

· Baseline: Algo does not trade
· Treatment: Algo executes its strategy

The difference in mid-price trajectory, spread, and depth between the two runs is the market impact:

```java
public class ImpactAnalyzer {
    public ImpactResult measure(List<PricePoint> baseline, List<PricePoint> treatment) {
        double permanentImpact = treatment.last().price - baseline.last().price;
        double temporaryImpact = average(treatment.midPrices - baseline.midPrices);
        double spreadWidening = average(treatment.spreads) - average(baseline.spreads);
        double depthReduction = average(baseline.depths) - average(treatment.depths);
        
        return new ImpactResult(permanentImpact, temporaryImpact, 
                                spreadWidening, depthReduction);
    }
}
```

4.3 Calibration from Historical Data

Agent parameters are calibrated so that the synthetic market reproduces historical stylized facts :

Stylized Fact Calibration Target
Bid-ask spread distribution Match historical spread percentiles
Order arrival rate Match historical message rate
Trade size distribution Match historical trade size histogram
Price return autocorrelation Match historical ACF
Volume-volatility correlation Match historical relationship

---

5. Discrete Event Kernel

5.1 Event Types

```java
public enum SimEventType {
    AGENT_WAKEUP,        // Agent scheduled to make a decision
    ORDER_SUBMIT,        // Order sent to exchange
    ORDER_ACK,           // Exchange confirms receipt
    ORDER_REJECT,        // Exchange rejects order
    TRADE_EXECUTE,       // Fill notification to agent
    MARKET_DATA_UPDATE,  // Book update to agents
    SESSION_CHANGE,      // Trading session transition
    VCM_TRIGGER          // Volatility control event
}
```

5.2 Kernel Loop

```java
public class SimulationKernel {
    private final PriorityQueue<SimEvent> eventQueue;
    private long globalVirtualTime;
    private final Random rng;
    
    public void run() {
        while (!eventQueue.isEmpty()) {
            SimEvent event = eventQueue.poll();
            globalVirtualTime = event.timestamp;
            
            switch (event.type) {
                case AGENT_WAKEUP:
                    Agent agent = (Agent) event.target;
                    List<Order> orders = agent.decide(marketState);
                    for (Order o : orders) {
                        scheduleOrderSubmission(o, agent);
                    }
                    break;
                case ORDER_SUBMIT:
                    exchange.processOrder(event.order, event.source);
                    break;
                case TRADE_EXECUTE:
                    event.agent.onFill(event.trade);
                    break;
            }
        }
    }
}
```

5.3 Determinism Guarantee

Following ABIDES' approach :

· Each agent has its own PRNG seeded from a master seed
· Agent A's random consumption does not affect Agent B's random stream
· This enables A/B testing where only the treatment agent's behavior changes

```java
public class Agent {
    private final Random agentRng;
    
    public Agent(long masterSeed, int agentId) {
        this.agentRng = new Random(masterSeed + agentId * 31);
    }
}
```

---

6. Exchange Agent

6.1 Reuse from Matching Engine

The exchange agent wraps the same matching engine used in MBO replay mode:

· Session state machine
· Order books per instrument
· IEP calculation for auctions
· VCM logic
· Price validation pipeline

The only difference: instead of consuming historical MBO events, it consumes orders from synthetic agents.

6.2 Market Data Dissemination

After each matching cycle, the exchange publishes:

· Book snapshot (top N levels)
· Last trade price
· Session status
· IEP/IEV during auctions

Agents receive this market data and use it in their next decision.

---

7. Algo Under Test Integration

7.1 Agent Interface

Your algo implements the same Agent interface as synthetic agents:

```java
public interface Agent {
    void onMarketData(MarketDataUpdate update);
    void onFill(Trade trade);
    List<Order> decide(MarketState state);
    void onWakeup();
}
```

7.2 Latency Modeling

Each agent has a configurable computation delay (time to process market data and generate orders) and network latency (time for orders to reach the exchange). Your algo's latency can be set independently of synthetic agents.

---

8. Session-Aware Behavior

8.1 Auction Sessions

During Pre-opening and CAS:

· Synthetic agents submit At-Auction and At-Auction Limit orders
· No continuous matching occurs
· IEP is calculated continuously
· At session end, auction matching executes

8.2 VCM

When VCM triggers:

· Exchange enters cooling-off period
· All agents receive VCM status update
· Trading restricted to pre-defined band
· Agents adjust their behavior accordingly

---

9. Validation Strategy

9.1 Stylized Facts Check

After each simulation run, verify that the synthetic market reproduces:

· Realistic bid-ask spread
· Realistic order arrival rate
· Heavy-tailed return distribution
· Volatility clustering

9.2 Impact Curve Validation

Compare the simulated impact curve (impact vs. order size) against:

· Historical impact curves from your own execution data
· Published square-root law relationships 

9.3 Cross-Validation with MBO Replay

Run the same algo strategy through:

1. MBO replay (no impact)
2. Agent-based simulation (with impact)

The difference quantifies the impact cost that MBO replay misses.

---

10. Implementation Roadmap

Phase 1: Kernel + Basic Agents

· Discrete event kernel
· Market maker and noise trader agents
· Exchange agent wrapping existing matching engine
· Validate stylized facts on a single instrument

Phase 2: Algo Integration

· Agent interface for your algo
· Latency modeling
· Session-aware behavior

Phase 3: Impact Measurement

· Baseline vs. treatment simulation runner
· Impact metrics and reporting
· Calibration against historical impact data

Phase 4: Advanced Agents

· Momentum and value traders
· Agent parameter optimization
· Multi-instrument support

---

11. Summary

This ABIDES-alternative provides a pragmatic path to market impact assessment for HKEX. It reuses the matching engine from the MBO replay simulator, ensuring rule consistency, while adding synthetic agents that react endogenously to your algo's orders. The design prioritizes HKEX rule fidelity over ABIDES' research generality, and is implemented in Java to share code with the existing matching engine.

The key insight is that impact measurement requires two simulations (baseline and treatment) with identical seeds, differing only in whether your algo trades. The difference in market trajectories quantifies the impact that MBO replay cannot capture.
