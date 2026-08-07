# SQL crash course (DuckDB + this warehouse)

Basic SQL syntax, with notes on where DuckDB differs from other engines, and
examples using this project's actual tables (`runs`, `samples`,
`expression_gene`, `de_gene`, `junction`, `splicing_event`, `qc_metric`, etc.
— see `SCHEMA.md` for the full list).

## Connecting

```bash
python3 -c "import duckdb; con = duckdb.connect('warehouse.duckdb')"
```
or the CLI: `duckdb warehouse.duckdb`

DuckDB is embedded (like SQLite) — no server, no login. The `.duckdb` file
*is* the database.

## The core SELECT anatomy

```sql
SELECT column1, column2          -- what to return
FROM table_name                  -- where to get it
WHERE condition                  -- row filter (before grouping)
GROUP BY column1                 -- collapse rows into groups
HAVING condition                 -- group filter (after grouping)
ORDER BY column1 DESC            -- sort
LIMIT 10;                        -- cap row count
```
This is also the *logical* order of evaluation — `WHERE` can't see aliases
defined in `SELECT`, but `GROUP BY`/`ORDER BY` can (DuckDB is lenient about
this, unlike stricter engines).

Example against this warehouse — top DE genes for one run:
```sql
SELECT gene_id, log2_fold_change, padj
FROM de_gene
WHERE run_id = 'R68fecadda929' AND padj < 0.05
ORDER BY padj ASC
LIMIT 20;
```

## Filtering (WHERE)

```sql
=  !=  <  >  <=  >=
AND  OR  NOT
IN ('a', 'b', 'c')
BETWEEN 1 AND 10
LIKE 'ENSG%'          -- % = any chars, _ = one char
IS NULL / IS NOT NULL -- never use = NULL, it silently matches nothing
```
Note from `SCHEMA.md`: 174 transcripts have a null `gene_id` — use
`IS NOT NULL` (or `!= 'NA'`, since some are the string `'NA'` not true NULL)
when you need gene-mapped rows only.

## Joins

```sql
SELECT s.sample_id, s.condition, q.metric_name, q.metric_value
FROM samples s
JOIN qc_metric q ON s.sample_id = q.sample_id AND s.run_id = q.run_id;
```
- `JOIN` / `INNER JOIN` — only matching rows in both tables.
- `LEFT JOIN` — all rows from the left table, NULLs where no match.
- Always join on **both** `sample_id`/`comparison_id` AND `run_id` in this
  warehouse — every table is keyed by `run_id` so multiple runs coexist;
  forgetting it silently cross-joins runs together.

## Aggregation

```sql
COUNT(*), COUNT(col), SUM(col), AVG(col), MIN(col), MAX(col)
```
```sql
SELECT run_id, comparison_id, COUNT(*) AS n_sig_genes
FROM de_gene
WHERE padj < 0.05
GROUP BY run_id, comparison_id;
```
`FILTER (WHERE ...)` — DuckDB/Postgres feature, conditional aggregate
without a subquery (used in this project's `splicing_summary` view in
`Key_Notes.md`):
```sql
SELECT event_type,
       COUNT(*) AS total_events,
       COUNT(*) FILTER (WHERE fdr < 0.05) AS significant_events
FROM splicing_event
GROUP BY event_type;
```

## CTEs (WITH) — name a subquery, use it like a table

```sql
WITH sig_genes AS (
  SELECT gene_id FROM de_gene WHERE run_id = 'R68fecadda929' AND padj < 0.05
)
SELECT e.*
FROM expression_gene e
JOIN sig_genes g ON e.gene_id = g.gene_id;
```
Prefer this over nesting subqueries three deep — reads top to bottom, each
block can be tested independently by running it alone as a SELECT.

## Window functions — per-row calc that sees other rows without collapsing them

```sql
SELECT gene_id, sample_id, tpm,
       RANK() OVER (PARTITION BY sample_id ORDER BY tpm DESC) AS rank_in_sample
FROM expression_gene
WHERE run_id = 'R68fecadda929';
```
`PARTITION BY` = restart the calc per group (like GROUP BY but keeps every
row). Useful for "top N per sample" type questions — the same shape as
`report_top_gene_tpms` in this schema, but done live instead of relying on
the pre-baked report table.

## Creating views vs. tables

```sql
CREATE VIEW my_view AS SELECT ...;   -- saved query, recomputed each time, no storage
CREATE TABLE my_table AS SELECT ...; -- materialized, stored on disk
```
This project deliberately uses views for `splicing_summary` and
`expression_gene` (see `Key_Notes.md`) because they're cheap-to-recompute
rollups of other loaded tables — avoids storing derivable data twice and
risking drift if the underlying rollup logic changes.

## DuckDB-specific things worth knowing

- **`read_csv_auto('file.tsv')`** — query a TSV/CSV directly without loading
  it first, with automatic type inference (this is what makes `padj` a REAL
  instead of TEXT here, unlike the sqlite3 preview file which has no type
  inference — see `SCHEMA.md`'s note on `preview_R68fecadda929.sqlite`).
  ```sql
  SELECT * FROM read_csv_auto('tables/de_gene.tsv', delim='\t') LIMIT 5;
  ```
- **`EXCLUDE`/`REPLACE`** in SELECT — grab all columns except one, without
  spelling out the rest:
  ```sql
  SELECT * EXCLUDE (parameters, tool_versions) FROM runs;
  ```
- **`DESCRIBE table_name;`** or `PRAGMA table_info('table_name');` — see a
  table's columns/types without a separate schema browser.
- **`SUMMARIZE table_name;`** — instant min/max/avg/nulls/distinct-count
  per column. Fast way to sanity-check a table you just loaded.
- Query JSON columns directly — `runs.stage_status`/`tool_versions`/
  `parameters` are JSON blobs; DuckDB can reach into them with `->>`:
  ```sql
  SELECT run_id, stage_status->>'gsea' AS gsea_status FROM runs;
  ```
- Can query Parquet/CSV/JSON files directly in `FROM` without an import
  step — DuckDB treats files and tables interchangeably in most contexts.

## Quick reference table

| Want to... | Clause |
|---|---|
| filter rows | `WHERE` |
| filter after grouping | `HAVING` |
| combine tables | `JOIN ... ON` |
| collapse into summary rows | `GROUP BY` |
| sort | `ORDER BY` |
| cap results | `LIMIT` |
| name a reusable subquery | `WITH x AS (...)` |
| per-row rank/running total without collapsing | window function `OVER (...)` |
| saved query vs. stored copy | `CREATE VIEW` vs. `CREATE TABLE AS` |
