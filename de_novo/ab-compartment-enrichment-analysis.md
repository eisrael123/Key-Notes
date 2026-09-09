# A/B Compartment Enrichment Analysis of De Novo Promoters

Status note for the analysis in `~/ethan/de_novo/analysis`. Written 2026-09-09.
Companion to [de-novo-promoter-cage-dataset.md]. Analysis in progress: steps 01-04
built and run, steps 05-06 outstanding.

**Question being answered:** for each of the 4 de novo promoter classes (Zta, vPIC,
Zta_vPIC, Unclassified), is it enriched in the A or B Hi-C compartment?

**Inputs:** `1_De_novo_promoter_identification_and_classification/de_novo_promoters.classified.bed`
(453,576 promoters) against `A_B_analysis/BMRF1p_GFP_sorted_latency_consensus.50kb.compartments.bed`
(60,630 x 50kb bins; A / B / AMBIGUOUS / UNSCORABLE), with
`CMV_GFP_unsorted_latency_consensus.50kb.compartments.bed` as a replicate.

---

## The chain of reasoning

The analysis is deliberately built as three nested steps, each answering the
objection raised by the previous one. That structure is the point - the raw
result on its own does not survive review.

### Step 02 - the observation

A-fraction among promoters in confidently called bins, genomic baseline 43.4% A:

| Class | n(A) | n(B) | %A | 95% CI (block bootstrap) | log2 obs/exp | z |
|---|---|---|---|---|---|---|
| Zta | 40,437 | 3,009 | **93.1%** | [92.2, 94.0] | 1.10 | 17.4 |
| vPIC | 99,456 | 99,277 | **50.0%** | [48.1, 51.9] | 0.21 | 7.9 |
| Zta_vPIC | 3,989 | 1,443 | 73.4% | [71.4, 75.4] | 0.76 | 17.4 |
| Unclassified | 104,259 | 15,894 | 86.8% | [85.5, 87.9] | 1.00 | 18.0 |

All four A-enriched vs the rotation null; all pairwise class differences exclude
zero. Zta vs vPIC = 43.0 points [41.5, 44.6].

### Step 03 - the composition objection

A bins average 44.9% GC, B bins 37.8%. The class labels are **motif-defined** and
the motifs differ in GC (Zta ZREs GC-rich, vPIC TATT box AT-rich). Mean local GC
by class - Zta 50.2% > Unclassified 48.5% > Zta_vPIC 45.6% > vPIC 41.6% - is
*exactly* the rank order of A-fraction. So the step-02 result has a mundane
alternative explanation: isochores.

Direct standardization to a common GC profile (deciles):

| Class | raw %A | GC-adjusted %A | 95% CI |
|---|---|---|---|
| Zta | 93.1% | 89.6% | [88.4, 90.7] |
| vPIC | 50.0% | 58.7% | [57.2, 60.3] |
| Zta_vPIC | 73.4% | 70.5% | [68.4, 72.5] |
| Unclassified | 86.8% | 81.1% | [79.8, 82.3] |

Zta vs vPIC gap shrinks 43.0 -> **30.8 points** [29.5, 32.0]. Survives.

### Step 04 - the motif-matched null (the decisive control)

Every genomic occurrence of a class-defining motif, split into occurrences that
DID spawn a de novo promoter vs occurrences that spawned nothing. Both arms
contain the identical motif, so composition cannot explain a difference.

| Motif set | USED | UNUSED | UNUSED GC-standardized | log2 OR (GC-adj) |
|---|---|---|---|---|
| vPIC (TATTWAA, n=1,970,022) | 47.7% A | 29.8% A | **46.5% A** | **0.07** [0.03, 0.10] |
| Zta (ZRE 7-mers, n=1,704,791) | 91.2% A | 53.2% A | **69.3% A** | **2.21** [2.08, 2.34] |

**THE RESULT IS A DISSOCIATION, NOT A GRADIENT.** Given a TATT box of matched
composition, whether a vPIC promoter forms there is essentially independent of
compartment (OR 1.05). Given a ZRE of matched composition, a Zta promoter is
~4.6x more likely to form in A than B. **Compartment gates Zta promoter
formation; it does not gate vPIC promoter formation.** Consistent with the
biology: Zta is a host-style TF needing accessible chromatin, vPIC brings its own
basal machinery and reads AT-rich DNA wherever it finds it.

---

## Methodological decisions worth defending in the paper

