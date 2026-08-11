# Experiment links table design

Design for how to record that two (or more) experiments are biologically linked — first
concrete case: `Mutu_Zta_EXOSC3-KO_2025-08-31` and `Mutu_Zta_EXOSC3-control_2025-08-31` (an
EXOSC3 knockout vs. its matched Cas9-no-guide control, same Zta-reactivation design run under
two co-treatment arms). They're still analyzed separately, but a user querying one should be
told the other exists.

This is the first of several metadata/schema revisions Ethan and his PI are planning; the rest
haven't been discussed yet. See [revised-experiment-parameter-schema.md](revised-experiment-parameter-schema.md)
and [experiment-metadata-schema-review.md](experiment-metadata-schema-review.md) for the
adjacent per-experiment field work — this note is scoped to cross-experiment links only.

## Options considered

**A. A dedicated `co_treatment`-style param** (e.g. a `linked_experiment_id` field) — rejected.
Too rigid: forces a decision today about exactly what kinds of relationships exist (pairs only?
what about a dose-response series, or the same biological material also run through CAGE-seq?),
and a single field can't hold more than one link.

**B. Free text in `Notes`, with an AI/MCP tool parsing it at query time to detect linkage** —
Ethan's initial idea, rejected as the *mechanism*, kept as a good instinct about not
over-constraining relationship types. The problem: it's not queryable by anything that isn't an
LLM (a plain SQL client, a dashboard, a report script gets nothing), phrasing drifts ("linked
to" vs "paired with" vs "see also") so detection is probabilistic, and a renamed `experiment_id`
silently breaks the reference with no error.

**C. A separate `experiment_links` table, keyed on `experiment_id`** — chosen. Structured enough
to query deterministically, loose enough (`relationship` is free text, not an enum) to not force
a taxonomy of link types up front.

## Why it's a new table, not a column on an existing one

None of the four existing warehouse tables (`runs`, `samples`, `comparisons`,
`artifacts_manifest`) has the right grain — a link is a fact about a *pair of experiments*,
which none of them represent:

- `runs` — one row per pipeline execution (`run_id`); `experiment_id` is just a column on it. A
  link is about the experiment, not a specific run — if `EXOSC3-KO` gets reprocessed on a future
  pipeline version (new `run_id`), the biological link to `EXOSC3-control` still holds. Tying the
  link to `run_id` would mean re-declaring it on every reprocess.
- `samples` — per-sample within a run.
- `comparisons` — per `(run_id, comparison_id)` test/cntl labeling within a single run.
- `artifacts_manifest` — per-file.

## Schema sketch

```sql
CREATE TABLE experiment_links (
    experiment_id_a  VARCHAR NOT NULL,
    experiment_id_b  VARCHAR NOT NULL,
    relationship      VARCHAR,              -- free text, not an enum, e.g.
                                             -- "co-treatment pair (EXOSC3-KO vs its
                                             --  Cas9-no-guide control)"
    note              VARCHAR,              -- optional extra context
    added_by          VARCHAR,              -- investigator who declared the link
    created_at        TIMESTAMP DEFAULT current_timestamp
);
```

One row per pair, `a`/`b` unordered — doesn't matter which experiment goes in which column, a
given pair is only ever stored once. No hard foreign key against `runs.experiment_id`, since
`experiment_id` isn't declared unique/primary anywhere else (no table with one row per experiment
exists yet) — it's a loose reference like other cross-links already in this schema, not
enforced by the database.

## Query-time behavior

1. The MCP tool runs the user's actual query as normal (e.g. `SELECT * FROM de_gene WHERE
   experiment_id = 'Mutu_Zta_EXOSC3-KO_2025-08-31' ...`).
2. Separately, for whichever `experiment_id`(s) surfaced in that result, it does one cheap
   lookup:
   ```sql
   SELECT experiment_id_b AS linked, relationship, note FROM experiment_links
     WHERE experiment_id_a = 'Mutu_Zta_EXOSC3-KO_2025-08-31'
   UNION
   SELECT experiment_id_a AS linked, relationship, note FROM experiment_links
     WHERE experiment_id_b = 'Mutu_Zta_EXOSC3-KO_2025-08-31'
   ```
   The `UNION` exists only because a pair is stored unordered — the queried experiment_id could
   be sitting in either column of the matching row, so both have to be checked to not miss it.
   For a result set with several experiment_ids, this batches to a single `IN (...)` lookup
   rather than one query per id.
3. **This is a set-difference operation**: of the experiment_ids that already surfaced in the
   query's own results, which ones have a linked partner that did *not* surface? That's the only
   case worth interrupting the user about — if both linked experiments already showed up (e.g. a
   broad query that happened to match both), there's nothing to offer to add.
4. What's left after that subtraction gets attached to the tool's response as a small structured
   field, separate from the data rows — e.g. `linked_experiments: [{experiment: "...control_2025-08-31",
   relationship: "co-treatment pair..."}]`.
5. The AI reads that structured field directly and surfaces it in plain language ("this has a
   linked control run, want it included?") — it's relaying a fact the tool handed it, not
   inferring one from prose.

## Status

Design only — not built. `Notes` stays a free-text field for human context as it already is;
it just isn't the thing a query mechanism depends on for correctness. No DuckDB warehouse exists
yet to migrate (per
[experiment-metadata-schema-review.md](experiment-metadata-schema-review.md)), so this table
gets created alongside `runs`/`samples`/`comparisons`/`artifacts_manifest` whenever
`build_warehouse.py`/`loader.py` actually get built out.
