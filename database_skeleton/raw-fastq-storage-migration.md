# Raw fastq storage migration (TUNGSACore3 -> efknas1/FlemingtonLabMain1)

## Why

Freeing space on `TUNGSACore3` for pipeline analysis. Raw fastq input
folders there (renamed to match this project's schema) total **~1.9TB**.
Lab collaborator ("him") responded that all new data should live on
`efknas1` — confirmed this is the same mount referred to elsewhere as
`/Volumes/FlemingtonLabMain1` — and asked for a mindful, durable directory
structure since it needs to last.

## Decision: destination structure

Copy (not move, not in-place rename) into a new folder next to the
existing `1_RNA_seq`, so his original files are never touched:

```
/Volumes/FlemingtonLabMain1/2b_Flemington_Lab_Experiments/RNA_seq_raw_data/
  <cell_line>/
    <run_name>/
      cntl/...
      test/...
      input_args.json
```

This mirrors the existing local layout (`<run_name>/<cntl|test>/<file>`)
exactly, just nested one level deeper under `<cell_line>`, so the loader
change is additive (new path = `RNA_seq_raw_data/<cell_line>/` + old
relative path) rather than a full path scheme rewrite.

`1_RNA_seq` (his original, untouched) is meant to be deleted **eventually**,
but only once the copy is verified complete and all data is reprocessed
from the new location — and that deletion is a separate, much higher-risk
decision than clearing local `TUNGSACore3` space, since it's the lab's
canonical raw data on shared infrastructure. Get his explicit go-ahead at
that specific moment, not now.

## cell_line mapping (derived from data, not folder-name parsing)

Each run folder has an `input_args.json` with a clean top-level
`cell_line` field — used that instead of guessing from the folder name
string (fragile: naming patterns aren't fully consistent across runs).

| run folder | cell_line |
|---|---|
| `Akata_anti-IgG_24hr_2022-12-08` | `Akata` |
| `Akata_anti-IgG_48hr_2022-12-08` | `Akata` |
| `BCBL1_Rta_2022-05-16` | `BCBL1` |
| `Mutu_anti-IgM_2020-09-16` | `Mutu` |
| `Mutu_Zta_2019-12-20` | `Mutu` |
| `Mutu_Zta_2020-04-02` | `Mutu` |
| `Mutu_Zta_2020-11-26` | `Mutu` |
| `Mutu_Zta_EXOSC3-control_2025-08-31` | `Mutu` |
| `Mutu_Zta_EXOSC3-KO_2025-08-31` | `Mutu` |
| `Mutu_Zta-PAA_2020-11-26` | `Mutu` |
| `SNU719_Rta-Zta_2025-04-10` | `SNU719` |

Excluded from this list: `Flemington_RNAseq_analysis_output` (pipeline
output, not raw input) and `testing` (smoke-test scratch, not a real run)
under `/Volumes/TUNGSACore3/rnaseq_runs`.

## Copy script

```bash
#!/bin/bash
set -euo pipefail

SRC_BASE="/Volumes/TUNGSACore3/rnaseq_runs"
DEST_BASE="/Volumes/FlemingtonLabMain1/2b_Flemington_Lab_Experiments/RNA_seq_raw_data"

declare -A RUN_TO_CELLLINE=(
  ["Akata_anti-IgG_24hr_2022-12-08"]="Akata"
  ["Akata_anti-IgG_48hr_2022-12-08"]="Akata"
  ["BCBL1_Rta_2022-05-16"]="BCBL1"
  ["Mutu_anti-IgM_2020-09-16"]="Mutu"
  ["Mutu_Zta_2019-12-20"]="Mutu"
  ["Mutu_Zta_2020-04-02"]="Mutu"
  ["Mutu_Zta_2020-11-26"]="Mutu"
  ["Mutu_Zta_EXOSC3-control_2025-08-31"]="Mutu"
  ["Mutu_Zta_EXOSC3-KO_2025-08-31"]="Mutu"
  ["Mutu_Zta-PAA_2020-11-26"]="Mutu"
  ["SNU719_Rta-Zta_2025-04-10"]="SNU719"
)

for run in "${!RUN_TO_CELLLINE[@]}"; do
  cell_line="${RUN_TO_CELLLINE[$run]}"
  dest_dir="$DEST_BASE/$cell_line"
  mkdir -p "$dest_dir"
  echo "Copying $run -> $dest_dir/$run"
  cp -R "$SRC_BASE/$run" "$dest_dir/$run"
done
```

## Caveats / open items

- Hit `Operation not permitted` just `ls`-ing
  `/Volumes/FlemingtonLabMain1/2b_Flemington_Lab_Experiments/` from this
  session — likely a macOS Full Disk Access restriction on the calling
  process, not an actual account permission gap, but worth testing on one
  run before trusting the full script.
- At ~1.9TB total, `rsync -avP` per run is probably safer than `cp -R` —
  resumable if interrupted mid-transfer; `cp` is not. Not yet switched to
  this, still using `cp -R` as written above.
- After copying, still need: (1) verify copy integrity (checksums, not
  just "looks done"), (2) regenerate/rewrite each run's `metadata.tsv`
  fastq paths to point at the new `RNA_seq_raw_data/<cell_line>/...`
  location, (3) reload affected runs so `samples.fastq_r1`/`fastq_r2` in
  the warehouse reflect the new paths. Not yet built.
- Separately noted: `samples.fastq_r1`/`fastq_r2` in the warehouse
  currently store **container paths** (e.g. `/data/<run>/<condition>/
  <file>.fq.gz`, a Docker bind-mount path), not real filesystem paths on
  any mounted drive — already disconnected from reality before this
  migration. The `fastq_r1_md5`/`fastq_r2_md5` columns are the reliable
  way to confirm file identity regardless of what path string sits next to
  them.
