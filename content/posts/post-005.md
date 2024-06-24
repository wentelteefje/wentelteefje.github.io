---
title: "ZK004: Optimal Ate Pairing in Python"
date: 2022-08-30T20:29:00+02:00
draft: true
tags: ["zk-snarks", "python", "mathematics", "groth16"]
summary: In this part of the series we review and start implementing elliptic curve pairings.
math: true
---

{{< color color="red" >}}
> Note: This is a draft article.
{{< /color >}}

# 1. Mathematical Review

## 1.1 The Optimal Ate pairing
Let $n$ be the curve order as determined in the last article, and let $\pi_p$ be the $p$-th power Frobenius endomorphism, i.e. $\pi_p(x,y) = (x^p, y^p)$. An Optimal Ate pairing is a non-degenerate bilinear pairing on $E$ with the following setup:
$$ \begin{align*} \mathbb{G}_1 &:= E[n] \cap \ker{(\pi_p - [1])} = E(\mathbb{F}_p)[n] \\\\
\mathbb{G}_2 &:= E[n] \cap \ker{(\pi_p - [p])} \subseteq E(\mathbb{F}\_{p^{12}})[n] \\\\
\mathbb{G}_3 &:= \mu_n = \lbrace \zeta\\,|\\,\zeta^n = 1 \rbrace \subset \mathbb{F}\_{p^{12}}^{\times} \end{align*}$$

For any $P \in \mathbb{G}_1, Q \in \mathbb{G}_2$ let $\alpha\_{\textrm{opt}} : \mathbb{G}_2 \times \mathbb{G}_1 \to \mu_n$ be defined as:
$$ \alpha\_{\textrm{opt}}(Q,P) := \left( f\_{6u+2,Q}(P) \cdot h\_{6u+2,Q}(P) \right)^{\frac{p^{12}-1}{n}} $$
where $h\_{6u+2,Q}(P) = l\_{[6u+2]Q,Q_1}(P) \cdot l\_{[6u+2]Q+Q_1,−Q_2}(P)$. In this definition the value $l\_{R,S}(P) \in \mathbb{F}\_{p^{12}}$ is obtained by evaluating the equation of the line through the points $R$ and $S$ on $E$ at the point $P$. The special points $Q_1, Q_2$ are defined using the Frobenius endomorphism as $Q_1 := \pi_p(Q)$ and $Q_2 := \pi_p^2(Q) = \pi_p(Q_1)$ and $f, h$ are rational functions on $\mathbb{F}\_{p^{12}}$.

Additionally, all(really everthing?) curve arithmetic is done on the sextic twist $E': y^2 = x^3 + b/\xi$ defined over $\mathbb{F}\_{p^2}$. We represent each point $R \in \mathbb{G}_2$ as an image $R = \psi(R')$ of a point $R' \in E'(\mathbb{F}\_{p^2})$, where $\psi : E' \to E, (x', y') \mapsto (\omega^2 x', \omega^3 y' )$ is the corresponding twist isomorphism.

```python
# w = 0x^0 +1x^1 + ... 0x^11
w = FQ12([0, 1] + [0] * 10)
```



## 1.2 Computing the Ate - Miller's Algorithm
To actually calculate the pairing we need Miller's algorithm (more commonly just Miller loop), the basics of which we would like to briefly motivate here. Suppose $f$ is a rational function on $E$, i.e. $f$ can be written as a ratio of polynomials
$$f(X) = \frac{a_0 + a_1X + a_2X^2 + ... + a_n X^l}{b_0 + b_1X + b_2X^2 + ... + b_mX^m} = \frac{a(X-\alpha_1)^{e_1} \cdot ... \cdot (X-\alpha_r)^{e_r} }{b(X-\beta_1)^{d_1} \cdot ... \cdot (X-\beta_s)^{d_s} }$$
with zeroes $\alpha_1, ..., \alpha_r$ and poles $\beta_1, ..., \beta_s$ with corresponding multiplicities $e_1, ..., e_r$ and $d_1, ..., d_s$.
We define the divisor $(f)$ of a rational function $f$ as a formal sum to keep track of the zeroes and poles associated with $f$:
$$ (f) = \sum_i e_i[\alpha_i] +  \sum_j -d_j[\beta_j].$$

