---
title: "ZK003: Elliptic Curve Arithmetic in Python"
date: 2022-08-27T14:32:44+02:00
draft: false
tags: ["zk-snarks", "python", "mathematics", "groth16"]
summary: In this part of the series we review and start implementing elliptic curve arithmetic.
math: true
---

# 1. Mathematical Review
## 1.1 Elliptic Curves
An elliptic curve $E/K$ over an arbitrary field $K$ of characteristic $\neq 2,3$ is given by the set of solution $E(K)$ to the Weierstrass equation together with a point at infinity $P_\infty$.
$$ E(K) := \lbrace (x,y) \in K^2\\,|\\, y^2 = x^3 + ax + b \rbrace \cup \lbrace P_\infty \rbrace,$$
where $a,b \in K$ and $\Delta = 4a^3 + 27b^2 \neq 0$. The condition $\Delta \neq 0$ is required to prevent cusps and self-intersections of the curve.

We will often consider the solutions to a given elliptic curve equation over different closely-related fields. If $L$ is any extension field of $K$, then the set of $L$-rational points on $E$ is the set given by
$$E(L) = \lbrace (x,y) \in L\times L \\,|\\, y^2 - x^3 - 3 = 0 \rbrace \cup \lbrace \infty \rbrace$$

## 1.2 The Group Law
The set $E(K)$ forms an abelian group under a certain addition law and $P_\infty$ is its neutral element. For two points $P = (x_P, y_P)$ and $Q = (x_Q, y_Q)$ we declare:

1. If $P = P_\infty$ then $P + Q = Q$, if $Q = P_\infty$ then $P + Q = P$.
2. If $x_P = x_Q$, but $y_P \neq y_Q$ then $P + Q = P_\infty$.
3. Point doubling: If $P = Q = (x,y)$ then
$$P + Q := (x_D, y_D) = (\lambda^2 -2x, -\lambda^3 + 3\lambda x - y),$$
where $\lambda = \frac{3x^2 + a}{2y}$ is the slope of the tangent at $P$.
4. If none of the previous cases apply, then $P + Q := (x_S, y_S)$ is given by
$$ (x_S, y_S) = (\lambda^2 - x_P - x_Q, -\lambda^3 + 2\lambda x_P + \lambda x_Q - y_P),$$
where $\lambda = \frac{y_Q - y_P}{x_Q - x_P}$ is the slope of the line connecting $P$ and $Q$.

Showing that $E(K)$ with this addition law is indeed an abelian group is not hard. For example, case 1 establishes $P_\infty$ as the unique neutral element of the group. The opposite point $-P$ to a point $P$ is given by $-P := (x_P, -y_P)$, since the curve is symmetrical about the $x$-axis. It follows from case 2, that $P + (-P) = P_\infty$ as required from a well-defined group law. Proving that it is associative is also straightforward, but the case distinctions make it rather messy, so we won't do it here.

Since the third case, i.e. the doubling of a point, is of particular interest to us we are going to implement it in a separate function later on. The reason for this will become apparent in the article on pairings.

## 1.3 The BN128 Curve
Barreto and Naehrig [2] described a family of pairing-friendly elliptic curves parametrized by a $u \in \mathbb{Z}$. These are the curves of the form
$$E_b: y^2 = x^3 + b $$
over the finite field $\mathbb{F}_p$ with $p(u) = 36u^4 + 36u^3 + 24u^2 + 6u + 1$ and curve order $n(u) = 36u^4 + 36u^3 + 18u^2 + 6u + 1$. The approach here is to find an integer $u$ such that both $p(u)$ and $n(u)$ are prime. Afterwards, we select $b$, such that a few additional conditions are met. We do not have to deal with the details of construction here. Instead, we will concentrate on the special case of curve `BN128` given by the equation $y^2 = x^3 + 3$ with the following parameters

$$\begin{align*} p = &218882428718392752222464057452572750886\\\\
                     &96311157297823662689037894645226208583,\\\\
                 n = &2188824287183927522224640574525727508854\\\\
                     &8364400416034343698204186575808495617. \end{align*}$$

These values are obtained with $u = 4965661367192848881$.


## 1.3 The Frobenius endomorphism
Let $\mathbb{F}_{q}$ be a finite field with algebraic closure $\overline{\mathbb{F}}_q$. The Frobenius endomorphism is the map $\pi_q : \overline{\mathbb{F}}_q \to \overline{\mathbb{F}}_q$ given by $x \mapsto x^q$. For any elliptic curve $E/\mathbb{F}_q$ this endomorphism acts on coordinates of points in $E(\overline{\mathbb{F}}_q)$ as $$ \pi_q(x,y) = (x^q, y^q) \quad\textrm{and}\quad \pi_q(\infty) = \infty.$$ In particular we have
$$\pi_q(x,y) \in E(\overline{\mathbb{F}}_q)$$ for any $(x,y) \in E(\overline{\mathbb{F}}_q)$, and
$$(x,y) \in E(\mathbb{F}_q)$$ if and only if $\pi_q(x,y) = (x,y)$.
By composing $\pi_q$ with itself $n \geq 1$ times, i.e. setting $\pi_q^{n} := \pi_q \circ \pi_q \circ ... \circ \pi_q$ we obtain the Frobenius map for the field $\mathbb{F}\_{q^{n}}$. Hence, it follows from the second property stated above that the eigenspace of $\pi_q^n$ with respect to the eigenvalue $1$ is equal to $E(\mathbb{F}\_{q^{n}})$, that is $\ker{(\pi_q^n - [1])} = E(\mathbb{F}\_{q^{n}})$. Here, $[1]$ denotes a special case of the multiply-by-$m$ endormophism $[m] : E/\mathbb{F}_q \to E/\mathbb{F}_q$ defined as $P \mapsto mP$. Another important special case of this map arises for $m = -1$, i.e. $P \mapsto -P$ (the negation map).

