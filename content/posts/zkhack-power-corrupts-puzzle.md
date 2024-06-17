---
title: 'ZK Hack - Power Corrupts Puzzle'
date: 2024-05-31T14:44:26+02:00
draft: false
math: true
summary: In this article, we implement a Cheon attack in Rust to solve the "Power Corrupts" puzzle, contributed by Stanford's Applied Cryptography group. The focus is on the mathematical intricacies of the attack.
cover: 
  image: /img/zkhack/white-logo.png
---

## 1 Puzzle Description

> Bob has invented a new pairing-friendly elliptic curve, which he wanted to use with Groth16.
> For that purpose, Bob has performed a trusted setup, which resulted in an SRS containting
> a secret $\tau$ raised to high powers multiplied by a specific generator in both source groups.
> The exact parameters of the curve and part of the output of the setup are described in the
> document linked below.
> 
> Alice wants to recover $\tau$ and she noticed a few interesting details about the curve and
> the setup. Specifically, she noticed that the sum $d$ of the highest power $d_1$ of $\tau$ in
> $\mathbb{G}_1$ portion of the SRS, meaning the SRS contains an element of the form
> $\tau^{d_1} G_1$ where $G_1$ is a generator of $\mathbb{G}_1$, and the highest power $d_2$
> of $\tau$ in $\mathbb{G}_2$  divides $q-1$, where $q$ is the order of the groups.
> 
> Additionally, she managed to perform a social engineering attack on Bob and extract the
> following information: if you express $\tau$ as $\tau = 2^{k_0 + k_1((q-1/d))} \mod r$,
> where $r$ is the order of the scalar field, $k_0$ is 51 bits and its fifteen most
> significant bits are 10111101110 ($15854$ in decimal). That is $A < k_0 < B$ where
> $A = 1089478584172543$ and $B = 1089547303649280$.
> 
> Alice then remembered the Cheon attack...
- from the Puzzle README.md on [GitHub](https://github.com/ZK-Hack/puzzle-power-corrupts)

Note: There's a little quirk in the puzzle's description. It mentions the first 15 bits of $k_0$ (which is supposed to be a $51$ bit long integer), but the supplied bitstring is only $11$ bits long. Moreover, this bitstring is equal to the decimal $1518$, not $15854$. However, $15854$ is $14$ bits long. It turns out that $k_0$ is actually $50$ bits long and we should use the decimal representaton of $15854$ as the most significant bits of this length $50$ bitstring. In this way we obtain the lower bound mentioned in the text plus one, i.e $A + 1$.
```rust
let MSBs = format!{"{:b}", 15854};
let padded_bits = format!("{:0<50}", MSBs);
let Ap1 = u64::from_str_radix(&padded_bits, 2).unwrap();
assert_eq!(1089478584172543, Ap1 - 1);
```

## 2 Solution
### 2.1 Cheon's Discrete Logarithm Attack
[Cheon](http://www.math.snu.ac.kr/~jhcheon/publications/2010/StrongDH_JoC_Final2.pdf) proposed an algorithm for the discrete logarithm problem (DLP) with auxiliary inputs. In general, the DLP for a group $\mathbb{G}$ is defined as: For a given input $(g, g^\tau) \in \mathbb{G}^2$, compute $\tau \in \mathbb{F}_q$. The auxiliary input in Cheon's algorithm refers to the knowledge of an additional group element $g^{\tau^d}$. Suppose $g, g^{\tau}, g^{\tau^d} \in \mathbb{G}$ are given for a divisor $d$ of $q-1$. Let $$\tau = \zeta^{k_0 + k_1 \frac{q-1}{d}}$$ for $0 \leq k_0 < \frac{q-1}{d}$ and $0 \leq k_1 < d$ for a primitive element $\zeta$ of $\mathbb{F}_q^{\times}$. Then the secret element $\tau \in \mathbb{F}_q$ can be computed deterministically by a succession of two rounds of Baby-step Giant-step in $\mathcal{O}(\sqrt{q/d} + \sqrt{d})$ exponentiations and by using $\mathcal{O}(\max{(\sqrt{q/d}, \sqrt{d})})$ storage. In the following we will make use of this attack to solve the puzzle.

### 2.2 Puzzle Setup
We are [given](https://gist.github.com/kobigurk/352036cee6cb8e44ddf0e231ee9c3f9b) the elements:
- $P, \tau P$ and $\tau^{d_1} P$ of the group $\mathbb{G}_1$, and
- $Q$ and $\tau^{d_2} Q$ of the group $\mathbb{G}_2$.

Here, $P$ and $Q$ are generators of the respective groups. We only know that $d_1 + d_2$ divides $q - 1$, so the requirements for Cheon's attack are not fulfilled. However, since the given curve is pairing-friendly we can make use of the bilinear pairing $$e: \mathbb{G}_1 \times \mathbb{G}_2 \to \mathbb{G}_T$$ to transform the problem to obtain suitable group elements. First, we set $g := e(P, Q).$
With this we obtain $$e(\tau P, Q) = e(P, Q)^{\tau} = g^{\tau}.$$
Now, remember Cheon's attack requires us to have an element $g^{\tau^{d}}$, such that $d | (q- 1)$. For this we let $d:= d_1 + d_2$ and use the points $\tau^{d_1}P$ and $\tau^{d_2}Q$: $$e(\tau^{d_1}P, \tau^{d_2}Q) = e(P, Q)^{\tau^{d_1+d_2}} = e(P,Q)^{\tau^d} = g^{\tau^d}.$$
Having obtained the three group elements $g, g^{\tau}$ and $g^{\tau^d}$ we are ready to implement Cheon's attack. 
According to Alice, we can write $$\tau = 2^{k_0 + k_1\frac{q-1}{d}} \pmod{q},$$
where $0 \leq k_0 < \frac{q-1}{d}$ and $0 \leq k_1 < d$. This means that $\zeta := 2$ is a primitive element of $\mathbb{F}_q^{\times}$, i.e. $2$ generates $\mathbb{F}_q^{\times}$. Any non-zero element $\alpha \in \mathbb{F}_q$ can be written as $2^i$ for some $i \in \mathbb{N}$. Since $d$ divides $q-1$, the expression $k_0 + k_1\frac{q-1}{d}$ yields a natural number. We are now going to compute $k_0$ and $k_1$ to recover $\tau$. This is done by confining the search space using the two subgroups $H_1$ and $H_2$ to be defined in the next sections. 

### 2.3 Solving the DLP in $H_1$
The first step of Cheon's attack is to compute $k_0$ using BSGS. Luckily, we already know the first $14$ bits of $k_0$, so we only need to find the remaining $36$ bits. We use the knowledge of $\tau$'s representation from Alice to restrict the search space as follows:
$$\tau = 2^{k_0 + k_1 \frac{q-1}{d}} \implies \tau^d = 2^{dk_0 + k_1(q-1)} = 2^{dk_0}(2^{k_1})^{q-1} = (2^{d})^{k_0}.$$
Here we used Fermat's Little Theorem to obtain $(2^{k_1})^{q-1} = 1 \pmod{q}$. So, in short this means that finding $k_0$ is equivalent to solving the DLP given by the equation $(2^{d})^{k_0} = \tau^d$.

Moreover, this DLP is restricted to the subgroup[^1] $H_1 := \\{x^d | x \in \mathbb{F}_q^{\times} \\},$ i.e. the elements of $d$-th powers in $\mathbb{F}_q^{\times}$. The order $|H_1|$ of this subgroup is $\frac{q-1}{d}$, which is easy to show[^2].

Using BSGS we can solve this DLP in $H_1$ by making a space-time tradeoff, splitting the remaining $2^{36}$ possibilities in half. To account for the known $14$ bits stored in $A$ we re-write $k_0$ as $$k_0 = A + i\cdot 2^{18} + j,$$where $0 \leq i,j \leq 2^{18}$. Therefore, we have $$\tau^d = (2^d)^{k_0} = (2^d)^{A + i\cdot 2^{18} +j} = (2^d)^{A+i\cdot 2^{18}}(2^d)^{j}.$$
We re-arrange the terms a bit more to obtain the common BSGS formula:  $$\tau^d(2^{-d})^{A+i\cdot 2^{18}} = (2^d)^{j}.$$
In this expression the right side corresponds to the "baby steps", whereas the left side are the "giant steps". There's just one problem: Unfortunately, we don't know $\tau^d$, we only know $g^{\tau^d}$, that's why another exponentiation with $g$ is needed: 
$$g^{\tau^d}g^{(2^{-d})^{A+i\cdot 2^{18}}} = (g^{2^d})^{j}.$$

With this setup, we can finally use BSGS to determine a value for $k_0$, which turns out to be equal to $1089539821761426$.
### 2.4 Solving the DLP in $H_2$
In the final step, we'll use $k_0$ to determine $k_1$. This time we operate in the subgroup $H_2 := \\{x^{\frac{q-1}{d}} | x \in \mathbb{F}_q^{\times} \\}$ generated by the element $2^{\frac{q-1}{d}}$. This group is of order $d$ and $d$ has $30$ bits, so we start with the following split of $k_1$:
$$k_1 = i\cdot 2^{15} + j,$$
with $0 \leq i,j \leq 2^{15}$. Keeping this in mind we go on to formulate the DLP in $H_2$:
$$\tau = 2^{k_0 + k_1\frac{q-1}{d}} = 2^{k_0} \cdot (2^{\frac{q-1}{d}})^{k_1}.$$
We once again re-arrange this into the typical BSGS formula and exponentiate with $g$ to obtain:
$$(g^{\tau})^{2^{-k_0}} (g^{-{2^{\frac{q-1}{d}}}})^{i\cdot 2^{15}} = (g^{2^{\frac{q-1}{d}}})^{j}$$
Again, the right side corresponds to the “baby steps” and the left side corresponds to the “giant steps”. Using these parameters for BSGS we obtain $k_1 = 690599720$.

### 2.5 Baby-Step Giant-Step Implementation
In the BSGS implementation we utilize the functions supplied in the `utils.rs` file, as well as the relevant traits from the `ark-ff` crate. To calculate the "baby steps" we can conveniently use the supplied `pow_sp2` function, which returns a `HashMap<S, u64>` object with values $p^{\text{exp}}, p^{\text{exp}^{2}}, ...., p^{\text{exp}^{n}}$. Additionally, we are introducing a `split` parameter to adjust the number of "baby steps", and accordingly the "length" of the "giant steps". For example, for the calculation of $k_0$ above we decided on letting `split = 1 << 18` ($2^{18}$), but depending on the machine other values may improve performance.

```rust
fn baby_step_giant_step<S: Field>(a: S, b: Fr, group_ord: u64, mut giant: S, split: u64) -> Option<u64> {
    let baby_steps = pow_sp2(a, b, split);
    let giant_step = pow_sp(b.inverse().unwrap(), split.into(), 64).into_bigint();
    let n = group_ord / split;

    for i in 0..n {
        if let Some(j) = baby_steps.get(&giant) {
            // return Some(i * split + (*j));
            return Some(i * split + (*j));
        }
        giant = giant.pow(giant_step);
    }
    None
}
```
In summary, the first round of BSGS is run with `baby_steps` $= (g^{2^{d}})^j$, `giant` $= (g^{\tau^d})^{(2^{-d})^A}$ and `giant_step` $= (2^{-d})^{2^{18}}$. 

```rust
let A: u64 = 1089478584172544;
let two_pow_d = pow_sp(Fr::from(2u64), d.into(), 31);
let giant = g_tau_d.pow(pow_sp(two_pow_d.inverse().unwrap(), A.into(), 51).into_bigint());

// Solve DLP in H_1, i.e. find k0
let k0 = match baby_step_giant_step(g, two_pow_d, q1_d, giant, 1 << 20) {
    Some(result) => A + result,
    None => {
        eprintln!("Error: BSGS failed to find k0.");
        std::process::exit(1);
    }
};
```
The second round of BSGS is run with `baby_steps` $= (g^{2^{\frac{q-1}{d}}})^j$, `giant` $= (g^\tau)^{2^{-{k_0}}}$ and `giant_step` $= (2^{-\frac{q-1}{d}})^{2^{18}}$.
```rust
let two_pow_q1_d = pow_sp(Fr::from(2u64), q1_d.into(), 51);
let giant = g_tau.pow(pow_sp(Fr::from(2u64).inverse().unwrap(), k0.into(), 51).into_bigint());
// Solve DLP in H_2, i.e. find k1
let k1 = match baby_step_giant_step(g, two_pow_q1_d, d, giant, 1 << 16) {
    Some(k1) => k1,
    None => {
        eprintln!("Error: BSGS failed to find k1.");
        std::process::exit(1);
    }
};
```
### 2.6 Recovering $\tau$
With the computed values for $k_0$ and $k_1$ we can now finally compute $\tau$:
$$ \tau = 2^{k_0 + k_1 \frac{q-1}{d}} = 284865198031253921498207.$$
```rust
// Calculate τ = 2^(k0 + k1 (q-1)/d)
let exp = (k0 as u128) + (k1 as u128) * (q1_d as u128);
let tau = pow_sp(Fr::from(2u64), exp, 80);
```
## 3 Complete Rust Implementation
```rust
use ark_bls12_cheon::{Bls12Cheon, Fq12, Fr, G1Projective as G1, G2Projective as G2};

use crate::utils::{bigInt_to_u128, pow_sp, pow_sp2};
use ark_ec::pairing::Pairing;
use ark_ff::{Field, PrimeField};

const d1: u64 = 11726539;
const d2: u64 = 690320833;
const d: u64 = d1 + d2;
const q: u128 = 1114157594638178892192613;
const q1_d: u64 = ((q - 1) / d as u128 ) as u64;

pub fn attack(P: G1, tau_P: G1, tau_d1_P: G1, Q: G2, tau_d2_Q: G2) -> i128 {
    // Using pairing to obtain g := e(P,Q), g^tau := e(tau P,Q), g^tau^d = e(tau^d P, Q)
    let g: Fq12 = Bls12Cheon::pairing(P, Q).0;
    let g_tau: Fq12 = Bls12Cheon::pairing(tau_P, Q).0;
    let g_tau_d: Fq12 = Bls12Cheon::pairing(tau_d1_P, tau_d2_Q).0;

    let A: u64 = 1089478584172544;
    let two_pow_d = pow_sp(Fr::from(2u64), d.into(), 31);
    let giant = g_tau_d.pow(pow_sp(two_pow_d.inverse().unwrap(), A.into(), 51).into_bigint());
    // Solve DLP in H_1, i.e. find k0
    let k0 = match baby_step_giant_step(g, two_pow_d, q1_d, giant, 1 << 20) {
        Some(result) => A + result,
        None => {
            eprintln!("Error: BSGS failed to find k0.");
            std::process::exit(1);
        }
    };
    println!("Found k0 = {}", k0);

    let two_pow_q1_d = pow_sp(Fr::from(2u64), q1_d.into(), 51);
    let giant = g_tau.pow(pow_sp(Fr::from(2u64).inverse().unwrap(), k0.into(), 51).into_bigint());
    // Solve DLP in H_2, i.e. find k1
    let k1 = match baby_step_giant_step(g, two_pow_q1_d, d, giant, 1 << 16) {
        Some(k1) => k1,
        None => {
            eprintln!("Error: BSGS failed to find k1.");
            std::process::exit(1);
        }
    };
    println!("Found k1 = {}", k1);

    // Calculate τ = 2^(k0 + k1 (q-1)/d)
    let exp = (k0 as u128) + (k1 as u128) * (q1_d as u128);
    let tau = pow_sp(Fr::from(2u64), exp, 80);
    println!("Found τ = {}", tau);

    return bigInt_to_u128(tau.into_bigint()) as i128;
}

fn baby_step_giant_step<S: Field>(a: S, b: Fr, group_ord: u64, mut giant: S, split: u64) -> Option<u64> {
    let baby_steps = pow_sp2(a, b, split);
    let giant_step = pow_sp(b.inverse().unwrap(), split.into(), 64).into_bigint();
    let n = group_ord / split;

    for i in 0..n {
        if let Some(j) = baby_steps.get(&giant) {
            // return Some(i * split + (*j));
            return Some(i * split + (*j));
        }
        giant = giant.pow(giant_step);
    }
    None
}
```
## 4 References
[1] "Cheon's discrete log attack, and its relevance to zk-SNARKs" - https://hackmd.io/2oUhPtzWSRulLQ83Ctoy_g

[^1]: This is because evidently $\tau^d \in H_1$, so $H_1$ is non-empty and for elements $g := x^d, h := y^d \in H_1$ we have $$gh^{-1} = x^d(y^d)^{-1} = x^d(y^{-1})^d = (xy^{-1})^d.$$It follows that $x^d(y^d)^{-1} \in H_1$ and from the One-Step Subgroup Test we conclude, that $H_1$ is indeed a subgroup of $\mathbb{F}_q^{\times}$.
[^2]: This is true, since $\mathbb{F}_q^{\times}$ is a cyclic group of order $q-1$. Let $g$ be a generator of this group, then to obtain the order of $H_1$ it is enough to calculate the order of $g^d$. In other words, we need to find the smallest $n \in \mathbb{N}$, such that $$(g^{d})^n = g^{dn} = 1$$Since $g$ is a generator, the equation implies that $dn$ must be a multiple of the order of the group, which is $q−1$. In other words: $$dn = m(q-1)$$The smallest positive $n$ that satisfies this equation occurs when $n = \frac{q-1}{d}$. This is because $d$ divides $q-1$.
