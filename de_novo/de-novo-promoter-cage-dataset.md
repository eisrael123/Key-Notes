# De Novo Promoter CAGE Dataset

Description of the data in `~/ethan/de_novo/1_De_novo_promoter_identification_and_classification`
and `~/ethan/de_novo/2_De_novo_promoter_associated_CAGE_reads`. Written 2026-09-08.
Claims marked "verified" were checked directly against the files (including pulling hg38
sequence); they are not stated in the bundled READMEs.

---

## The biology first

### What the experiment is

Epstein-Barr virus (EBV) sits quietly inside B cells in **latency** - almost no viral genes
on. A trigger flips it into the **lytic cycle**, where the virus runs a strict three-stage
gene program and eventually bursts the cell to release new virions:

1. **Immediate-early**: the master switch protein **Zta** (gene *BZLF1*) is made. Zta is a
   DNA-binding transcription factor of the AP-1 family. Its famous trick is that it binds and
   activates DNA *even when that DNA is CpG-methylated* - methylation normally silences
   promoters, so this lets Zta wake up a genome the cell has chemically shut down. Its binding
   sites are called **ZREs** (Zta Response Elements).
2. **Early**: viral DNA replication machinery; the viral genome gets copied.
3. **Late**: capsid and structural proteins. Late genes use completely different machinery:
   the **vPIC** (viral Pre-Initiation Complex), a virus-encoded mini-version of the cell's own
   basal transcription apparatus, built around **BcRF1**, a viral mimic of TATA-binding
   protein. vPIC does not read a normal TATA box - it reads a **TATT box** sitting ~30 bp
   upstream of the transcription start site.

So there are two distinct engines of transcription running in a lytic cell: **Zta binding
ZREs** (enhancer-like, position-flexible) and **vPIC binding TATT boxes** (core-promoter,
rigidly positioned at -30).

### What this dataset is about

Both engines are supposed to act on the *viral* genome. This dataset asks what they do to the
**human** genome. Every coordinate in both folders is human (chr1-22, X, Y, MT - no viral
contig anywhere). The finding being catalogued is that in a lytic cell, hundreds of thousands
of positions in the *host* genome start acting as transcription start sites that do not exist
in normal cells - **de novo promoters** - and they carry the sequence signatures of viral Zta
and viral vPIC.

**Verified:** fewer than 1% of these sites sit within 100 bp of any annotated human transcript
start site, and only 3-8% within 500 bp. These are genuinely unannotated cryptic start sites,
not rediscoveries of known gene promoters.

### What CAGE is (the assay behind every number here)

**CAGE** = Cap Analysis of Gene Expression. When RNA polymerase II starts a transcript, a
chemical "cap" is added to the very first nucleotide. CAGE captures only capped 5' ends and
sequences them. The consequence: **the start coordinate of a CAGE read is a transcription
start site, at single-base resolution.** Pile up many CAGE reads and a peak of read-5'-ends
marks a promoter. That is why every interval in folder 1 is tiny (median 3 bp) - it's a TSS,
not a gene.

This is paired-end CAGE. **Verified** from the SAM flags: **R1 always carries the 5'/TSS end**
(flags 83/99 = first in pair) and R2 sits downstream in the nascent transcript. So one read
pair = one observed transcription initiation event, plus a little sequence telling you where
that transcript went.

### The samples

Six libraries, three per group:

| Group | Samples | Read pairs in the data |
|---|---|---|
| `Cntl` | `Cntl.Zta.1/2/3` | 5,838,468 (68%) |
| `KO` | `EX3.KO.Zta.1/2/3` | 2,722,439 (32%) |

Both groups are lytic (`.Zta.` = Zta-induced). The KO group carries some knockout labelled
`EX3`.

**OPEN QUESTION: what `EX3` knocks out is not recorded anywhere in the files.** Confirm from
lab notes. The data is suggestive: 141,436 vPIC-classified promoters appear only in `Cntl`
versus 14,877 only in `KO`, i.e. the KO collapses vPIC-driven initiation. That is the
signature expected if EX3 disrupts late-gene/vPIC function, or anything upstream of it (e.g.
viral DNA replication, which late gene expression strictly requires).

