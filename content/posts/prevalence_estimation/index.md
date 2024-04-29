+++
title = 'Stochastic Model for Population Estimation on Social Platforms'
date = '2022-07-01'
readTime = 'true'
math = 'true'
+++

When I was at Snapchat, I worked on a team that was responsible for detecting and mitigating bad actors on the platform (aka spam detection). One of the challenges we faced was estimating how many bad actors were active on the platform.

This was not a trivial problem because if we knew how many bad actors were active, it meant we had a way to detect them, and if we could detect them, we could've removed them to begin with... So we needed a way to take an "educated guess."

During my tenur, I came across a [white paper](https://arxiv.org/abs/2211.03754) published by the manager of my team from a previous generation. The model was already in production but lacked maintenance and explanations (no one knew how it worked), hence I took a stab at dusting it off and documenting it.

This technique was interesting because it could be applied to many other problems, not just spam detection. For example, in ecology, you could use it to estimate the population of a species without counting every single individual (usually an impossible task).

_Original white paper - [A Stochastic Model for Estimating the Number of Offenders and Targets on Snapchat Platform by Vasyl Pihur](https://arxiv.org/abs/2211.03754)_

## Introduction

**TPDAU** (Targets Per **DAU**) is a probabilistic model outlined in the paper above aimed at estimating how many accounts have been affected by bad actors with regards to **DAU** on a social platform. Where the term "affected" is loosely defined as someone that’s been reached by bad actors (offenders). **OPDAU** (Offenders Per **DAU**) is a similar term that describes the number of bad actors with regards to **DAU**.

This model does not take into consideration the "intensity", or the degree to which the victims have been affected. Someone subject to a thousand spam messages is regarded as a single target (victim). Furthermore, no distinctions are made between the "type" of attacks. Common sense tells us bullying or harassment are worse than receiving friend requests from spammers, but both are weighted the same in this model.

**TPDAU** is a factor of two variables: **how many unique accounts are reported** and **how many times did each account get reported**. Common sense tells us that more unique accounts reported means more users affected, but also, more reports per account also means more users affected. If we count how many accounts were reported a total of 1, 2, 3, ... times for a particular day, we can expect the graph to look like this:

![intro_chart](intro_light.webp#light)
![intro_chart](intro_dark.webp#dark)

Where most offenders are reported once, some are reported twice, and so on. **The model is affected by this curve. Longer tails or taller headers means more people are affected**. The intuition is that 100 targets affected is the same regardless of whether 1 or 100 offenders did it.

## Explanation

Both **TPDAU** and **OPDA** are just the number of targets and offenders divided by the **DAU** for that particular day. Let's define the number of targets and offenders as $T$ and $O$ respectively, to keep the language consistent, then:

$$
TPDAU = \frac{T}{DAU}
$$

$$
OPDAU = \frac{O}{DAU}
$$

Let $f(k)$ be the probability that offenders are reported exactly $k$ times and let $g(n)$ be the probability that an offender has reached exactly $n$ targets. If we know $f(k)$ for all possible values of $k$, call it $P(K)$, then the number of reports we receive is just:

$$
R = O \cdot P(K)
$$

$K$ starts from 1, since offenders with 0 reports don’t contribute to $R$. So we have (much easier to compute):

$$
R = O(1 - f(0)) \tag{1}
$$

Where $R$ is the total number of user reports, and if we can figure out $f(0)$, we can find $O$, the number of offenders. Assuming we found $O$, and that we know $g(n)$, then there are $g(1) \cdot O$ offenders who’ve each reached 1 user for a total of $g(1) \cdot O \cdot 1$ targets. Similarly, there are $g(2) \cdot O$ offenders who’ve each reached 2 users and a total of $g(2) \cdot O \cdot 2$ targets. So we can find $T$ by summing through all possible $n$. i.e.,

$$
T = \sum_{n=1}^{n_{max}} n \cdot g(n) \tag{2}
$$

_($n\_{max}$ is the theoretical limit of how many targets an offender can reach)_

$𝑛$ also starts from 1 since an offender needs to have attacked at least 1 user to be considered an offender (registration spams that idle after activation are not considered), but note that the reason $𝑛$ starts from 1 is different than the reason why $𝐾$ starts from 1. The former is by definition whereas the latter is not (there are plenty of offenders with zero user reports).

The model makes two assumptions (both of these assumptions are empirically observed and discussed in the white paper. I won’t discuss that here):

1. $𝑔(𝑛)$ follows a [Power Law Distribution](https://en.wikipedia.org/wiki/Power_law) that looks like the graph in the background section. i.e. $𝑔(𝑛) = 𝑐 \cdot 𝑛^𝑠$ for some unknown constants $𝑐$ and $𝑠$.

2. The number of reports we get from an offender that’s reached exactly $𝑛$ targets follows a [Binomial Distribution](https://www.google.com/search?client=safari&rls=en&q=Binomial+Distribution&ie=UTF-8) for some constant $𝑝$, where $𝑝$ is the probability that the target will report the offender.

## Formulate $g(n)$

Because $𝑛 \in N$ rather than $𝑛 \in \mathbb{R}^{+}$, 𝑔(𝑛) is a [Zeta Distribution](https://www.google.com/search?client=safari&rls=en&q=Zeta+Distribution&ie=UTF-8) (a more specific type of the power law) in the form of:

$$
g(n) = \frac{n^{-s}}{\zeta(s)}
$$

Where $s$ is the power constant from before and $\zeta(s)$ is the [Riemann Zeta Function](https://en.wikipedia.org/wiki/Riemann_zeta_function):

$$
\zeta(s) = \sum_{i=1}^{\infty} i^{-s}
$$

But since our $n$ is not infinite (offenders don’t have infinite users to target), we have (called a [Zipf’s Law](https://simple.wikipedia.org/wiki/Zipf%27s_law#:~:text=Zipf's%20law%20is%20an%20empirical,rank%20in%20the%20frequency%20table)):

$$
g(n, s, n_{max}) = \frac{n^{-s}}{\sum_{i=1}^{n_{max}} i^{-s}} \tag{3}
$$

So we have formulated $𝑔(𝑛)$, which is really $𝑔(𝑛, 𝑠, n_{max})$ where $𝑠$ and $n_{max}$ are two constants for us to discover.

## Formulate $f(k)$

Because the number of reports from an offender that’s reached exactly $n$ targets follows a binomial distribution (assumption 2). The probability of receiving exactly $k$ reports from $n$ targets is $Bernoulli(k, n, p)$ where $p$ is the probability that a target will report the offender. Furthermore, for such $n$, we know $g(n)$, the probability that an offender has reached $n$ targets. So the probability of getting exactly $k$ reports is $Bernoulli(k, n, p) \cdot g(n)$ which is the $f(k)$ we’re after!

However, since $n$ isn't fixed (not all offenders have exactly $n$ targets), we'll need to sum this up for all possible $n$. Using the formulated $g(n, s, n\_{max})$, we obtain:

$$
f(k) = \sum_{n=1}^{n_{\text{max}}} \text{Bernoulli}(k_0 | n, p) \cdot g(n) = \sum_{n=1}^{n_{\text{max}}} \binom{n}{k} \cdot p^k (1 - p)^{n-k} \cdot g(n)
$$

We can write this as

$$
f(k; s, n_{max}, p) = \sum_{n=1}^{n_{max}} \binom{n}{k} \cdot p^k(1 - p)^{n-k} \cdot \frac{n^{-s}}{\sum_{i=1}^{n_{max}} i^{-s}} \tag{4}
$$

So we now have $f(k)$, which is really $f(k; s, n\_{max}, p)$, and we now have one extra constant to discover and that is $p$.

## Estimate $s$, $n\_{max}$, and $p$

The model uses the [Maximum Likelihood](https://en.wikipedia.org/wiki/Maximum_likelihood_estimation#:~:text=In%20statistics%2C%20maximum%20likelihood%20estimation,observed%20data%20is%20most%20probable.) method (MLE) to estimate $s$, $n_{max}$, and $p$. The intuition is that we first assume the data we observe does follow our distribution $f(k; s, n_{max}, p)$. We then write the probability of observing this exact set of data in the form of $f(k; s, n_{max}, p)$ for every $k$. i.e.,

For all $k \in M$, where $M$ is the set of observed $k$ from the data collected:

$$
P(M) = f(k; s, n_{max}, p) =\prod_{k \in M} f(k; s, n_{max}, p)
$$

where $P(M)$ is the probability of getting the set observed.

We then find $s$, $n_{max}$, and $p$ that maximizes $P(M)$ by calculating the derivative of $P(M)$ with respect to $s$, $n_{max}$, and $p$. In practice, this is computationally expensive, so we take the logarithm of $P(M)$ because it allows us to transform the function expressed in the form of its product to its summations. Hence:

$$
\ln P(M) = \sum_{k \in M} \ln f(k; s, n_{max}, p)
$$

Which is much easier to compute. This is fine because $\ln(x)$ is a monotonically increasing function, and so maximizing its value is the equivalent of maximizing the original function.

The model calculates this maximum by using [L-BFGS-B](https://bergant.github.io/nlexperiment/flocking_bfgs.html), which to be honest, is an abomination to look at...

## Putting it together

We found $s$, $n\_{max}$, and $p$, we can now calculate $f(0)$ from (4), the probability of offenders that are never reported. Which is:

$$
f(0; s, n_{max}, p) = \sum_{n=1}^{n_{max}} (1 - p)^n \cdot \frac{n^{-s}}{\sum_{i=1}^{n_{max}} i^{-s}}
$$

We also have $R$ for free, the total number of reports. From (1), we have:

$$
O = \frac{R}{f(0; s, n_{max}, p)}
$$

Substituting this into (2) and (3), we have the final form:

$$
T = \frac{R}{f(0; s, n_{max}, p)} \cdot \sum_{n=1}^{n_{max}} n \cdot \frac{n^{-s}}{\sum_{i=1}^{n_{max}} i^{-s}}
$$

## Notes

- **TPDAU** is the ratio of estimated targets and our **DAU** (same with **OPDAU**), but our **DAU** estimation is the total number of daily active users minus the accounts we think are fake. We don’t have a precise way of measuring the number of fake accounts (note that offenders are not necessarily fake accounts) and so hence the ratio may fluctuate. Indeed, the **DAU** will almost always be a shrinking factor since the number of fake accounts we can measure is always strictly less than the truth.

- **TPDAU** is inaccurate when $p$, the probability that a target will report the offender, is small. Empirical data shows that $p$ needs to be above $0.01$ for the estimations to be reliable, **but higher the better**:

![notes_chart](notes_light.webp#light)
![notes_chart](notes_dark.webp#dark)

- The model assumes that the amount of fake user reports are negligible. This may not always be true. The model also assumes that the users report offenders by the correct report reasons, which also may not be true (e.g. some users will report something as "I don’t like it", which is ultimately excluded from this calculation).
