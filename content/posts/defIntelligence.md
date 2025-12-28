---
title: "On the Definition of Intelligence"
date: 2025-12-28
---

## Ontology
We proceed with a description of the universe as a dynamical system. The universal state $u\in\mathcal U$ evolves under an evolutionary law
$$
\dot u = f_\mathcal{U}(u).
$$
The state space $\mathcal U$ and the existence of time is implicit in such a description. We work under a discretization of time and a finite discretization of the state space $\mathcal U$, an ontological burden we impose to borrow tooling from Computer Science. As a result, the evolutionary law is rewritable as a state transition across a temporal unit
$$
u_+ = f_{\mathcal U} (u).
$$
We must unfortunately bear some more burden. In that we assume a decomposition of the universal state space $\mathcal U = \mathcal S \oplus \mathcal E$ which we shall call the system and the environment whereby $u = (s, e)$ for $s\in\mathcal S$, $e\in\mathcal E$ and the evolution decomposes as
$$
s_+ = f_{\mathcal S}(s, e)
$$
$$
e_+ = f_{\mathcal E}(s, e)
$$
Let $T\subset \mathbb Z$ be a finite interval on the integers which can be supposed as $\{1, ..., N\}$ without loss of generality for our purposes. Define the trajectories of the system and the environment respectively on this interval as
$$
s_T = (s_1, ..., s_N),
$$
$$
e_T = (e_1, ..., e_N).
$$

## Recast as a Computation
Note that $\mathcal S$ and $\mathcal E$ are discrete and finite and fix the bijective encodings
$$
\text{enc}_{\mathcal S}: \mathcal{S}\to \{0, 1\}^{\lceil\log_2m\rceil},
$$
$$
\text{enc}_{\mathcal E}: \mathcal{E}\to \{0, 1\}^{\lceil\log_2n\rceil}.
$$
Encode trajectories $s_T$ and $e_T$ via concatenation:
$$
\text{enc}(s_T):= \text{enc}_{\mathcal S}(s_1)\cdot...\cdot \text{enc}_{\mathcal S}(s_N);
$$
$$
\text{enc}(e_T):= \text{enc}_{\mathcal E}(e_1)\cdot...\cdot \text{enc}_{\mathcal E}(e_N)
$$
each $N\log m$ and $N\log n$ bits long with perhaps some overhead $\mathcal O(\log N + \log m)$ (to allow encoding N, m and avoid decoding ambiguity).

## Preliminary: Kolmogorov Complexity
The above reformulation of the universal evolution as an automatic process on finite strings allows us to utilize the machinery provided by the Theory of Computation in defining Intelligence.

Fix a universal Turing machine $U$ and recall the Kolmogorov complexity in its absolute and conditional forms:
$$
K(x) := \min\{|p|:U(p)=x\},
$$
$$
K(y|x) := \min \{|p|: U(p, x)= y\}
$$
respectively describe the minimum description length such that $U$ outputs $x$ and minimum additional description length atop $x$ such that $U$ outputs $y$.

## Intelligence
We define intelligence as a property of a systemic trajectory $s_T$ evaluated as a function on the pair $(s_T, e_T)$:
$$
\mathcal I(s_T):= K(e_T) - K(e_T|s_T).
$$
Intelligence is thus the reduction in the descriptive complexity of the environmental trajectory when the systemic trajectory is known.
