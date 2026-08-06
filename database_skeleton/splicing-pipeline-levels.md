# splicing-pipeline-levels

The four levels of splicing data in this warehouse, raw to summarized,
and where a real statistical test enters (not just grouping/averaging).

`junction` captures every splice junction observed, full stop — both the
routine, constant intron removal every gene does, and any alternative
splicing junctions, with no distinction made between them.
`splicing_event_replicate`/`splicing_event`/`splicing_summary` all come
from rMATS, and rMATS only detects and tests **alternative** splicing —
it doesn't report on ordinary intron removal at all, since that never
varies and isn't interesting. So `junction` is the only one of the four
that includes routine splicing; the upper three are alternative-splicing
only.

## junction
Raw. Every splice junction seen in a sample, at a genomic coordinate, no
interpretation attached.

## splicing_event_replicate
Junction reads sorted into "supports inclusion" / "supports skipping"
for one specific tested splicing choice (event), per individual
replicate. Not simple grouping from `junction` — requires knowing which
junctions belong to which event's inclusion/skip architecture, a
categorization step, not a SQL aggregate.

## splicing_event
Replicates averaged per group (`inc_level_1_mean`/`inc_level_2_mean` =
`AVG(inc_level)` grouped by `sample_group` — this part IS simple
grouping), plus `pvalue`/`fdr` — rMATS's actual statistical test on the
underlying read counts. This is the real break in the chain: the test
result can't be recreated by grouping/averaging
`splicing_event_replicate`, only the mean columns can. Reproducing
`pvalue`/`fdr` means re-running rMATS's test, not writing a `GROUP BY`.

## splicing_summary (view, not a table)
`COUNT()` over `splicing_event`, grouped by event_type/counting_mode,
filtered on the fdr/inc_level_difference thresholds. Pure aggregate of
already-loaded data, no new computation — why it's a view instead of a
stored table.

## Where "just grouping" stops working

`splicing_event` is the level that introduces a real statistical test.
Everything below it (junction → replicate) and above it (event →
summary) can be reproduced by categorizing or aggregating data that's
already loaded. `splicing_event`'s `pvalue`/`fdr` can't — that step
needs rMATS to actually run its test on the read counts, not a database
query.

Chain: `junction` → `splicing_event_replicate` → `splicing_event`
(test applied here) → `splicing_summary` (view).

## What splicing is, and what each level means biologically

When a cell copies a gene into RNA, it copies the whole thing, including
filler pieces (introns) sitting between the useful pieces (exons) it
actually needs. Splicing is the editing step where the cell cuts out the
filler and glues the useful pieces together into the final, usable
message. The cell can also choose to keep or cut certain exons, not just
the introns — same gene, different edit, different final message. That
choice is what these four tables track, at increasing levels of
interpretation:

- **junction** — physical proof a splice happened somewhere. Two exons
  glued together leave a seam; a sequencing read spanning that seam is a
  junction. Just says a splice occurred, here, this many times — no
  judgment about whether it matters.
- **splicing_event_replicate** — proof organized around one specific
  choice the cell can make: for one exon it sometimes keeps and
  sometimes drops, did this one sample keep it or drop it, and how
  often. Turns raw "a splice happened" into "here's what one sample
  actually decided" at a real fork in the road.
- **splicing_event** — the actual scientific answer: did cells in one
  condition make that choice differently and reliably than the other, or
  could the difference be chance. This is where something real gets
  learned — e.g. "when EBV goes lytic, the cell starts keeping this exon
  more often," a discovery about the virus altering how the cell edits
  its own messages.
- **splicing_summary** — a scoreboard, not new biology. Out of
  everything tested, how many choices actually changed in a trustworthy
  way. Useful for getting oriented before looking at individual genes,
  not for learning anything new on its own.

## Why alternative splicing specifically, not splicing in general

Constitutive splicing (plain intron removal) is the same every time, for
every copy of a gene, in every condition — it never varies, so comparing
it between `test` and `cntl` couldn't reveal anything. Alternative
splicing is the only part of the process that's regulated and
condition-dependent, which is why it's the only part worth testing.

**Why it matters biologically, in general:** it's how a limited number of
genes produces a much larger number of distinct proteins — including one
exon vs. skipping it can produce two proteins with different domains, or
even opposite functions, from the exact same gene. It's also fast and
reversible: a cell can change its splicing choices in response to a
signal without altering its DNA or waiting on new gene expression from
scratch, making it a quick way to reprogram what proteins get made. This
is why alternative splicing shows up heavily in development, cancer, and
infection — situations where a cell needs to change its identity or
behavior quickly.

**Why it matters for EBV specifically:** two directions. First, EBV's own
genome relies on alternative splicing to punch above its weight — a
compact viral genome produces many different proteins across its
infection stages by splicing the same primary transcript different ways,
rather than needing a separate gene per protein. Second, and more
relevant to the host side of this data: EBV is known to interfere with
the host cell's own splicing machinery during lytic infection, not just
shut host genes off wholesale. That's a subtler kind of interference than
turning a gene down — a gene can still look normally transcribed (no
change in `de_gene`) while the virus quietly breaks it by disrupting
which exons get included, producing a nonfunctional protein instead of a
missing one. That's exactly the kind of effect `de_gene` alone can't
catch, and why `splicing_event`/`transcript_de` exist as a separate
layer rather than being assumed to follow from gene-level expression.
