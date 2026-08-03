# Revised experiment parameter schema

Revised version of the per-experiment metadata parameters, plus the `metadata.py` command-line
arguments the pipeline already reads today, included for reference alongside the proposed field
set below.

## Per-experiment parameters (revised)

| Field | Allowed values / format | Notes |
|---|---|---|
| `CELL_LINE` | `Mutu`, `Akata`, `DG75`, `SNU719`, `YCCEL1`, `BCBL1`, `Raji`, `P3HR1`, `HepG2`, `HEK293` | |
| `PERTURBATION_TYPE` | `transfection`, `chemical`, `BCR-crosslink`, `none`, `unknown` | |
| `PERTURBATION_AGENT` | e.g. `Zta`, `Rta`, `BRRF1`, `Zta+Rta`, `anti-IgG`, `anti-IgM`, `CC115`, `siRNA`, `CRISPR`; `NA` when `PERTURBATION_TYPE = none`; `unknown` if not recorded | |
| `PERTURBATION_TARGET` | e.g. `BMRF1`, `SRSF1`, `CNOT1`, `UPF1`; `NA` when the agent has no separate molecular target (e.g. `Zta`/`Rta` overexpression); `unknown` if not recorded | **New field** — see note below |
| `PERTURBATION_DOSE` | e.g. `5ug+5ug`, `100nM`; `NA` when `PERTURBATION_TYPE = none`; `unknown` if not recorded | |
| `CO_TREATMENT` | `none`, `PAA`, `siRNA`, `CRISPR`, `unknown` | |
| `CO_TREATMENT_TARGET` | `PAA_replication`, `CNOT1`, `CNOT9`, `UPF1`, `EXOSC3`, `NAT10`, `TET1`, `BRRF1`; `NA` when `CO_TREATMENT = none`; `unknown` if not recorded | |
| `FACS_PURIFIED` | `yes`, `no`, `unknown` | |
| `FACS_GFP_PROMOTER` | `pCMV`, `BMRF1p`, `none`, `unknown` | |
| `LIBRARY_SELECTION` | `polyA`, `ribodepleted`, `unknown` | |
| `LIBRARY_STRANDEDNESS` | `stranded`, `unstranded`, `unknown` | See note below |
| `LIBRARY_TYPE` | `PE`, `SE` | |
| `TIMEPOINT_HOURS` | e.g. `6`, `12`, `24`; `NA` | |
| `SEQUENCING_RUN_DATE` | `YYYY-MM-DD`; `NA` | |
| `NOTES` | free text | |

### For review

1. **`PERTURBATION_TARGET` is a new field.** `CO_TREATMENT` already pairs with
   `CO_TREATMENT_TARGET` (e.g. a co-treatment of `siRNA` against `CNOT1`), which keeps "every
   CNOT1 knockdown" a queryable filter. The primary `PERTURBATION_AGENT` didn't have an
   equivalent — if the main perturbation is itself a knockdown (e.g. an siRNA against a specific
   gene, not as a co-treatment), there was nowhere to record which gene. Added
   `PERTURBATION_TARGET` to mirror `CO_TREATMENT_TARGET`.

2. **`LIBRARY_STRANDEDNESS` is intentionally separate from the pipeline's existing `strandedness`
   column**, not a duplicate. `strandedness` is measured empirically per sample after alignment
   (RSeQC inspects the aligned reads and reports what the library actually turned out to be, e.g.
   `fr-firststrand`). `LIBRARY_STRANDEDNESS` records what was intended/ordered at library prep,
   entered as metadata up front. The two normally agree; a mismatch between them is a useful flag
   for a mislabeled sample or wrong kit, which is why both are kept.

## `metadata.py` command-line arguments (current pipeline, for reference)

Positional arguments:

| Argument | Description |
|---|---|
| `fastq_root_dir` | Directory containing the `test/` and `cntl/` subdirectories |
| `reference_dir` | Path to `referenceFiles` |
| `species_name` | Genome build; must match a directory under `reference_dir` |
| `investigator_name` | Investigator name |
| `experiment_type` | `PE` or `SE` |
| `results_dir` | Output directory |

Experiment option flags (prompted interactively if omitted):

| Flag | Description |
|---|---|
| `--cell-line` | One of the controlled cell line values |
| `--organism` | One of the controlled organism values |
| `--perturbation-type` | One of the controlled perturbation type values |
| `--induced-program` | Biological program induced (e.g. lytic reactivation, latency, none) |
| `--library-selection` | RNA selection before library prep |
| `--perturbation-target` | e.g. `BMRF1`, `SRSF1`, `CC115` |
| `--perturbation-dose` | e.g. `100nM` |
| `--timepoint-hours` | Hours post-perturbation, or `NA` |
| `--sequencing-run-date` | ISO 8601 date, or `NA` |
| `--notes` | Free text |
| `--non-interactive` | Never prompt; missing values are an error |
