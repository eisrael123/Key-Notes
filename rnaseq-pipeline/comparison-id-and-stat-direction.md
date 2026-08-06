# comparison-id-and-stat-direction

How `comparison_id` is built in `rnaseq.py`/`rnaseq_helper_scripts`, and what a positive vs.
negative statistic means in each queryable stats table (`de_gene.tsv`, `transcript_de.tsv`,
`splicing_event*.tsv`/`splicing_summary.tsv`, `gene_set_enrichment.tsv`).

## Comparison IDs

Built by `RunContext.comparison_id` (`rnaseq_helper_scripts/outputs.py:76-78`):

```python
f"{self.experiment_id}__{self.test_group}_vs_{self.cntl_group}"
```

In practice this is **always** `{experiment_id}__test_vs_cntl` — `build_run_context()` in
`rnaseq.py:2055-2070` constructs `RunContext` without overriding `test_group`/`cntl_group`, so the
dataclass defaults (`"test"`, `"cntl"`) are used every time. There's no multi-comparison support
per run; one experiment = one fixed test-vs-cntl comparison. (The only place a *different*
comparison_id string gets parsed is `tables_rmats.py`'s standalone reprocessing `main()`, not the
normal pipeline path.)

Every stats table keys on `(run_id, comparison_id, ...)` — see `schemas.py:153,208,230,253,282,303`.

## Direction convention, table by table

All three differential tables land on the **same convention**: positive = higher in test, negative
= higher in cntl. But each gets there independently via its own tool's factor-releveling or BAM
group assignment — worth knowing so a bug in any one path wouldn't be assumed to affect the others.

**`de_gene.tsv` (DESeq2) — `log2_fold_change`**
`metadata.py`'s `condition_from_subdir()` emits lowercase `"cntl"`/`"test"`.
`deseq2_analysis_ercc.R:97` does `as.factor(condition)` with **no explicit relevel**, so R falls
back to alphabetical ordering — `"cntl"` sorts first and becomes the reference level.
`results(dds)` then reports the last level vs. the reference: `log2(test / cntl)`.
→ positive = more expressed in test, negative = more expressed in cntl.

**`transcript_de.tsv` (Sleuth) — `b` / `test_stat` (Wald rows only; LRT rows carry `NA`, per
`schemas.py:218`)**
`create_sleuth_metadata()` in `rnaseq.py:533-535` emits `"control"`/`"test"`, and
`sleuth_analysis_ercc.R:58` **explicitly** relevels `ref = "control"`. The Wald test extracts
coefficient `'conditiontest'` (`sleuth_analysis_ercc.R:103,137`), the effect of the test level
relative to control.
→ positive `b` = more expressed in test, negative = more expressed in cntl. Same convention as
DESeq2, independently arrived at.

**`splicing_event*.tsv` / `splicing_summary.tsv` (rMATS) — `inc_level_difference`**
rMATS is invoked with `--b1` = test BAMs, `--b2` = control BAMs (`tables_rmats.py:27-33`, comment
flags this explicitly as a place a silent inversion could sneak in). `GROUP_COLUMNS` maps
`IncLevel1`→test, `IncLevel2`→cntl, and `inc_level_difference` is rMATS' own
`IncLevelDifference = IncLevel1 - IncLevel2`.
→ positive = higher inclusion in test, negative = higher inclusion in cntl. `splicing_summary.tsv`
pre-splits this into `sig_higher_inclusion_test` / `sig_higher_inclusion_cntl`
(`tables_rmats.py:162-163`) so no sign reasoning is needed there at all.

**`gene_set_enrichment.tsv` (GSEA) — `es`/`nes`**
Not set independently — inherits the DESeq2 convention. The `.rnk` file (`rnaseq.py:809-813`) is
sorted by `log2FoldChange` descending, so test-upregulated genes sit at the top of the ranked list.
→ positive NES = gene set enriched among test-upregulated genes, negative NES = enriched among
cntl-upregulated genes.

## One inconsistency worth flagging

`metadata.py` writes a *different*, capitalized `Condition` column (`"test"`/`"control"`,
`metadata.py:524`) alongside the lowercase `condition` column (`"test"`/`"cntl"`). DESeq2's
`generate_deseq2_metadata()` prefers lowercase `condition` if present (`rnaseq.py:420`), so it sees
`"cntl"`, while Sleuth's path uses `Control?` to synthesize its own `"control"`/`"test"` labels.
Both happen to alphabetize/relevel to the same reference group, so the sign convention stays
aligned in practice — but it's two independently-derived label sets converging on the same answer
by coincidence of naming, not a shared source of truth. If either tool's metadata-generation logic
changes, check that the relevel/alphabetization still lands `cntl`/`control` as the reference.
