## What is the RL in my project? – A short, direct answer

My RL agent is a **replacement for the fixed, hand‑written congestion control rules** (like Reno’s AIMD or CUBIC’s cubic function). Instead of telling the sender *“on loss, cut your window in half,”* you let the agent learn its own rule from experience.

---

## 1. What does the network (the agent) see?

At every decision moment (once per RTT), the agent looks at three numbers:

| State variable | What it measures | Units |
| --- | --- | --- |
| **cwnd** | How many packets are currently allowed to be in flight. | packets |
| **sRTT** | Smoothed round‑trip time – includes queueing delay. | milliseconds |
| **delivery rate** | Fraction of packets successfully delivered in the last RTT. | 0 to 1 |

These three numbers are the **state**. They tell the agent *everything it needs to know* about the current network condition.

---

## 2. How does the agent know if it did well? (The reward)

After the agent picks an action and the network reacts, it receives a **reward** – a single number that scores the outcome. I defined the reward as:

$$\text{reward} = (1.0 \times \text{delivery\_rate}) - \left(0.5 \times \frac{\text{sRTT}}{100}\right) - (1.0 \times (1 - \text{delivery\_rate}))$$

This is a simple formula that balances three goals:

* **High delivery rate** → good (positive)
* **Low RTT** → good (negative penalty for RTT)
* **Packet loss** (which is $1 - \text{delivery\_rate}$) → bad (negative penalty)

There are **no fixed thresholds** like “RTT must be below 50 ms.” Instead, the agent learns to **maximise the cumulative reward over time** – it figures out by itself what RTT and delivery rate lead to the highest score.

---

## 3. What is the “optimal” point?

The theoretical best performance would be:

* **delivery rate = 1.0** (no loss)
* **sRTT = base propagation delay + tiny queue** – in my setup, the propagation delay is 20 ms, so an RTT around 25 ms would be excellent.
* **cwnd** such that the bottleneck is kept busy but the queue never overflows – roughly the bandwidth‑delay product (~18 packets in my network).

That would give a reward around:

$$1.0 - \left(0.5 \times \frac{25}{100}\right) - (1.0 \times 0) = 1.0 - 0.125 = 0.875$$

If the agent causes a queue to fill up (RTT climbs to 80 ms) and starts losing packets (delivery rate 0.98), the reward becomes:

$$0.98 - \left(0.5 \times \frac{80}{100}\right) - (1.0 \times 0.02) = 0.98 - 0.4 - 0.02 = 0.56$$

The agent learns to avoid states that give lower rewards and to move towards the ones that give higher rewards.

---

## 4. How does the RL find the optimal behaviour without thresholds?

It learns **through trial and error**, not by following a rule like “if $RTT > X$, slow down.”

1. The agent starts with a random policy (random actions).
2. It runs many simulations (episodes), each time taking actions, observing the next state, and receiving a reward.
3. Using the PPO algorithm, it gradually updates its neural network to make actions that lead to higher long‑term rewards more likely.
4. Over thousands of steps, the policy converges: it learns to increase the window when the queue is empty, back off gently when RTT starts rising, and avoid pushing the queue to overflow.

The result is a **learned mapping from the three numbers (cwnd, RTT, delivery rate) to one of the five discrete actions** – no human‑set thresholds anywhere.

---

## Summary

> “I replaced the standard TCP congestion control with an RL agent that observes cwnd, RTT, and packet delivery rate. It receives a reward that combines throughput (delivery rate) with penalties for delay and loss. The agent is trained offline in ns‑3 using PPO, and it learns to maximise that reward over time. It finds by itself the optimal balance between sending fast and not overloading the queue – no fixed thresholds, just learning.”