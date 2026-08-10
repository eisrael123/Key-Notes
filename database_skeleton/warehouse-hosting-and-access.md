# warehouse-hosting-and-access

Where `warehouse.duckdb` and the raw pipeline artifacts should live long
term, so every lab member can query the same data from their own Claude
account without mounting anything.

## The measurement that reframes the problem

Measured against the real `rnaseq_runs/` tree, not estimated:

| | Size |
|---|---|
| Everything in `rnaseq_runs/` | **2.8 TB** |
| All `tables/*.tsv` across 10 real runs | **12.5 GB** |
| Estimated `warehouse.duckdb` after load + compression | **~2–4 GB** |

Per-table TSV totals across all runs: `splicing_event_replicate` 3.72 GB,
`expression_transcript` 2.01 GB, `signal_over_gene` 1.75 GB, `junction`
1.71 GB, `splicing_event` 1.07 GB, `transcript_de` 0.85 GB,
`expression_gene` 0.70 GB (dropped — becomes a view), `de_gene` 0.19 GB,
everything else negligible.

**The warehouse is ~0.1% of the data.** The database and the raw data are
three orders of magnitude apart and are two separate hosting problems.
Solving them with one mechanism is what makes this feel hard.

## The three mount objections, graded

1. **Windows breaks Finder volume mounts** — true, but stops mattering
   once no user machine mounts anything.
2. **Mounts drop and need reconnecting** — true, and the most real of the
   three. SMB/AFP mounts die on sleep, Wi-Fi switch, and VPN
   renegotiation. No config fixes this. Any design requiring a
   laptop-side persistent mount is wrong.
3. **Network share is slow at TB scale** — true for the TBs, false for
   the warehouse. Wrong unit of analysis: answering "which EBV genes went
   up in SNU719" touches a few MB inside a 3 GB file, not 2.8 TB.

**The assumption underneath all three** was that the MCP tool needs
filesystem access on the user's machine. It doesn't. An MCP tool is a
remote procedure call — the query travels out, executes where the data
already lives, and a few KB of answer comes back. Server-side `grep`/
`samtools` are exposed the same way: as tools that run remotely and
return text. The 2.8 TB never moves and no laptop mounts anything, on any
OS.

## Two things that force a real endpoint

### Artifacts need a resolvable location, not just a path

The warehouse stores *paths* to artifacts in `artifacts_manifest`. A path
like `/Volumes/TUNGSACore3/rnaseq_runs/.../foo.bw` is meaningless to a
lab member on a Windows laptop. Something has to turn a stored path into
something a person or IGV can actually open.

The good news: the two formats that matter are **built for exactly this**.
The tree contains 288 `.bw` bigWig files and 76 BAMs with 75 `.bai`
indexes alongside them. bigWig and indexed BAM are random-access binary
formats designed to be read over HTTP range requests — IGV can load
either straight from a URL and fetch only the bytes for the region on
screen. Viewing BZLF1 coverage pulls kilobytes, not the whole multi-GB
file. So a plain HTTPS file server over the `artifacts/` tree, with range
requests enabled, fully serves the manual-inspection funnel described in
`signal-over-gene-coverage-vs-expression.md` — no download step, no
mount.

One gap worth checking: 76 BAMs but only 75 indexes. An unindexed BAM
can't be streamed this way and would have to be indexed or downloaded
whole.

**Schema implication:** `artifacts_manifest` should treat `rel_path` as
canonical and portable, and resolve the actual location at query time by
prefixing a configured base (a local mount path, or an HTTPS base URL)
depending on where the query runs. Storing a machine-specific `abs_path`
in the database bakes in one deployment's layout and breaks the moment
the data is served from anywhere else.

### Update cadence — measured, not assumed

The concern was that re-distributing the database on every new experiment
would be unworkable. Actual run dates in the tree: 2019-12, 2020-04,
2020-09, 2020-11, 2020-11, 2022-05, 2022-12, 2022-12, 2025-04, 2025-08 —
**10 runs in ~5.7 years, roughly one every 7 months.**

At that cadence, syncing a 3 GB file is a non-event, and it's an
automated background sync (rsync/Syncthing/S3), never a manual "send
everyone a file." The distribution objection is real in principle but
doesn't match the lab's actual history. It becomes real if throughput
increases substantially — worth revisiting then, not now.

Note one wrinkle if you do go the sync route: a rebuilt DuckDB file
delta-syncs poorly (internal pages get reshuffled), so each sync is
effectively a full ~3 GB transfer rather than an incremental one. Fine
twice a year; annoying weekly.

