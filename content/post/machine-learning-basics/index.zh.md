---
title: 机器学习基础
date: 2025-09-22 22:50:31
draft: false
params:
    math: true
tag: DL
categories: [Machine Learning/Deep Learning]
---

# Machine Learning Basics

这篇文章基于Deep Learning [@Goodfellow-et-al-2016] 一书 Chapt5，介绍机器学习的一些基本概念和方法。

## Supervised Learning

本文主要讨论监督学习算法。

首先，介绍一些基本概念：

- **训练集 (Training Set)**：用于训练模型的数据集，包含输入数据 (sample, $X$)和对应的标签（label，$y$）。
- **样本 (Sample)**：训练集中的每一个数据点，通常表示为一个向量 $x^{(i)}$，其中 $i$ 是样本的索引。
- **测试集 (Test Set)**：用于评估模型性能的数据集，包含不被用在训练中，未见过的输入数据和对应的标签。
- **输入 (Input)**：模型的输入数据，通常表示为一个向量 $x \in \mathbb{R}^n$，其中 $n$ 是输入的维度。
- **输出 (Output)**：模型的输出结果，通常表示为一个标量 $y$，可以是连续值（回归问题）或离散值（分类问题）。
- **特征 (Feature)**：输入数据的各个维度，表示为 $x_1, x_2, \ldots, x_n$。
- **标签 (Label)**：输入数据对应的真实输出值，一般表示为 $y$。

监督学习算法的主要目的是，从一个训练集中学习 $P(y|x)$ ，得到一个模型，来获得近似值 $\hat{P}(y|x)$，使得对于新的输入 $x$，可以预测出对应的输出 $y$。

表示数据集的常用方法是将所有样本的输入和标签分别存储在矩阵和向量中，例如 28*28 的灰度图像可以表示为一个 (1,28,28) 的张量，包含 $m$ 个样本的训练集可以表示为一个 (m,1,28,28) 的张量，标签可以表示为一个 (m,) 的向量。

## Example: Linear Regression

### Model

线性回归是最简单的监督学习算法之一，假设输入 $x$ 和输出 $y$ 之间存在线性关系，可以表示为(这里先不考虑截距项)：

$$y = w^T x $$

其中 $w$ 是模型的参数，表示输入各个特征的权重, 即学习的目标

假设我们已经训练得到参数 $w$，对于新的输入 $x$，可以通过计算 $\hat{y} = w^T x$ 来预测输出 $\hat{y}$。

### Loss Function

为了评估模型的性能，我们需要定义一个损失函数 (Loss Function)，用于衡量模型预测值 $\hat{y}$ 和真实值 $y$ 之间的差距。

常用的损失函数有

L1损失 (Mean Absolute Error, MAE)：

$$L1(y, \hat{y}) = \frac{1}{m} \sum_{i=1}^{m} |y^{(i)} - \hat{y}^{(i)}|$$

均方误差 (Mean Squared Error, MSE)：

$$MSE(y, \hat{y}) = \frac{1}{m} \sum_{i=1}^{m} (y^{(i)} - \hat{y}^{(i)})^2$$

其中 $m$ 是样本数量，$y^{(i)}$ 和 $\hat{y}^{(i)}$ 分别是第 $i$ 个样本的真实值和预测值。

在这里，我们简单的认为 MSE 越小，模型的性能越好，当然也可以使用别的 Loss 函数，不同的 Loss 函数也适用于不同的场景。

### Optimization

下面我们的目标就是最小化 MSE 损失函数，找到最优的参数 $w$ 。

为了达成这个目标，我们希望找到通过训练集，找到满足或尽可能接近 $\triangledown_w MSE_{train}=0$ 的 $w$ 。

$$
\begin{aligned}
\nabla_w MSE_{train} = 0\\
\Rightarrow & \frac{1}{m} \nabla_w ||X_{train} \cdot w-y_{train}||_2^2 = 0\\
\Rightarrow & \nabla_w (X_{train} \cdot w-y_{train})^T (X_{train} \cdot w-y_{train}) =0\\
\Rightarrow & 2X_{train}^T (X_{train} \cdot w-y_{train}) =0\\
\Rightarrow & w = (X_{train}^T X_{train})^{-1} X_{train}^T y_{train}\\
\end{aligned}
$$

以上推出的解被称为正规方程 (Normal Equation)，它给出了线性回归问题的闭式解 (Closed-form Solution)。

值得注意的是，一般情况下，linear regression 通常还要附带参数 $b$ 作为截距项，即
$$y = w^T x + b$$

通常， $b$ 也被称为偏置 (bias) 参数

## Learning

在实际情况中，训练集往往非常大，计算正规方程的闭式解非常耗时，且当前仅为一个神经元的线性回归问题，实际的神经网络往往包含大量参数，计算闭式解更加困难。

考虑如何去拟合出一个好的 $w$，我们可以使用梯度下降 (Gradient Descent) 方法。具体请参考 [梯度下降](/gradient-descent) 文章。

## References

[Goodfellow-et-al-2016]: Ian Goodfellow, Yoshua Bengio, and Aaron Courville. *Deep Learning*. MIT Press, 2016. [http://www.deeplearningbook.org](http://www.deeplearningbook.org)
