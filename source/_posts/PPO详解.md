---
title: PPO详解
date: 2024-06-24 17:14:14
tags: 深度强化学习
categories: 强化学习
mathjax: True
---


## 1. PPO算法简介

Proximal Policy Optimization (PPO) 是一种广泛使用的强化学习算法，旨在通过限制策略更新的幅度来提高训练的稳定性和效率。PPO结合了策略梯度方法和信赖域策略优化 (Trust Region Policy Optimization, TRPO) 的优势，提出了一种简单但有效的策略更新方法。

### PPO的核心思想

PPO通过引入“剪辑”机制来限制策略更新的幅度，从而避免策略在每次更新时发生过大变动。其目标函数为：

$$ L^{\text{CLIP}}(\theta) = \mathbb{E}_t \left[ \min \left( r_t(\theta) \hat{A}_t, \text{clip}(r_t(\theta), 1 - \epsilon, 1 + \epsilon) \hat{A}_t \right) \right] $$

其中：
- $r_t(\theta) = \frac{\pi_{\theta}(a_t|s_t)}{\pi_{\theta_{\text{old}}}(a_t|s_t)}$ 是策略比率。
- $\hat{A}_t$ 是优势函数。
- $\text{clip}(r_t(\theta), 1 - \epsilon, 1 + \epsilon)$ 将策略比率剪辑到 $[1 - \epsilon, 1 + \epsilon]$ 范围内。

通过这种剪辑机制，PPO有效平衡了策略更新的效率与稳定性，使其成为强化学习中的一种高效实用的策略优化算法。

---

## 2. 优势函数计算

在强化学习中，优势函数（Advantage Function）用于衡量某个动作相对于平均水平的优越程度。计算优势函数的方法有多种，常用的方法之一是基于时间差分（Temporal Difference，TD）误差的方法。

### 优势函数的定义

优势函数 $A(s_t, a_t)$ 表示在状态 $s_t$ 下采取动作 $a_t$ 比平均水平好多少：

$$ A(s_t, a_t) = Q(s_t, a_t) - V(s_t) $$

其中：
- $Q(s_t, a_t)$ 是状态-动作值函数，表示在状态 $s_t$ 下采取动作 $a_t$ 后的预期回报。
- $V(s_t)$ 是状态值函数，表示在状态 $s_t$ 下的预期回报。

### 优势函数的近似计算

常用的两种优势函数计算方法如下：

#### 1. TD(0) Advantage

使用一步TD误差来估计优势函数：

$$ A_t = r_t + \gamma V(s_{t+1}) - V(s_t) $$

#### 2. Generalized Advantage Estimation (GAE)

GAE通过对多步TD误差进行加权平均，得到更加平滑和准确的优势函数估计：

$$ A_t = \sum_{l=0}^{\infty} (\gamma \lambda)^l \delta_{t+l} $$

其中：
- $\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)$ 是一步TD误差。
- $\lambda$ 是一个平滑参数，控制多步TD误差的权重，通常在0到1之间。

### 计算GAE的代码示例

```python
def compute_gae(rewards, values, gamma=0.99, lambda_=0.95):
    advantages = np.zeros_like(rewards)
    last_gae_lam = 0
    for t in reversed(range(len(rewards))):
        if t == len(rewards) - 1:
            next_value = 0  # Value of terminal state is 0
        else:
            next_value = values[t + 1]
        delta = rewards[t] + gamma * next_value - values[t]
        advantages[t] = last_gae_lam = delta + gamma * lambda_ * last_gae_lam
    return advantages

def compute_returns(rewards, gamma=0.99):
    returns = np.zeros_like(rewards)
    G = 0
    for t in reversed(range(len(rewards))):
        G = rewards[t] + gamma * G
        returns[t] = G
    return returns
```

---

## 3. PPO算法伪代码

以下是PPO算法的伪代码，展示了PPO的主要步骤，包括数据收集、优势函数计算和策略更新。

### PPO 伪代码

