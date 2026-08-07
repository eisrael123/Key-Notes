# signal-over-gene-coverage-vs-expression

Why `signal_over_gene` (coverage) isn't redundant with `expression_gene`
(counts/TPM), how researchers actually use coverage in practice, and
whether this could be automated.

## Count vs. coverage — the actual difference

`expression_gene` gives one number: how much of a gene, total, added up
across all its reads. `signal_over_gene` gives a shape: how evenly
coverage is spread across the gene's full length, not just the total.

Example: two samples show the same TPM for gene X in `expression_gene` —
identical. But sample A has reads spread evenly across the whole gene
body; sample B has reads piled almost entirely at one end, barely any
coverage elsewhere — a classic sign of degraded RNA or a technical
artifact, not real biology. `expression_gene` can't tell these apart,
since it only sees the total. `signal_over_gene` catches it: sample B
would show low `covered_bases` and `max_coverage` far higher than
`mean_coverage` (a spike, not a flat profile); sample A would look even
across both.

## Why not just auto-filter uneven-coverage genes

Reasonable-sounding idea, but three problems with baking it in
automatically:

- No universal "even enough" threshold — short genes naturally look
  spikier than long ones (fewer possible read positions) without being
  degraded. A flat cutoff would misjudge based on gene length.
- Uneven coverage isn't always junk — can reflect real biology (an
  alternative start site, partial transcription in one condition).
  Auto-filtering would discard real signal along with actual artifacts.
- Degradation is usually a whole-sample problem, not a per-gene one — if
  RNA in a sample degraded, essentially every gene in that sample shows
  the same lopsided pattern. The right fix is flagging/excluding the
  whole sample, not filtering genes one at a time within it. Checked
  `qc_metric.tsv`'s actual tools (`fastp`, `fastqc`, `star`, `kallisto`,
  `rseqc`) — the RSeQC metrics loaded here are about strandedness, not a
  sample-wide coverage-evenness score. This pipeline may not currently
  compute that sample-level check at all.

Conclusion: keep the raw coverage data available (already the case,
`signal_over_gene`) and let evenness be a threshold a researcher applies
for their specific question at query time, rather than something
silently and irreversibly dropped at load time. Precedent already exists
for this split: DESeq2 does apply its own automatic filtering in
`de_gene`, but on mean expression level (to improve statistical power),
not coverage shape.

## How researchers actually use coverage in practice

Not by manually checking every gene — that would be impractical, and
isn't how it's done. It's a funnel:

1. Genome-wide numeric screen (`de_gene`) narrows ~20,000 genes down to
   a short list of significant/interesting hits.
2. `signal_over_gene`'s numbers can flag which of those candidates look
   suspicious (e.g. `max_coverage` far above `mean_coverage`) before
   anyone opens a browser.
3. Only that short list — a handful to a few dozen genes — gets manually
   inspected in a genome browser (IGV or similar), loading the
   corresponding bigWig track to visually confirm the signal looks real
   before trusting the result enough to publish or plan a follow-up
   experiment.

This is exactly why `bigwig_manifest` carries `sample_id`/`strand`/
`file_path` as real columns rather than being redundant with
`artifacts_manifest` — its purpose is making "which bigWig file do I
load for gene X, sample Y, this strand" a fast lookup for this specific
step.

## Has this been automated / has ML been tried

Yes, it's a real research area, though not yet a routine default step in
ordinary single-lab pipelines like this one.

- [Detecting, Categorizing, and Correcting Coverage Anomalies of RNA-Seq
  Quantification](https://pubmed.ncbi.nlm.nih.gov/31786209/) —
  computational methods that flag genes where coverage doesn't match
  what the quantification algorithm assumed (e.g. unannotated
  isoforms), automatically rather than by eye.
- `IRcall`/`IRclassifier` — combines expression level, region-specific
  coverage, and read counts to separate real intron-retention events
  from false positives.
- Autoencoder-based [unsupervised anomaly
  detection](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8467021/) in
  genomic data — learns what "normal" looks like from a large real
  dataset, flags what reconstructs poorly.
- The more powerful modern angle: [Enformer/Basenji-lineage
  models](https://www.nature.com/articles/s41576-025-00887-2) predict
  what a coverage track *should* look like directly from DNA sequence.
  Compare observed coverage to the model's prediction, and a big
  mismatch is itself the anomaly signal — no hand-tuned "evenness" rule
  needed.
- Reality check: none of this is a default step in typical single-lab
  pipelines like this one yet. Most still use cheaper statistical
  heuristics (RSeQC coverage-evenness, Picard bias metrics) because
  they're simpler and cheap to run; ML approaches show up more in
  large-scale reference-building efforts or specialized detection
  tasks, not routine per-experiment QC.