Miller's algorithm exploits a simple but powerful theorem of elliptic curves. When $f$ and $g$ are two non-zero rational functions on $E$ with $(f) = (g)$, then we already have $f = cg$ for some non-zero constant $c$ (Source: [X], Thm Y.Z). This means, that the actual choice of the rational functions involved in the computations doesn't matter, rather it's their divisors which are of main interest.

For any $Q \in E(\mathbb{F}\_{p^{12}})$ and integer $s$, let $f\_{s,Q}$ be a $\mathbb{F}\_{p^{12}}$-rational function with divisor:

$$(f\_{s,Q}) = s(Q) - ([s]Q) - (s-1)\mathcal{O}.$$

This function $f\_{s,Q}$ is a so-called Miller function, which is determined uniquely up to a non-zero constant $c \in \mathbb{F}\_{p^{12}}$. Miller’s algorithm builds up these functions $f\_{s,Q}$ in a loop according to the following formula: If $l\_{[m]Q,[n]Q}$ is the equation of the line through $[m]Q$ and $[n]Q$ (or the tangent when $[m]Q = [n]Q$) and $v\_{[m+n]Q}$ is the equation of the vertical line trough $[m + n]Q$, then we have
$$ f\_{m+n,Q} = f\_{m,Q} \cdot f\_{n,Q} \frac{ l\_{[m]Q,[n]Q} } { v\_{[m+n]Q} }. $$

The interested reader can find a more detailed explanation in [2]. In summary, we compute the Optimal Ate pairing using the Miller loop as $\alpha\_{\textrm{opt}}(Q,P) = \operatorname{ML}(P,Q)$. The rest of the article will be dedicated to implementing Miller's algorithm for `BN128`.
Note: The term __optimal__ in Optimal Ate refers to the fact, that the pairing can be computed in $\log_2(n)/\phi(k) + e(k)$ Miller iterations, where $n = |\mathbb{G}_1| = | \mathbb{G}_2 | = |\mathbb{G}_T|$, $\phi(k)$ is Euler's totient function evaluated on the embedding degree $k$ and $e(k) \leq log_2 k$ [3].

## 1.1 Miller Loop

Following Aranha et al. [1] we are now going to implement a version of Miller's algorithm to evaluate the Optimal Ate pairing function on $Y^2 = X^3 + 3$. The following is a picture of its pseudocode from the publication.

![Miller Loop for Optimal Ate with BN curves](/img/post-005/millerloop.png)

We are going to step by step through the algorithm and simultaneously translate it into Python code. We start with the instantiations on line one.

```python
def miller_loop(Q,P):
  T = Q
  f = FQ12.one()
```
Recall from the last article, that we obtained the `BN128` curve parameters $p(u), n(u)$ with $u = 4965661367192848881$. We can compute the Optimal Ate loop count $r$ and its logarithmic equivalent $t$ as as
$$ \begin{align*} r &= |6u + 2| = &29793968203157093288,\\\\
t &= \lfloor \log_{2}{r} \rfloor - 1 = &63.\end{align*}$$
We save $r$ in a Python variable named `ate_loop_count` and $t$ in a variable named `log_ate_loop_count`. Additionally, we will make use of the coefficients $r_i$ in the binary representation of $r = \sum_{i=0}^{\lfloor \log_{2}{r} \rfloor} r_i 2^i$ inside the loop. In particular we want to check in each loop whether $r_i = 1$, i.e. if $r$'s binary representation includes $2^i$ as a summand. We can easily achieve this by using $2^i$ as a bit mask and the bitwise AND operator `&`.
```python
for i in range(log_ate_loop_count, -1, -1):
  f = f * f * line_func(T,T,P)
  T = double(T)
  if ate_loop_count & (2**i):
    f = f * line_func(T,Q,P)
    T = add(T,Q)
```
This concludes the loop. Now we obtain two new points by applying the Frobenius endomorphism to $Q$:
```python
Q1 = frobenius(Q, p)
nQ2 = frobenius(neg(Q1), p)
```
Here we have used the fact that...

