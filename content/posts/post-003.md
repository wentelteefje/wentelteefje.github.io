---
title: "ZK002: Extension Field Arithmetic in Python"
date: 2022-08-19T14:47:32+02:00
draft: false
tags: ["zk-snarks", "python", "mathematics", "groth16"]
summary: In this part of the series we review and start implementing extension field arithmetic.
math: true
---

# 1 Extension Field Arithmetic

## 1.1 Finite Fields of Prime Power
Recall that an irreducible polynomial over a field $K$ is a polynomial which cannot be factored into two non-constant polynomials of lesser degree over $K$ (in much the same way as a prime number cannot be factored into two numbers). For a given prime $p$ and a monic irreducible polynomial $P(x)$ over $\mathbb{F}\_{p}[x]$ of degree $k\geq 2$, the quotient ring $\mathbb{F}\_p[x]/(P(x))$ is a finite field of order $p^k$. This can be seen by considering that the equivalence classes (cosets) modulo $P(x)$ are given by polynomials of the form
$$a_0 + a_1 x + a_2 x^2 + ... + a_{k-1}x^{k-1} $$
where $a_i \in \mathbb{F}_p$. Since $\\#\mathbb{F}_p = p$ and there are $k$ separate coefficients $a_i$ this gives $p^k$ different combinations. That the ring $\mathbb{F}\_p/(P(x))$ is indeed a field follows from the fact that $P(x)$ is irreducible.

Let's start with a quick example. In the field $\mathbb{F}_2 = \lbrace 0, 1 \rbrace$ we can easily write down the  addition and multiplication tables:

| $\boldsymbol{+}$ | $0$   | $1$   | $\quad$ | $\boldsymbol{\times}$ | $0$ | $1$ |
|------------------|-------|-------|----|----------|-----|-----|
| $0$              | $0$   | $1$   | $\quad$ | $0$      | $0$ | $0$ |
| $1$              | $1$   | $0$   | $\quad$ | $1$      | $0$ | $1$ |

According to the tables the equation $x^2 + x = 0$ over $\mathbb{F}_2$ can be solved by $x = 0$ and $x = 1$. This means that we can factor the polynomial as $x^2 + x = (x - 0)(x - 1)$. It is therefore reducible over $\mathbb{F}_2$. The polynomial $P(x) = x^2 + x + 1$ does not have a root in the field, since $P(0) = P(1) = 1$. Hence, it is irreducible.

The quotient $\mathbb{F}_2[x]/(x^2 + x + 1)$ is a field extension of $\mathbb{F}_2$ with a root to $P(x)$. It consists of the $2^2 = 4$ elements $0, 1, x, x+1$ and has the addition table:

| $\boldsymbol{+}$ | $0$     | $1$   | $x$   | $x + 1$ |
|-----------------------|---------|-------|-------|---------|
| $0$                   | $0$     | $1$   | $x$   |  $x+1$  |
| $1$                   | $1$     | $0$   | $x+1$ | $x +1$  |
| $x$                   | $x$     | $x+1$ | $0$   | $1$     |
| $x + 1$               | $x+1$   | $x$   | $1$   | $0$     |

Let's go through the calculation of a few of its entries. Since every polynomial has coefficients from $\mathbb{F}_2$ we need to keep this fields arithmetic in mind. Therefore, since $2 \equiv 0 \pmod{2}$ we have $2x \equiv 0$, $2(x+1) \equiv 0$ and $x + x+1 \equiv 2x + 1 \equiv 1$. Next, the multiplication table has the form:

| $\boldsymbol{\times}$ | $0$     | $1$   | $x$   | $x + 1$|
|-----------------------|---------|-------|-------|---------|
| $0$                   | $0$     | $0$   | $0$   |  $0$    |
| $1$                   | $0$     | $1$   | $x$   | $x+1$   |
| $x$                   | $0$     | $x$   | $x+1$ | $1$     |
| $x + 1$               | $0$     | $x+1$ | $1$   | $0$     |

For $x\cdot x = x^2$ we know by referring to the polynomial $P$, that $x^2 = -x -1 = -(x +1)$, but $-1 \equiv 1$ in $\mathbb{F}_2$, so $x^2 \equiv x + 1$. Note, how the addition and multiplication tables of $\mathbb{F}_2$ are embedded in their counterpart in $\mathbb{F}_2[x]/(x^2 + x + 1)$. This visualises that $\mathbb{F}_2$ is a subfield of the extension. In this larger field we now have an element $x$ for which
$$P(x) = x^2 + x + 1 = (x + 1) + x + 1 = 2(x+1) = 0.$$
This may appear a bit strange at first, since normally a small $x$ denotes a variable and not a concrete element. However, here $x$ is really a new independent element with the property that $P(x) = 0$.

