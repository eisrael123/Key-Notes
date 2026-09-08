# DG75 Corrupt FASTQ Inventory

Integrity scan of all 36 DG75 experiments in `1_RNA_seq/DG75/x_next_batch/`, run 2026-09-05
to 2026-09-07 with `gzip -t` (validates CRC32 + length of every compressed block).

**81 of 342 raw fastq files are corrupt (24%, ~1415 GiB of 5.1 TiB.)**

All damage is PRE-EXISTING in the NAS originals — verified by testing the untouched source
at `x_next_batch/` directly. It was not caused by copying. Corrupt files have correct sizes
and correct metadata and fail only on decompression, so size/checksum-of-name verification
cannot detect them.

## Batch definitions
- **Batch A ("this computer")** — the 18 SMALLEST experiments, staged to TUNGSACore3 and
  copied to `VIRA/rnaseq/raw_data/DG75/`. 35 corrupt files.
- **Batch B ("other computer")** — the 18 LARGEST experiments, never staged; still only in
  `x_next_batch/` on the NAS. Destined for the new machine. 46 corrupt files.

## Batch A — staged 18 (this computer)

| Experiment | Corrupt / Total | Status |
|---|---|---|
| `DG75_BORF2_2023-04-26` | 8/8 | **TOTAL LOSS** |
| `DG75_BHLF1_2023-06-08` | 8/8 | **TOTAL LOSS** |
| `DG75_BALF5_2023-06-08` | 7/8 | mostly lost |
| `DG75_BALF1_2023-04-26` | 5/8 | mostly lost |
| `DG75_BTRF1_2023-04-26` | 3/8 | partial |
| `DG75_BNLF2a_2023-04-26` | 3/8 | partial |
| `DG75_BcRF1_2023-08-27` | 1/10 | partial |

Clean in batch A (11): 
`DG75_BXLF1_2023-06-08` `DG75_BGRF1_BDRF1_2023-04-26` `DG75_BBLF4_2023-04-26` 
`DG75_BRLF1-BRRF1_2023-06-08` `DG75_BFRF2_2023-08-27` `DG75_BDLF3_2023-08-27` 
`DG75_BcRF1_2023-11-08` `DG75_ORF49_2023-07-12` `DG75_BRRF1_2023-06-08` `DG75_BRLF1_2023-06-08` 
`DG75_BSLF1_2023-06-08` 

## Batch B — unstaged 18 (other computer)

| Experiment | Corrupt / Total | Status |
|---|---|---|
| `BMLF1` | 13/16 | mostly lost |
| `BNLF2b` | 8/8 | **TOTAL LOSS** |
| `BSRF1` | 7/8 | mostly lost |
| `BMRF2` | 7/8 | mostly lost |
| `BVRF1` | 6/8 | mostly lost |
| `BMRF1` | 3/8 | partial |
| `BGLF4` | 2/8 | partial |

Clean in batch B (11): 
`BMLF1_repeat` `BLLF3` `BHRF1` `BRRF1_repeat` `BGLF4_repeat` `BNLF2b_repeat` `BLLF3_repeat` `BKRF3` 
`BKRF4` `BBLF2_BBLF3` `BGLF3_5` 

## Key findings

**All 2024-08-26 repeat runs are clean.** `BGLF4_repeat`, `BLLF3_repeat`, `BMLF1_repeat`,
`BNLF2b_repeat`, `BRRF1_repeat` — 12 files each, zero corruption — while several of their
2023 originals are damaged. Damage is confined to 2023 data, which points at the older
files' storage history rather than at sequencing.

Directly usable substitutions:
- `BNLF2b` (8/8 corrupt) -> `BNLF2b_repeat` clean
- `BMLF1` (13/16 corrupt) -> `BMLF1_repeat` clean
- `BGLF4` (2/8 corrupt) -> `BGLF4_repeat` clean

