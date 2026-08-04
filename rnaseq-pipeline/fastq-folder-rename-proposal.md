# fastq-folder-rename-proposal

Confirmed: two Akata dirs (`fastq/1_24hr`, `fastq/2_48hr`) have `cntl`/`test` but no `input_args.json`, so per instruction they're excluded — no row for Akata at all.

For the rest, the key issue isn't cosmetic — the directory that actually becomes `experiment_id` (the one directly holding `cntl`/`test`/`input_args.json`) is often a generically-named leaf (`fastq`, `1_fastq`) nested under a descriptive but functionally irrelevant wrapper folder (`2_Mutu_polyA_Zta_24hrs_ribodepletion`, etc.). The wrapper name is currently doing all the descriptive work, but the pipeline never reads it.

## Final scheme: `<cell_line>_<agent>_<date>`

Simplified per feedback: timepoint, library_selection, co_treatment etc. don't need to be baked into the folder name since they're already queryable columns in `input_args.json` / `metadata.tsv` / the warehouse. The name only needs to be unique enough to serve as `experiment_id`.

| Cell line | Current path (relative to `1_RNA_seq/`) | Current fastq-root name | Proposed new name |
|---|---|---|---|
| SNU719 | `SNU719/SNU719_Rta-Zta_2025-04-10` | `SNU719_Rta-Zta_2025-04-10` | *(already conforms — no rename needed)* |
| Mutu | `Mutu/2_Mutu_polyA_Zta_24hrs_ribodepletion/1_fastq` | `1_fastq` | `Mutu_Zta_2019-12-20` |
| Mutu | `Mutu/1_Mutu_polyA_Zta_24hrs_polyA_stranded/fastq` | `fastq` | `Mutu_Zta_2020-04-02` |
| Mutu | `Mutu/4_Mutu_polyA_BCR_24hrs_stranded/1_fastq` | `1_fastq` | `Mutu_anti-IgM_2020-09-16` |
| Mutu | `Mutu/3_Mutu_polyA_Zta-plus-or-minus-PAA_24hrs_polyA/1_no_PAA_Zta_vs_cntl/1_fastq` | `1_fastq` | `Mutu_Zta_2020-11-26` |
| Mutu | `Mutu/3_Mutu_polyA_Zta-plus-or-minus-PAA_24hrs_polyA/2_PAA_Zta_vs_cntl/1_fastq` | `1_fastq` | `Mutu_Zta-PAA_2020-11-26` |

Naming logic:
- Model + descriptor, single first underscore, matching the `SNU719_Rta-Zta_...` reference. Descriptor = `perturbation_agent` from `input_args.json` (or `anti-IgM` for the BCR-crosslink row, since agent is more specific than `perturbation_type`), then the real `sequencing_run_date`.
- The one real collision: the two PAA-comparison folders share cell_line, agent (`Zta`), and the same real `sequencing_run_date` (`2020-11-26`) — only `co_treatment` (`none` vs `PAA`) tells them apart, so date alone can't disambiguate them even once real dates are used. Resolved by appending `-PAA` only to the row that has it; the `none` row stays plain.
- Every other pair already has distinct real dates, so no other qualifiers (timepoint, library_selection, etc.) are needed for uniqueness.
- Only the leaf directory itself (the one with `cntl`/`test`) gets renamed, not its ancestor wrapper folders — nothing downstream reads those, so there's no need to flatten the tree.

This was a proposal table only — no renames were performed.
