# Key notes — RNA-seq warehouse design decisions

Running log of decisions made while designing the DuckDB warehouse, so the
reasoning doesn't have to be re-derived later.

## Table inventory (verified for overlap)

Every file below was checked against every other file for duplicated
information, not just skimmed by column name. `splicing_summary.tsv` and
`expression_gene.tsv` are the two overlaps found — flagged and excluded
below.

| file | what it measures | biological question it answers |
|---|---|---|
| `expression_gene.tsv` | per-sample read count/TPM for each gene (kallisto) | how active is this gene in this sample? |
| `expression_transcript.tsv` | per-sample read count/TPM for each individual mRNA isoform of a gene | how much of this *specific* spliced version of the gene is being made? |
| `de_gene.tsv` | statistical test (DESeq2) comparing a gene's expression between test vs. control | did this gene's output change significantly, or is that noise? |
| `transcript_de.tsv` | same statistical test (sleuth), per isoform | did this specific isoform change, even if the gene's total output looks flat? |
| `junction.tsv` | raw per-sample evidence: every spot a read was seen jumping over an intron, with genomic coordinates | where exactly is splicing physically happening, and how many reads support it? |
| `splicing_event.tsv` | statistical test (rMATS): is a specific splicing choice (skip this exon, use this alternate site, etc.) happening at different rates between conditions | did the cell's splicing *decision* change, not just where splicing occurs? |
| `splicing_event_replicate.tsv` | individual per-sample inclusion values that get combined into splicing_event's group-level test | what did each replicate actually look like, before averaging? |
| `signal_over_gene.tsv` | read depth averaged across each gene's full length, from the coverage tracks | is coverage even across the gene (degradation/bias check), and a coverage-based expression proxy |
| `gene_set_enrichment.tsv` | GSEA test: is a whole predefined group of related genes (a known pathway) collectively shifted between conditions | is this biological process activated or suppressed overall, not gene-by-gene |
| `bigwig_manifest.tsv` | not biology — index of the coverage-track files signal_over_gene summarizes, with location and checksum | none directly; metadata about files, not a measurement |
| `qc_metric.tsv` | read/alignment quality stats from fastp, fastqc, STAR, kallisto, RSeQC | is the sequencing data itself trustworthy, not what's happening biologically |
| `reports/mycoplasma_report.tsv` | bacterial contamination screen of the cell culture, per species | was the cell line contaminated before sequencing even started — a check on the sample, not the data |

**Excluded: `splicing_summary.tsv`.** Verified against real data (run
R6dc470632f84) that every number in it is an exact `GROUP BY`/`COUNT`
rollup of `splicing_event.tsv`, using the fdr/inclusion-diff thresholds it
documents in its own columns (e.g. SE/JC: 1722 total, 20 significant, 3
higher-in-test, 17 higher-in-cntl — matches a live count exactly). No new
information, cheap to recompute, same drift risk as the `reports/`
convenience tables. Build as a view instead of loading it as a table:

```sql
CREATE VIEW splicing_summary AS
SELECT run_id, comparison_id, event_type, counting_mode,
       count(*) AS total_events,
       count(*) FILTER (WHERE fdr <= 0.05 AND abs(inc_level_difference) >= 0.1)
           AS significant_events,
       count(*) FILTER (WHERE fdr <= 0.05 AND abs(inc_level_difference) >= 0.1
                          AND inc_level_difference > 0) AS sig_higher_inclusion_test,
       count(*) FILTER (WHERE fdr <= 0.05 AND abs(inc_level_difference) >= 0.1
                          AND inc_level_difference < 0) AS sig_higher_inclusion_cntl,
       0.05 AS fdr_threshold,
       0.1  AS inclusion_diff_threshold
FROM splicing_event
GROUP BY run_id, comparison_id, event_type, counting_mode;
```

**Corrected 2026-08-14 — the thresholds above were wrong in the first draft
of this note.** Two mistakes, both found by diffing the view against the real
`splicing_summary.tsv` at load time:

- The comparisons are **inclusive** (`fdr <= 0.05`, `abs(diff) >= 0.1`), not
  strict. `rnaseq_helper_scripts/tables_rmats.py:_summarize` computes
  `significant = (fdr <= RMATS_FDR_THRESHOLD) & (difference.abs() >=
  RMATS_INC_DIFF_THRESHOLD)`.
- The two directional counts re-use that **same** significance mask and differ
  only in the *sign* of `inc_level_difference` (`mask & (difference > 0)`).
  Applying the 0.1 threshold a second time in the directional filters is a
  second, independent error.

