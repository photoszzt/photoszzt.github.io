---
layout: post
title: attention math
date: 2026-03-01T16:39:58-07:00
description: reading notes on attention math
tags: paper-note
categories: paper-note
giscus_comments: true
---

### Attention math

#### Why $frac{1}{\sqrt(dk)}$

X, Y are two independent random variable. E[X] = 0, E[Y] = 0. Var[X] = 1, Var[Y] = 1. 

$Var\[XY\] = E[X]^2 \dot Var[Y] + E[Y]^2 \dot Var[X] + Var[X] \dot Var[Y]$

Since E[x] and E[Y] = 0, $Var[XY] = Var[X] * Var[Y]$

For $C = Q * K^T$, $C_{ij} = \sum_{k=1}^{N} Q_{ik} * K_{kj}$. Each entry is a sum of N products of random variables. 

$Var[QK^T] = dk * 1 * 1$, where N = dk, Var[Q] = 1, Var[k] = 1. 

We want to find a value to scale the $QK^T$ so that Var[a \dot QK^T] = 1$

$Var[aX] = a^2 * Var[X]$, $Var[aQK^T] = a^2 * Var[QK^T] = a^2 * dk = 1$. $a = \frac{1}{\sqrt(dk)}$

