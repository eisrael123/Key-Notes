# RNA-seq Pipeline Walkthrough & Transcript-to-Gene Mapping

## Pipeline story (short-read `rnaseq-pipeline`)

1. **Sample sheet (`metadata.py`)** — records which FASTQ files belong to which sample and condition (`cntl` vs `test`). Everything downstream reads from this.
2. **Quality check (FastQC, fastp)** — inspects raw reads for quality, adapter contamination, etc. QC-only in this pipeline (fastp runs without trimming) — diagnostic, not corrective.
3. **Genome alignment (STAR)** — maps reads to genomic coordinates. Gates the next two steps (strandedness inference, splicing), since both need genome coordinates.
4. **Strandedness inference (RSeQC)** — uses STAR's alignments to determine which strand each transcript's reads came from; needed before quantification to avoid misattributing expression.
5. **Transcript quantification (kallisto)** — estimates counts/TPM per known transcript by pseudo-aligning reads against a transcript-sequence FASTA (not through STAR/genome coordinates at all — a separate, parallel path). Also run against a contaminant FASTA as a screen (e.g. mycoplasma).
6. **Transcript → gene rollup** — kallisto's transcript-level counts get summed into gene-level counts using a transcript-to-gene (tx2gene) lookup table exported from the reference annotation. See worked example below.
7. **Differential expression, gene level (DESeq2)** — tests each gene for significant up/down change between conditions, using the gene-level counts matrix from step 6. Optional ERCC spike-in normalization.
8. **Differential expression, transcript level (Sleuth)** — parallel branch, works directly on kallisto's transcript-level output to catch isoform-level shifts that gene-level sums would hide.
9. **Gene set enrichment (GSEA)** — asks whether whole pathways/programs shift coherently, built from the ranked gene list DESeq2/TPM output produced.
10. **Alternative splicing (rMATS)** — goes back to STAR's genome alignments (not kallisto) to detect exon inclusion/skipping between conditions.
11. **Report assembly** — merges QC, gene-level, transcript-level, pathway, and splicing results into one HTML/Excel report.

## STAR/minimap2 vs. GTF vs. kallisto — who does what

- **STAR (short reads) / minimap2 (long reads, ONT)** align reads to **genomic coordinates** — "this read came from chromosome X, position Y." Neither tool inherently knows what gene or transcript that position belongs to.
- **The GTF reference annotation** is what supplies gene/transcript identity for those coordinates. It's built over years from accumulated transcript evidence (RNA-seq, cDNA/EST libraries, manual curation) and curated into a lookup: "these coordinates belong to gene G, specifically transcript T."
- **kallisto** doesn't use genome coordinates or STAR/minimap2 output at all. It pseudo-aligns reads directly against a **transcript-sequence FASTA** (a separate reference file, one sequence per known transcript/isoform) and estimates abundance per transcript from that matching alone. It's a fully independent quantification path from the genome-alignment path.

## Worked example: transcript → gene rollup

Say gene **ABC** has three known transcript isoforms in the reference annotation: `ABC-201`, `ABC-202`, `ABC-203`. Kallisto quantifies each independently, since it only sees transcript sequences, not gene identity:

| Transcript | est_counts | TPM |
|---|---|---|
| ABC-201 | 120 | 45.2 |
| ABC-202 | 45 | 15.8 |
| ABC-203 | 10 | 3.1 |

The tx2gene table (from the Biomart export in `referenceFiles/<build>/biomart/`) says all three transcript IDs map to gene `ABC`. The pipeline's gene-level rollup is a straight sum per gene:

- **Gene-level est_counts** = 120 + 45 + 10 = **175**
- **Gene-level TPM** = 45.2 + 15.8 + 3.1 = **64.1**

(Summing TPM across isoforms is a common approximation, not mathematically exact — TPM is defined to sum to 1,000,000 across the whole transcriptome, not independently additive per gene — but it's standard practice for gene-level TPM reporting.)

This is the whole trick behind "gene-level" numbers in the pipeline: kallisto never computes anything at the gene level itself — the reference annotation's tx2gene table is what lets transcript-level numbers be regrouped into gene-level numbers after the fact.
