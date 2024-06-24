---
title: "ZK001: Finite Field Arithmetic in Python"
date: 2022-08-16T15:45:13+02:00
draft: false
tags: ["zk-snarks", "python", "mathematics", "groth16"]
summary: In this series of articles we try to implement a toy version of the groth16 zk-snark in python.
math: true
---

{{< color color="red" >}}
> Note: This is a draft article.
{{< /color >}}

Idea of this series: We try to obtain a solid grasp on current zk-snark protocols by implementing groth16 from the ground up. The field has evolved a lot in the past years and for a part-time project it's not feasible to come up with a production-grade implementation. The drawback is that we won't learn much about possible side-channel attacks, but instead we focus more on the mathematical workings of the protocols.

# 1. Mathematical Review
## 1.1 Modular Arithmetic

Given an integer $n > 1$, we say two $a,b \in \mathbb{Z}$ are congruent modulo $n$, if $n$ is a devisor of their difference, that is if there exists an integer $k$, such that $kn = a - b$. This defines an equivalence relation $\equiv_n \subseteq \mathbb{Z}\times\mathbb{Z}$. A little more explicit, this means for any $a,b,c \in \mathbb{Z}$:
- $\equiv_n$ is reflexive: $a \equiv_n a$,
- $\equiv_n$ is symmetric: $a \equiv_n b$ if and only if $b \equiv_n a$,
- $\equiv_n$ is transitive: If $a \equiv_n b$ and $b \equiv_n c$ then $a \equiv_n c$.

More commonly, congruence modulo $n$ is denoted as $a \equiv b \pmod{n}$. A few examples: Since $14 - 2 = 12$ which is a multiple of $4$ we have $14 \equiv 2 \pmod{4}$ and since $8^2 - 1 = 63$ is a multiple of $3$ we have $8^2 \equiv 1 \pmod{3}$. The definition applies to negative integers as well: Since $2-(-3) = 1\cdot 5$ we have $2 \equiv -3 \pmod{5}$ and from $-8-7 = -15 = -3 \cdot 5$ it follows that $-8 \equiv 7 \equiv 2 \pmod{5}$.

The equivalence classes of the congruence relation $\equiv_n$ are called congruency classes and are comprised of integers congruent modulo $n$. Sticking with the previous examples, here is the congruence class $\overline{2}$ of 2 modulo 5:
$$\overline{2}_5 = \lbrace ..., -13, -8, -3, 2, 7, 12, 17, ...\rbrace.$$
More generally, for $a \in \mathbb{Z}$ its congruence class modulo $n$ is the set
$$\overline{a}_n = \lbrace a + k\cdot n\\,|\\,k \in \mathbb{Z} \rbrace.$$

## 1.2 Ring of Integers modulo n

For a fixed $n > 0$ we collect the congruence classes modulo $n$ in a set
$$\mathbb{Z}/n \mathbb{Z} := \lbrace \overline{a}_n\\,|\\,a \in \mathbb{Z} \rbrace = \lbrace \overline{0}_n, \overline{1}_n, \overline{2}_n, ..., \overline{n-1}_n \rbrace$$
and equip it with an addition operation $\overline{a}_n + \overline{b}_n := \overline{a + b}_n$ and a multiplication operation $\overline{a}_n \cdot \overline{b}_n := \overline{(a \cdot b)}_n$. Additionally, multiplication is distributive with respect to addition. Together with these two operations the set $\mathbb{Z}/n\mathbb{Z}$ is called the ring of integers modulo $n$. Here $(\mathbb{Z}/n\mathbb{Z}, +)$ is an abelian group and we can therefore define a subtraction operation as $\overline{a}_n - \overline{b}_n := \overline{(a + (-b))}_n$. However, in general $(\mathbb{Z}/n\mathbb{Z}, \cdot)$ doesn't have a group structure. Multiplication is associative, commutative and we have a neutral element $\overline{1}_n$, but not every element has a unique inverse. This is where the trouble starts.