## 1.1 Twists
Let $\xi \in \mathbb{F}\_{p^2}$ such that $X^6 - \xi$ is irreducible over $\mathbb{F}\_{p^2}[X]$ whenever $p \equiv 1 \pmod{6}$.
Let $\mu \in \mathbb{F}_{p^{12}}$ be a root of $X^6 - \xi$, then the curve $E'/\mathbb{F}\_{p^2}: Y'^2 = X'^3 - b/\xi$ is a sextic twist of $E/\mathbb{F}\_{p}$ with order $n(2p -n)$. The map $\psi$ defined by
$$\begin{align*} \psi: E'(\mathbb{F}\_{p^2}) &\to E(\mathbb{F}\_{p^{12}})\\\\
(x',y') &\mapsto (\mu^2 x', \mu^3 y') \end{align*}$$
is an isomorphism.

Source: https://eprint.iacr.org/2010/429.pdf
Read: https://crypto.stackexchange.com/questions/14669/sextic-twist-optimization-of-bn-pairing-cubic-root-extraction-required?noredirect=1&lq=1
Source2: https://eprint.iacr.org/2007/390.pdf

## 1.2 Torsion points
Let $E/K$ be an elliptic curve defined over a field $K$ and let $r$ be a positive integer. The set of $r$-torsion points is defined as the set of curve points of finite order $r$. More formally,
$$E[r] := \lbrace P \in E(\overline{K})\\,|\\, rP = \infty \rbrace. $$
Note that $E[r]$ contains points with coordinates in the algebraic closure $\overline{K}$ of $K$.

For example, consider the curve given by $Y^2 = X^3 + 2$ over $\mathbb{F}_7$. Then we have
$$E(\mathbb{F}_7) = \lbrace (0,3), (0,4), (3, 1), (3, 6), (5, 1), (5, 6), (6, 1), (6, 6) \rbrace \cup \lbrace \infty \rbrace.$$ An easy computation but somewhat tedious computation shows that each of these points $P$ satisfy $3P = \infty$, so $E[3] =  E(\mathbb{F}_7) \simeq \mathbb{Z}_3 \times \mathbb{Z}_3$.

# 2. Implementation
## 2.1 Doubling of Projective Points
In order to avoid expensive divisions in the calculation of the slope $\lambda$ in the doubling and addition algorithms we will use homogeneous coordinates in the projective plane. Any point $P = (x,y)$ in the Euclidean plane corresponds to a line through the origin with points $Z(X,Y,1)$ and $Z \neq 0$ in projective space. The set of solutions to the curve equation is given by homogenization of the polynomial $y^2 = x^3 + b$. That is

$$E(\mathbb{F}_p) = \lbrace (X,Y,Z) \in \mathbb{F}_p^3 \\,|\\,Y^2Z = X^3 + bZ^3 \rbrace. $$

By construction this set includes the point at infinity $P\_\infty = (0,1,0)$. With this we are going to adjust the algorithms given in the introduction. For example, the resulting $x$-coordinate in the doubling algorithm is transformed by replacing $X$ by $X/Z$ and $Y$ by $Y/Z$ (note that $a = 0$):
$$\left( \frac{3(\frac{X}{Z})^2}{2\frac{Y}{Z}} \right)^2 - 2\frac{X}{Z} = \frac{(3X^2)^2}{4Y^2Z^2} - 2\frac{X}{Z} $$
To get rid of the divisions we multiply by $8Y^2Z^3$ and obtain
$$ 2YZ(3X^2)^2 - 16XY^3Z^2  = 2YZ((3X^2)^2 - 8XY^2Z).$$
Let $A := 3X^2$, $B := YZ$, $C := XYB$ and $D := A^2 - 8C$ then we obtain for the resulting projective coordinates $(X_D, Y_D, Z_D)$ of the point doubling:
$$\begin{align*} X_D &= 2BD, \\\\
Y_D &= A(4C-D) - 8Y^2B^2, \\\\
Z_D &= 8B^3. \end{align*} $$
Therefore, our implementation looks like this:
```python
def double(P):
    X, Y, Z = P
    A = 3 * X * X     
    B = Y * Z         
    B2 = B * B        
    C = X * Y * B     
    D = A * A - 8 * C

    XD = 2 * B * D
    YD = A * (4 * C - D) - 8 * Y * Y * B2
    ZD = 8 * B * B2
    return XD, YD, ZD
```