**Corruption occurred during copying, not at sequencing.** Three experiments share the same
control files (`C1.2_1.fq.gz`, `C2.2_1.fq.gz` in BTRF1/BORF2/BNLF2a). All three copies are
corrupt but at DIFFERENT offsets with different checksums — independent damage events on
copies of one source file.

**Duplicate masquerading as a replicate.** `BHLF1`'s two control replicates are the same file
(identical full-file SHA-256): `PCDNA3_1_1 == PCDNA3_2_1`, `PCDNA3_1_2 == PCDNA3_2_2`. That
experiment has ONE control, not two.

**`BcRF1_2023-08-27`'s `2b` files are a corrupted copy of sample 2**, not an independent
replicate (same size, identical first 500 MB, differing full-file checksums; `2b` fails
CRC while sample 2 passes). Analyses treating `2` and `2b` as separate replicates have
inflated n.

## Actions taken (2026-09-07)
- Deleted all 7 affected experiment folders from TUNGSACore3 local staging (freed 790 GiB).
  11 verified-clean experiments remain there.
- Renamed the 35 corrupt files in `VIRA/rnaseq/raw_data/DG75/` with a `.CORRUPT` suffix so
  no pipeline picks them up. Good files in those experiments are untouched.
- `x_next_batch/` originals left as-is — they are the record of what was received.

## TO DO on the other computer (batch B)
Do NOT stage these 7 experiments, or delete them if already copied:
`BNLF2b` (8/8), `BMLF1` (13/16), `BSRF1` (7/8), `BMRF2` (7/8), `BVRF1` (6/8),
`BMRF1` (3/8), `BGLF4` (2/8).
The other 11 batch-B experiments are clean and safe to stage.

## Update from the other computer (2026-09-08)

Batch B has been staged and independently `gzip -t` verified here. The source-level numbers
in the table above are confirmed correct — but the destination copy briefly showed 3
additional false positives from copy-side truncation, not source damage:

- `BVRF1/test/BVRF1.2_2.fq.gz` — copy was truncated (`unexpected end of file`); the source
  passes `gzip -t` clean. This inflated the local BVRF1 count to 7/8 before re-copy.
- `BMLF1_repeat/cntl/pLVX1_1.fq.gz` and `BNLF2b_repeat/cntl/pLVX1_1.fq.gz` — same failure
  mode, each inflating an otherwise-clean repeat folder to 1/12 locally.

All three were confirmed intact at `x_next_batch/` directly (matching sizes, `gzip -t` exit
0), then re-copied and re-verified clean. Batch B on this machine now matches the table above
exactly: 46 genuinely corrupt files, the 11 clean experiments untouched, both repeat folders
(`BMLF1_repeat`, `BNLF2b_repeat`) reconfirmed fully clean.

Root cause: the NAS connection dropped repeatedly during the original copy of batch B (SMB
timeouts, one outage lasting several hours), leaving a small number of in-flight files short.
Worth adding to the Method note below: destination-side truncation from a flaky SMB path is a
second failure mode distinct from pre-existing source corruption, and both produce identical
`gzip -t` failures unless the source is checked directly.

**TO DO status:** batch B fully staged. The 7 experiments listed above still should not be
used without regenerating good replicates from `x_next_batch/`; all other 11 — now including
both repeat folders, reconfirmed clean — are safe to use.

## Not yet checked
The 18 `pipeline_output` folders (these DO have `checksums.sha256` manifests) and the other
cell lines (Akata, BCBL1, Mutu, SNU719) under `VIRA/rnaseq/raw_data/`.

## Method note
`gzip -t` is the only content-level check available — raw fastq folders have no
`checksums.sha256`. When running it: exit 1 = real CRC failure, exit >128 = killed by signal
(inconclusive, NOT corruption). Run serially against SMB; concurrent reads trigger fd
invalidation and produce false failures. Full per-file results are in
`/Volumes/TUNGSACore3/rnaseq_runs/.handoff/crc_results_full.tsv` and `CORRUPT_FILES_ALL36.txt`.
