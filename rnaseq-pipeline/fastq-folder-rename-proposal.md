# fastq-folder-rename-proposal

Confirmed: two Akata dirs (`fastq/1_24hr`, `fastq/2_48hr`) have `cntl`/`test` but no `input_args.json`, so per your instruction they're excluded — no row for Akata at all.

For the rest, the key issue isn't cosmetic — the directory that actually becomes `experiment_id` (the one directly holding `cntl`/`test`/`input_args.json`) is often a generically-named leaf (`fastq`, `1_fastq`) nested under a descriptive but functionally irrelevant wrapper folder (`2_Mutu_polyA_Zta_24hrs_ribodepletion`, etc.). The wrapper name is currently doing all the descriptive work, but the pipeline never reads it.

| Cell line | Current path (relative to `1_RNA_seq/`) | Current fastq-root name | Proposed new name |
|---|---|---|---|
| SNU719 | `SNU719/SNU719_Rta-Zta_2025-04-10` | `SNU719_Rta-Zta_2025-04-10` | `SNU719_Rta-Zta_yyyy-mm-dd` *(already conforms — only the date segment needs a real value)* |
| Mutu | `Mutu/2_Mutu_polyA_Zta_24hrs_ribodepletion/1_fastq` | `1_fastq` | `Mutu_Zta_24hrs_ribodepleted_yyyy-mm-dd` |
| Mutu | `Mutu/1_Mutu_polyA_Zta_24hrs_polyA_stranded/fastq` | `fastq` | `Mutu_Zta_24hrs_polyA_yyyy-mm-dd` |
| Mutu | `Mutu/4_Mutu_polyA_BCR_24hrs_stranded/1_fastq` | `1_fastq` | `Mutu_anti-IgM_24hrs_yyyy-mm-dd` |
| Mutu | `Mutu/3_Mutu_polyA_Zta-plus-or-minus-PAA_24hrs_polyA/1_no_PAA_Zta_vs_cntl/1_fastq` | `1_fastq` | `Mutu_Zta-minus-PAA_24hrs_yyyy-mm-dd` |
| Mutu | `Mutu/3_Mutu_polyA_Zta-plus-or-minus-PAA_24hrs_polyA/2_PAA_Zta_vs_cntl/1_fastq` | `1_fastq` | `Mutu_Zta-plus-PAA_24hrs_yyyy-mm-dd` |

Naming logic:
- Model + descriptor, single first underscore, matching the `SNU719_Rta-Zta_...` reference.
- Descriptor built from `perturbation_agent` (or `anti-IgM` for the BCR-crosslink row, since agent is more specific than `perturbation_type`), plus a distinguishing qualifier so the two `1_fastq` names under the PAA-comparison parent don't collide once dates are real — `co_treatment` (`PAA` vs `none`) was the actual differentiator there, so it's encoded as `-plus-PAA`/`-minus-PAA`, echoing the original wrapper's own `plus-or-minus-PAA` language.
- The `ribodepleted` vs `polyA` qualifier on the first two rows is load-bearing too: same cell line, same agent, same timepoint — `library_selection` is the only field that keeps them from colliding.
- Only the leaf directory itself (the one with `cntl`/`test`) gets renamed, not its ancestor wrapper folders — nothing downstream reads those, so there's no need to flatten the tree.

Two things worth flagging: the `input_args.json` files read here actually already contain distinct, plausible `sequencing_run_date` values (`2019-12-20`, `2020-04-02`, `2020-09-16`, `2020-11-26` x2) rather than one identical current-date value — that doesn't match the "PI forgot to write the actual run dates, all identical to current date" description, so it's worth double-checking whether the PI's edits landed correctly or a different batch was intended. `yyyy-mm-dd` was used as a placeholder regardless, per instruction.

This was a proposal table only — no renames were performed.