---

## Folder 1 - `1_De_novo_promoter_identification_and_classification`

**`de_novo_promoters.classified.bed`** - 110 MB, **453,576 rows, one per de novo promoter**.
BED12 format plus 4 extra columns. (`.idx` is just an IGV index for genome-browser loading.)

Each row was built by merging nearby CAGE peaks from the six libraries into one consensus
promoter interval. Median width 3 bp, max 70 bp. Perfectly strand-balanced (226,789 minus /
226,787 plus), a good sign there is no strand artifact.

### Column by column

| # | Column | What it holds | What it means |
|---|---|---|---|
| 1-3 | `chrom`, `chromStart`, `chromEnd` | e.g. `chr1 31061 31076` | The promoter interval. 0-based, half-open (BED convention). |
| 4 | `name` | `chr1;31065;31066;peak_4_strand_-;20;-\|group=KO;...\|\|Cntl_coverage=28;KO_coverage=89` | Everything before `\|\|` is the **audit trail**: every individual CAGE peak merged into this promoter, each with its own coordinates, peak name, score, strand, and source group. After `\|\|` is the group coverage summary (same as col 15). 64% of promoters were built from 1 peak, 34% from 2, the rest up to 12. |
| 5 | `score` | integer, 6-1224 | **Promoter activity.** **Verified** to be exactly `max(Cntl_coverage, KO_coverage)` in all 453,576 rows - the stronger of the two groups, not the sum. Median 15. |
| 6 | `strand` | `+` / `-` | Direction of transcription. For `+` the TSS is at `chromStart`; for `-` it is at `chromEnd - 1`. This matters constantly downstream. |
| 7-8 | `thickStart`, `thickEnd` | | The single **selected representative peak** - the best-supported exact base within the interval. Renders as the thick block in a browser, so you see the precise TSS inside the fuzzy interval. |
| 9 | `itemRgb` | `255,0,0` etc. | Browser color encoding col 16: red = Zta, blue = vPIC, green = both, grey = unclassified. |
| 10-12 | `blockCount`, `blockSizes`, `blockStarts` | always `1`, width, `0` | Filler so the file is valid BED12. No information. |
| 13 | `selected_peak_position` | `chr1;31075;31076` | Same as cols 7-8 in text form. **Cleanest single-base TSS to anchor analyses on.** |
| 14 | `selected_peak_score` | integer | Raw score of that one selected peak. Not the same as col 5 - col 5 aggregates the group, col 14 is one peak. |
| 15 | `coverage` | `Cntl_coverage=28;KO_coverage=89` | Summed CAGE signal per group across its 3 replicates. `NA` = not detected in that group at all. **This is the differential-expression raw material.** |
| 16 | `classification` | `Zta` / `vPIC` / `Zta_vPIC` / `Unclassified` | See below. |

### The classification - what it actually is

The README does not say how classification was assigned. **Verified by extracting hg38
sequence around 3,000 random TSSs per class: it is sequence/motif-based, not
expression-based.** Classification is essentially independent of which group the peaks came
from.

| Class | n | Share | What the sequence shows |
|---|---|---|---|
| **vPIC** (blue) | 254,178 | 56.0% | **100%** carry a `TATT` upstream, and the position histogram spikes hard at **-32/-31** relative to the TSS. 75% match the canonical late-promoter consensus `TATTWAA`. Textbook BcRF1/vPIC core promoter architecture. |
| **Zta** (red) | 50,557 | 11.1% | No positioned TATT (9.4%, scattered). Instead enriched for **ZREs**: `TGAGTCA`/`TGACTCA` (12.8% vs 5.7% background), `TGTGCAA`, and notably the **CpG-containing methylated ZREs** `TGAGCGA`/`TCGCTCA` and `TGTGCGA`/`TCGCACA` at 3-6x background. Positions spread across -200..+100, as expected for a TF binding site rather than a core element. |
| **Zta_vPIC** (green) | 6,805 | 1.5% | Both signatures: 100% positioned TATT *and* elevated ZRE. |
| **Unclassified** (grey) | 142,036 | 31.3% | Neither signature above background. |

