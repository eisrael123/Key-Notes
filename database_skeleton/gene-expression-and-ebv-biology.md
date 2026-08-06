# gene-expression-and-ebv-biology

What "gene expression" actually measures, and why it's the right thing to
measure for this experiment.

## What gene expression measures

DNA gets copied into mRNA, then mRNA gets built into protein. RNA-seq
counts mRNA, not protein. The two are related but not the same — a gene
can make a lot of mRNA that never turns into much protein, or the
reverse.

Researchers measure mRNA instead of protein directly for two real
reasons, not just cost:

- mRNA sequencing reliably catches low-abundance molecules. Many of the
  most important proteins in biology — transcription factors especially
  — are naturally low in number and easy to miss with direct protein
  measurement unless you go out of your way to enrich for them.
- mRNA and protein aren't answering the same question. mRNA shows what
  the cell is currently transcribing. Protein shows what it's actually
  built. For an experiment about transcription factors reprogramming a
  cell — which is exactly what this one is — mRNA is the direct, correct
  thing to measure, not a cheap stand-in for the "real" answer.

If the actual question were about protein stability or how efficiently
mRNA gets turned into protein, RNA-seq wouldn't answer that no matter the
budget — that needs a protein-specific method regardless. Serious studies
often follow up an RNA-seq result with something protein-specific (a
Western blot, targeted mass spec) to confirm a change also shows up at
the protein level, not just the mRNA level.

## Why gene expression matters

Every cell has the same DNA. What makes cell types different is which
genes are turned on. Comparing expression between two conditions shows
which genes the cell switched on or off in response to something.

## What this experiment is actually testing

EBV lives in two modes:

- **Latent**: the virus stays quiet, makes only a few of its own genes,
  doesn't build new virus, and the cell keeps dividing normally. This is
  how EBV survives in a person for life without being noticed by the
  immune system.
- **Lytic**: the virus flips a switch, turns on most of its genes,
  hijacks the cell to build copies of itself, then kills the cell to
  release them.

Two viral proteins control that switch: Zta (`BZLF1`) and Rta (`BRLF1`).
This experiment directly forces that switch by adding Zta+Rta to the
cells. Every `test` vs `cntl` comparison in this database is measuring
what changes — in the cell and in the virus — when that switch gets
flipped.

## What people are looking for

- Does the cell fight back (turn on antiviral genes), or does the virus
  successfully shut the cell's own genes down? EBV is known to actively
  suppress the host's normal genes during the lytic switch.
- The reference genome used here includes both human and EBV DNA, so the
  same results table has human genes and viral genes side by side. This
  warehouse's own data already shows it: the top significant genes in
  the smoke-test run include `BHLF1_1`, `BMRF1_1`, `BORF2_1`, `BRLF1_1`,
  `BcLF1_1` — EBV genes, not human ones. `BRLF1` is Rta itself.
  `BMRF1`/`BORF2` build viral DNA. `BcLF1` builds the virus's outer
  shell — meaning the cell had gotten far enough along to start building
  new virus.
- This cell line (SNU719) comes from a real EBV-linked cancer (gastric
  carcinoma). The cancer is thought to be driven by the virus staying
  latent. One treatment idea is to force the lytic switch on purpose to
  kill the infected cancer cells. Understanding exactly what changes
  during that switch is directly relevant to whether that idea works.
- EBV is also known to mess with how the cell splices its own mRNA, not
  just how much of it gets made — why this pipeline also tracks splicing
  changes, not just overall gene activity.