```python
f = f * line_func(T, Q1, P)
T = add(T, Q1)
f = f * line_func(T, nQ2, P)
T = add(T, neg(Q2))
# Final Exponentiation
return f ** (( p ** 12 - 1) // curve_order)
```

This concludes the implementation of the Miller loop. Let's turn to the implementation of the line function subroutine.

## 1.2 Line Function

For simplicity, we first describe the algorithm in affine coordinates and then we extend it to projective coordinates. Let $P = (x_1, y_1), Q = (x_2, y_2)$ and $T = (x_t, y_t)$ be points, then `line_func(P,Q,T)` constructs a line through $P$ and $Q$ and evaluates it at $T$. Fortunately, this function is very easy to fathom and we have already supplied all the relevant math in the previous article about Elliptic curves (see section the group law). There are three separate cases we need to take care of.

1. (General Line) In the case that $x_1 \neq x_2$ we have a well-formed line, which can be described by the equation $y = \lambda_1(x - x_1) + y_1$, where $\lambda_1 = \frac{y_2 - y_1}{x_2 - x_1}$ is the slope. We can evaluate it at the point $T$ by computing $ \lambda_1 \cdot (x_t - x_1) + (y_1 - y_t).$
2. (Tangent Line) When $x_1 = x_2$ and $y_1 = y_2$ the line is a tangent to the point $P = Q$ with slope given by $\lambda_2 = \frac{3x^2}{2y}$. The evaluation term has the same form as in the first case with $\lambda_1$ replaced by $\lambda_2$.
3. (Vertical Line) Lastly, when $x_1 = x_2$ and $y_1 \neq y_2$ we have the case of a vertical line $X - x_1$, which can be evaluated by computing $x_t - x_1$.

As you can see the function is zero whenever $T$ is a point on the line passing through $P$ and $Q$ and non-zero otherwise.

How do we need to change this function when working with projective coordinates? We can derive every equation by remembering that two points $P = (X_1, Y_1, Z_1)$ and $Q = (X_2, Y_2, Z_2)$ in projective space are equal, whenever $X_1/Z_1 = X_2/Z_2$ and $Y_1/Z_1 = Y_2/Z_2$. For example, to obtain the slope $\lambda_1$ of the general line equation:
$$\lambda_1 = \frac{Y_2/Z_2 - Y_1/Z_1}{X_2/Z_2 - X_1/Z_1} $$
However, to avoid unnecessary divisions we expand the term with $Z_1Z_2$ and save numerator and denominator separately:
$$ \lambda_1 = \frac{Z_1 Z_2}{Z_1 Z_2} \lambda_1 = \frac{Z_1Y_2 - Z_2Y_1}{Z_1X_2 - Z_2X_1} =: \frac{\lambda_{1n}}{\lambda_{1d}}. $$
Using the same procedure (but expanding by $Z^2$) we can obtain an expression for $\lambda_2$ in projective coordinates as
$$\lambda_2 = \frac{3X^2}{2ZY} =: \frac{\lambda_{2n}}{\lambda_{2d}}. $$

With this we can re-formulate the three cases of the algorithm as follows:
1. (General Line) If $\lambda_{1d} \neq 0$, then evaluate as
$$ \lambda_1 (X_t/Z_t - X_1/Z_1) + (Y_1/Z_1 - Y_t/Z_t).$$
2. (Tangent Line) If $\lambda_{1d} = 0$ and $\lambda_{1n} = 0$, then evaluate as
$$ \lambda_2 (X_t/Z_t - X_1/Z_1) + (Y_1/Z_1 - Y_t/Z_t).$$
3. (Vertical Line) Else evaluate as $X_t/Z_t - X_1/Z_1$.