It is customary to suppress the distinction of the congruence class and its elements in practice and we will therefore also relax our notation somewhat in the following. That is instead of $\overline{a}_n$ we are just going to write $a$. An element $a \in \mathbb{Z}/n\mathbb{Z}$ has a multiplicative inverse if and only if $\gcd(a,n) = 1$ (they are coprime). Elements that have a multiplicative inverse are called units, and they are denoted $a^{-1}$. The set of units is typically denoted as
$$(\mathbb{Z}/n\mathbb{Z})^{\times} := \lbrace a \in \mathbb{Z}/n\mathbb{Z}\\,|\\, a \textrm{ has an inverse mod n} \rbrace.$$
Since this set is closed under multiplication, that is the product of two units is again a unit (exercise), it is a group. On the group of units we can finally define a division operation as $a \cdot b^{-1}$.
## 1.3 Finite Fields

From the condition for the existence of a multiplicative inverse follows that as soon as $n = p$, for some prime $p$, every element (except for $0$) has such an inverse. That is
$$ (\mathbb{Z}/p\mathbb{Z})^{\times} = \lbrace 1, 2, 3, ..., p-1 \rbrace.$$
Hence, for every prime $p$ the set $\mathbb{Z}/p\mathbb{Z}$ together with addition and multiplication fulfils the definition of a (finite) field: A set $F$ equipped with an addition operation $+$ and a multiplication operation $\cdot$ is called a field, whenever $(F, +)$ is an abelian group (with neutral element $0$), $(F\setminus{\lbrace 0\rbrace}, \cdot)$ is an abelian group (with neutral element $1$) and $$a\cdot(b + c) = a\cdot b + a\cdot c \\\\ (a + b)\cdot c = a\cdot c + a\cdot c$$ for any $a,b,c \in F$. So $\mathbb{Z}/p\mathbb{Z}$ is a finite field with $p$ elements. We typically denote this field by $\mathbb{F}_p$ or $GF(p)$ for Galois field (after Évariste Galois, who first introduced finite fields and died in a duel during the French revolution of 1830).

In addition to the case described above, for each prime $p$ and every positive integer $k > 1$ there exists a prime field of order $p^k$. Constructing a finite field of order $p^k$ boils down to finding an irreducible polynomial $P$ of degree $k$ over $\mathbb{F}_p[x]$ and forming the quotient $\mathbb{F}_p[x]/P$. We are going to describe this procedure more in depth in the next article. Here we are only concerned with the more boring fields obtained from integers.

# 2. Python Implementation

## 2.1 Field methods: Add, Sub, Mult, Div
In this article we are going to focus on implementing a convenience class for finite field arithmetic. While our implementation is going to be independent of the exact prime order of the field, we will be working over the prime field $GF(q)$ with $q$ set to

```python
q = 21888242871839275222246405745257275088696311157297823662689037894645226208583.
```

We have chosen this modulus because the resulting finite field is the basis for the elliptic curves we will implement later. We start by implementing our class by defining:

```python
class GFq():
    def __init__(self, n):
        self.n = n % q
```
The idea here is that by wrapping a number in the `GFq` class automatically reduces it modulo $q$ using Python's built-in modulo function `%`. Next, we are going to implement the field operations, i.e. addition, subtraction, multiplication, division as well es exponentiation. Let's start with addition. One approach would be to define addition as follows.
```python
def add(self,other):
    return GFq(self.n + other.n)
```
This takes two instances of our class and adds their `n` variables, again wrapping the number in `GFq`. This approach meets our requirements, but it does not allow us to write field addition using infix notation. Infix notation is the common way of writing an operator between two operands, e.g. `2 + 5`. Instead we would have to write `GFq(2).add(GFq(5))`. Fortunately, Python allows us to overload its built-in operators. This can be achieved by overwriting them in a class. The names of these methods start and end with double underscores. Therefore, we need to alter our method in the following way:
```python
def __add__(self,other):
    return GFq(self.n + other.n)
```
Now we can write `GFq(2) + GFq(5)` and obtain a correct result. The subtraction and multiplication methods follow the same layout.
```python
def __mul__(self, other):
    return GFq(self.n * other.n)
def __sub__(self, other):
    return GFq(self.n - other.n)
```
Now, since division in prime fields is defined as $a\cdot b^{-1}$ we need a utility function to find the inverse of any element. The right tool for this task is the extended Euclidean algorithm. For now, we can just copy/paste the pseudo-code from Wikipedia and define a new function as follows.
```python
def calc_inverse(a, q):
    """Use the extended Euclidean algorithm to compute the multiplicative inverse"""
    (t, newt) = (0, 1)
    (r, newr) = (q, a)

    while newr != 0:
        quotient = r // newr
        (t, newt) = (newt, t - quotient * newt)
        (r, newr) = (newr, r - quotient * newr)
    if r > 1:
        error("a is not invertible!")
    if t < 0:
        t = t + q
    return t
```
Note, that this function is not a part of our class. With this we can implement the division operation:
```python
def __truediv__(self, other):
    return GFq(self.n * calc_inverse(other.n, q))
def __div__(self, other):
    return __truediv__(self, other)
```
We overwrite both truediv and div with the same implementation to be able to use the operators `/` (truediv) and `//` (div = floor division) interchangeably. The last operation we need to implement is exponentiation.

