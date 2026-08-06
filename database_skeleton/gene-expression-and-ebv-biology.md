# gene-expression-and-ebv-biology

What "gene expression" (RNA-seq) actually measures, and why it's the core
readout for this experiment's EBV latency-to-lytic reactivation design.

## What RNA-seq expression measures

DNA → mRNA (transcription) → protein (translation). RNA-seq counts mRNA
molecules — a readout of transcription/mRNA stability, not protein
abundance. The two usually correlate but aren't identical: translation
efficiency and protein degradation rate both happen downstream of what
RNA-seq sees. mRNA is measured genome-wide because it's cheap and
scalable (~20,000 genes in one run); genome-wide protein measurement
(mass spec) is harder and far more expensive. So expression data is a
proxy for what the cell is currently transcribing, not a direct
measurement of what it has built.

## Why it matters

Every cell has the same DNA; what differs between cell types/states is
which genes are expressed. Comparing expression between two conditions
(`test` vs `cntl`) shows which parts of that program the cell switched on
or off in response to a stimulus.

## EBV latency vs. lytic — what this experiment is actually testing

EBV has two infection modes:

- **Latency**: viral genome sits quiet inside the cell, expresses only a
  handful of its ~80 genes, makes no new virus, cell keeps dividing —
  how EBV persists lifelong and evades the immune system.
- **Lytic**: a switch flips, most of the viral genome activates in a
  coordinated cascade, the cell is hijacked to replicate viral DNA and
  build new virus particles, then dies to release them.

The latency→lytic switch is controlled by two viral proteins, Zta
(`BZLF1`) and Rta (`BRLF1`). This experiment's `perturbation_target =
Zta+Rta` forces that switch directly — every `test` vs `cntl` comparison
in this warehouse is measuring what changes, host and viral, when EBV is
forced out of latency.

## What researchers are looking for

- **Host response**: does the cell mount an antiviral/interferon
  response, or does EBV suppress host expression broadly ("host
  shutoff"), a documented herpesvirus behavior during lytic replication.
- **Viral gene cascade**: the reference genome (`hg38plusAkataInverted`)
  includes both human and EBV (Akata strain) sequence, so `de_gene`
  contains viral genes alongside host genes. Confirmed in this
  warehouse's own data — the top genes by `padj` in the smoke-test run
  include `BHLF1_1`, `BMRF1_1`, `BORF2_1`, `BRLF1_1`, `BcLF1_1`:
  canonical EBV lytic genes, not human ones. `BRLF1` is Rta itself
  (directly transfected in); `BMRF1`/`BORF2` are early viral
  DNA-replication machinery; `BcLF1` is the major capsid protein, a late
  gene — meaning the cascade had progressed far enough to begin building
  new virions.
- **Clinical relevance**: EBV-associated cancers (this cell line, SNU719,
  is EBV+ gastric carcinoma) are thought to be driven by latent viral
  gene expression acting as an oncogene program. Deliberately forcing
  lytic reactivation to kill infected tumor cells ("lytic induction
  therapy") is a real therapeutic strategy, making the order and
  magnitude of host/viral gene changes during this switch directly
  relevant.
- **Splicing disruption**: herpesviruses are known to alter host RNA
  splicing during lytic infection — why this pipeline runs
  `splicing_event`/`transcript_de` alongside `de_gene`, not gene-level
  counts alone.
