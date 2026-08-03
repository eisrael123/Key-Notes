# Metadata form and --quick-input

Implemented 2026-08-03. Lets a non-CLI collaborator (Erik) supply experiment parameters through a
browser form instead of a dozen command-line flags or interactive prompts.

## The flow

1. Open `metadata_form.html` (repo root) — a single self-contained page. No server, no install,
   double-click to open.
2. Fill in every field. The download button stays disabled until nothing is missing, and it names
   what is still outstanding.
3. Download produces `input_args.json`. Drag it into the experiment's FASTQ folder, alongside the
   `cntl` and `test` subdirectories. **The filename is the whole interface** — it must stay
   exactly as downloaded.
4. Whoever runs the pipeline passes only the two directory arguments plus the flag:

```bash
python metadata.py \
  --root-fastq-dir /data/SNU719_Zta-plus-Rta \
  --output-dir /data/output \
  --quick-input
```

Erik never types or sees a filesystem path. The file rides along when the FASTQ tree is copied
locally for Docker.

## What the form covers

Dropdowns for every controlled-vocabulary field (cell line, perturbation type, induced program,
library selection, genome build, PE/SE). Number input + "not applicable" checkbox for timepoint;
date picker + checkbox for sequencing run date. Target and dose are hidden when perturbation type
is `none`. `referenceFiles` is prefilled read-only as `/data/referenceFiles`. Investigator name
has whitespace stripped as you type, because it becomes part of the output filename that the
pipeline later globs for.

`organism` is deliberately absent — it is derived from the genome build via
`vocab.organism_for_build()`, so there is nothing to ask.

`--root-fastq-dir` and `--output-dir` are deliberately absent too: they name the local
Docker-speed copy of the data and where its output goes, which only the person running the
pipeline knows.

## metadata.py argument changes

All six positional arguments became named flags. There are no positional arguments left.

| Old positional | New flag | Required? |
|---|---|---|
| `fastq_root_dir` | `--root-fastq-dir` | always |
| `results_dir` | `--output-dir` | always |
| `reference_dir` | `--reference-dir` | flag, file, or prompt |
| `species_name` | `--species-name` | flag, file, or prompt |
| `investigator_name` | `--investigator-name` | flag, file, or prompt |
| `experiment_type` | `--experiment-type` | flag, file, or prompt |

The experiment-structure flags (`--cell-line`, `--perturbation-type`, etc.) are unchanged.

## Behaviour worth remembering

- **`--quick-input` is opt-in.** Without the flag the file is never read, even if it is sitting
  right there. Existing flag-driven and interactive workflows are untouched.
- **An explicit flag beats the file.** `--quick-input --library-selection ribodepleted` overrides
  what the file says, so a one-off rerun does not need the file edited and re-downloaded.
- The file sits *next to* the condition folders, not inside them, so it does not disturb sample
  discovery — `metadata.py` walks directories and ignores loose files.

## Two deliberate tradeoffs

**1. No double validation of vocabulary values.** Values arriving from `input_args.json` skip
`vocab.validate()` entirely; the form's `<select>` is the only verifier. Consequence: a
**hand-edited** `input_args.json` can put a value into the archive that the vocabulary would have
rejected. Re-download from the form rather than editing the JSON. Re-adding the check is a
two-line change in `resolve()` if it ever bites.

**2. Two checks were kept anyway**, on the grounds that they are not vocabulary duplication:

- `timepoint_hours` → `str(float(...))` and `sequencing_run_date` → `date.fromisoformat(...)` are
  *canonicalisation*. Without them, `24` from the form and `24.0` from a flag land differently in
  the column and a query grouping on it splits one timepoint into two.
- `experiment_type` membership in `LIBRARY_LAYOUTS` picks which FASTQ pairing branch runs, so an
  unrecognised value would silently parse a paired-end run as single-end.

## Live maintenance risk

`metadata_form.html`'s `<option>` lists are a **hand-maintained copy** of
`rnaseq_helper_scripts/vocab.py` (`CELL_LINES`, `PERTURBATION_TYPES`, `INDUCED_PROGRAMS`,
`LIBRARY_SELECTIONS`, and the `GENOME_BUILD_ORGANISM` keys). They agreed exactly at
implementation time, verified programmatically. **No sync script exists, by design** — a
generator would have added a build step to something deliberately kept static.

So: adding a vocabulary value to `vocab.py` requires adding the matching `<option>` to
`metadata_form.html` **in the same commit**, or the form silently cannot offer a value the
pipeline supports. This is also stated in the README.

## Still positional

`rnaseq.py` keeps its four positional arguments (`metadata_file`, `reference_dir`, `scripts_dir`,
`results_dir`). Out of scope — the form only feeds `metadata.py`. Converting it the same way is a
small follow-up if the inconsistency becomes annoying.

## Tests

`tests/test_metadata_quick_input.py` covers the happy path, opt-in-ness, missing file, misspelled
key, malformed JSON, flag-overrides-file, sample discovery being undisturbed, and investigator
whitespace stripping. Full suite: 78 passing.