That the meZRE variants come out as the *top* discriminative k-mers is a good internal
validation - those are the sites only Zta can use, in methylated host chromatin.

---

## Folder 2 - `2_De_novo_promoter_associated_CAGE_reads`

The evidence layer. Folder 1 says *where* the promoters are; folder 2 gives **the individual
sequencing reads supporting each one**. 446,196 of 453,576 promoters (98.4%) have at least one
qualifying read pair.

Each row is one paired-end fragment collapsed into a single genomic interval: `chromStart` =
leftmost mapped base, `chromEnd` = rightmost, so the interval spans from the TSS out into the
transcript. Column 5 is the span length (**verified** to equal `end - start` exactly). Column 6
is the strand, which tells you which end is the TSS: `-` -> TSS at `chromEnd`, `+` -> TSS at
`chromStart`.

**Verified link between the two folders:** on chr1, 834,734 of 834,734 read-pair 5' anchors
fall inside a folder-1 promoter interval. Zero exceptions. The join is exact.

### The name field, decoded

```
VH01236:144:222C7WHNX:2:2308:42141:13709 | R1=83,75M,1,8C66 | R2=163,43M,0,43 | sample=Cntl.Zta.3 | group=Cntl
|__ Illumina read ID __________________|   |_ R1 alignment _| |_ R2 alignment _| |_ library ____| |_ group _|
```

Each `R1=`/`R2=` block is `flag, CIGAR, NM, MD`:

- **flag** - SAM bitfield. Only 4 values appear (83/163 for minus-strand pairs, 99/147 for
  plus), meaning every read pair here is properly-paired and correctly-oriented.
- **CIGAR** - how the read aligned: `75M` = 75 matched bases; `74M1S` = 74 matched, 1
  soft-clipped; `1S32M3450N42M` - an **`N` block is an intron**, i.e. the read spans a splice
  junction. ~26% of long-span pairs contain an `N` versus 0.5% of short ones.
- **NM** - number of mismatches to the reference (86% are 0).
- **MD** - where those mismatches are (`8C66` = 8 matches, a C in the reference, 66 more
  matches).

### Span lengths - read this before filtering

Median fragment span is **417 bp** (normal library insert), but the 95th percentile is **9,553
bp** and the max is **1.47 Mb**. The long tail mixes real biology with noise: spliced
transcripts (the `N` CIGARs) versus mismapping/chimeras. **A span cutoff is needed, and where
it goes is a real analytical decision.**

### The three files

| File | Rows | What it is | Use it for |
|---|---|---|---|
| `all_de_novo_read_pairs.bed` | **8,560,907** | Every qualifying read pair, no deduplication - identical coordinates kept as separate rows | Counting. The quantitative matrix: reads per promoter per sample. Median 5 reads per TSS anchor, max 1,041. |
| `all_de_novo_read_pairs.short.bed` | 8,560,907 | Row-for-row identical, columns 4-5 blanked to `.` | Fast interval work (bedtools/coverage). 255 MB instead of 1.1 GB. **Row *i* here is row *i* there** - strip and rejoin by line number. |
| `longest_de_novo_read_pairs.bed` | **541,998** | Per promoter, the single longest-span pair within its group; all ties kept | Structural questions: how far does the transcript from this de novo promoter reach, and what does it run into? Ties are why 541,998 > 446,196. |

Group skew in the longest file: 381,508 Cntl vs 160,490 KO rows - the KO libraries are smaller
overall, so **any between-group comparison needs depth normalization.**

---

## Practical notes before running an analysis

1. **Strand handling is the #1 source of silent errors.** TSS is at `chromEnd - 1` for `-` and
   `chromStart` for `+`. Anything treating the interval midpoint as the TSS will smear motif
   and metagene plots.
2. **Column 13 is the best TSS anchor** - a single base, already chosen as best-supported.
3. **Coverage `NA` is not 0.** It means "no peak called in that group." Substituting 0 is often
   defensible but is an explicit assumption: 190k of 453k promoters have an `NA` on one side.
4. **`score` is a max, not a sum** - biased toward whichever group is deeper. Do not use it as
   a normalized cross-group activity measure; use column 15 with proper normalization.
