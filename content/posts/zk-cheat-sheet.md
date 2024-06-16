+++
title = 'Zk Cheat Sheet'
date = 2024-06-14T21:19:40+02:00
draft = true
+++

# 1. Elliptic Curve Cryptography Concepts
## 1.1 Domain Parameters
$$y^2 = x^3 + ax + b,$$
where $a, b$ are elements of a finite field $\mathbb{F}_p$ with $\operatorname{char}(\mathbb{F}_p) \neq 2,3$.

The **domain parameters** of an ECC scheme are defined by the following data:
- size $p$ of the finite field $\mathbb{F}_p$,
- choice of the constants $a$ and $b$,
- the generator $g$ for the cyclic subgroup, also called base point,
- the order $n$ of $g$, i.e. $n := \operatorname{ord}(g)$,
- the cofactor $h := \frac{1}{n}|E(\mathbb{F}_p)|$.
## 1.2 Discrete Logarithm Problem
The **discrete logarithm problem (DLP)** in $\mathbb{G}$ is defined as: For a given input $(g, g^\alpha) \in \mathbb{G}^2$, compute $\alpha \in \mathbb{F}_p$.
## 1.3 Pairing-friendly Curve, Bilinear Pairings
# 2 Finite Fields
## 2.1 Overview
For every integer $n > 0$ the set of integers modulo $n$ that are relatively prime to $n$ is written as $(\mathbb{Z}/n\mathbb{Z})^{\times}$. This set forms a group under the operation of multiplication.
- For a prime $p$, the group $(\mathbb{Z}/p\mathbb{Z})^{\times}$ is always cyclic and we have $|(\mathbb{Z}/n\mathbb{Z})^{\times}| = p - 1$
- $(\mathbb{Z}/p\mathbb{Z})^{\times}$ consists precisely of the non-zero elements of $\mathbb{Z}/p\mathbb{Z}$
## 2.2 Important Theorems
## 2.3 Toolkit
# 3 Diffie-Hellman Key Exchange
The goal of DH is for Alice and Bob to establish a **shared secret key**. For this purpose they agree on a prime $p$ and a generator $g$ in the finite cyclic group $G$ of order $p$.
## 3.1 Protocol
1. Alice picks a random $a \in \mathbb{N}$ with $1 < a < p$ and sends the element $g^{a}$ to Bob.
2. Bob picks a random $b \in \mathbb{N}$ with $1 < b < p$ and sends the element $g^{b}$ to Alice.
3. Alice computes the element $g^{ba} = (g^{b})^{a}$.
4. Bob computes the element $g^{ab} = (g^{a})^{b}$.

After the protocol's completion both obtained the shared secret value $g^{ab} = g^{ba}$.
## 3.2 Elliptic-Curve-Diffie-Hellman (ECDH)
ok
## 3.2 Computational Diffie-Hellman Problem (CDHP)
> **CDHP**: Given a group element $g \in G$ and the values of $g^{a}$ and $g^{b}$, what is the value of $g^{ab}$?

Here, $G$ typically is an elliptic curve group and $g$ is a generator of this group. The integers $x,y$ have been chosen randomly during a Diffie-Hellman key exchange.
# 4 BLS Digital Signatures