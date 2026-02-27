# Technical Deep-Dive: ACONS-PettingZoo Integration
This document provides a granular, script-by-script explanation of the project architecture, the mathematical models used for communication, and the reinforcement learning training pipeline.
## 1. Project Objective
The primary goal is to provide a standardized interface for **Multi-Agent Reinforcement Learning (MARL)** that incorporates realistic communication constraints. This allows researchers to train agents that are robust to real-world networking issues such as narrow bandwidth, signal latency, and packet drops.
---
## 2. Component-Level Breakdown
### 2.1. Network Layer: `proto/acons_sim.proto`
This file defines the gRPC (Google Remote Procedure Call) interface. It is the "contract" between the Python-based AI and the C++ based Simulator.
- **`Reset`**: Reinitializes the simulation with a specific seed and scenario.
- **`Step`**: The core loop. Sends actions for all agents and receives the subsequent state, rewards, and "done" flags.
- **`CommRecv`**: Defines how messages are received by agents, including an `age_steps` field to track latency.
### 2.2. Environment Layer: `src/pettingzoo_env/acons_env.py`
This script wraps the gRPC client into a **PettingZoo AEC (Agent-Environment Cycle)** environment.
- **Action Buffering**: Since PettingZoo processes agents one by one but the simulator expects a "joint action" for all agents, this script buffers individual actions until everyone has moved.
- **Observation Merging**: It merges the physical sensor data from the simulator with the filtered communication messages from the `CommEmulator`.
- **`observation_space`**: Defines exactly what the AI can "see" (a 64-element feature vector plus a communication buffer).
### 2.3. Communication Layer: `src/pettingzoo_env/comm_emulator.py`
This is a standalone module that simulates the physics of the network. It implements three primary constraints:
- **Bandwidth ($B$)**: 
  - **Mechanism**: Sorts all out-bound messages and drops those that exceed the bit-budget for the current step.
  - **Code**: `_enforce_bandwidth(src_id, sends)`
- **Latency ($L$)**:
  - **Mechanism**: Instead of delivering messages immediately, it places them in a `delivery_queue` keyed by the future timestep $t + L$.
  - **Code**: `_schedule_delivery(src_id, send)`
- **Packet Loss ($p_{loss}$)**:
  - **Mechanism**: A stochastic gate that uses a Random Number Generator (RNG) to determine if a packet is destroyed.
  - **Code**: `_apply_loss(send)`
### 2.4. Reward Layer: `src/reward/extractor.py` & `config.yaml`
Translates high-level mission events into raw numerical data.
- **Event Mapping**: Uses a YAML config to assign weights (e.g., `mine_neutralized: +1.0`).
- **Terminal Rewards**: Handles large bonuses/penalties for mission success or failure.
### 2.5. Training Layer: `src/training/runner.py`
Demonstrates a full training loop using a dummy policy.
- **Reward Shaping**: Implements **Execution-Consistent Potential-Based Shaping**.
  - **Formula**: $F_i = \gamma V(s') - V(s)$
  - **Why?**: This helps the agent learn in "sparse" reward environments without changing the optimal behavior.
- **AEC Loop**: Iterates through agents, fetches actions from the policy, and steps the environment.
---
## 3. The Communication Pipeline
1. **Agent Step**: Agent $i$ generates a physical action $a_i$ and a message $m_{ij}$ for agent $j$.
2. **Bandwidth Filter**: `CommEmulator` checks if $m_{ij}$ fits in the budget $B$.
3. **Loss/Latency Filter**: `CommEmulator` rolls for loss ($p$) and schedules delivery at $t+L$.
4. **Simulator Step**: Actions are sent to ACONS via gRPC. ACONS returns physical state $s_{t+1}$.
5. **Observation Assembly**: `AconsEnv` combines $s_{t+1}$ with any messages scheduled for delivery at time $t$.
---
## 4. Testing and Quality Assurance
### `tests/unit/test_comm_emulator.py`
This is a critical script that verifies:
- **Deterministic loss**: Ensures that with a fixed seed, the loss is reproducible.
- **Queuing**: Confirms that messages sent at $t=0$ with $L=2$ actually arrive at $t=2$.
- **Bandwidth Truncation**: Ensures that if the budget is 10 bits, an 11-bit message is rejected.
---
## 5. Technical Requirements
- **Python 3.8+**
- **gRPC & Protobuf**: For network communication.
- **Gymnasium/PettingZoo**: For RL standard interfaces.
- **PyTorch/NumPy**: For mathematical operations and policy logic.
---
## 6. Project Roadmap
1. [x] Core gRPC Integration.
2. [x] PettingZoo AEC Wrapper.
3. [x] Comm Simulator ($B, L, p$).
4. [/] Advanced Multi-Agent Policy (MAPPO/QMIX) Integration.
5. [ ] Real ACONS Simulator Connection (currently using Mock Server).
