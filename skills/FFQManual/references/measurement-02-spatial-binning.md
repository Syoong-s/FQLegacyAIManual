# Measurement: Data Loading & Spatial Binning

## Data Loading (Stage 1)

### Distributed Read

Each MPI worker node reads one exposure's shear catalog (`*_all.cat`) via a manager-worker dispatch loop. The catalog is filtered through quality cuts:

- Remove NaN/Inf values
- SNR cut
- Star/galaxy classification cut
- PSF χ² cut
- Flag-based cuts

### Variable Construction

Two parallel variable sets are constructed, one for each shear component:

| Variable | Catalog Column | Meaning |
|:---|:---:|:---|
| `x1` / `x2` | gf1 / gf2 | Field distortion shear (known "input") |
| `y1` / `y2` | g1 / g2 | Galaxy measured shear (Fourier_Quad signal) |
| `de1` | de − h1 | Isotropic response minus h1 correction (for g1) |
| `de2` | de + h1 | Isotropic response plus h1 correction (for g2) |

**Physical rationale**: The response variables differ for the two shear components because of how the rotation from pixel to celestial coordinates combines $de$ and $h_1$. The g1 component uses $de - h_1$; the g2 component uses $de + h_1$.

## Spatial Binning (Stage 2)

### Purpose

Sources are binned by their field distortion value $gf$ to create $N_{\text{bin}}$ spatial bins. Within each bin, the program will later extract the mean galaxy shear, using the known $gf$ as the "true" shear reference.

### Parameters

- $N_{\text{bin}} = \texttt{fd\_num} = 21$
- Range: $[x_{\min}, x_{\max}] = [-\texttt{gf\_lim}, +\texttt{gf\_lim}] = [-0.0015, +0.0015]$

### Binning Formula

$$\Delta x = \frac{x_{\max} - x_{\min}}{N_{\text{bin}}}, \qquad x_{\text{center}}^{(i)} = x_{\min} + \Delta x \left(i - \tfrac{1}{2}\right)$$

Source $j$ is assigned to bin $i$ when:
$$x_{\min} + \Delta x (i-1) \;\le\; x_j \;<\; x_{\min} + \Delta x \cdot i$$

For each spatial bin, a local array $(y_k, de_k)$ and count `is` are constructed. The global count `ntot` is obtained via `MPI_AllReduce` (sum) over all nodes.

### Parallelism

The g1 and g2 components are processed independently, each going through its own spatial binning → equal-probability binning → χ² minimization pipeline, using the same spatial bin parameters but different response variables (de1 vs de2).