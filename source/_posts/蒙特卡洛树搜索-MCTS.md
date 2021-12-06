---
title: 蒙特卡洛树搜索(MCTS)
date: 2021-12-06 21:56:44
tags:
---
**最近开始学习Game AI和强化学习相关的内容，发现蒙特卡洛树搜索(MCTS)在围棋等领域发挥了关键的作用。以前在做围棋项目的时候也曾接触过MCTS，但对它的具体原理和实现方法只是一知半解，将它作为第一篇博客内容也算是对过往知识的补充。**

## MCTS
$$Q(s, a)=\frac{1}{N(s,a)}\sum_{i=1}^{N(s)}\mathbb I{(s,a)z_i}$$

## 参考资料
* [Cameron Browne, Edward Jack Powley, Daniel Whitehouse, Simon M. Lucas, Peter I. Cowling, Philipp Rohlfshagen, Stephen Tavener, Diego Perez Liebana, Spyridon Samothrakis, Simon Colton:
A Survey of Monte Carlo Tree Search Methods. IEEE Trans. Comput. Intell. AI Games 4(1): 1-43 (2012)](https://ieeexplore.ieee.org/document/6145622)
* [如何学习蒙特卡罗树搜索（MCTS）](https://zhuanlan.zhihu.com/p/30458774)
