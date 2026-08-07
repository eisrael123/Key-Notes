# gsea-pathway-enrichment

What a "pathway" / "gene set" is, what GSEA actually tests, and whether the pipeline's GSEA
stage matches the classic textbook description of the method.

## What a gene set / pathway is

A single gene rarely does anything alone. Cells run coordinated programs — dozens to hundreds
of genes that get turned on or off together to accomplish one biological job. A "gene set" (also
called a "pathway," loosely) is just a pre-existing, curated list of genes known to work together
for one purpose. Examples relevant to this lab's EBV work:

- A list of every gene known to be a target of a specific transcription factor (e.g. "genes that
  NF-kB turns on").
- A list of every gene involved in the interferon antiviral response.
- A list of every gene in a specific signaling cascade (a "canonical pathway," e.g. the MAPK
  cascade).

These lists come from decades of separate experiments (ChIP-seq, knockouts, literature curation)
and are bundled into MSigDB, the standard reference collection GSEA reads from. The point of
pathway analysis is that a *single* gene changing 20% often means nothing statistically, but
finding that 80% of a *known coordinated program* all moved in the same direction is a much
stronger, more interpretable signal than staring at one gene at a time. See
[[gene-expression-and-ebv-biology]] for what "gene expression" is measuring in the first place.

## What GSEA actually tests — confirmed against this pipeline's code

Ethan's summary going in was: *rank all genes from most-up to most-down in test vs. control, then
ask whether a known pathway's genes cluster toward one end of that list rather than being
scattered randomly.* That's exactly right, and it's confirmed directly in `rnaseq.py` — but the
pipeline actually runs GSEA in **one of two modes**, and only one of them literally builds and
consumes a single sorted list the way the description implies.

`run_enrichment()` (`rnaseq.py:1951-1994`) picks the mode based on replicate count:

- **Fewer than 3 replicates in either condition → "preranked" mode.** `create_rnk_file()`
  (`rnaseq.py:783-823`) takes the DESeq2 results, keeps `gene` and `log2FoldChange`, and sorts
  descending — most upregulated-in-test genes at the top, most upregulated-in-control genes at
  the bottom (`rnaseq.py:812-813`, comment: *"descending = upregulated at top"*). That `.rnk` file
  is handed straight to `GSEAPreranked` (`rnaseq.py:825-866`). This is the textbook description,
  word for word.
- **3+ replicates in both conditions → "standard" mode.** `create_gsea_input_file()` writes raw
  per-sample TPMs (not a pre-sorted list) plus `create_gsea_cls_file()` writes a `control`/`test`
  phenotype-label file, and classic `GSEA` (not `GSEAPreranked`) is invoked
  (`rnaseq.py:721-781`). In this mode GSEA computes its **own** ranking metric internally from the
  expression matrix (defaulting to Signal2Noise since no `-metric` flag is passed) — it still
  produces and uses a single ranked list, just not the DESeq2 one, and it isn't ours to inspect
  directly. Significance is tested by permuting **gene sets**, not the phenotype labels
  (`-permute gene_set`, `rnaseq.py:764`) — a deliberate choice given how few samples per condition
  these experiments typically have; permuting the actual sample labels needs a larger `n` to be
  reliable.

Both modes run the same underlying statistic once they have *a* ranked list: a running-sum
enrichment score that rewards a pathway's genes for appearing bunched near one end and penalizes
them for being spread out evenly — this is the actual "does it cluster" test, and it's identical
either way. The only difference is where the ranking comes from and how the p-value is computed.

**Which mode a given run used is on the record**: `manifest.record_stage("gsea_mode", ...)` writes
`"preranked"` or `"standard"` into that run's `run_manifest.json` (`rnaseq.py:2195`). Confirmed
live during the `Mutu_Zta_2019-12-20` batch run (3 cntl + 4 test replicates, so "standard" mode):
`docker exec ... ps aux` showed the actual GSEA Java command running with `-cls test.cls
-permute gene_set`, matching `run_gsea()`'s command exactly, not `GSEAPreranked`.

## Direction convention

`gene_set_enrichment.tsv`'s `nes` (normalized enrichment score) column follows the same sign
convention as everything else differential in this pipeline: **positive = enriched among genes
upregulated in test/perturbed, negative = enriched among genes upregulated in cntl/control.**
Full details (and why DESeq2/Sleuth/rMATS independently land on the same convention) are in
[[comparison-id-and-stat-direction]] (in the `rnaseq-pipeline` notes, not duplicated here).

## Gene set collections this pipeline actually tests against

Whichever mode runs, it loops over every `.gmt` file in
`referenceFiles/<species>/GSEA/` and produces one row per gene set per collection in
`gene_set_enrichment.tsv`, with `collection` distinguishing them (`tables_gsea.py:70-123`). For
the hg38+Akata-EBV reference currently in use, that's:

| Collection | What it is | EBV relevance |
|---|---|---|
| `biocarta` | Curated signaling/metabolic pathway diagrams | General pathway context |
| `canonical_pathways` | Reactome/KEGG/PID-style curated pathways | Signaling cascades EBV proteins hijack |
| `kegg_medicus` | KEGG's clinically-curated pathway set | Disease-relevant pathway context |
| `gene_ontology` (full), `biological_process`, `molecular_function` | GO terms — what a gene *does* | Broadest, least specific signal; full GO is skipped when the BP/MF splits are present (`rnaseq.py:738-746`) — it OOM'd a prior smoke run once the splits already covered the same ground |
| `chem_and_genetic_perturbations` | Genes that moved in published drug/knockout experiments | Useful for spotting "this looks like a known drug's signature" |
| `microRNA_targets` | Genes targeted by specific microRNAs | EBV encodes its own microRNAs that manipulate host gene expression — this collection is the direct way to test "did a known EBV or host miRNA's targets move as a block" |
| `transcription_factor_targets` | Genes downstream of a specific transcription factor | **The most directly relevant collection to this lab's experiments.** Zta (`BZLF1`) and Rta (`BRLF1`) — the two viral proteins these experiments force on to flip the latent→lytic switch — are themselves transcription factors. A positive/negative NES hit here can point at exactly which downstream TF program the switch is routing through, human or viral. |

## Why pathway-level results matter more than usual for this lab's question

Per [[gene-expression-and-ebv-biology]], the central question these experiments ask is whether
the cell mounts a coordinated antiviral response or gets its own machinery coordinately shut down
during the lytic switch — that's inherently a "did a whole program move together" question, not a
single-gene one. GSEA is the tool built specifically to answer that, which is why it's a standing
stage in this pipeline rather than an optional add-on.