5. **Classification is not dependency.** vPIC-classified means "has a TATT box at -32." Whether
   that promoter's activity actually *depends* on vPIC is what the Cntl/KO contrast answers,
   and the two do not agree perfectly - 14,877 vPIC-motif promoters are KO-only. That gap is
   probably where the interesting biology is.
6. **`chrMT` has 16 entries** - likely artifact, worth dropping.

---

## Which file is "the" file, and the folder-1 / folder-2 reconstruction trap

Added 2026-09-08.

`de_novo_promoters.classified.bed` (folder 1) is the spine: the only file with one row per
biological entity, carrying the coordinates, the classification and the authoritative
quantitation. Nearly every analysis will be shaped as "for each of these 453,576 promoters,
...". Folder 2 has no rows that aren't traceable to a folder-1 promoter (**verified**: of
8,560,907 read pairs, exactly **1** fails to anchor inside a folder-1 promoter interval).

But folder 2 is not merely redundant support. It holds three things folder 1 cannot give:

1. **Per-replicate counts.** Folder 1 collapses to two numbers per promoter
   (`Cntl_coverage`, `KO_coverage`) - enough for a fold change, not enough for a statistical
   test, since within-group variance cannot be estimated from a collapsed number. Folder 2
   keeps the six libraries separate. **Verified**: 399,328 of 453,576 promoters (88%) have
   qualifying pairs in >=3 of the 6 libraries; 41,717 in 2; 5,151 in 1.
2. **Where the transcript goes.** Folder 1 is a point. Folder 2 gives span and splice
   junctions - the only route to "what does this cryptic promoter actually transcribe."
3. **The ability to re-filter.** Folder 1's coverage is baked; folder 2 permits read-level QC.

### The trap: folder 2 is a strict, non-uniformly filtered subset

Attempted to reconstruct folder-1 coverage by counting folder-2 read pairs per promoter per
group, genome-wide. It does not reconstruct.

- **Folder 2 is a strict subset**: declared coverage >= observed read count in *both* groups
  for **100.0%** of promoters (453,360 / 453,576; 216 exceptions, 0.05%). So folder 2 never
  adds signal, only loses it.
- Only **7.2%** of promoters have folder-1 coverage exactly equal to folder-2 read count.

| | declared (folder 1) | observed (folder 2) | retention |
|---|---|---|---|
| Cntl | 8,604,997 | 5,838,468 | **67.8%** |
| KO | 6,044,067 | 2,722,438 | **45.0%** |

A 22.8-point gap. Signal vanishes entirely for a group at 38,366 promoters (8.5%) in Cntl and
**121,725 (26.8%)** in KO.

**Retention is strongly class-dependent, and not in one direction:**

| Class | Cntl retention | KO retention |
|---|---|---|
| Zta | 51.3% | **64.3%** |
| vPIC | 73.5% | **13.3%** |
| Zta_vPIC | 67.5% | 37.6% |
| Unclassified | 59.5% | 60.2% |

Checked and ruled out promoter width as the explanation - stratifying by interval width
(1 / 2-4 / 5-9 / 10+) does not account for the group asymmetry, and width-1 promoters show
the same pattern (Cntl 71.2% vs KO 34.5%).

**The cause is unresolved.** Two readings, with opposite implications:

- *Technical*: the "qualifying pair" filter (proper pair, both mates mapped) discards
  reads at a rate that happens to correlate with class and group - in which case the gap is
  pure artifact.
- *Biological*: vPIC promoters in the KO genuinely fail to produce complete paired
  fragments - in which case the retention difference is itself a result.

Worth resolving before leaning on either file for the vPIC/KO contrast.

### Practical rule

- **Folder 1** = unit of analysis, classification, and **all between-group quantitation**
  (column 15).
- **Folder 2** = replicate-level statistics, transcript structure, read-level QC.
- **Never compute Cntl-vs-KO fold changes from folder-2 read counts.** The 22.8-point
  retention gap would inflate every Cntl-over-KO effect, and for vPIC promoters it would
  manufacture a ~5.5x artifactual depletion out of nothing.
