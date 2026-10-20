ABIDES employs a Discrete Event Multi-Agent Simulation (DEMAS) architecture, where the core mechanism is: multiple agents interact with a central exchange agent by sending messages, and the simulation kernel advances simulation time through an event priority queue, rather than fixed time steps .

🏗️ Core Architecture: Discrete Event Kernel

The "heart" of ABIDES is an event kernel that maintains a priority queue sorted by time. It does not advance time in fixed intervals (such as every second); instead, time "jumps" directly to the moment when the next event occurs .

When an agent needs to act, the kernel schedules a WAKEUP event for it. Once the simulation time reaches that moment, the kernel invokes the agent's wakeup method. The agent can then:

· Read market data (such as the latest mid-price, order book depth)
· Decide whether to place an order, cancel an order, or do nothing
· Send an OrderMessage or CancelMessage to the exchange agent

All messages are routed through the kernel, and agents are not allowed to directly call each other's methods—they only know each other's ID and can only communicate through messages .

👥 Agent Composition: Background Population and Experimental Agents

ABIDES is designed to create a realistic "background market" environment for researchers to inject experimental agents for testing .

Background Agents (used to construct realistic market dynamics) typically include :

· Market Maker: Provides liquidity and maintains quotes on both sides
· Noise Agent: Submits random orders, representing uninformed traders
· Value Agent: Has its own estimate of fundamental value and trades when price deviates
· Momentum Trader: Follows price trends

The quantity of these agents is configurable. For example, one study used 300 noise agents, 100 value agents, and 25 momentum traders to produce a U-shaped intraday volume pattern .

Experimental Agents: This is the core of your strategy testing. You need to inherit the base Agent class and implement at least :

· wakeup(currentTime): Called when the agent "wakes up," where it makes trading decisions and sends orders
· receiveMessage(currentTime, msg): Handles market data and execution reports sent back from the exchange

🔄 Interaction Flow (Simplified)

1. Initialization: The kernel registers all agents and schedules their first WAKEUP event.
2. Wakeup: The kernel pops the earliest event from the priority queue and invokes the corresponding agent's wakeup method.
3. Order Dispatch: The agent generates an order message and hands it to the kernel. The kernel delivers it to the exchange agent based on the configured latency model (which can vary per agent) .
4. Exchange Processing: The exchange agent receives the order, performs matching (price-time priority), and generates an execution report or market data update.
5. Feedback: The exchange agent sends the results back through the kernel to the relevant agents, and simultaneously broadcasts market data to subscribed agents.
6. Loop: The kernel continues to process the next event until the simulation ends.

💡 Implications for You

If you want to implement a similar LLM execution framework in your HKEX simulator, the key is to learn from ABIDES's message-driven and event scheduling design:

· Your LLM agents (Planner/Executor) don't need to be invoked on every tick; they can be scheduled as "wakeup" events at a slower frequency (such as every 5 seconds).
· Between two wakeups, the matching engine processes all market data and orders at full speed (microsecond scale).
· The LLM agent's orders are injected into the matching engine as "messages," and execution reports are fed back to the agent as "messages" when it wakes up next.

This way, slow LLM reasoning and fast matching engine execution are decoupled through the event queue, just as ABIDES handles slow background agents and fast matching.