Together they undercount every significance column by roughly half a percent
while leaving `total_events` exactly right — which is what makes it easy to
miss. On a real run (`R0097bf6d0f7d`, Akata_anti-IgG_48hr) all 10 summary rows
disagreed: SE/JC read 16,374 significant against the view's 16,283. The
`total_events` column matched on all 10, so a spot-check of row counts alone
would have passed.

The view also emits `fdr_threshold`/`inclusion_diff_threshold` as literals so
it stays column-compatible with the `.tsv` it replaces.

**Excluded: `expression_gene.tsv`.** Verified against real data (run
R6dc470632f84), full table not a sample — checked all 907,902 gene rows
against `expression_transcript.tsv` summed by `(sample_id, gene_id)`. Zero
mismatches on `est_counts` or `tpm`. The only unmatched keys were 6
`(sample, gene_id='NA')` aggregates from the unmapped-transcript rows,
which correctly don't get a gene-level row — not a counterexample, the
aggregation working as expected. No new information, cheap to recompute.
One difference from `splicing_summary`: that table is produced by the same
tool (rMATS) in the same step as its source, so it can't structurally
drift. `expression_gene` and `expression_transcript` come from separate
pipeline steps (kallisto's transcript quantification vs. a gene-level
rollup step), so zero drift today isn't structurally guaranteed to stay
zero if that rollup step ever changes — if anything, a stronger reason to
make it a view, not a weaker one. Build as a view instead of loading it as
a table:

```sql
CREATE VIEW expression_gene AS
SELECT run_id, sample_id, gene_id, gene_id AS gene_symbol,
       sum(est_counts) AS est_counts,
       sum(tpm) AS tpm
FROM expression_transcript
WHERE gene_id IS NOT NULL AND gene_id != 'NA'
GROUP BY run_id, sample_id, gene_id;
```

**Corrected 2026-08-14 — the first draft of this note did not run.** It
selected and grouped by `gene_symbol` *from `expression_transcript`*, which has
no such column. Its columns are `run_id, sample_id, transcript_id, gene_id,
length, eff_length, est_counts, tpm` — the symbol only ever appears in
`expression_gene.tsv`, the very table the view is replacing. DuckDB rejects it
outright:

```
Binder Error: Referenced column "gene_symbol" not found in FROM clause!
Candidate bindings: "gene_id", "run_id", "eff_length", "est_counts"
```

Emitting `gene_id AS gene_symbol` is correct **only while the pipeline runs
with `gene_id_source = 'gene_symbol'`**, which it does today — `gene_id` holds
symbols like `5S_rRNA` and `BZLF1_1`, not `ENSG…` accessions, and the two
columns are identical on all 605,268 rows of a full run. That is a live
assumption, not a structural guarantee, so `loader.verify_derived_views()`
re-checks the equality against every run's own `expression_gene.tsv` at load
time. A switch to Ensembl gene ids then surfaces as a reported mismatch rather
than a silently wrong column.

The rollup itself was right: verified across all 11 experiments, zero
mismatches on `est_counts` or `tpm`.

## Decisions so far

- Load all of `tables/*.tsv` **except** `splicing_summary.tsv` and `expression_gene.tsv` (views above instead).
- Load `reports/mycoplasma_report.tsv` only — the one file in `reports/` with no equivalent elsewhere.
- Everything else in `reports/` excluded as redundant: `topGenes.tsv`/`topTranscripts.tsv`/`top_gene_tpms.tsv` are filtered joins of tables already loaded; `alignmentSummary.tsv`/`transcriptCoverage.tsv` are wide reshapes of `qc_metric.tsv`'s `star`/`kallisto` rows; `de_gene.xlsx` is a format duplicate of `de_gene.tsv`; PNGs/`report.html` are rendered artifacts, not tabular data.
- `artifacts_manifest` stays a materialized table, not a view — unlike the redundant cases above, it's not duplicating other loaded data (nothing else records artifact paths/checksums), and materializing it preserves a record of files that may later be archived or deleted off disk.
- `comparisons` (built 2026-08-14): one row per `(run_id, comparison_id)`, columns `test_group`/`cntl_group` (currently always `'test'`/`'cntl'` per `rnaseq_helper_scripts/outputs.py`, but stored as data rather than assumed from the `comparison_id` string). Sample membership in a comparison is not stored separately — join to `samples` on `condition = comparisons.test_group`/`cntl_group`, since every comparison includes all of a run's samples with no subsetting.
