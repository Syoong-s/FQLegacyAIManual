# Skill Architecture Audit

## Current architecture

- `SKILL.md`: compact router + high-value invariants.
- `dataset-01-initializer-layout.md`: initializer, exposure-list, dataset-root, product-layout and path-helper contract.
- stage references: detailed algorithms and stage behavior.
- `config-01-parameter-map.md`: parameter-oriented lookup.
- `variant-01-full-vs-lite.md`: feature boundary and shared I/O contract.
- deployment references: full runner vs minimal Direct HPC.
- `development-01-change-map.md`: requested behavior → source/interface → verification.
- measurement references: auxiliary analysis scope isolated from the main F77 runtime.

## 1.4.0 compatibility audit

The manual was rechecked against the modernized Legacy F77 reference state on 2026-09-21, including the compatibility/layout migration and subsequent Lite synchronization. The audit found several material stale assumptions in 1.3.0:

1. **Initializer missing from the manual.** The current pipeline has a single `init_program.py` replacing the older split decompression/list-generation workflow.
2. **Filesystem contract changed.** Current products are organized under a WFST-compatible target tree with exposure-scoped science/DQ/stamp/astrometry paths.
3. **Full/Lite filesystem behavior converged.** Lite no longer uses a separate old legacy layout in the current reference implementation.
4. **Centralized path API was undocumented.** `path_layout.inc` and `fq_*_product_path` helpers now carry the product-path contract across stage code.
5. **Stage-1 DQ/F6 description was stale.** Current code loads DQ before background/F6 and marks DQ pixels invalid in `weight`; `set_sig` consumes that weight mask even though it does no DQ file I/O itself.
6. **Path capacity was stale.** Current reference `strl` is 512 rather than the older 150 value.
7. **External catalog prefix was missing.** Current reference defines `SOURCE_CAT_TILE_PREFIX='extern_'`.
8. **Build dependency guidance was stale.** Current Full/Lite Makefiles include `path_layout.inc` in the executable dependency set.
9. **Deployment examples did not surface the current initializer output.** The natural initialized F77 entry list is `<output-root>/expo_<target>.list`.

## 1.4.0 design decisions

The fix deliberately does **not** turn the skill into a pinned copy of one repository revision. Instead:

- current compatibility facts are documented as a reference contract;
- exact editing still requires inspecting the user's live source tree;
- initializer/layout knowledge is isolated in one reference rather than duplicated across every stage file;
- the top-level router only gains enough invariants to prevent old-layout and old-DQ assumptions;
- Full/Lite feature differences remain separate from their now-shared I/O contract.

This keeps the low-context architecture while making the manual usable with the current Pipeline layout.
