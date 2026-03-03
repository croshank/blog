---
title: "Temporal-Priority Action Tokenization for Generative Controllers"
date: 2026-03-03
---

Autoregressive policies for robotic control predict action sequences as discrete, sequentially decoded tokens. To train such policies, continuous action signals must be tokenized into sequences of discrete symbols. State-of-the-art tokenization schemes such as the DCT-based [FAST](https://arxiv.org/abs/2501.09747) exploit smooth structure in action signals to represent a full horizon with fewer tokens. However, this compression comes at a cost: all tokens spanning a horizon must be decoded before execution over that horizon can begin, a requirement that penalizes reactive control when inference time is comparable to the execution horizon.

At the other extreme, per-timestep binning (the Dirac basis) provides perfect temporal priority — early tokens encode near-term actions exactly — but sacrifices compression, inflating the token count and with it the inference burden.

In this post, we summarize our analysis of this compression–localization tradeoff and present a wavelet-based tokenization scheme with temporal-priority ordering (DWT-TP) that navigates it favorably relative to both extremes.

## The prefix reconstruction loss

Action chunk policies typically operate in a receding-horizon fashion, executing only a prefix of each chunk before replanning with a fresh observation. In this regime, the quality of *partially-decoded* predictions over the immediate execution window determines the controller's reactive capability.

We define the **prefix reconstruction loss** at execution horizon $W$ and decode rate $r$:

$$\mathcal{L}(W, r) = \frac{1}{W} \sum_{k=1}^{W} \| u[k] - \hat{u}_{W,r}[k] \|^2$$

where $\hat{u}_{W,r}[k]$ is the reconstruction at timestep $k$ from the coefficients recoverable from the first $\lfloor W \cdot r \rfloor$ decoded BPE tokens.

Lower $\mathcal{L}(W, r)$ at small $W$ means the agent can sustain tighter replan loops without fresh predictions being unusably coarse.

## Analysis under a parametric signal model

We analyze the expected prefix loss for each of three bases — DCT, Dirac (per-timestep), and DWT-TP (wavelet with temporal-priority ordering) — under a stationary signal with power spectrum $S(k) = k^{-\beta}$, where larger $\beta$ corresponds to smoother trajectories.

### DCT prefix loss

The DCT approximately diagonalizes the covariance of a stationary process, so coefficient energies are $\mathbb{E}[c_m^2] \approx m^{-\beta}$. Since each DCT basis function distributes energy uniformly across the time axis, the expected prefix loss after $K$ coefficients is:

$$\mathbb{E}\left[\mathcal{L}^{\text{DCT}}(K)\right] \approx \frac{1}{H} \sum_{m=K}^{H-1} m^{-\beta}$$

This is essentially **independent of $W$**: whether the execution prefix is short or long, the per-timestep error is the same. Only $K$ matters.

### Dirac prefix loss

The Dirac basis coefficients are the raw signal values. After $K$ coefficients are decoded in temporal order, the reconstruction is exact for $k \leq K$ and zero for $k > K$:

$$\mathbb{E}\left[\mathcal{L}^{\text{Dirac}}(K)\right] = \frac{\max(W - K, 0)}{W} \cdot \sigma^2$$

This is linear in the fraction of the prefix not yet decoded. Once $K \geq W$, the prefix loss is exactly zero.

### DWT-TP prefix loss

For the Haar wavelet basis, coefficient energy at scale $j$ satisfies $E_j \propto 2^{-j(\beta - 1)}$, independent of location by stationarity. Under temporal-priority ordering, the first $K$ coefficients are those with earliest-starting support. The expected prefix loss is:

$$\mathbb{E}\left[\mathcal{L}^{\text{DWT-TP}}(K)\right] = \frac{1}{W} \sum_{j=0}^{J-1} 2^{-j(\beta-1)} \cdot \max\!\left(0, \;\left\lceil \frac{W}{2^{J-j}} \right\rceil - n_j(K)\right)$$

where $n_j(K)$ is the number of scale-$j$ coefficients among the first $K$ decoded. Coefficients whose temporal support lies entirely beyond $W$ contribute exactly zero — they can be dropped with no effect on the prefix reconstruction.

### Analytical comparison

Each basis decodes $K_B = \lfloor W \cdot \alpha_B \cdot R \rfloor$ coefficients per execution horizon, where $\alpha_B$ models the per-basis compression efficiency ($\alpha_{\text{DCT}} = \alpha_{\text{Haar}} = 1$, $\alpha_{\text{Dirac}} = 0.5$, reflecting the Dirac basis's poorer BPE compression).

![Analytical prefix loss for the three bases under a 1/f^β power spectrum](/blog/images/acttoken/analytical_prefix_loss.png)

*Prefix loss $\mathcal{L}(W)/\sigma^2$ (%) for the three bases with $H=32$. At moderate decode rates ($R \geq 1.25$), DWT-TP achieves lower prefix loss than both DCT and Dirac across most execution horizons. However, at lower rates or steeper spectra ($\beta = 1.5$), DCT's superior energy concentration in the leading coefficients begins to edge out DWT-TP.*

Three observations emerge:

1. At moderate decode rates, **DWT-TP achieves the lowest prefix loss across most execution horizons**, at comparable compression rates to DCT while delivering temporal priority that Dirac cannot match without similar compressive performance.

2. **DCT is strongest at very small $W$**, where its first few high-energy low-frequency coefficients provide a competitive coarse approximation despite lacking localization.

3. **The advantage of DWT-TP narrows as $\beta$ increases**: steeper spectra concentrate more energy in the leading DCT coefficients, reducing the penalty of frequency-first ordering.

## Empirical results on six manipulation datasets

We evaluate on six datasets spanning 7 to 40 action dimensions and 5 to 50 Hz control rates, at a decode rate of $R = 3$ BPE tokens per timestep.

| Dataset | $d$ | Hz | $H$ | DCT tokens | DWT-TP tokens | Dirac tokens |
|---------|-----|----|------|------------|---------------|--------------|
| Fanuc | 7 | 10 | 10 | 19 | 7 | 43 |
| DROID | 7 | 15 | 15 | 20 | 27 | 76 |
| Cable Routing | 7 | 10 | 10 | 21 | 22 | 37 |
| NYU Rot | 7 | 5 | 5 | 5 | 4 | 16 |
| UTree Fold | 40 | 50 | 50 | 333 | 472 | 1221 |
| UTree Warehouse | 40 | 50 | 50 | 366 | 483 | 1094 |

![Prefix reconstruction loss vs. execution horizon for six datasets](/blog/images/acttoken/paper_batch_mse_1x_3x2.png)

*Prefix reconstruction loss $\mathcal{L}(W, R{=}3)$, normalized by signal energy, vs. execution horizon $W$ for six datasets at native control rate. Lower is better. DWT-TP (red) achieves the lowest prefix loss at early horizons across all six datasets; DCT (blue) narrows the gap or overtakes only as $W$ approaches the full chunk length $H$.*

Across all six datasets, DWT-TP achieves lower prefix loss than DCT at early execution horizons, confirming the analytical prediction. On Fanuc, the mean prefix loss over the first half of the horizon drops from 13.1% with DCT to 4.7% with DWT-TP — a **64% reduction** — and DWT-TP maintains a lead across all horizons. NYU Rot shows a 38% reduction with no DCT crossover. On DROID, DWT-TP reduces the mean by 34%; DCT narrows the gap and overtakes at $W^*{=}9$. Cable Routing exhibits a steady 23% reduction throughout.

On the high-dimensional Unitree datasets ($d{=}40$), DWT-TP reduces the mean prefix loss by 16% (UTree Fold) and 14% (UTree Warehouse). DCT overtakes at $W^*{=}33$ on both, reflecting its superior compression on these high-dimensional datasets.

| Dataset | DCT $\bar{\mathcal{L}}_{\text{half}}$ | DWT-TP $\bar{\mathcal{L}}_{\text{half}}$ | Reduction | $W^*$ |
|---------|-------|--------|-----------|-------|
| Fanuc | 13.1% | 4.7% | 63.9% | -- |
| DROID | 18.4% | 12.2% | 33.5% | 9 |
| Cable | 32.7% | 25.2% | 23.0% | -- |
| NYU Rot | 20.8% | 12.9% | 38.1% | -- |
| UTree Fold | 41.2% | 34.7% | 15.7% | 33 |
| UTree Wh | 33.1% | 28.4% | 14.2% | 33 |

*$\bar{\mathcal{L}}_{\text{half}}$: mean prefix loss over $W \in [1, H/2]$. $W^*$: execution horizon at which DCT overtakes DWT-TP (-- if DWT-TP leads throughout).*

## Summary

At a realistic decode rate of $R=3$, DWT-TP consistently achieves the lowest prefix loss at short to moderate execution horizons on all six datasets. The mean prefix loss over the first half of the execution horizon is reduced by **14–64%** relative to DCT. On three of six datasets, DWT-TP leads throughout the entire horizon; on the remaining three, DCT overtakes only at $W^* \geq 9$, well beyond the typical replanning window. For practitioners operating in a receding-horizon regime with $W/H \leq 0.5$, DWT-TP offers a consistent advantage over DCT as a drop-in replacement in the FAST tokenization pipeline.