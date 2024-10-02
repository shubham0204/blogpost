---
title: Bias and Variance in ML
draft: true
tags:
  - machine-learning
  - mathematics
---
#### Statistic
**A statistic is a quantity derived from a sample**. A sample is any subset of the data (a collection of data-points) collected for statistical purpose.
Given a sample, consisting of height measurements of ten students in a class, the mean or median height of the ten students is a statistic derived from that sample.
Note, statistic refers both, to the function (i.e. *mean* or *median*) and the output value of the function (i.e. *statistic = mean of heights of ten students*).
$$
\text{Statistic}: \ \text{Sample} \to \text{Quantity of Interest}
$$

#### Estimator
An **estimator is a statistic** that is used to infer the value of an unknown quantity. For instance, if the population mean is the unknown quantity, the sample mean could be its estimator. Computing the sample mean *could* give us the population mean, but we expect it to provide us an approximation mostly. 
$$
\text{Estimator}: \ \text{Sample} \to \text{Approximation for 'Quantity of Interest'}
$$
The $\text{Quantity of Interest}$ is the **estimand** and its approximation provided by the estimator is called the **estimate**.