## 2.2 Exponentiation by Squaring
The naive approach would be to implement exponentiation simply by successively multiplying the base by itself. However, this yields an inefficient $\mathcal{O}(n)$ algorithm and with a little work we can do better. As an example, let us compute $4^{118} \pmod{1000}$. First, write down the binary expansion of the exponent: $$118 = 2^1 + 2^2 + 2^4 + 2^5 + 2^6$$
Now, we can substitute this back to obtain: $$ 4^{118} = 4^{2^1 + 2^2 + 2^4 + 2^5 + 2^6} = 4^{2^1} \cdot 4^{2^2} \cdot 4^{2^4} \cdot 4^{2^5} \cdot 4^{2^6}$$
The resulting powers $4^{2^i}$ are easily computed as each can be obtained by squaring the preceding power. That is $4^{2^2} = (4^{2^1})^2 = 4^{2\cdot2^1}$, $4^{2^3} = (4^{2^2})^2$, etc. We can collect the powers in a table like the following.
| $i$  | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|---|
| $4^{2^i} \pmod{1000}$ | 4 | 16 | 256 | 536 | 296 | 616 | 456 |

Using the table we can easily compute the desired quantity:
$$4^{118} \equiv 4^{2^1} \cdot 4^{2^2} \cdot 4^{2^4} \cdot 4^{2^5} \cdot 4^{2^6} \equiv 16 \cdot 256 \cdot 296 \cdot 616 \cdot 456 \equiv 736 \pmod{1000}.$$
To create the table we needed 6 multiplications and to get the final result we needed another 4 multiplications, i.e. 10 multiplications in total. If we had followed the naive approach we would have had 117 multiplications. More generally, if we want compute $$g^n = g^{n_0\cdot 2^0 + n_1\cdot 2^1 + n_2 \cdot 2^2 + ... + n_r \cdot 2^r}$$
we need $r$ multiplications to calculate all powers up to $g^{2^r}$. In order to calculate the final result, we need at most another $r$ multiplications. Since $n \geq 2^r$ we have $\log_2(n) \geq r$ and so we obtain a time complexity of $\mathcal{O}(\log_2(n))$, i.e. the number of bits in the binary representation of $n$. This algorithm is known as The Fast Powering Algorithm or Exponentiation by Squaring.

Instead of calculating a table, we implement the algorithm recursively by exploiting the following relationship.
$$x^n = \begin{cases} x(x^{2})^{\frac {n-1}{2}} &\text{if $n$ is odd}\\\\ (x^{2})^{\frac {n}{2}} &\text{if $n$ is even} \end{cases}$$

Using this formula we can recursively remove the least significant digit of the binary representation of $n$. The full implementation looks like this:
```python
def __pow__(self, other):
    """Implements exponentiation by squaring"""
    if other == 0:
        if self != GFq(0):
            return GFq(1)
        else:
            error("undefined: 0^0")
    elif other == 1:
        return GFq(self.n)
    elif other % 2 == 0:
        return (self * self) ** (other //2)
    elif other % 2 == 1:
        return self * (self * self) ** ((other - 1) // 2)
```
## 2.3 Overloading Equality and Printing
Finally, we want to be able to determine if two elements of our class represent the same element of $GF(q)$. For this we follow the same approach as with the other class methods. We overload `==` by rewriting the `__eq__` method.
```python
def __eq__(self, other):
    """Overrides the default implementation"""
    if isinstance(other, GFq):
        return self.n == other.n
    return NotImplemented
```
The function `isinstance(obj, class)` returns `true` if and only if the supplied object `obj` is an instance of `class`. Internally, we reduce the problem to checking equality of two integers. Starting with Python3 this implementation already allows us to use `!=` on objects of `GFq` as well.

Lastly, to aid our intuition we overwrite `__str__` to be able to use `print()`:
```python
def __str__(self):
    return str(self.n)
```
