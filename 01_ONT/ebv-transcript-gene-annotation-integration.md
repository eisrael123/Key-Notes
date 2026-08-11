# EBV Transcript Gene Annotation Integration

## Background problem

PI (Erik) asked how much effort it would take to redo the poly(A) length and
RNA-modification (m6A/m5C/pseU/inosine) analyses so they include EBV
transcript data, as an alternative to paying the ONT sequencing vendor
(project `CGEKDRSacRIP2634`) to reanalyze from scratch.

Genome alignment already included EBV (dedicated `chrEBV_Akata_inverted`
contig) and modification calling already separated virus reads per sample
with genomic coordinates. But poly(A) and expression tables had **zero real
EBV gene names**. Root cause: SQANTI3 reconstructed 792 distinct transcript
models on `chrEBV_Akata_inverted` but classified them against a
**human-only GENCODE annotation** (no EBV gene models to match against), so
every chrEBV transcript got an anonymous placeholder ID like
`100:474|f905473d-7458-47c0-9d26-6e839d380edd` instead of a real gene name.
That placeholder ID is what poly(A) and expression tables inherited.

Erik supplied a real EBV transcriptome annotation
(`EBV_annotation.gtf` / `supplemental_file_1_core_EBV_transcriptome_annotation.gtf`,
239 unique gene_ids covering the full lytic + latent EBV transcriptome:
BZLF1, BRLF1, EBNA1-3/LP, LMP1/2A/B, BHRF1, BART miRNAs, RPMS1, etc.),
correct `chrEBV_Akata_inverted` coordinates, standard 9-column GTF.

## What I did

1. Extracted the 792 anonymous SQANTI3 chrEBV transcript models
   (`refAnnotation_SQANTI3_Final.genePred`) -> BED.
2. Converted Erik's GTF gene features -> BED (246 rows, 239 unique gene_ids).
3. `bedtools intersect -s -wao` (same-strand only -- EBV genes are densely,
   often antisense, overlapping) + a custom tie-break (exact boundary match >
   max overlap > tightest/smallest containing gene). This cut raw-overlap
   ties from 589/792 down to 100/792; all remaining ties are between named
   splice/promoter variants of the same locus (e.g. BZLF1 vs BZLF1-v1), not
   real cross-gene confusion. 788/792 (99.5%) matched.
4. **Poly(A)**: joined the mapping against each sample's per-read
   `*.polyA.txt`. Found the per-read files list multiple "compatible"
   candidate transcripts per physical read (dense near-identical placeholder
   isoform models) -- naive crediting would inflate signal (93% of SVP reads,
   ~55% of ZTA reads touched >1 gene). Built a conservative
   (unambiguous-reads-only) table and a separate inflated all-reads table,
   clearly labeled.
5. **Modifications**: intersected `*.virus.txt` genomic coordinates directly
   against the EBV gene BED, same-strand restricted via the originating
   transcript. >98% of positions annotated across all 24 sample x mod-type
   files.

## Sanity check result

BZLF1/BRLF1 (master lytic switch genes) had to show clean induction in
ZTA_Test (reactivated) vs SVP_Ctrl (latent) for this to be trustworthy.

- Poly(A), unambiguous reads: **0** BZLF1/BRLF1 reads in all 3 SVP_Ctrl
  samples; 108-165 BRLF1-family / 14-38 BZLF1-family reads in all 3 ZTA_Test
  samples, poly(A) tails 80-130 nt (reasonable range).
- Total EBV reads jumped from 42-53 (SVP_Ctrl) to 1.5-1.75M (ZTA_Test) --
  consistent with known massive lytic genome/transcript amplification.
- m6A sites at the BZLF1/BRLF1 locus: 117-255 (SVP_Ctrl, plausible low-level
  leaky lytic transcription) vs 14,536-14,945 (ZTA_Test), ~60-90x higher.

**Conclusion: the coordinate-mapping approach cleanly recovers the expected
biology** -- viable without paying the vendor to redo the analysis.

## Output location

`CGEKDRSacRIP2634/06.EBV_gene_annotation/` on the shared volume, with a
README.md explaining the directory contents:
- `transcript_to_gene_map.tsv`, `mapping_QC_summary.txt`
- `gene_polyA_summaries_unambiguous/` (recommended) and
  `gene_polyA_summaries_all_reads/` (exploratory/inflated, flagged as such)
- `mod_annotated/` (all 24 sample x mod-type files with gene names added)
- `ambiguous_reads/`, `intermediate/` (QC/reproducibility)

## Open items for Erik

1. Whether to collapse splice/promoter variant gene_ids (BZLF1/BZLF1-v1/
   BZLF1-v2/lncBZLF1 etc.) into one parent-gene rollup for final summary
   tables, or keep transcript-variant resolution throughout.
2. A proper SQANTI3 re-run with GENCODE+EBV-GTF combined as reference
   (rather than this post-hoc coordinate matching) remains the more rigorous
   option if time allows before the paper -- would resolve read assignment
   via splice-junction logic and likely reduce the multi-transcript-per-read
   ambiguity seen in the poly(A) data.
3. Expression tables (`counts_gene.txt`/`counts_transcript.txt`) not yet
   updated with EBV rows -- optional if needed for the paper.
