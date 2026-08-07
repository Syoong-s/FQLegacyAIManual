# Measurement: χ² Minimization & Quadratic Fitting (Core)

## Purpose (Stage 6)

Find the best-fit mean shear $c_{\text{best}}$ that minimizes the χ² sign test statistic, and estimate its statistical uncertainty. This is done in three steps: coarse interval search, fine grid sampling, and quadratic fitting at the minimum.

## Step 1: Coarse Interval Clamping Search

### Initial Interval

$[c_1, c_2] = [-0.2, +0.2]$

### Iterative Narrowing

Repeat until no further narrowing is possible:

1. Compute the midpoint: $cc = (c_1 + c_2)/2$.
2. Evaluate χ² at three points: $(c_1 + cc)/2$, $cc$, $(c_2 + cc)/2$, giving values $v_1, v_c, v_2$.
3. **Left-side rejection**: If $v_1 > v_c + \text{thresh}$ (with $\text{thresh} = 20$, corresponding to ≈ 3σ $\Delta\chi^2$), discard the left half: $c_1 \leftarrow (c_1 + cc)/2$.
4. **Right-side rejection**: If $v_2 > v_c + \text{thresh}$, discard the right half: $c_2 \leftarrow (c_2 + cc)/2$.
5. If neither side is rejected (`change = 0`), the interval is converged.

The threshold of 20 ensures only statistically significant side-lobes are discarded, guarding against prematurely narrowing the interval due to noise fluctuations.

## Step 2: Fine Grid Sampling

Within the narrowed interval $[c_1, c_2]$, uniformly sample $\texttt{NMAX} = 200$ points:

$$c_i = c_1 + \frac{i-1}{\texttt{NMAX}-1}(c_2 - c_1), \qquad i = 1, \ldots, \texttt{NMAX}$$

For each $c_i$, compute $\chi^2(c_i)$ via the sign test. This produces 200 $(c_i, \chi^2_i)$ data points centered around the minimum.

## Step 3: Quadratic Fitting

### Model

$$\chi^2(c) = a_1\, c^2 + a_2\, c + a_3$$

### Least-Squares Solution

The 200 data points define a $3 \times 3$ normal equation system. For numerical stability, the independent variable is first shifted and scaled:

$$u = \frac{c - cc}{b}, \qquad cc = \frac{c_1 + c_2}{2}, \qquad b = \frac{c_2 - c_1}{2}$$

The normal equations are solved via Cramer's rule (3×3 determinant method). After solving for coefficients in $u$-space, they are transformed back to the original $c$ coordinate.

### Best-Fit and Uncertainty

From the fitted quadratic coefficients:

$$c_{\text{best}} = -\frac{a_2}{2\, a_1}$$

$$\sigma_c = \frac{1}{\sqrt{2\, a_1}}$$

### Uncertainty Derivation

The uncertainty formula follows from the standard likelihood-ratio approach under a quadratic approximation. Let $\chi^2(c)$ be twice the negative log-likelihood (up to an additive constant). The 1σ confidence interval satisfies:

$$\chi^2(c_{\text{best}} + \delta) - \chi^2(c_{\text{best}}) = 1$$

At the minimum, $\chi^2(c) \approx a_1 (c - c_{\text{best}})^2 + \text{const}$, so:

$$a_1 \delta^2 = 1 \quad\Rightarrow\quad \delta = \frac{1}{\sqrt{a_1}}$$

However, the code uses $\sigma_c = 1/\sqrt{2a_1}$. This corresponds to $\Delta\chi^2 = 1/2$ rather than $\Delta\chi^2 = 1$, which is the appropriate normalization when the χ² statistic is defined with a factor of $1/2$ in the denominator (as it is here: $\sum D^2/(2N)$). Equivalently, the effective number of degrees of freedom is approximately 2 per bin (positive and negative counts), and the $1/(2N)$ normalization scales the curvature accordingly.

## Summary: Three-Step Pipeline for Each Spatial Bin

```
[c1, c2] = [-0.2, 0.2]
→ Clamping search narrows interval around minimum
→ 200-point uniform grid evaluates χ²(c) across narrowed interval
→ Quadratic fit yields c_best and σ_c
→ Output: (gf_center, c_best, σ_c) for this spatial bin
```

This is repeated for each of the 21 spatial bins, independently for the g1 and g2 components.