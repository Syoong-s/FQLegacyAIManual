# Measurement: Python Post-Processing & Calibration

## Purpose (Stage 7)

Transform the per-bin best-fit shear values into calibrated multiplicative and additive bias parameters, and manage the batch parameter search workflow.

## Fortran Output

The combined results for all spatial bins are written to `FD_test_comb.dat` by rank 0:

| Column | Content |
|:---:|:---|
| 1 | $N_1$ — source count in g1 spatial bin |
| 2 | $gf_1$ bin center value |
| 3 | $c_1$ — best-fit mean shear (g1 component) |
| 4 | $\sigma_1$ — uncertainty on $c_1$ |
| 5 | $N_2$ — source count in g2 spatial bin |
| 6 | $gf_2$ bin center value |
| 7 | $c_2$ — best-fit mean shear (g2 component) |
| 8 | $\sigma_2$ — uncertainty on $c_2$ |

## Python Weighted Linear Calibration

### Model

The measured galaxy shear $g_{\text{GAL}}$ is fit as a linear function of the field distortion shear $g_{\text{FD}}$:

$$g_{\text{GAL}} = a \cdot g_{\text{FD}} + b$$

### Method

Using `scipy.optimize.curve_fit` with per-bin $\sigma$ as weights (`absolute_sigma=True`).

### Bias Parameters

From the linear fit:

- **Multiplicative bias**: $m = a - 1 \pm \sigma_a$
- **Additive bias**: $b \pm \sigma_b$
- **Shape noise** (per-component): $\sigma_{\text{shape}} = \sqrt{N} \cdot \sigma$, where $N$ is the per-bin source count and $\sigma$ is the measured uncertainty. This gives the shape noise per galaxy (the uncertainty-weighted dispersion).

### Output

Two-panel comparison plot (g1, g2 subplots), each showing:
- Data points with error bars ($g_{\text{GAL}}$ vs $g_{\text{FD}}$)
- $y = x$ reference line (perfect calibration)
- Weighted linear fit with $m, b$ labeled

## Batch Parameter Search Driver

### Workflow

The Python driver `control.py` iterates over a grid of analysis parameters:

- SNR thresholds (low/high)
- Star/galaxy classification thresholds
- PSF χ² cut thresholds
- Other quality-cut parameters

### For Each Parameter Set

1. Generate `para.inc` from a template file `para.inc.temp` with `{placeholder}` substitution.
2. Generate `bad_ccds.inc` (list of known-bad CCDs to exclude).
3. Compile: `mpif77 *.f -o main -mcmodel=medium -w`
4. Run: `mpiexec -n 200 main <expo_list> <output_dir>`
5. Call Python calibration routine on the output.
6. Rename `FD_test_comb.dat` to a parameter-tagged unique filename (e.g. `FD_test_SNR_LOW_5.0_..._default.dat`).

### Purpose

The batch search systematically explores the parameter space to find the optimal quality cuts that minimize multiplicative and additive bias while retaining sufficient statistical power. The renamed output files allow retrospective comparison of all parameter combinations without recomputation.