## 2.2 Addition of Projective Points
In the same way as for the doubling algorithm, the projective coordinates for the addition $P + Q$ of two points of the curve can be obtained from the cartesian counterpart discussed in the mathematical review. However, this would take up a lot of space and would not give us any new knowledge, so we will not do the explicit calculation and give the result right away.

Denote the two points as $P = (X_1, Y_1, Z_1)$ and $Q = (X_2, Y_2, Z_2)$. Further, we define $A = Y_2Z_1 − Y_1Z_2$, $B = X_2Z_1 − X_1Z_2$ and $C = A^2Z_1Z_2 − B^3 − 2B^2X_1Z_2$. With these definitions in place, the projective coordinates $(X_S, Y_S, Z_S)$ of the sum of the points $P$ and $Q$ can be written as:

$$\begin{align*} X_S &= BC, \\\\
Y_S &= A(B^2X_1Z_2 − C) − B^3Y_1Z_2, \\\\
Z_S &= B^3Z_1Z_2. \end{align*} $$

This is the implementation:

```python
def add(P,Q):
    X1, Y1, Z1 = P
    X2, Y2, Z2 = Q

    # Case 1
    if is_point_at_infinity(P):
      return Q
    if is_point_at_infinity(Q):
      return P

    # Case 2 & 3
    if X1 == X2:
      if Y1 != Y2:
        return (zero, one, zero) #Point at inf
      else:
        return double(P)

    # Case 4
    A = Y2 * Z1 − Y1 * Z2
    B = X2 * Z1 − X1 * Z2
    B2 = B * B
    C = A ** 2 * Z1 * Z2 − B * B2 − 2 * B2 * X1 * Z*2

    XS = B * C
    YS = A * (B2 * X1 * Z2 − C) − B * B2 * Y1 * Z2
    ZS = B * B2 * Z1 * Z2

    return XS, YS, ZS
```
Where we have used the convenience function `is_point_at_infinity` defined as
```python
def is_point_at_infinity(P):
  X, Y, Z = P
  return X == zero and Y == one and Z == zero
```

## 2.4 Twisting

"Twist" a point in E(FQ2) into a point in E(FQ12), i.e. $\psi : E'(\mathbb{F}\_{p^2}) \to E(\mathbb{F}\_{p^{12}})$. A twist of $E/K$ is a smooth curve $E'/K$ that is isomorphic to $E$ over $\overline{K}$.

Sextic twist to represent elements of $\mathbb{G}_2 \subset E(\mathbb{F}\_{p^k})[r]$ as elements of an isomorphic group $\mathbb{G}\_2^{'} = E'(\mathbb{F}\_{p^{k/6}})[r]$. In our case $k = 12$ so we can represent elements of $E(\mathbb{F}\_{p^{12}})[r]$ as elements of $E'(\mathbb{F}\_{p^{2}})[r]$. This means that we can perform group operations in $\mathbb{G}_2$ on points with coordinates in an extension field with degree $\frac{1}{6}$-th the size.

 Vitalik macht so: $P = (x,y) \in \mathbb{F}\_{p^2}$ with $x = \begin{pmatrix} a_0 \\ a_1 \end{pmatrix}^T$ and $y = \begin{pmatrix} b_0 \\ b_1 \end{pmatrix}^T$. Then first map
$$ \left(\begin{pmatrix} a_0 \\\\ a_1 \end{pmatrix}, \begin{pmatrix} b_0 \\\\ b_1 \end{pmatrix} \right) \mapsto \left( \begin{pmatrix} a_0 - 9a_1 \\\\ a_1 \end{pmatrix},\begin{pmatrix} b_0 - 9b_1 \\\\ b_1 \end{pmatrix}  \right)$$
Afterwards, "embed" the result in $\mathbb{F}\_{p^{12}}$ with
$$\left( \begin{pmatrix} a_0 - 9a_1 \\\\ a_1 \end{pmatrix},\begin{pmatrix} b_0 - 9b_1 \\\\ b_1 \end{pmatrix}  \right) \mapsto \left( \begin{pmatrix} a_0 - 9a_1 \\\\ 0_5 \\\\ a_1 \\\\ 0_5 \end{pmatrix},\begin{pmatrix} b_0 - 9b_1 \\\\ 0_5 \\\\ b_1 \\\\ 0_5 \end{pmatrix}  \right) $$

# References
[1] Mrabet, N.E., & Joye, M. (Eds.). (2016). Guide to Pairing-Based Cryptography (1st ed.). Chapman and Hall/CRC. https://doi.org/10.1201/9781315370170

[2] Barreto, P.S.L.M., Naehrig, M. (2006). Pairing-Friendly Elliptic Curves of Prime Order. In: Preneel, B., Tavares, S. (eds) Selected Areas in Cryptography. SAC 2005. Lecture Notes in Computer Science, vol 3897. Springer, Berlin, Heidelberg. https://doi.org/10.1007/11693383_22

[3] https://git.with.parts/mirror/go-ethereum/src/branch/master/crypto/bn256/google/constants.go