Since we can easily modify the Miller loop to operate separately on numerator and denominator we can just return them as a tuple without performing the costly division. For this we need to use another small algebraic transformation. Consider again the case of the general line in the projective setting, then we can write:
$$ \begin{align*}\lambda_1 \left(\frac{X_t}{Z_t} - \frac{X_1}{Z_1}\right) + \left(\frac{Y_1}{Z_1} - \frac{Y_t}{Z_t}\right) &= \frac{\lambda_{1n}}{\lambda_{1d}} \frac{X_t Z_1 - X_1 Z_t}{Z_1 Z_t} + \frac{Y_1 Z_t - Y_t Z_1}{Z_1 Z_t} \\\\
&= \frac{\lambda_{1n}(X_t Z_1 - X_1 Z_t) + \lambda_{1d}(Y_1 Z_t - Y_t Z_1)}{\lambda_{1d} Z_1 Z_t} \end{align*}$$
```python
def line_func(P, Q, T):
    zero = P[0].__class__.zero()
    x1, y1, z1 = P
    x2, y2, z2 = Q
    xt, yt, zt = T

    l_n = y2 * z1 - y1 * z2
    l_d = x2 * z1 - x1 * z2

    if l_d != zero:
        return l_n * (xt * z1 - x1 * zt) - l_d * (yt * z1 - y1 * zt), l_d * zt * z1
    elif l_n == zero:
        l_n = 3 * x1 * x1
        l_d = 2 * y1 * z1
        return l_n * (xt * z1 - x1 * zt) - l_d * (yt * z1 - y1 * zt), l_d * zt * z1
    else:
        return xt * z1 - x1 * zt, z1 * zt
```
TODO: replace small x with big X etc.

## 1.4 Adapting the Miller Loop
Now we have to adapt the implementation of the Miller loop to take into account the separate calculation of numerator and denominator of the line function. Since the output of the line function always occurs only in connection with the variable `f`, it is sufficient to consider these lines. Thus, we start by splitting `f` into numerator and denominator:
```python
def miller_loop(Q,P):
  T = Q
  f_n, f_d = FQ12.one(), FQ12.one()
```
We proceed analogously with the occurrences in the actual loop:
```python
for i in range(log_ate_loop_count, -1, -1):
  lf_n, lf_d = line_func(T,T,P)
  f_n = f_n * f_n * lf_n
  f_d = f_d * f_d * lf_d

  T = double(T)
  if ate_loop_count & (2**i):
    lf_n, lf_d = line_func(T,Q,P)
    f_n = f_n * lf_n
    f_d = f_d * lf_d
    T = add(T,Q)
```
Lastly, we replace the computations after the loop and merge the results before the final exponentiation:
```python
  Q1 = (Q[0] ** field_modulus, Q[1] ** field_modulus, Q[2] ** field_modulus)
  # assert is_on_curve(Q1, b12)
  nQ2 = (Q1[0] ** field_modulus, -Q1[1] ** field_modulus, Q1[2] ** field_modulus)
  # assert is_on_curve(nQ2, b12)
  _n1, _d1 = linefunc(R, Q1, P)
  R = add(R, Q1)
  _n2, _d2 = linefunc(R, nQ2, P)
  f = f_num * _n1 * _n2 / (f_den * _d1 * _d2)
```

## 1.5 Pairing Function
Finally, we write a wrapper function that we call to calculate the pairing of two points $P$ and $Q$:

```python3
def pairing(Q, P):
    assert is_on_curve(Q, b2)
    assert is_on_curve(P, b)
    if P[-1] == P[-1].__class__.zero() or Q[-1] == Q[-1].__class__.zero():
        return FQ12.one()
    return miller_loop(twist(Q), cast_point_to_fq12(P))
```


# References

[1] https://www.iacr.org/archive/eurocrypt2011/66320047/66320047.pdf

[2] https://crypto.stanford.edu/pbc/thesis.pdf

[3] https://eprint.iacr.org/2008/096.pdf
