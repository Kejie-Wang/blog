---
date: 2017-08-17 22:19:09
title: Machine Learning Related Distribution
mathjax: true
tags:
---

最近在学习LDA (Latent Dirichlet Allocation)，其中涉及到了很多的数学知识，这里主要对于机器学习中一些常用的概率分布，比如$Gamma$分布、多项分布、$Dirichlet$分布等等进行一个总结和分析。

## 伯努利分布

Simple inline $a = b + c$.
伯努利分布(*Bernoulli distribution*)通常被称之为**两点分布**，表示的是一个只有两种状态的随机变量的概率分布。首先介绍**伯努利实验**，其指的只有两个结果的单次随机实验，比如抛一枚硬币。那么伯努利实验结果的概率分布就是伯努利分布。假设随机变量$X$为一个伯努利分布，记其两个状态分别为0和1，那个它的概率分布为：

| $X$  |  0   |   1   |
| :--: | :--: | :---: |
| $P$  | $q$  | $1-q$ |

