# warehouse-hosting-and-access

Where `warehouse.duckdb` and the raw artifacts live, and how each lab
member's Claude reaches them.

## Sizes (measured, not estimated)

| | Size |
|---|---|
| Everything in `rnaseq_runs/` | **2.8 TB** |
| All `tables/*.tsv` across 10 real runs | **12.5 GB** |
| Estimated `warehouse.duckdb` after load + compression | **~2–4 GB** |

The warehouse is ~0.1% of the data. The database and the raw data are
three orders of magnitude apart and are two separate problems. Trying to
solve both with one mechanism is what made this feel hard.

## The architecture

| | Where it lives | How it's reached |
|---|---|---|
| `warehouse.duckdb` (~3 GB) | Local disk, each member's machine | Local MCP server, opened `READ_ONLY` |
| `artifacts/` (2.8 TB) | Lab NAS (`FlemingtonLabMain1`) | SMB mount — human opens in IGV, Claude reads via tools |

**No HTTPS server.** Nothing to build, secure, certificate, or maintain.

Each member runs Claude Desktop with a local MCP server. "Local" doesn't
mean a daemon that never turns off — it's a subprocess the Claude client
launches on start and kills on quit. What makes it reliable is that there
is no network in the path for warehouse queries, so there's nothing that
can disconnect. (Local MCP servers only work in Claude Desktop/Cowork,
not claude.ai in a browser — everyone needs the desktop app.)

Because the MCP server runs on the same machine that has the NAS mounted,
Claude can **read** artifact files directly — grep chimeric junctions,
parse a QC report, pull a column from a count matrix — not just report
where they are.

## Why artifacts don't need their own server

Checked what's actually in `artifacts/` rather than assuming. It's mostly
IGV tracks and plots, plus a small tail of machine-readable data.

The expected counter-example failed: GSEA looked like artifacts-only
territory (4,067 files vs. a tiny `gene_set_enrichment` table), but the
warehouse's `leading_edge_genes` column already holds real symbols
(`NFKB1,TRAF2,IKBKB,...`), not just GSEA's `tags=65%, list=7%` summary.
The pipeline already pulls the useful part forward.

Genuinely artifacts-only:

- **`Chimeric.out.junction`** — ~163 MB/sample, confirmed absent from
  every `tables/` file. Matters here: viral-host chimeric transcripts and
  EBV integration sites are only answerable from this.
- DESeq2 normalized count matrices, kallisto `.h5` bootstraps, rMATS
  `JC.raw.input.*`, raw fastp/fastqc/RSeQC reports, ~2,400 PNGs.

Also worth knowing: bigWig and indexed BAM are random-access formats, so
IGV over a mount reads only the bytes for the region on screen. Viewing
BZLF1 coverage pulls kilobytes, not the multi-GB file. (Census: 288
bigWigs, 76 BAMs, 75 `.bai` — one BAM is unindexed and can't be streamed
this way.)

## Why "everyone works on campus" is the load-bearing assumption

It's the deciding factor, not data size:

- **Latency, not bandwidth.** SMB is chatty — opening a file is dozens of
  small round trips. ~1 ms each on LAN, ~30 ms over VPN. Same file, ~30x
  slower.
- **VPN sessions time out** on idle and re-auth, and the mount dies with
  them.
- **VPN routes through one shared campus gateway** — a bottleneck nobody
  in the lab controls.

On campus, none of these apply. Off campus, all three do at once. That's
the trigger for revisiting this design.

## Two checks that belong in code, not CLAUDE.md

Anything that must happen 100% of the time goes in the MCP server.
CLAUDE.md is advisory — Claude may skip it, a member may not have it
loaded, a long conversation may bury it. A mount check that runs 95% of
the time is worse than useless, because you'll trust it.

