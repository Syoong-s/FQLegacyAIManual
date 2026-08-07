# Measurement Program: Architecture & Execution Model

> **Scope:** auxiliary documentation; this program's source was not part of the reference material used to build this manual. Use `measurement-00-scope.md` before treating filenames/defaults below as executable source truth, and verify against the user's actual auxiliary source when available.

## Overview

This program takes the per-exposure shear catalogs (`*_all.cat`) produced by the pipeline and measures the mean shear signal as a function of field distortion. It operates on the principle that field distortion $(gf_1, gf_2)$ — a known quantity from the astrometric solution — provides a proxy for "input shear." By recovering the galaxy shear response in bins of $gf$, the program calibrates the multiplicative and additive bias of the shear estimator.

## Execution Model

**Hybrid MPI-parallel Fortran 77 + Python driver:**

- **Fortran core**: Manager-worker MPI model distributes exposures across nodes. Each node reads one exposure's shear catalog, builds local data arrays, and participates in collective statistics.
- **Python driver** (`control.py`): Iterates over a grid of quality-cut parameters (SNR thresholds, star classification cuts, χ² thresholds, etc.), generates `para.inc` from a template, compiles the Fortran, runs MPI, collects output, and renames result files to prevent overwrites.

## Processing Stages

| Stage | Function | Core Algorithm |
|:---|:---|:---|
| 1 | Data Loading | Distributed read of shear catalogs, quality filtering, variable construction |
| 2 | Spatial Binning | Bin sources by field distortion value $gf$ into $N_{\text{bin}}$ equal-width bins |
| 3 | Equal-Probability Binning | Within each spatial bin, sort sources by $|g|$ and compute equal-probability inner bin boundaries |
| 4 | Boundary Iteration | Iteratively adjust inner bin boundaries so each bin has approximately equal source count |
| 5 | χ² Sign Test | For each trial $c$, compute the non-parametric sign-test χ² statistic measuring distribution symmetry |
| 6 | χ² Minimization | Coarse interval search + fine grid sampling + quadratic fitting to find best-fit $c$ and uncertainty $\sigma$ |
| 7 | Output & Python Calibration | Write combined results, weighted linear fit for multiplicative/additive bias |

## Key Global Parameters

| Parameter | Default | Meaning |
|:---|:---:|:---|
| `fd_num` | 21 | Number of spatial bins (by $gf$ value) |
| `PDF_BINS` | 4 | Number of inner equal-probability bins (including overflow) |
| `gf_lim` | 0.0015 | Spatial bin range: $\pm g_{\text{f,lim}}$ |
| `NMAX` | 200 | Fine grid sampling points / quadratic fitting points |
| `nmax_per_core` | $2 \times 10^7$ | Maximum sources per MPI node |

## Compilation & Execution

- **Compile**: `mpif77 *.f -o main -mcmodel=medium -w` (inside WSL2)
- **Run**: `mpiexec -n <N> main <expo_list> <output_dir>`
- Exact compilation environment must be taken from the actual auxiliary program checkout when code-level work is requested.