## 1.2 Modular Multiplication of Polynomials
Let $k := \operatorname{deg}(P)$ then multiplication of elements $a, b \in \mathbb{F}\_{p^k} = \mathbb{F}\_{p}[x]/(P(x))$ can be achieved by computing the ordinary polynomial multiplication along with coefficient reductions in $\mathbb{F}_p$ and a final reduction by the modulus $P$. More explicitly, we can write this multiplication as
$$ c(x) = a(x) \cdot b(x) = \left( \sum\_{i = 0}^{k-1} a_i x^i \right) \cdot \left( \sum\_{j = 0}^{k-1} b_j x^j \right) \equiv \sum\_{l = 0}^{2k-2} c_l x^l \pmod{P(x)}$$
where $c_l = \sum\_{i + j = l} a_i b_j \pmod{p}$.

As an example we are now going to compute the product $(x + 1)\cdot (x + 1)$ over $\mathbb{F}_2[x]/(x^2 + x + 1)$. In the notation of the previous equations we have $a_0 = b_0 = a_1 = b_1 = 1$. Therefore we need to compute $c_l$ for $0 \leq l \leq 2k-2 = 2$:
$$ \begin{align*} c_0 &= a_0 b_0 = 1 &\pmod{2} \\\\
c_1 &= a_0 b_1 + a_1 b_0 = 2 \equiv 0 &\pmod{2} \\\\
c_2 &= a_1 b_1 = 1 &\pmod{2} \end{align*}$$
This corresponds to the polynomial $c(x) = c_0 x^0 + c_1 x^1 + c_2 x^2 = 1 + x^2$. Now, since $x^2 \equiv x + 1 \pmod{P(x)}$ and $2 \equiv 0 \pmod{2}$ this polynomial is reduced to $x$.

## 1.3 The Field Extensions $\mathbb{F}\_{p^{2}}$ and $\mathbb{F}\_{p^{12}}$
In our implementation of the pairing on elliptic curves we will be working over the fields $\mathbb{F}\_{p^{2}}$ and $\mathbb{F}\_{p^{12}}$. The quadratic extension field will be defined as $\mathbb{F}\_{p^{2}} = \mathbb{F}_p[x]/(x^2 + 1)$. For this approach to work it is necessary that $x^2 + 1$ is irreducible over $\mathbb{F}_p$. It is sufficient to choose an odd prime $p$, such that $p \equiv 3 \pmod{4}$. Recall, that the multiplicative group $\mathbb{F}\_{p}^{\times}$ is a cyclic group of order $p - 1$. If $p \equiv 1 \pmod{4}$, then $4|(p-1)$ and therefore $\mathbb{F}\_{p}^{\times}$ has an element $i$ of order $4$, that is $i^4 = 1$ and $i^2 = -1$. Hence, $$x^2 + 1 = (x - i)(x + i)$$ is reducible. The requirement $p \equiv 3 \pmod{4}$ precisely prevents the existence of an element of order $4$ in $\mathbb{F}\_{p}^{\times}$.

For $\mathbb{F}\_{p^{12}}$ it's a bit more complicated. Recall, that an element $\zeta \in K$ of a field $K$ is called an $n$-th root of unity, if $\zeta^n = 1$. Equivalently, this means $\zeta$ is a root of $X^n - 1$ over $K$. We say $\zeta$ is primitive if no smaller power accomplishes this.

Now, consider the polynomial $X^6 - \zeta$ with some $\zeta \in \mathbb{F}\_{p^2}$. Suppose $\zeta$ is a cube, that is there exists some element $a \in \mathbb{F}\_{p^2}$, such that $a^3 = \zeta$. Then
$$ X^6 - \zeta = X^6 - a^3 = (X^2 - a)(X^4 + aX^2 + a^2)$$
and $X^6 - \zeta$ is not irreducible. Likewise, consider the case where $\zeta$ is a square, i.e. there exists an element $b$ with $b^2 = \zeta$. In this case $X^6 - \zeta$ is also irreducible because
$$ X^6 - \zeta = (X^3)^2 - a^2 = (X^3 - a)(X^3 + a).$$

Theory: $6|(p^2-1)$ and $6 = 2\cdot 3$, i.e. $2$ and $3$ both devide $p^2-1$ so only squares and cubes can exist in the field but not biquadrats.

Therefore the polynomial $(x^6 - 9)^2 + 1 = x^{12} - 18x^6 + 82$ is irreducible over $\mathbb{F}_p$ and we have
$$\mathbb{F}\_{p^{12}} = \mathbb{F}_p[x]/(x^{12} - 18x^6 + 82).$$

# 2. Implementation
## 2.1 Base Class for Polynomials
We start by implementing a base class for polynomial extension field arithmetic, which we later on use to define our quadratic extension field. Knowing a polynomial is knowing its coefficients and therefore an object of this class is represented by the vector (list) of its coefficients. The only other important information is of course the coefficients of the modulus.

