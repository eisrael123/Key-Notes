# Key notes — RNA-seq warehouse design decisions

Running log of decisions made while designing the DuckDB warehouse, so the
reasoning doesn't have to be re-derived later.

## Table inventory (verified for overlap)

Every file below was checked against every other file for duplicated
information, not just skimmed by column name. `splicing_summary.tsv` was
the one real overlap found — flagged and excluded below.

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
       count(*) FILTER (WHERE fdr < 0.05 AND abs(inc_level_difference) > 0.1) AS significant_events,
       count(*) FILTER (WHERE fdr < 0.05 AND inc_level_difference > 0.1) AS sig_higher_inclusion_test,
       count(*) FILTER (WHERE fdr < 0.05 AND inc_level_difference < -0.1) AS sig_higher_inclusion_cntl
FROM splicing_event
GROUP BY run_id, comparison_id, event_type, counting_mode;
```

## Decisions so far

- Load all of `tables/*.tsv` **except** `splicing_summary.tsv` (view above instead).
- Load `reports/mycoplasma_report.tsv` only — the one file in `reports/` with no equivalent elsewhere.
- Everything else in `reports/` excluded as redundant: `topGenes.tsv`/`topTranscripts.tsv`/`top_gene_tpms.tsv` are filtered joins of tables already loaded; `alignmentSummary.tsv`/`transcriptCoverage.tsv` are wide reshapes of `qc_metric.tsv`'s `star`/`kallisto` rows; `de_gene.xlsx` is a format duplicate of `de_gene.tsv`; PNGs/`report.html` are rendered artifacts, not tabular data.
- `artifacts_manifest` stays a materialized table, not a view — unlike the redundant cases above, it's not duplicating other loaded data (nothing else records artifact paths/checksums), and materializing it preserves a record of files that may later be archived or deleted off disk.