- **No Fisher's exact test.** Promoters are not independent (4-5 per 50kb bin;
  compartments autocorrelate across bins). Measured design effect is **3x to 73x**
  variance inflation - a Fisher test would be overconfident by that much.
- **p-values from circular rotation permutation.** The compartment label track is
  rotated within each chromosome; preserves compartment run-length structure, the
  genome-wide A:B ratio, and promoter clustering. Promoters in AMBIGUOUS bins are
  NOT pre-filtered before rotating, or the null gets conditioned on the observed
  labelling.
- **CIs from a 1 Mb block bootstrap** (20 bins), so co-located promoters travel
  together.
- **Direct standardization over logistic regression** for GC: the GC effect is
  strongly non-linear, and the per-stratum table can be audited by a reader.
- **Promoter position = column 13** (selected_peak_position), a single base. Also
  disposes of bin-straddling: only 32 of 453,576 intervals cross a 50kb boundary.
- **Replicate concordance:** the two compartment call sets agree on 355,838 /
  356,012 promoters where both call confidently = **99.95%**.
- Dropped: chrY (1,074) and chrMT (16) - absent from the Hi-C BED.

## Caveats to state explicitly in the paper

1. **The vPIC adjusted CI excludes zero but the effect is negligible.** log2 OR
   0.07 = OR 1.05. With ~2M occurrences almost anything reaches significance.
   Report as *no meaningful effect*, lean on effect size, do not write
   "significant".
2. **Detection bias cannot be fully removed.** "Used" means CAGE detected a
   promoter, and CAGE detects transcription, which is easier in A. BUT: the bias
   applies identically to both motif sets and vPIC shows essentially nothing. If
   detectability drove the Zta result, vPIC should show it too. Strong internal
   control - state it rather than wait to be asked.
3. **Cell-state mismatch, unresolved.** Hi-C is *latency*, CAGE is *lytic*. If the
   same cell system this is arguably the right design ("pre-existing latent
   compartment structure predicts where lytic de novo promoters form"), since
   lytic compartments would be a consequence rather than a predictor. Needs to be
   stated in methods. **Still need to confirm what system each is from.**
4. hg38.fa here is not soft-masked, so no repeat-content sensitivity analysis is
   available from it; would need an external RepeatMasker track.

## Two bugs hit and fixed (both silent)

- **Reverse-strand offset sign.** Implied TSS for a minus-strand motif is
  `match_start - 26`, not `+ 26`. Symptom: promoter recovery 28.8% when ~75% carry
  the motif. After fix, 73.0% of vPIC promoters in confident bins recovered. The
  recovery check is now a permanent assertion in the script.
- **Mismatched estimate and interval.** First version printed a GC-standardized
  point estimate beside an *unadjusted* OR and CI. The bootstrap now standardizes
  inside each replicate with the reference GC profile held fixed.
- (Also corrected: a diagnostic counter credited only one promoter per motif
  occurrence, understating Zta motif recovery as 31.8% when it is 59.7%. Affected
  only the log line, never the statistics.)

## Open decisions

1. **Which claim leads the figure?** Descriptive ("classes occupy different
   compartments", step 02) vs mechanistic ("compartment gates Zta but not vPIC
   promoter formation", step 04). Recommendation: mechanistic, with step 02 as
   panel A setting it up. Depends on what the paper as a whole argues.
2. **ZRE motif strictness.** Strict 7-mer list covers 68.3% of Zta promoters but
   only 46.7% of Unclassified (enrichment 1.46x); degenerate `TG[AT]G[CT][CG]A`
   covers 79.7% / 62.6% (1.27x). Looser motif = better promoter coverage but a
   weaker control, because the "unused" background fills with sequences that were
   never ZREs. Plan: strict primary, degenerate as sensitivity.

## Still to build

Step 05 sensitivity (AMBIGUOUS three ways, chrX in/out, one-vote-per-bin, promoter
score thresholds), step 06 figures (SVG), and `analysis/README.md`.

## Layout

Stdlib-only Python 3.9, no numpy/pandas/scipy/R - runs on a bare machine with
`python3 script.py`. `analysis/scripts/{config.py, lib/genome.py, lib/resample.py,
01_build_promoter_table.py, 02_primary_enrichment.py, 03_gc_adjusted.py,
04_motif_matched_null.py}`, outputs to `analysis/{results,figures,logs}`.
