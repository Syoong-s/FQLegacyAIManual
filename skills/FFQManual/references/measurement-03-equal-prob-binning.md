# Measurement: Equal-Probability Binning & Boundary Iteration

## Purpose (Stages 3 & 4)

Within each spatial bin, sources are further divided into inner "equal-probability" bins based on $|g|$ (absolute measured shear). This ensures the χ² sign test has balanced statistical weight across amplitude ranges, maximizing sensitivity.

## Stage 3: Equal-Probability Boundary Initialization

### Inner Bin Count

$n_{\text{bin}} + 1 = \texttt{PDF\_BINS} = 4$ bins (3 boundaries + 1 overflow bin at high $|g|$). The parameter passed to the statistics routine is $n_{\text{bin}} = \texttt{PDF\_BINS} - 1 = 3$.

### Boundary Calculation

1. Take $|y_k|$ (absolute measured shear) for all sources in the spatial bin.
2. Sort in ascending order (using a hybrid quicksort + insertion sort).
3. Compute the quantile spacing: $dn = n / (n_{\text{bin}} + 1)$, where $n$ is the per-node source count.
4. Set initial boundaries at the sorted values:

$$x_b^{(i)} = |y|_{(\lfloor dn \cdot i \rfloor)}, \qquad i = 1, \ldots, n_{\text{bin}}$$

### Cross-Node Averaging

Since each MPI node holds only a subset of sources for the spatial bin, boundaries from all effective nodes are averaged:

$$\bar{x}_b^{(i)} = \frac{1}{P_{\text{eff}}} \sum_{p \in \text{eff}} x_{b,p}^{(i)}$$

where $P_{\text{eff}}$ is the number of nodes with sufficient sources ($n \ge n_{\text{bin}} + 1 = 4$). Nodes with too few sources are marked ineffective and excluded.

The averaged boundaries are broadcast to all nodes via `MPI_Bcast`.

### Design Note

Boundaries are based on $|y_k|$ (not $|r_k| = |y_k - c \cdot de_k|$), making them independent of the trial value $c$. This means boundaries are computed once and reused for all $c$ values during χ² scanning. When $c \neq 0$, bins are no longer strictly equal-probability, but the sign test remains valid and the balanced source counts maintain consistent statistical weight.

## Stage 4: Boundary Iteration for Balance

After cross-node averaging, the per-bin source counts may not be exactly equal. An iterative adjustment ensures approximate balance.

### Imbalance Metric

$$v^{(i)} = \frac{N^{(i)}}{N_{\text{tot}}} - \frac{1}{n_{\text{bin}}+1}$$

where $N^{(i)}$ is the global source count in bin $i$ (via `MPI_AllReduce` sum) and $N_{\text{tot}}$ is the total.

### Convergence Criterion

If $|v^{(i)}| < 0.03$ for a given boundary, it is considered converged and skipped.

### Iterative Adjustment Logic

For non-converged boundaries (max 20 iterations):

- **$v > 0$** (bin $i$ has too many sources): Need to lower boundary to push sources into bin $i+1$. Search interval: $[x_{b1}, x_{b2}]$ where $x_{b2} = x_b^{(i)}$ (current), $x_{b1}$ = midpoint with previous boundary (linear extrapolation for $i=1$).
- **$v < 0$** (bin $i$ has too few sources): Need to raise boundary to pull sources from bin $i+1$. Search interval: $[x_{b1}, x_{b2}]$ where $x_{b1} = x_b^{(i)}$ (current), $x_{b2}$ = midpoint with next boundary (linear extrapolation for $i=n_{\text{bin}}$).

Within each search interval:
1. Evaluate $v$ at both endpoints ($v_1, v_2$).
2. If $v_1 v_2 < 0$ (zero crossing bracketed): Narrow the interval toward the side with larger $|v|$ (weight 0.75 : 0.25).
3. If $v_1 v_2 > 0$ (same sign): Extend the search outward (double the step + fine adjustment).

The final converged boundaries form the `xbin` array shared via common block.