---
title: 强化学习算法1：REINFORCE
date: 2022-10-29 16:06:10
tags: 强化学习算法系列
categories: 强化学习
mathjax: true
---

&emsp;&emsp;策略梯度（Policy Gradient）方法是强化学习方法中的一个大类，通过期望回报并以梯度上升的方式直接优化策略函数，避免了传统强化学习中的一些困难，无需准确的价值函数，也可以处理连续动作问题，应用范围很广。常见的策略梯度方法包括REINFORCE、Actor-Critic(AC)、TRPO以及著名的PPO方法，本篇主要从最简单的REINFORCE方法开始，逐步理解策略梯度方法的思想。

# 一、主要内容
&emsp;&emsp;策略梯度方法是直接优化策略$\pi_\theta$，其中$\theta$为参数，那么目标便是求$\theta^*=\mathop{\arg\max}_\theta \mathbb{E}_\tau[R(\tau)]$，轨迹$\tau\sim\pi_\theta$，$R(\tau)$表示为轨迹$\tau$的累积期望汇报。 我们可以采用梯度上升方法进行优化：
$$\theta=\theta+\eta\nabla_\theta \mathbb{E}(R(\tau))$$

&emsp;&emsp;从上式可知，问题转化为如何求解期望回报对于参数$\theta$的梯度， 可将关于策略$\pi_\theta$的期望回报表示为：
$$J(\theta)=\mathbb{E}_\tau[R(\tau)] = \sum_\tau R(\tau)p_\theta(\tau)$$
根据期望的定义，可以进一步拆开，用$p_\theta(\tau)$表示轨迹$\tau$出现的概率，轨迹的出现与策略$\pi_\theta$相关，所以也应当是关于$\theta$的函数。于是可以计算梯度：
$$\begin{align}
\nabla_\theta J(\theta)
&=\nabla_\theta\sum_\tau p_\theta(\tau)R(\tau)\\
&=\sum_\tau\nabla_\theta p_\theta(\tau)R(\tau)\\
&=\sum_\tau\frac{p_\theta(\tau)}{p_\theta(\tau)}\nabla_\theta p_\theta(\tau)R(\tau)\\
&=\sum_\tau p_\theta(\tau)\frac{\nabla_\theta p_\theta(\tau)}{p_\theta(\tau)}R(\tau)\\
&=\sum_\tau p_\theta(\tau)\nabla_\theta \log p_\theta(\tau)R(\tau)\\
&=\mathbb{E}_\tau[\nabla_\theta\log p_\theta(\tau)R(\tau)]
\end{align}$$
我们可以利用当前的策略$\pi_\theta$采样$m$条轨迹，用经验平均来估计梯度：
$$\nabla_\theta J(\theta)=\frac{1}{m}\sum_{i=1}^m\nabla_\theta\log p_\theta(\tau)R(\tau)$$
直观而言，沿着梯度正方向，轨迹出现的概率会增大；沿着梯度负方向，轨迹出现的概率会减小。进一步的，$R(\tau)$控制了增大减小的步长，当回报越大，轨迹出现的概率也增大的越多。

&emsp;&emsp;轨迹出现的概率$p_\theta(\tau)$可以用状态转移概率$p(s_{t+1}|s_t,a_t)$和策略$\pi_\theta$来表示，根据链式法则：
$\begin{align}
p_\theta(\tau) 
&=p(s_1)\pi_\theta(a_1|s_1)p(s_2|s_1,a_1)\pi_\theta(a_2|s_2)p(s_3|s_2,a_2)\pi_\theta(a_3|s_3)p(s_4|s_3,a_3)...\\
&=p(s_1)\prod_{t=1}^{T}\pi_\theta(a_t|s_t)p(s_{t+1}|s_t,a_t)
\end{align} $
于是有：
$\begin{align}
\nabla_\theta\log p_\theta(\tau)
&=\nabla_\theta\log[p(s_1)\prod_{t=1}^{T}\pi_\theta(a_t|s_t)p(s_{t+1}|s_t,a_t)]\\
&=\nabla_\theta\log[\prod_{t=1}^T\pi_\theta(a_t|s_t)p(s_{t+1}|s_t,a_t)]\\
&=\nabla_\theta[\sum_{t=1}^T\log p(s_{t+1}|s_t,a_t)+\sum_{t=1}^T\log\pi_\theta(a_t|s_t)]\\
&=\nabla_\theta[\sum_{t=1}^T\log\pi_\theta(a_t|s_t)]\\
&=\sum_{t=1}^T\nabla_\theta\log\pi_\theta(a_t|s_t)
\end{align}$
推导过程中，与参数$\theta$无关的量均可省去。综上，梯度的表达也可以进一步优化：
$\begin{align}
\nabla_\theta J(\theta)
&=\mathbb{E}_\tau[\sum_{t=1}^T \nabla_\theta\log\pi_\theta(a_t|s_t)R(\tau)]\\
&=\frac{1}{m}\sum_{i=1}^m\sum_{t=1}^T\nabla_\theta\log\pi_\theta(a_t^{(i)}|s_t^{(i)})R(\tau^{(i)})
\end{align}$

&emsp;&emsp;在计算累计回报时，通常需要完整的轨迹，是基于蒙特卡洛方法采样来计算回报的，所以计算得到的值的偏差较小（无偏的）而方差较大，降低了学习的速度。

# 二、知识延伸
1. 在计算梯度时，除了轨迹的累积回报，我们更加常用的是使用优势函数作为梯度方向的“步长”，常见的方式有：
   * 轨迹的累积回报：$\sum_{t=0}^Tr_t$
   * 采用动作$a_t$后的累积回报：$\sum_{t'=t}^Tr_{t'}$
   * 采用动作$a_t$后的累积回报，使用baseline：$\sum_{t'=t}^Tr_{t'}-b(s_t)$
   * 状态动作价值函数：$Q_\pi(s_t,a_t)$
   * 优势函数：$A_\pi(s_t,a_t)$
   * TD residual：$r_t+V_\pi(s_{t+1})-V_\pi(s_t)$
2. 在实现中，需要根据实际情况设计策略网络来表示 $\pi_\theta(a|s)$。
   * 在离散动作空间中，输入为状态的表示，输出节点与动作个数相等，后接Softmax层。
   * 在连续动作空间中，输入为状态的表示，输出的设计方式有多种。一般假设每个动作的输出服从高斯分布，因此可以输出每个动作的均值。动作之间可以共用方差或各自分别学习方差。近期也有研究指出输出使用Beta分布比Gussian分布要好

# 参考资料
* REINFORCE算法推导
* David Silver 增强学习——Lecture 7 策略梯度算法（一）