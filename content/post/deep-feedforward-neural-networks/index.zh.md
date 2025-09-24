---
title: deep-feedforward-neural-networks
date: 2025-09-22 22:40:11

---
# Deep Feedforward Neural Networks

本文是对[Deep Learning](http://www.deeplearningbook.org/)一书中第6章内容的学习笔记。

## Introduction

深度前馈网络 (Deep Feedforward Networks)，也称为多层感知机 (Multilayer Perceptrons, MLPs)，是最经典的神经网络模型。

从图论角度出发，深度前馈网络是一个有向无环图 (Directed Acyclic Graph, DAG)，其中每个节点表示一个neuron（神经元），每条边表示一个连接权重 (weight)。节点之间的连接是有方向的，信息只能沿着边的方向流动。因此，深度前馈网络没有循环 (cycles) 或反馈 (feedback) 连接。

这里简单给出几个概念：

- **神经元 (Neuron)**：神经网络中的基本计算单元，接收输入并生成输出。这里可以将每个Neuron看作一个函数，接收输入向量并输出一个标量。
- **层 (Layer)**：神经网络中的一组神经元，通常按功能划分为输入层、隐藏层和输出层。
- **输入层 (Input Layer)**：网络的第一层，接收外部输入数据。
- **隐藏层 (Hidden Layers)**：位于输入层和输出层之间的中间层。隐藏层的数量和每层的神经元数量是网络设计的重要参数。
- **输出层 (Output Layer)**：网络的最后一层，生成最终的输出结果。
- **权重 (Weights)**：神经元的参数，决定了输入数据对输出的影响程度。权重是通过训练过程学习得到的。
- **偏置 (Biases)**：神经元的另一个参数，允许模型更灵活地拟合数据。偏置也是通过训练过程学习得到的。
