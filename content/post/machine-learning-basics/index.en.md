---
title: Machine Learning Basics
date: 2025-09-22 22:50:31
draft: false
math: true
---

# Machine Learning Basics

This article is based on Chapter 5 of [Machine Learning Yearning](https://www.deeplearning.ai/machine-learning-yearning/) and introduces some basic concepts and methods of machine learning.

## Supervised Learning

This article mainly discusses supervised learning algorithms.

First, let’s introduce some basic concepts:

- **Training Set**: The dataset used to train the model, containing input data (sample, $X$) and corresponding labels ($y$).
- **Sample**: Each data point in the training set, usually represented as a vector $x^{(i)}$, where $i$ is the sample index.
- **Test Set**: The dataset used to evaluate model performance, containing unseen input data and corresponding labels.
- **Input**: The input data to the model, usually represented as a vector $x \in \mathbb{R}^n$, where $n$ is the input dimension.
- **Output**: The output result of the model, usually represented as a scalar $y$, which can be a continuous value (regression) or a discrete value (classification).
- **Feature**: Each dimension of the input data, represented as $x_1, x_2, \ldots, x_n$.
- **Label**: The true output value corresponding to the input data, generally represented as $y$.

The main goal of supervised learning algorithms is to learn $P(y|x)$ from a training set, obtain a model to get an approximate value $\hat{P}(y|x)$, so that for new input $x$, the corresponding output $y$ can be predicted.

A common way to represent datasets is to store all samples’ inputs and labels in matrices and vectors. For example, a 28*28 grayscale image can be represented as a (1,28,28) tensor, a training set with $m$ samples can be represented as a (m,1,28,28) tensor, and labels can be represented as a (m,) vector.

## Example: Linear Regression

### Model

Linear regression is one of the simplest supervised learning algorithms. It assumes a linear relationship between input $x$ and output $y$, which can be expressed as (ignoring the intercept term for now):

$$y = w^T x $$

where $w$ is the model parameter, representing the weights of each input feature, which is the learning objective.

Assuming we have trained the parameter $w$, for a new input $x$, we can predict the output $\hat{y}$ by calculating $\hat{y} = w^T x$.

### Loss Function

To evaluate the performance of the model, we need to define a loss function to measure the difference between the predicted value $\hat{y}$ and the true value $y$.

Common loss functions include:

L1 loss (Mean Absolute Error, MAE):

$$L1(y, \hat{y}) = \frac{1}{m} \sum_{i=1}^{m} |y^{(i)} - \hat{y}^{(i)}|$$

Mean Squared Error (MSE):

$$MSE(y, \hat{y}) = \frac{1}{m} \sum_{i=1}^{m} (y^{(i)} - \hat{y}^{(i)})^2$$

where $m$ is the number of samples, $y^{(i)}$ and $\hat{y}^{(i)}$ are the true and predicted values for the $i$-th sample, respectively.

Here, we simply consider that the smaller the MSE, the better the model performance. Of course, other loss functions can be used, and different loss functions are suitable for different scenarios.

### Optimization

Next, our goal is to minimize the MSE loss function and find the optimal parameter $w$.

To achieve this goal, we hope to find $w$ through the training set that satisfies or is as close as possible to $\triangledown_w MSE_{train}=0$.

$$
\begin{align*}
    
& \triangledown_w MSE_{train} = 0\\

\Rightarrow & \frac{1}{m} \triangledown_w ||X_{train} \cdot w-y_{train}||^2_2 = 0\\

\Rightarrow & \triangledown_w (X_{train} \cdot w-y_{train})^T (X_{train} \cdot w-y_{train}) =0\\

\Rightarrow & 2X_{train}^T (X_{train} \cdot w-y_{train}) =0\\

\Rightarrow & w = (X_{train}^T X_{train})^{-1} X_{train}^T y_{train}
\end{align*}
$$

The solution derived above is called the Normal Equation, which gives the closed-form solution to the linear regression problem.