```python
class PolyExtF():
    def __init__(self, coeffs, modulus_coeffs):
        assert len(coeffs) == len(modulus_coeffs)
        self.coeffs = coeffs
        self.degree = len(self.modulus_coeffs)
        self.modulus_coeffs = modulus_coeffs
```
We agree on the convention that the least significant coefficients are on the left side of the vector, that is the polynomial $Q(x) = \sum_\{i = 0}^{k-1} a_i x^i$ is represented by the vector $(a_0, a_1, ..., a_{k-2}, a_{k-1})$. Additionally, since the modulus is the only polynomial of degree $k$ and its leading coefficient $a_k$ will always be equal to one, we don't save $a_k$ in `modulus_coeffs` vector. This way we can represent the modulus by a vector of length $k-1$ like all the other elements of the field. Of course we need to remember this implied $a_k = 1$ in the algorithms we are going to implement, but in the end it will save us some overhead.

## 2.2 Modular Addition and Subtraction
Since there's no reduction mod $P$ needed, both addition and subtraction are easy to implement. We illustrate this with the example of addition. Let $a,b \in \mathbb{F}\_{p}[x]/(P(x))$ then

$$ a(x) + b(x) = \sum\_{i = 0}^{k - 1} c_i x^i,$$
where $c_i = (a_i + b_i) \pmod{p}$. Therefore, we only need to reduce the coefficient $c_i$ modulo $p$ when $a_i + b_i \geq p$. Just as we did in the case of the base field $\mathbb{F}\_{p}$, we overload Python's operands $+$ and $-$ by overwriting the methods `__add__` and `__sub__`.

```python
def __add__(self, other):
    assert self.deg == other.deg
    return PolyExtF([ a + b for a,b in zip(self.coeffs,other.coeffs)])

def __sub__(self, other):
    assert self.deg == other.deg
    return PolyExtF([ a - b for a,b in zip(self.coeffs,other.coeffs)])
```

## 2.3 Modular Multiplication
This one is a bit trickier, but we have already laid most of the groundwork in the introduction. We break this one into two parts. First, we implement the computation of the $c_l$, then we think about how to reduce the resulting polynomial modulo $P$.

A multiplication of two polynomials with degree at most $k - 1$ results in a new polynomial of degree at most $2k -2$. Therefore we start by initialising a coefficient vector of length $2k - 2 + 1$, then we use two for loops to calculate the $c_l$:
```python
k = self.deg
c = [GFq(0)] * (2*k - 1)
for i in range(k):
    for j in range(k):
        c[i+j] += a[i] * b[j]
```
Now, starting with the most significant coefficient we reduce the $c_l$ via the modulus coefficients until only terms of degree at most $k-1$ remain.
```python
# Reduction part
while len(c) > k:
    exp = len(c) - k - 1
    top = c.pop()

    for i in range(k):
        c[exp + i] -= top * modulus_coeffs[i]
return c
```
This is really just like polynomial long division except that we only keep track of the remainder. As an example consider the product of the polynomials $u(x) = 2x^2 +x + 2$ and $v(x) = x^2 + 2x$ modulo $x^3 + 1$. A traditional long division of the product $u(x)v(x)$ modulo $x^3 + 1$ looks like this:

$$\begin{align*} (2x^4& +& 5x^3& +& 4x^2& +& 4x& +& 0&):(x^3 + 1) = 2x + 5\\\\
-(3x^4&& && & +& 2x&&&) \\\\
&&(5x^3& +&4x^2& +& 2x& +& 0&) \\\\
&&-(5x^3& && & & +& 5&) \\\\
&&&&( 4x^2&+& 2x& -& 5&) \\\\
\end{align*}$$
If we compare the coefficients with the intermediate results of the reduction algorithm, we see that each pass of the inner loop corresponds to one step of the polynomial division:
```python
Input: c = [0, 4, 4, 5, 2]
After Loop 1: c = [0, 2, 4, 5]
After Loop 2: c = [-5, 2, 4]
```
TODO: Take care of base field modulus reduction.

# References
[1] Mrabet, N.E., & Joye, M. (Eds.). (2016). Guide to Pairing-Based Cryptography (1st ed.). Chapman and Hall/CRC. https://doi.org/10.1201/9781315370170

[2] Barreto, P.S.L.M., Naehrig, M. (2006). Pairing-Friendly Elliptic Curves of Prime Order. In: Preneel, B., Tavares, S. (eds) Selected Areas in Cryptography. SAC 2005. Lecture Notes in Computer Science, vol 3897. Springer, Berlin, Heidelberg. https://doi.org/10.1007/11693383_22
