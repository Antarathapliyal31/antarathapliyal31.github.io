+++
title = "What even is a Parquet table? Notes on data lakes and Delta tables"
date = 2026-09-29
description = "My learning notes on row vs column storage, data lakes, Delta tables, and when data lives in RDS vs a lakehouse."
[taxonomies]
tags = ["databricks", "delta-lake", "parquet", "data-engineering"]
+++

I kept seeing "Parquet table" come up in database discussions, and it bugged me. A table is a table, right? Rows, columns, and every column has a type. So what's different?

It turned out I'd been using Databricks, Unity Catalog and Delta tables without really knowing what sits underneath them. These are my notes from untangling it.

## 1. Same table, different layout on disk

The answer to my first question: logically, nothing is different. A Parquet table has the same rows, columns and types as any other table. The difference is how the bytes are physically laid out on disk.

Take this small table:

| id | name  | city  | salary |
|----|-------|-------|--------|
| 1  | Asha  | Pune  | 50000  |
| 2  | Ravi  | Delhi | 60000  |
| 3  | Meena | Pune  | 55000  |

A normal database like Postgres or MySQL stores it **row by row**:

```
1, Asha, Pune, 50000 | 2, Ravi, Delhi, 60000 | 3, Meena, Pune, 55000
```

Parquet stores it **column by column**:

```
1, 2, 3 | Asha, Ravi, Meena | Pune, Delhi, Pune | 50000, 60000, 55000
```

Also, Parquet is a file format, not a database. A "Parquet table" just means a table whose data lives in `.parquet` files, which an engine like Spark, Databricks, Athena or DuckDB reads as a table.

## 2. The data lake is the ground floor

A data lake is big, cheap cloud storage (S3, ADLS, GCS) where you can keep files of any kind: CSV, JSON, logs, PDFs, Parquet. You don't define a schema before writing. You decide how to interpret the data when you read it, which is called schema-on-read.

On its own, though, it's dumb storage. There's no ACID, so a job that crashes halfway can leave half-written data visible. Permissions only work at the file and folder level. And there's no catalog telling you what lives where. Teams that just dumped data in ended up with what people call a "data swamp."

The lakehouse is the fix. In Databricks, the data still lives in the lake, but Delta Lake adds ACID and table behavior on top, and Unity Catalog adds the `catalog.schema.table` namespaces, permissions and governance. Everything I knew from Databricks turned out to be the upper floors of a building whose ground floor is the data lake.

## 3. A Delta table = Parquet files + a transaction log

This connected back to my first question. The data in a Delta table is stored as plain Parquet files. Next to them sits a `_delta_log` folder with one numbered JSON file per commit, recording which Parquet files were added or removed.

```
sales_table/
   _delta_log/
      00000000000000000000.json   ← commit 0 (table created)
      00000000000000000001.json   ← commit 1 (rows inserted)
   part-0000-a1b2.parquet
   part-0001-c3d4.parquet
```

The rule that makes it work: **if a change isn't in the log, it didn't happen.** Say a job writes 3 of 5 files and then crashes. Those 3 files really are sitting in the folder, but no commit was written, so readers never see them. The table stays clean, as if the job never ran.

So a Parquet table is just Parquet files in a folder. A Delta table is Parquet files plus the log, and the log is what turns plain files into a proper table with ACID.

## 4. RDS vs Delta: both structured, different jobs

My next confusion: RDS tables are structured and live in the cloud too. So why bother with Delta?

|                | RDS table                          | Delta table                          |
|----------------|------------------------------------|--------------------------------------|
| Where it lives | Inside the database server's disks | Parquet files in S3/ADLS             |
| Layout         | Row by row                         | Column by column                     |
| Who can read it | Only that database engine         | Any engine that understands Delta    |
| Good at        | Many small reads/writes (OLTP)     | Big scans and aggregations (OLAP)    |

Both are queried with SQL, so SQL isn't the difference. The workload is.

They also aren't alternatives. The app writes to RDS, and a pipeline **copies** (not moves) that data into the lakehouse. Keeping two copies is worth it because big analytics queries on RDS would slow down the live app, row storage is slow at "scan everything and sum one column," the lakehouse can combine many sources (RDS, CRM, logs, vendor files) in one place, and it can keep years of history cheaply while RDS usually only keeps the current state.

The way I remember it: RDS is where the business *runs*, and the lakehouse is where the business *analyzes*.

## 5. Where the Medallion layers fit

```
App → RDS ──copy──► S3 landing (raw files)
                        │
                  Bronze (Delta, as-is)
                        │  dedup, nulls, constraints
                  Silver (Delta, clean)
                        │  aggregate, join
                  Gold (Delta, business-ready) → dashboards, ML, Genie
```

What I got wrong at first: Delta isn't only the "cleaned" stage. In Databricks, Bronze, Silver and Gold are usually all Delta tables. What changes between layers is how clean the data is, not the format.

## 6. Why column storage makes analytics fast

There are three tricks, and all of them come down to reading less data.

**Read only the columns you need.** Say a `policies` table has 50 columns, 500 million rows, and about 500 GB of data:

```sql
SELECT state, SUM(premium)
FROM policies
GROUP BY state;
```

This query needs 2 of the 50 columns. Row storage has to read every full row, so roughly all 500 GB. Column storage reads just those two column chunks, roughly 20 GB (assuming the columns are similar in size). Reading from disk or S3 is usually the slowest part of a query, so that's a huge win.

The flip side: "give me everything about policy #12345" is better with rows, because all 50 values sit together. With columns, they're scattered across 50 chunks. That's exactly why apps use RDS and analytics uses Delta.

**Compression.** A column holds values of one type, and they often repeat. A `state` column might look like `NJ, NJ, NJ, NY, NJ...`. Parquet uses dictionary encoding (store "NJ" once and replace each occurrence with a small number) and run-length encoding (store "NJ × 1,000" instead of repeating it a thousand times), then applies a compression codec like Snappy or ZSTD on top. Rows that mix names, dates and numbers don't compress nearly as well. Smaller files mean less to read.

**Skipping with min/max stats.** Parquet splits each file into row groups and stores the min and max of every column for each group. For a query with `WHERE premium > 10000`, if a row group's max premium is 8,000, the engine skips that whole group without reading it. Delta goes one step further and keeps min/max stats for each file in its transaction log, so Databricks can skip entire files without opening them. This works best when similar values are stored close together, which is what features like Z-ordering and liquid clustering in Databricks are for.

## What I'm taking away

- A Parquet table has the same logical structure as any table. The difference is column-by-column storage on disk.
- A data lake is cheap storage for any kind of file. A lakehouse adds ACID (Delta) and governance (Unity Catalog) on top.
- A Delta table is Parquet files plus `_delta_log`. If it's not in the log, it didn't happen.
- RDS runs the app with row storage and small reads/writes. Delta powers analytics with column storage and big scans. Data flows from one to the other.
- Columns win for analytics because you read less: only the columns you need, compressed, with whole chunks skipped.
