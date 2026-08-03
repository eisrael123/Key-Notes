# Experiment metadata schema review

Review of the new experiment-level metadata schema Ethan and his PI drafted (fields:
`CELL_LINE`, `PERTURBATION_TYPE`, `PERTURBATION_AGENT`, `PERTURBATION_DOSE`, `CO_TREATMENT`,
`CO_TREATMENT_TARGET`, `FACS_PURIFIED`, `FACS_GFP_PROMOTER`, `LIBRARY_SELECTION`,
`LIBRARY_STRANDEDNESS`, `LIBRARY_TYPE`, `TIMEPOINT_HOURS`, `SEQUENCING_RUN_DATE`, `NOTES`),
checked against the current pipeline code (`vocab.py`, `metadata.py`) and the DuckDB warehouse
design (`/Volumes/TUNGSACore3/database_skeleton/SCHEMA.md`).

## Issues found, ranked by importance

1. **No `PERTURBATION_TARGET` field.** `CO_TREATMENT` correctly pairs with
   `CO_TREATMENT_TARGET` (e.g. `CO_TREATMENT=siRNA`, `CO_TREATMENT_TARGET=CNOT1`), which keeps
   "every CNOT1 knockdown" queryable. But `PERTURBATION_AGENT` has no equivalent target slot —
   if the *primary* perturbation is itself an siRNA/CRISPR knockdown, there's nowhere to record
   which gene. Add `PERTURBATION_TARGET`, mirroring `CO_TREATMENT_TARGET`. (The current pipeline
   already has this as `perturbation_target` — don't drop it.)

2. **`none`/`unknown`/`NA` semantics applied inconsistently.** `vocab.py` already documents the
   principle (for `INDUCED_PROGRAMS`): `unknown` = unrecorded value, `none` = recorded absence.
   New schema gives `CO_TREATMENT` and `FACS_PURIFIED` an `unknown` option but not
   `PERTURBATION_TYPE` or `LIBRARY_SELECTION`. Apply the same principle to every field.

3. **`PERTURBATION_TYPE` vocab drifted from code.** Current `vocab.py`:
   `transfection, siRNA, drug, BCR-crosslink, none`. New schema:
   `transfection, chemical, BCR-crosslink, none`. `drug`→`chemical` is fine; `siRNA` dropping out
   as a *type* (folded into `PERTURBATION_AGENT` instead) is plausible but needs to be confirmed
   as intentional before updating `vocab.py`.

4. **`LIBRARY_STRANDEDNESS` will collide in meaning with the existing `strandedness` column.**
   The pipeline's per-sample `metadata.tsv` already has `strandedness`, populated from RSeQC's
   *empirical* inference post-run (e.g. `fr-firststrand`). The new `LIBRARY_STRANDEDNESS` looks
   like *design intent* (what was ordered at library prep) instead. Keep both — a mismatch
   between them is a useful QC signal — but document the distinction explicitly so it isn't
   read as a duplicate column later.

5. **`genome_build`/`organism` missing from the experiment-level table.** Both are fixed per
   experiment and already asked once in `metadata.py`. The archive includes mouse builds
   (`mm39`, `mm39plusMHV68`) as well as human, so this isn't safe to assume/omit.

6. **`CELL_LINE` vocab is ahead of the code.** New list adds `YCCEL1` and `P3HR1`, not yet in
   `vocab.py::CELL_LINES`. One-line fix once the list is finalized.

## Side question: backfilling the already-processed SNU719 run

`SNU719_Rta-Zta_2025-04-10` was processed under the old (current-code) schema before this
update. Conclusion: **backfill, don't rerun.**

- All the new fields (`CO_TREATMENT`, `CO_TREATMENT_TARGET`, `FACS_PURIFIED`,
  `FACS_GFP_PROMOTER`, `PERTURBATION_AGENT`, `LIBRARY_STRANDEDNESS`) are facts about
  experimental design, not anything derived from FASTQ/BAM data — nothing about them touches
  alignment, DESeq2, rMATS, or any other compute stage.
- No DuckDB warehouse has actually been built yet — `/Volumes/TUNGSACore3/database_skeleton/`
  has no `duckdb` install, only a SQLite preview (`preview_R68fecadda929.sqlite`) built by a
  fallback script (`_sandbox_preview.py`). So there's no live table to migrate.
- What's needed: hand-edit `metadata.tsv` (and the investigator-stamped copy) in that
  experiment's output directory to add the new columns with known values (cell line, "Rta+Zta"
  as agent, FACS-purified y/n, etc.) — a few minutes of editing, or a small one-off script once
  the new columns are finalized.