```python
# PPO Algorithm Pseudocode

initialize policy parameters θ
initialize value function parameters φ

for iteration = 1, 2, ..., N_iter do
    # Step 1: Data Collection
    trajectories = []  # list to store trajectories
    for actor = 1, 2, ..., N_actors do
        trajectory = []
        state = env.reset()
        for t = 1, 2, ..., T do
            action = policy.sample_action(state, θ)
            next_state, reward, done, _ = env.step(action)
            trajectory.append((state, action, reward, next_state, done))
            state = next_state
            if done:
                break
        trajectories.append(trajectory)
    
    # Step 2: Compute Advantages and Returns
    advantages = []
    returns = []
    for trajectory in trajectories do
        rewards = [step[2] for step in trajectory]
        values = [value_function(state, φ) for (state, _, _, _, _) in trajectory]
        advantages.append(compute_gae(rewards, values))
        returns.append(compute_returns(rewards, γ))
    
    # Flatten the trajectories and corresponding advantages and returns
    states, actions, rewards, next_states, dones = flatten(trajectories)
    advantages = flatten(advantages)
    returns = flatten(returns)
    
    # Step 3: Update Policy using PPO Objective
    old_log_probs = policy.get_log_probs(states, actions, θ)
    
    for epoch = 1, 2, ..., K do
        for batch in mini_batches(states, actions, returns, advantages, old_log_probs) do
            states_batch, actions_batch, returns_batch, advantages_batch, old_log_probs_batch = batch
            log_probs = policy.get_log_probs(states_batch, actions_batch, θ)
            ratios = torch.exp(log_probs - old_log_probs_batch)
            surr1 = ratios * advantages_batch
            surr2 = torch.clip(ratios, 1 - ε, 1 + ε) * advantages_batch
            policy_loss = -torch.mean(torch.min(surr1, surr2))
            
            # Gradient descent step on policy loss
            θ = θ - learning_rate * ∇_θ policy_loss
    
    # Step 4: Update Value Function
    for epoch = 1, 2, ..., K do
        for batch in mini_batches(states, returns) do
            states_batch, returns_batch = batch
            value_loss = torch.mean((returns_batch - value_function(states_batch, φ))^2)
            
            # Gradient descent step on value loss
            φ = φ - learning_rate * ∇_φ value_loss
```

### 伪代码说明

1. **初始化：** 初始化策略参数 $\theta$ 和价值函数参数 $\phi$。

2. **数据收集：**
   - 使用多个actor并行地从环境中收集数据，得到多个轨迹。
   - 每条轨迹包含状态、动作、奖励、下一个状态和done标志（表示episode是否结束）。

3. **计算优势函数和回报：**
   - 对每条轨迹，计算优势函数（例如使用GAE）和累积回报（Returns）。
   - 将所有轨迹的数据展开为单个列表，以便后续的批量处理。

4. **更新策略：**
   - 计算旧策略的对数概率。
   - 对每个epoch，使用小批量梯度下降更新策略参数。
   - 在每个批次中，计算新策略的对数概率和比率，应用PPO的剪辑目标函数更新策略。

5. **更新价值函数：**
   - 对每个epoch，使用小批量梯度下降更新价值函数参数。
   - 计算价值损失（均方误差），并更新价值函数参数。

---

## 4. 关键函数实现

### 计算GAE的函数

```python
def compute_gae(rewards, values, gamma=0.99, lambda_=0.95):
    advantages = np.zeros_like(rewards)
    last_gae_lam = 0
    for t in reversed(range(len(rewards))):
        if t == len(rewards) - 1:
            next_value = 0  # Value of terminal state is 0
        else:
            next_value = values[t + 1]
        delta = rewards[t] + gamma * next_value - values[t]
        advantages[t] = last_gae_lam = delta + gamma * lambda_ * last_gae_lam
    return advantages

def compute_returns(rewards, gamma=0.99):
    returns = np.zeros_like(rewards)
    G = 0
   

 for t in reversed(range(len(rewards))):
        G = rewards[t] + gamma * G
        returns[t] = G
    return returns
```

### 批处理函数

```python
def mini_batches(*args, batch_size=64):
    data_length = len(args[0])
    indices = np.arange(data_length)
    np.random.shuffle(indices)
    for start_idx in range(0, data_length, batch_size):
        excerpt = indices[start_idx:start_idx + batch_size]
        yield (arg[excerpt] for arg in args)

def flatten(trajectories):
    states = []
    actions = []
    rewards = []
    next_states = []
    dones = []
    for trajectory in trajectories:
        for step in trajectory:
            states.append(step[0])
            actions.append(step[1])
            rewards.append(step[2])
            next_states.append(step[3])
            dones.append(step[4])
    return states, actions, rewards, next_states, dones
```

---

### 总结

PPO是强化学习中的一种高效实用的策略优化算法，通过限制策略更新的幅度，提高了训练的稳定性和效率。本文详细介绍了PPO算法的核心思想、优势函数的计算方法，并提供了完整的伪代码和关键函数的实现。通过这些内容，读者可以深入理解PPO的工作原理，并应用于实际的强化学习任务中。