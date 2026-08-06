# transcript-expression-and-ebv-biology

Why `de_gene` isn't a subset of `transcript_de`, what a transcript
actually is, and why transcript-level resolution matters for EBV research
specifically.

## de_gene is not derivable from transcript_de

Unlike `expression_gene`/`expression_transcript` (gene-level really is
`SUM(transcript-level)` — verified and excluded as a table in
`Warehouse_Design_table.md`), `de_gene` and `transcript_de` are two
separate statistical tests run by two different tools (DESeq2 vs.
sleuth) on differently-prepared data. DESeq2 pools a gene's reads
together first, then tests that pooled count. Sleuth tests each
transcript's abundance on its own. Different inputs, different models —
not arithmetically related.

Confirmed with a real example (gene `TPT1`, smoke-test run): its 4
transcripts have sleuth `b` values `0.44, 1.56, -0.21, -0.45` — summing
or averaging gives a small positive number. `de_gene`'s
`log2_fold_change` for the same gene is `-1.07` — negative, larger
magnitude, no formula connects the two. Significance disagrees too: every
individual transcript's `qval` is non-significant (`0.17`, `0.74`,
`0.9997`, `0.9997`), but the gene-level `padj` is `0.032` — significant.
Pooling all of a gene's reads before testing gives `de_gene` more
statistical power than any single transcript has alone, which is why
this can happen.

## What a transcript actually is

A gene is a stretch of DNA made of exons (kept) and introns (cut out).
Splicing is the editing step that turns the raw copy into a final mRNA,
and the cell has choices during it — which exons to include, which to
skip, sometimes exactly where to cut. Different choices from the same
gene produce different final mRNA molecules. Those different molecules
are transcripts (isoforms) — same gene, different products.

## Why transcript-level differences matter biologically

Different isoforms of the same gene can do genuinely different, even
opposite, jobs — not just "more or less of the same thing." A classic
case: one isoform (with a particular exon included) promotes cell
survival, another isoform of that same gene (exon skipped) promotes cell
death instead. Measuring only total gene expression would show no
change, because the two isoforms cancel out in the total count — the
same blind spot as the `TPT1` example above, but with a real functional
consequence attached instead of just statistical noise. Isoforms can
also differ in whether they make a full working protein at all — one
version functional, another truncated, broken, or actively interfering
with the working one.

## Why this matters specifically for EBV

Splicing manipulation is central to how herpesviruses like EBV operate,
on both sides:

- EBV packs many genes into a small viral genome and gets extra protein
  variety by splicing the same viral transcript multiple different ways,
  rather than needing a separate gene per protein.
- EBV is known to interfere with the host cell's own splicing machinery
  during the lytic switch as part of taking the cell over.

So `splicing_event`/`transcript_de` aren't just a finer-grained version
of `de_gene` — they're tracking a mechanism (alternative splicing) that's
central to how this virus actually works, not a statistical nice-to-have.
