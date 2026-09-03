
---

## 1. What my project does in one sentence

I train one shared RL agent to control **two TCP flows at the same time** so they share a 10 Mbps bottleneck fairly, without me writing any fixed rules.

---

## 2. Where each RL part lives

| Part | Where it is defined | What it does |
|------|----------------------|--------------|
| **State / observation** | C++ file `marl-multi-env.cc`, function `GetObservation()` | Collects 4 numbers per flow: `cwnd`, smoothed RTT, loss, throughput. Python receives a vector of 8 numbers for 2 flows. |
| **Action** | C++ file `marl-multi-env.cc`, `ExecuteActions()` | Takes one number per flow (0–4), maps it to a multiplier: 0 = keep, 1 = +10%, 2 = −10%, 3 = +20%, 4 = −20%. |
| **Reward** | C++ file `marl-multi-env.cc`, `GetReward()` | Combines throughput, fairness, delay penalty, and loss penalty. |
| **Training loop** | Python file `train_marl_multi.py` | Uses Stable‑Baselines3 PPO to update the neural network. |
| **Environment wrapper** | Python file `marl_multi_env.py` | Normalises observations and rounds actions before sending to ns‑3. |
| **Evaluation** | Python file `eval_marl_multi.py` | Loads trained model, runs one episode, parses FlowMonitor, computes fairness. |

The trained model is saved as `ppo_marl_multi.zip`.

---

## 3. My current reward function (MARL)

For each decision step:

```
reward = 20 × (total throughput Mbps / 10)
         + 20 × Jain fairness
         − average delay penalty
         − loss penalty
```

In plain words:

- **Throughput reward:** up to 20 points if both flows together use the full 10 Mbps.
- **Fairness reward:** up to 20 points if the two flows share equally. This uses Jain’s Fairness Index, where 1.0 = perfectly equal.
- **Delay penalty:** subtracts points when RTT grows above the minimum RTT.
- **Loss penalty:** subtracts points for packet loss.

The reward is per step. The training console shows the **total reward over a 60‑second episode**, which is around **19,700**, meaning about 33 points per step.

---

## 4. Why my reward function changed during the project

I tried three versions before the current one:

### M3 Reward v1 — Decaying‑max normalisation

I wanted rewards to adapt to the network, so I tracked:
- the highest throughput seen so far
- the lowest and highest RTT seen so far

The throughput reward was `current throughput / max throughput`, and the delay penalty was based on where current RTT sat between min and max. I also used weights α=1.0, β=0.5, γ=2.0 to balance throughput, delay, and loss.

**Why it failed:** The throughput observation was broken. The agent saw throughput as zero most of the time, so it never learned to increase cwnd.

### M3 Reward v2 — Direct throughput in Mbps

I made reward simpler:
- reward = raw throughput in Mbps
- mild delay and loss penalties

**Why it still failed:** The action code was broken. Instead of multiplying cwnd by 10%, it set cwnd to a fixed absolute value. So cwnd could never grow.

### M3 Reward v3 — CWND target / BDP

I ignored the broken throughput and rewarded the agent for keeping cwnd near the Bandwidth‑Delay Product (25 KB for my link).

**Training reward became positive**, but real throughput stayed low because the action bug was still present.

### M4 — Fix the real bugs

I fixed:
1. Throughput measurement → now reads delivered bytes from `PacketSink`.
2. Action application → now multiplies current cwnd.

After these fixes, a single RL flow achieved **8565 kbps** (85.65% of the bottleneck).

Then for MARL, I added the fairness term, and the final reward became the one above.

---

## 5. Frameworks and tools, simply explained

| Tool / Abbreviation | Full meaning | What it does |
|----------------------|--------------|--------------|
| **ns‑3** | Network Simulator 3 | Simulates TCP/IP, routers, links, queues. My whole network runs here. |
| **ns3‑gym** | ns‑3 + OpenAI Gym bridge | Lets Python talk to ns‑3 using ZMQ messages. |
| **ZMQ** | ZeroMQ | A fast messaging library that connects Python and C++. |
| **Protobuf** | Protocol Buffers | A format for serialising structured data between the two programs. |
| **Gymnasium** | Gymnasium (OpenAI Gym fork) | Standard RL environment API: `reset()`, `step(action)`, `observation_space`. |
| **Stable‑Baselines3 (SB3)** | Stable Baselines 3 | RL algorithm library built on PyTorch. I use its PPO implementation. |
| **PPO** | Proximal Policy Optimization | A popular, stable RL algorithm that updates the policy in small steps. |
| **PyTorch** | PyTorch | Deep learning framework. SB3 uses it under the hood for neural networks. |
| **RL** | Reinforcement Learning | Learning by trial and error: agent takes actions, gets rewards. |
| **MARL** | Multi‑Agent Reinforcement Learning | Multiple agents learning in the same environment. |
| **CTDE** | Centralised Training, Decentralised Execution | One policy is trained with global info (both flows), but can act independently per flow later. |
| **TCP** | Transmission Control Protocol | Reliable transport protocol I’m controlling. |
| **cwnd** | Congestion Window | Maximum bytes a sender can have unacknowledged. This is what the agent adjusts. |
| **RTT** | Round‑Trip Time | Time for a packet to go from sender to receiver and back. |
| **BDP** | Bandwidth‑Delay Product | Ideal amount of data in flight = bandwidth × RTT. |
| **AIMD** | Additive Increase Multiplicative Decrease | Classic TCP rule: slowly increase, sharply decrease on loss. |
| **CUBIC** | CUBIC TCP | A Linux default TCP algorithm using a cubic window growth curve. |
| **BBR** | Bottleneck Bandwidth and Round‑trip propagation time | Model‑based TCP that paces packets, instead of reacting only to loss. |
| **Jain Fairness Index** | Jain’s Fairness Index | Formula measuring how equally throughput is shared. 1.0 = perfectly fair. |

---

## 6. Final results in one table

| Metric | MARL Agent 1 | MARL Agent 2 |
|--------|--------------|--------------|
| Throughput | 5112.1 kbps | 4824.8 kbps |
| Delay | 60.96 ms | 60.80 ms |
| Loss | 0.04% | 0.05% |
| **Jain Fairness** | **0.9992** | |
| **Total throughput** | **9936.9 kbps** (99.37% of 10 Mbps) | |

This shows the two RL flows learned to use the entire bottleneck while sharing almost perfectly fairly.
