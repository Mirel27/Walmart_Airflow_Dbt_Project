# Neon PostgreSQL to Databricks Ingestion Notes

This guide explains the planned path from the Walmart PostgreSQL source in Neon to a Databricks lakehouse. It is a learning and implementation reference; it does not contain database credentials.

## Architecture at a glance

```text
Neon PostgreSQL branch
        |
        | CDC, or incremental reads using a cursor
        v
Databricks ingestion pipeline
        |
        v
Unity Catalog: Bronze Delta tables
        |
        v
Silver / Gold transformations (for example, dbt)
        |
        v
Databricks jobs, optionally orchestrated by Airflow
```

Delta Lake is the default table format in this project. Iceberg is another open table format, but choose the format based on the Databricks features and interoperability the project needs rather than mixing formats without a reason.

## Neon branch, database, and table names

These are different levels of the source:

| Resource | Name |
|---|---|
| Neon project | `walmart-data-engineering` |
| Source branch (fork) | `walmart-db-fork` |
| PostgreSQL database on that branch | `walmart_db` |
| Example source table | `raw.stores` |

A Neon branch is an isolated, copy-on-write clone of a branch's data at the time it is created. The branch name does not replace the database name: connect to branch `walmart-db-fork` and database `walmart_db`. Changes made on the fork do not modify its parent branch.

To list branches and get the connection string for the fork:

```powershell
npx.cmd neon@latest branches list --project-id fragrant-dust-32119190
npx.cmd neon@latest connection-string walmart-db-fork --project-id fragrant-dust-32119190 --database-name walmart_db
```

The connection-string command prints a password-bearing URL. Run it in a trusted terminal; do not paste its output into this README, source code, Git, chat, or logs. For Databricks, configure the matching host, database, user, password, port `5432`, and SSL settings as secret-backed connection parameters. A username and password must belong together and be valid for the fork's endpoint. Use a least-privilege database role for ongoing ingestion where the selected connector supports it.

The repository is already linked to the Neon project, and the project MCP configuration and Neon skills are installed. VS Code may prompt you to sign in to Neon the first time you use the MCP server. Use the explicit branch name above when the command must target the fork; avoid changing local branch context just to retrieve its connection details.

## Databricks concepts

- **Unity Catalog** organizes and governs data assets. It contains catalogs, schemas, tables, and permissions; it is not itself the process that copies rows out of PostgreSQL.
- **The ingestion connector or pipeline** connects to PostgreSQL, reads changes, and writes them into Databricks.
- **Bronze** stores source-shaped data in Delta tables for traceability and replay.
- **Silver and Gold** contain cleaned, conformed, and analytics-ready data. dbt can manage SQL transformations and tests in this layer.
- **A pipeline** defines the ingestion and transformation work. A Databricks job schedules and orchestrates pipeline runs; Airflow can orchestrate the job if the project needs an external workflow manager.

Create the destination catalog, schemas, tables, and required grants before running ingestion. Keep pipeline code and job configuration in version control, but keep credentials in a Databricks secret scope or another approved secret manager.

## Choose an incremental ingestion method

### Option A: Change Data Capture (CDC)

CDC reads database changes from PostgreSQL's write-ahead log (WAL), rather than repeatedly querying whole tables. It can capture inserts, updates, and deletes when the chosen connector and its configuration support them.

Before using CDC, verify that the Databricks connector supports the PostgreSQL source and Neon endpoint, and configure the required logical replication settings, publication/replication permissions, and retention. These settings affect the source project and should be reviewed before enabling them. Monitor replication lag and WAL retention so a stalled pipeline does not cause unbounded source storage growth.

### Option B: Cursor-based incremental reads

If CDC is unavailable or not needed, query rows using a reliable change cursor such as `updated_timestamp`. Persist the last successfully committed cursor in the pipeline checkpoint/state, and use the table's primary key as a tie-breaker when multiple rows share a timestamp. The source schema uses `store_id` as the primary key for `raw.stores`.

Cursor-based reads can capture inserts and updates only when the cursor is consistently maintained. They do not detect hard deletes; use CDC, source tombstones, or a soft-delete field such as `is_active` when deletions must flow downstream. Consider a small lookback window for late-arriving updates, and make each run safe to replay.

Do not advance the saved cursor until the corresponding target write succeeds. Use an idempotent upsert/MERGE keyed by the source primary key so retries do not create duplicate rows.

## Suggested implementation sequence

1. Confirm the Neon fork is ready and contains the expected `walmart_db` database and `raw` schema.
2. Obtain the fork's connection details using the command above. Store the password as a Databricks secret and configure TLS/SSL.
3. Test that Databricks can connect and read the intended source tables with the chosen database role.
4. Choose CDC or cursor-based ingestion; document how updates and deletes are handled.
5. Write source-shaped rows to Bronze Delta tables in Unity Catalog, with checkpoints and replay-safe writes.
6. Add Silver/Gold transformations and data quality checks, for example with dbt.
7. Schedule the pipeline with a Databricks job, or have Airflow trigger that job. Avoid scheduling two independent orchestrators to run the same ingestion concurrently.
8. Validate row counts, primary-key uniqueness, update/delete behavior, retries, and recovery from a failed run.

## Local project setup

This repository already has a Python project and lockfile. For an existing checkout, synchronize its dependencies and activate the Windows PowerShell virtual environment:

```powershell
uv sync
.\.venv\Scripts\Activate.ps1
```

`uv init` is for creating a new Python project, not a step to repeat in this existing repository. The Neon project config (`neon.ts`), branch context (`.neon`), and MCP server config (`.vscode/mcp.json`) serve different purposes: service/branch policy, local CLI context, and editor-to-Neon tooling respectively.