1. **On startup — warehouse freshness.** Read `warehouse.meta.json` off
   the mount (`{"version", "sha256", "bytes", "built_at"}`). If the
   version differs from the local copy, copy the new `.duckdb` to a
   `.tmp`, verify the sha256, then atomically rename over the old one — a
   rename can't half-succeed, so nobody is ever left with a partial
   database. Mount unreachable? Use the existing local copy. **Don't ask
   the user to redownload — just do it.**
2. **Before each artifact call — mount presence.** Confirm the path
   exists; if not, return a clean "the drive isn't connected" rather than
   a confusing filesystem error.

Query the warehouse from the **local copy**, not across the mount — a
3 GB DuckDB file read over SMB is slow.

**CLAUDE.md is for what Claude needs to know, not do:** positive
`log2_fold_change` means higher in test, how `comparison_id` is built,
which tables are views, that `condition` is `cntl`/`test`.

## Failure modes

On-campus mounts still drop on sleep/wake, Wi-Fi roaming, NAS reboots,
and network blips — less often than over VPN, but not never. Mitigate
with auto-remount on login/wake, plus check #2 above.

The failure is graceful either way: the warehouse is a local file, so
**all normal queries keep working**; only artifact access breaks, and the
fix is remounting.

**Request read-only mounts for everyone but the pipeline operator.**
`FlemingtonLabMain1` is shared infrastructure co-owned with a
collaborator and holds the lab's canonical raw data — see
`raw-fastq-storage-migration.md`, which already documents
`Operation not permitted` oddities there. No reason to hand every member
write access to it.

## Path portability

`artifacts_manifest` stores `rel_path` only — never an absolute path.
Mount roots differ per machine (`/Volumes/FlemingtonLabMain1` on Mac,
`Z:\` on Windows), so location is resolved at query time by prefixing a
per-machine configured base. Same disease already noted in
`raw-fastq-storage-migration.md`, where `samples.fastq_r1/r2` store
Docker *container* paths that match no real filesystem.

## Fallback: if remote access becomes a real need

Then, and only then, put an HTTPS server in front of `artifacts/`. bigWig
and BAM stream over HTTP range requests, so IGV works from a URL with no
mount and no download.

Cost: **$0 marginal** self-hosted on campus hardware — you own the disks,
WSU provides network and power, nginx/Caddy are free, and Let's Encrypt
certificates are free. Size is irrelevant to a self-hosted server.

Cloud is the expensive path: [S3](https://www.cloudzero.com/blog/s3-pricing/)
storage is ~$0.023/GB/month, so 2.8 TB is ~$65/month indefinitely just to
sit there. Egress (~$0.09/GB) would be near-trivial thanks to range
requests — **the terabytes drive storage cost, not transfer cost**, which
is exactly the cost that vanishes on hardware WSU already hosts free.

One Claude-side constraint if this ever happens: Anthropic's cloud
connects to the MCP server, not from the user's device, so a custom
connector needs a publicly reachable HTTPS endpoint.
[MCP tunnels](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview)
solve the firewall problem but are **not available as claude.ai
connectors** — only Managed Agents and the Messages API.

## WSU HPC, if storage ever needs to move off the lab NAS

The [Grid](https://tech.wayne.edu/hpc) is free — 427 nodes, 8,212 cores,
1.2 PB disk, open to anyone with an AccessID.
[Storage tiers](https://services.wayne.edu/TDClient/277/Portal/KB/Article/20247/HPC-storage-solutions):
Tier 1 Panasas (groups to 10 TB, backed up, two weeks of snapshots) and
Tier 2 (**groups to 50 TB**, more on request) — 2.8 TB fits easily.
[Globus](https://services.wayne.edu/TDClient/277/Portal/KB/Article/20224/How-to-Setup-Globus)
(`wsugrid#globus`) handles bulk transfer.

Caveat: WSU's FAQ says the login node is "for the sole purpose of Slurm
job submissions, job status, and file transfers," so a persistent service
probably can't run there. Not a problem for the current design, which
needs no server.
