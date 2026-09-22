# Promoter Sequence and ATAC Feature Construction

Recorded 2026-09-22 for project `de_novo`. The user requested preserving the detailed explanation of how source files become interpretable model features, and found the concrete, step-by-step explanation thorough and easy to follow. This is a proposed feature design, not a completed model. Actual measurements and illustrative examples are distinguished below.

## One observation and its source files

For the proposed promoter-level comparison, one observation is one identified de novo promoter. The outcome is cluster-associated versus isolated, conditional on the promoter already existing. Feature importance would identify associations, not establish causal licensing.

- `1_De_novo_promoter_identification_and_classification/de_novo_promoters.classified.bed` supplies chromosome, promoter interval, selected TSS (column 13), and strand.
- Cluster membership supplies the outcome and grouping metadata. The top-percentile exports alone do not identify every clustered promoter; absence from those files is not evidence of isolation.
- `/Users/flemingtonlab/ethan/referenceFiles/hg38/fasta/hg38.fa` supplies reference DNA, with its `.fai` index enabling interval extraction.
- `3_ATAC-seq_insertions_and_coverage/*.ins.bw` supplies six separate insertion measurements over the same genomic coordinates.

Flow: promoter coordinates -> query DNA and six ATAC tracks -> calculate sequence features and per-sample insertion sums -> normalize samples -> summarize conditions -> join features and outcome by promoter ID. Preserve source files and per-replicate intermediate measurements.

Actual example: `DNP_0360069`, chr17, selected TSS 4861147 in the source coordinate convention, plus strand, cluster `C100_135751`. The DNP numeric identifier maps to one-based row 360069 in the original promoter BED; preserve that original ordering or establish an explicit ID mapping.

## ATAC filenames identify replicates, not pieces of a chromosome

The PI confirmed:

- MC = control; files MC2, MC4, MC5.
- MZ = Zta condition; files MZ1, MZ2, MZ4.
- Each is a biological replicate, with unnormalized insertion-point signal, not coverage.

The complete filenames are `<sample>_ATAC30122019.ins.bw`. These six files represent separate measurements over the same genomic coordinate system. They are not stitched together. Numeric suffixes do not establish sampling order or paired relationships; MC2 and MZ2 are not necessarily paired. Normalization adjusts numerical scales, not genomic alignment.

## Extracting insertion measurements

A candidate 200-bp core window is `[TSS-100, TSS+100)`. For the actual example, query chr17:4861047-4861247 in each of the six BigWigs. Coordinates are zero-based, end-exclusive.

Conceptually each query yields a signal vector, such as `[0, 0, 2, 0, 1, 5, ...]`, one insertion count per genomic position. Sum it for a regional insertion count. If the file represents a constant value over a multi-base interval, integrate value times overlap length. Handle unavailable positions explicitly rather than silently treating all missing values as measured zeros.

The regional count measures sampled insertion events, not the percentage of cells with open chromatin. The six biological replicates contribute repeated measurements for one promoter; they do not create six independent promoter observations.

## Normalization and illustrative arithmetic

A candidate depth normalization is:

`normalized_window_signal = window_insertions / total_eligible_sample_insertions * 1,000,000`

The denominator must consistently count eligible insertion events, not automatically read pairs or fragments. Choose a consistent genomic scope, such as eligible human sequence. Simple total-count scaling is a starting method whose adequacy must be checked; it does not automatically remove all technical biases or resolve global accessibility changes.

The following numbers are illustrative, NOT measurements extracted from the actual ATAC tracks:

| Sample | Window insertions | Eligible sample insertions | Insertions per million |
|---|---:|---:|---:|
| MC2 | 100 | 50 million | 2 |
| MC4 | 80 | 40 million | 2 |
| MC5 | 150 | 50 million | 3 |
| MZ1 | 600 | 60 million | 10 |
| MZ2 | 550 | 50 million | 11 |
| MZ4 | 720 | 60 million | 12 |

MC2 and MC4 have different raw counts but the same normalized local signal. Normalize each replicate before averaging so deeper sequencing alone does not give a replicate extra weight.

From this example:

- Control accessibility = mean(2, 2, 3) = 2.33.
- Zta accessibility = mean(10, 11, 12) = 11.
- Accessibility difference = Zta mean - control mean = 8.67.
- Accessibility log ratio = log2((Zta mean + 1)/(control mean + 1)) = approximately 1.85.

The added 1 is a pseudocount that avoids division by zero and moderates ratios at low signal; its value affects weak-signal comparisons. A separate possible transformation is log2(1 + accessibility), which compresses large values. Do not automatically include condition means, their difference, and their ratio together: these are redundant candidate representations. Preserve individual replicate values for consistency checks.

