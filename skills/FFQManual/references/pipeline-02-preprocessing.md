# Pipeline Stage 1: Pre-processing

> **Lite scope:** current `f77_Lite` freezes `include_FLAT=0` and DQ masking behavior equivalent to Full `include_Mask=2`; alternate Full branches are removed. `CCD_split` and F6 numerical controls remain relevant.

## Purpose

Stage 1 reads a science chip, constructs the base validity mask, applies the configured DQ/flat behavior, estimates/subtracts background, estimates the spatial noise plane with the F6 mode-bar method, prepares astrometry matching data, detects image defects, and writes the normalized chip product used downstream.

## Current call-order invariant

The current reference implementation loads the DQ image **before** `set_background` and `set_sig`.

Simplified flow:

```text
read science FITS
  -> initialize array / normap / weight
  -> optional Full flat handling
  -> load DQ mask when active
       nonzero DQ -> weight = 0
  -> set_background(...)
  -> set_sig(..., weight, ...)
  -> prepare astrometry data
  -> locate_defects / merge_defects
  -> serialize normalized output
```

This differs from older documentation that described DQ as a post-F6 operation.

## DQ mask path

Current stage code constructs the DQ input using the centralized exposure+physical-CCD helper:

```text
dqmask/<exposure>/<exposure>_<CCDNUM>.fits
```

This is intentionally different from the science basename sequence number. The initializer uses physical `CCDNUM` for DQ outputs.

When DQ is active, every pixel with nonzero DQ value is rejected by setting `weight(i,j)=0`. The current code also checks that the DQ image exists and has the same dimensions as the science image.

## Background subtraction

`set_background` operates per amplifier when `CCD_split=2`, otherwise over the configured region as a whole. The current reference uses `blocksize=200` as the background block side.

The exact polynomial/background implementation should be read from current `pre_process.f` before numerical changes.

## F6 noise-plane estimator

`set_sig` receives `normap` and `weight`, estimates a per-amplifier spatial noise plane, normalizes the chip, and returns the plane coefficients via `sigabc`.

### Critical DQ clarification

`set_sig` itself does **not** open or parse a DQ FITS file. Its internal code is therefore DQ-format-agnostic. However, in the current Stage-1 call order, `weight` has already been modified by DQ masking. `set_sig` skips pixels/triples whose weight is nonpositive.

Therefore:

- true: `set_sig` performs no direct DQ file I/O;
- false for current Stage 1: “F6 is unaffected by DQ”;
- changing whether DQ is applied before or after `set_sig` changes the eligible sample and is a numerical behavior change.

### Current numerical outline

The reference F6 implementation uses robust block statistics to seed a brightness mode/noise scale, builds a private symmetric clipping mask, and fits a spatial plane from surviving pixel triples. Important controls live in `sig_para.inc`, including:

- `sig_blocksize=200`
- `sig_clip_k=3.0`
- `sig_rdil=2`
- `sig_clip_niter=2`
- `sig_min_fit_triples=1000`
- `sig_max_plane_ratio=4.0`
- active `sig_scale=1.027786`

Do not use `sig_scale` to hide a changed selection/masking algorithm; a selection change requires numerical re-validation.

## Astrometry preparation

After noise/background processing, Stage 1 writes per-chip astrometry matching data through the current layout helper:

```text
astrometry/dat_Astro/<exposure>/<chip-prefix>_astro.dat
```

Full may use `ASTROMETRY_trivial=1`; Lite freezes the Gaia-based path.

## Defect detection

Current flow runs `locate_defects` and `merge_defects` after the astrometry-data preparation call, using the accumulated `weight`/normalized image state. When debugging missing sources or unexpected masks, inspect both DQ rejection and later defect rejection.

## Normalized output

The normalized science product is routed by `DIR_NORM='stamps/Norm'`:

```text
stamps/Norm/<exposure>/<chip-prefix>_norm.fits
```

The Stage-1 noise-plane coefficients remain serialized according to the current implementation so downstream source processing can recover the required noise information.

## Configuration ownership

Use `para.inc` for branch selectors, geometry and high-level thresholds; `sig_para.inc` for F6 numerical controls; `path_layout.inc` for product directory tokens.

For current dataset naming and initializer behavior, load [dataset-01-initializer-layout.md](dataset-01-initializer-layout.md).

## Safe modification checklist

When changing Stage 1:

1. inspect the live DQ load location relative to `set_background` and `set_sig`;
2. verify `weight` semantics at every call;
3. keep physical `CCDNUM` DQ naming distinct from science sequence numbering;
4. preserve Full/Lite synchronization for shared DQ/path/F6 changes;
5. compile both variants for shared code changes;
6. run at least one chip/exposure and compare the normalized image, F6 coefficients, astrometry input, and mask counts.

> Source-of-truth note: before a code or parameter change, verify the current `pre_process.f`, includes, and layout helpers in the user's actual tree.
