# Measurement: χ² Sign Test (Core)

## Purpose (Stage 5)

For a given trial mean shear $c$, evaluate the symmetry of the residual distribution around zero using a non-parametric sign test. The best-fit $c$ is the value that makes the signed residuals as symmetric as possible.

## Residual Definition

For each source $j$ in the spatial bin, with measured shear $y_j$ and response $de_j$:

$$r_j = y_j - c \cdot de_j$$

where $y_j$ is the galaxy measured shear (g1 or g2 component), $de_j$ is the corresponding response ($de \mp h_1$), and $c$ is the trial mean shear.

## Binning by Equal-Probability Boundaries

The absolute residuals $|r_j|$ are binned using the pre-computed equal-probability boundaries `xbin` (from Stages 3–4). For each of the $n_{\text{bin}} + 1 = 4$ bins (including the overflow bin for highest $|r|$):

### Per-Bin Counts

$$N^{(b)} = \#\{j: |r_j| \in \text{bin } b\}$$

$$N_+^{(b)} = \#\{j: r_j \ge 0,\ |r_j| \in \text{bin } b\}$$

$$N_-^{(b)} = \#\{j: r_j < 0,\ |r_j| \in \text{bin } b\}$$

The signed count difference:

$$D^{(b)} = N_+^{(b)} - N_-^{(b)}$$

## χ² Sign Test Statistic

$$\chi^2(c) = \sum_{b=1}^{n_{\text{bin}}+1} \frac{\left(D^{(b)}\right)^2}{2\, N^{(b)}}$$

This is a **non-parametric sign test**: it does not assume any distribution shape for the residuals. It only tests whether positive and negative residuals are balanced within each amplitude bin.

### Physical Interpretation

- When $c$ equals the true mean shear, the residual distribution should be symmetric about zero, so $N_+^{(b)} \approx N_-^{(b)}$ in each bin, giving $D^{(b)} \approx 0$ and $\chi^2(c) \to 0$.
- When $c$ is biased (too large or too small), residuals are systematically shifted, positive/negative counts become unbalanced in one or more bins, and $\chi^2$ increases.
- Equal-probability binning ensures each bin has approximately equal statistical weight, giving the test uniform sensitivity across the amplitude range.

### MPI Parallelism

Each non-rank-0 node computes its local $D^{(b)}$ and $N^{(b)}$, then `MPI_Reduce` (sum) aggregates to rank 0. Rank 0 computes $\chi^2$ and `MPI_Bcast` broadcasts to all nodes.

## Why This Test?

The χ² sign test has several advantages over parametric alternatives:

1. **No distribution assumption**: Does not require Gaussian or any other residual distribution — robust to non-Gaussian noise and outliers.
2. **Amplitude-binned sensitivity**: By binning by $|r|$, the test detects asymmetry at all signal-to-noise levels, not just near zero.
3. **Automatic weighting**: The $1/(2N^{(b)})$ normalization naturally accounts for Poisson counting variance, giving less weight to sparsely populated bins.
4. **Computational efficiency**: For each trial $c$, only counting operations are needed — no matrix inversion or iterative fitting.