## Wayne State HPC findings

The [WSU Grid](https://tech.wayne.edu/hpc) is **free** — 427 nodes, 8,212
cores, 56 TB RAM, 1.2 PB disk, open to any WSU student/faculty/staff with
an AccessID. Accounts are typically created within two business days.

[Storage tiers](https://services.wayne.edu/TDClient/277/Portal/KB/Article/20247/HPC-storage-solutions):

- **Tier 1** — Panasas ActiveStor Prime, ~1.6 PB raw. Users up to 4 TB,
  **groups up to 10 TB**, more on request. Two weeks of backups plus
  snapshots on a separate system.
- **Tier 2** — Panasas ActiveStor 14/18, ~1.5 PB raw. Users up to 10 TB,
  **groups up to 50 TB**, more on request. This tier comfortably fits
  2.8 TB with room to grow.
- **OSiRIS** — ~8 PB distributed, available if collaborating with U-M or
  MSU.

Group directories are a documented, requestable feature — the right unit
for a lab. [Globus](https://services.wayne.edu/TDClient/277/Portal/KB/Article/20224/How-to-Setup-Globus)
(`wsugrid#globus`) handles bulk TB transfers in without babysitting.

**On the VPN concern:** [Grid OnDemand requires the WSU VPN](https://tech.wayne.edu/kb/security/wsu-virtual-private-network),
and interactive VPN sessions do time out — but that only constrains
*humans* SSHing in. A server permanently inside the campus network isn't
"connecting through" a VPN, it's already there. WSU also documents a
**static-IP exception** requestable from hpc@wayne.edu, which is the
specific lever for a lab machine.

**The blocking caveat:** WSU's FAQ states the login node "is for the sole
purpose of Slurm job submissions, job status, and file transfers." Most
HPC centers prohibit long-running daemons on login nodes, so the MCP
server probably **cannot run on the Grid**, even though the Grid is the
ideal home for the 2.8 TB. This is the single question that determines
the architecture — ask before designing around it.

## Claude-side constraints

Anthropic's cloud connects to the MCP server, **not from the user's
device**. A server behind the WSU firewall won't connect; it needs to be
publicly reachable over HTTPS.

[MCP tunnels](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview)
solve this with outbound-only connections (no inbound ports, no IP
allowlisting) — **but tunnels are not available as connectors in
claude.ai**, only via Managed Agents and the Messages API. So the
"everyone has the rnaseq connector in their own account" model requires a
genuinely public HTTPS endpoint with OAuth.

The account model itself works as intended: on Team/Enterprise, an owner
adds the [custom connector](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)
org-wide once, and each member authenticates individually — shared
tooling, separate chat histories, per-user access control.

## Recommendation — phased

**Phase 1 (now).** Warehouse only, no server. Rebuild `warehouse.duckdb`
on the pipeline machine after each run; sync the file to each member's
laptop. Local, instant, offline, zero infrastructure. Covers ~95% of
questions.

**Phase 2 (when IGV access is needed by anyone but you).** Stand up an
HTTPS file server over `artifacts/` with range requests enabled. This is
the smaller and more urgent half of the server problem — and it's what
Phase 1 genuinely can't do.

**Phase 3 (when the lab outgrows sync).** Move the warehouse next to that
same endpoint and expose it as a proper MCP connector. At that point the
distribution problem dissolves entirely — nobody syncs anything, because
the data only exists in one place.

Note that Phase 2 mostly builds the host Phase 3 needs, so the sequence
compounds rather than throwing work away.

**If building the host:** a lab-owned Linux box hosted at the WSU
Computing Center (they [advertise hosting purchased equipment](https://tech.wayne.edu/kb/high-performance-computing/pi-resources/500159))
is the sweet spot — institutional network, power, and physical security,
but full control over the software and permission to run a persistent
daemon. Cloud VMs work but mean paying to store TBs that WSU stores free,
and genomic data may carry IRB or data-use restrictions that make a
personal AWS account a compliance conversation first.

## Questions for hpc@wayne.edu

1. Can we run a persistent (non-Slurm) network service anywhere on your
   infrastructure — login node, a dedicated VM, or hosted lab hardware?
2. Can a static-IP lab machine get a firewall exception for **inbound**
   HTTPS from outside campus?
3. What's the realistic ceiling on a Tier 2 group allocation beyond the
   documented 50 TB?
4. Any institutional restriction on serving this data over public HTTPS
   (IRB, data-use agreement, human-subjects derivation)?
