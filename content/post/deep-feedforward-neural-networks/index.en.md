---
title: Deep Feedforward Neural Networks
date: 2025-09-22 22:40:11
draft: false
---
# Deep Feedforward Neural Networks

This article is a study note for Chapter 6 of the book [Deep Learning](http://www.deeplearningbook.org/).

## Introduction

Deep Feedforward Networks, also known as Multilayer Perceptrons (MLPs), are the most classic neural network models.

From the perspective of graph theory, a deep feedforward network is a Directed Acyclic Graph (DAG), where each node represents a neuron and each edge represents a connection weight. The connections between nodes are directional, and information can only flow along the direction of the edges. Therefore, deep feedforward networks have no cycles or feedback connections.

Here are some basic concepts:

- **Neuron**: The basic computational unit in a neural network, receives input and generates output. Each neuron can be regarded as a function that takes an input vector and outputs a scalar.
- **Layer**: A group of neurons in a neural network, usually divided by function into input layer, hidden layers, and output layer.
- **Input Layer**: The first layer of the network, receives external input data.
- **Hidden Layers**: The intermediate layers between the input and output layers. The number of hidden layers and the number of neurons in each layer are important design parameters.
- **Output Layer**: The last layer of the network, generates the final output result.
- **Weights**: Parameters of neurons that determine the influence of input data on the output. Weights are learned during the training process.
- **Biases**: Another parameter of neurons that allows the model to fit data more flexibly. Biases are also learned during training.