MC represents control-state accessibility and MZ represents accessibility in the Zta condition. Their contrast describes a condition-associated change. Control is not automatically a longitudinal pre-treatment measurement of the same cells. Timing and experimental correspondence to CAGE remain relevant to biological interpretation.

## Core and surrounding accessibility

Candidate windows, relative to selected TSS:

- Core: [-100, +100), 200 bp.
- Left flank: [-500, -100), 400 bp.
- Right flank: [+100, +500), 400 bp.
- Combined flanks: 800 bp; full window: 1,000 bp.

Use normalized core signal, flanking signal, and potentially core-to-flank enrichment to distinguish focal accessibility from a broadly accessible neighborhood. When comparing differently sized windows, divide by length as well as sample depth, for example insertions per million per kilobase. On minus-strand promoters, swap genomic left/right when labeling upstream/downstream. Symmetric combined windows do not need that swap. Window sizes are proposals, not fixed decisions.

## Sequence extraction and actual composition measurements

Use the FASTA index to retrieve the same interval without loading the entire genome. The reference contains DNA letters, not sample-specific variants or methylation measurements.

The actual 200-bp window for DNP_0360069 was extracted read-only on 2026-09-22. It has 200 valid A/C/G/T bases, 112 G/C bases, and seven adjacent CG pairs:

- GC fraction = (number of G + number of C) / valid bases = 112/200 = 0.56.
- CpG frequency = CG pairs / valid adjacent-pair opportunities = 7/199 = approximately 0.0352.

GC fraction describes composition; CpG frequency describes adjacency and can differ between sequences with equal GC. CpG frequency is not methylation. Track unknown N bases and use valid denominators or exclude poorly defined windows.

## Motif counts and density

Extract DNA, scan each possible position on both strands, retain hit locations, and summarize per motif. Three hits in a 1,000-bp window would yield count 3, density 3 per kb, and presence 1. For equal-length windows, count and density are redundant.

Actual scan of the example 200-bp window found zero exact matches on either strand for each of the five ZRE strings previously used in the project: TGAGTCA, TGAGCCA, TGAGCGA, TGTGCAA, TGTGCGA. It also found zero exact TATTAAA or TATTTAA matches. This does not establish that binding is impossible: the motif list and window are limited, and exact matching excludes imperfect matches.

## Best motif-match score

A motif matrix describes base preferences at each position. Slide it along the sequence, score each candidate site, and retain features such as best score or number of hits above a predefined threshold. FIMO sums entries in a position-specific scoring matrix to calculate a match score.

A match score measures sequence compatibility, not experimentally measured binding affinity or occupancy. Scoring requires motif matrices and a consistent background/threshold definition; these scores are not already contained in the BED or ATAC tracks.

## Motif position

Two promoters with one motif each can differ because one match is 30 bp upstream and another is 450 bp upstream. Features can include nearest qualifying motif distance, presence in a specified upstream interval, and orientation relative to transcription.

Use a consistent position anchor for the motif and orient positions relative to promoter strand. Negative should consistently mean upstream. If no motif qualifies, distance is missing, not zero; add a presence indicator. Zero means a motif at the chosen TSS anchor.

## Motif combinations and spacing

For specified motifs A and B, candidate features include both present (binary), minimum distance between motif centers, an A-B pair within 50 bp (binary), or number of qualifying pairs. These make sequence arrangements explicit, interpretable variables.

Limit combinations to a manageable hypothesis set. Data-driven discovery or selection must occur inside training data, not using held-out observations. These features are ways to investigate patterns with traditional statistical or machine-learning models; an LLM is not required for pattern discovery.

## A proposed final row

| Column | Value | Status |
|---|---|---|
| promoter_id | DNP_0360069 | Actual identifier |
| chrom | chr17 | Actual metadata; useful for evaluation grouping |
| tss | 4861147 | Actual coordinate |
| cluster_id | C100_135751 | Actual membership metadata |
| clustered | 1 | Proposed outcome, supported by membership |
| gc_fraction_200bp | 0.56 | Actual reference-derived measurement |
| cpg_frequency_200bp | 0.0352 | Actual reference-derived measurement |
| exact_zre_count_200bp | 0 | Actual exact sequence scan |
| control_accessibility_200bp | 2.33 | Illustrative only |
| zta_accessibility_200bp | 11 | Illustrative only |
| accessibility_log2_ratio_200bp | 1.85 | Illustrative only |

Coordinates and cluster IDs are metadata, not automatically predictor variables. Keep nearby promoters and members of the same cluster together during training/evaluation splits. Features used to define an outcome should not be fed back as explanatory predictors of that same outcome.

## References from the discussion

- FIMO scoring: https://meme-suite.org/meme/doc/fimo-output-format.html
- Sequencing-depth normalization background (read-based documentation; the proposed insertion denominator must be defined separately): https://deeptools.readthedocs.io/en/3.2.1/content/tools/bamCoverage.html
