# LakeXpress – Commands and flags (0.6.0)

```text
LakeXpress [--version] [--no_banner] COMMAND [SUBCOMMAND] [OPTIONS]
```

Options go **after** the command. `--version` prints e.g. `LakeXpress 0.6.0 (build 2026-10-26T20:28:13Z)`.

| Command | Purpose |
|---|---|
| `lxdb init` / `drop` / `truncate` / `locks` / `release-locks` | Manage the LakeXpress DB |
| `config create` / `list` / `delete` | Manage sync configurations |
| `sync`, `'sync[export]'`, `'sync[publish]'` | Run a sync configuration (both steps, or one) |
| `status` | Runs and their status |
| `cleanup` | Remove stuck (`running`) or `failed` runs |
| `run` | Legacy: run a YAML configuration file |

## Common options (every command)

| Option | Description |
|---|---|
| `--auth_file PATH` (`-a`) | Credentials JSON. Required, except for `run` |
| `--lxdb_auth_id ID` | LakeXpress DB entry. Required, except for `run` |
| `--log_level` | `DEBUG`, `INFO` (default), `WARNING`, `ERROR` |
| `--log_dir PATH` | Log folder (default: current directory) |
| `--no_progress` | No progress bar |
| `--no_banner` | No startup banner |
| `--quiet_fbcp` | Don't copy FastBCP output into the LakeXpress log (statistics still recorded) |
| `--license TEXT` / `--license_file PATH` | License (see environment.md) |

## lxdb

`init`, `drop`, `truncate` and `release-locks` take only the common options plus:

| Command | Options |
|---|---|
| `lxdb init` | – Creates the tables, keeps existing ones (safe to rerun), checks the connection |
| `lxdb drop` | `--confirm` drops all LakeXpress tables and data. Without it: lists them |
| `lxdb truncate` | `--confirm` empties all tables (configurations, runs, watermarks). Without it: lists them. `--sync_id` is not implemented (nothing deleted): use `config delete` |
| `lxdb locks` | `--sync_id` filter. Lists tables locked by incremental syncs, with their table id |
| `lxdb release-locks` | `--max_age_hours N` (default 24), `--table_id ID` (one lock), `--confirm` (without it: dry run) |

## config create

Required: `--auth_file`, `--lxdb_auth_id`, `--source_db_auth_id`, and at `sync` time `--output_dir` or
`--target_storage_id` (not checked by `config create`). Prints the `sync_id` and the `sync` commands.

| Group | Option | Default | Description |
|---|---|---|---|
| Identity | `--sync_id ID` | generated `sync-YYYYMMDD-xxxxxx` | 1–64 letters, digits, `_`, `-`. Error (exit 10) if it exists |
| | `--env_name NAME` | `default` | Groups configurations (dev/test/prod), filter for `config list` |
| Source | `--source_db_auth_id ID` | – | Source entry |
| | `--source_db_name NAME` | `info.database` of the source entry | Source database |
| | `--source_schema_name LIST` | all schemas | Comma list, SQL `LIKE` patterns (`public`, `prod_%`, `"sales%, marketing%"`) |
| Tables | `--include LIST` (`-i`) | all | Table `LIKE` patterns, comma list (`"fact_%, dim_%"`) |
| | `--exclude LIST` (`-e`) | none | Table `LIKE` patterns to skip (`"temp%, test%"`) |
| | `--min_rows N` / `--max_rows N` | – | Skip tables with fewer / more rows |
| Incremental | `--incremental_table SPEC` | – | Repeat per table: `schema.table:column:type[:i\|:e][@start][!strategy]` (incremental.md) |
| | `--incremental_safety_lag SECONDS` | `0` | Upper bound = now − lag (date/datetime watermarks) |
| Output | `--output_dir PATH` | – | Local folder; relative paths are saved as absolute (0.4.9+) |
| | `--target_storage_id ID` | – | Cloud storage entry (S3, GCS, Azure, OneLake) |
| | `--sub_path PATH` | – | Folder between the storage base path and the schema folders; also the `{subpath}` token |
| Engine | `--fastbcp_dir_path PATH` | bundled `engine/` | Folder containing another FastBCP |
| | `--n_jobs N` | `1` | Tables exported at the same time (also publishing workers) |
| | `--fastbcp_p N` (`-p`) | `1` | FastBCP processes per table, used when > 1 and the table has ≥ `--large_table_threshold` rows |
| | `--large_table_threshold N` | `100000` | Smaller tables are exported in one stream |
| | `--fastbcp_table_config CONFIG` | – | Per table: `[schema.]table:method:key_column:degree`, `;`-separated or repeated (export.md) |
| | `--compression_type` | `Zstd` | `Zstd`, `Snappy`, `Gzip`, `Lz4`, `None` |
| Publish | `--publish_target ID` | – | Platform entry (publishing.md) |
| | `--publish_method` | `external` | `external` (tables over the files) or `internal` (data loaded) |
| | `--publish_database_name NAME` | – | Glue database / BigQuery dataset, literal, replaces `--publish_schema_pattern` |
| | `--publish_schema_pattern P` | `EXT_{schema}` (external), `{schema}` (internal) | Target schema; tokens below, never `{table}` |
| | `--publish_table_pattern P` | `{table}` | Target table; must contain `{table}` |
| | `--no_views` | views created | Snowflake external: no `V_` views |
| | `--pk_constraints` | off | Source primary keys on internal tables (Snowflake, Databricks, Fabric, BigQuery, MotherDuck, Redshift) |
| Metadata | `--generate_metadata` | off | CDM `manifest.json` + `.cdm.json` next to the Parquet files |
| | `--manifest_name NAME` | schema or database name | CDM manifest name |
| Errors | `--error_action` | `fail` | `fail`: stop at the first failed table. `continue`: export the other tables |

Pattern tokens: `{schema}`, `{table}` (table pattern only), `{database}`, `{date}` (YYYYMMDD),
`{timestamp}` (YYYYMMDD_HHMMSS, local time), `{uuid}` (one per run), `{subpath}` (`--sub_path` value).
Patterns are validated when `sync` runs (exit 10).

## config list / config delete

| Command | Options |
|---|---|
| `config list` | `--env_name NAME` filter |
| `config delete` | `--sync_id ID` (required), `--confirm` (without it: shows the counts that would be deleted). Deletes the configuration, its runs, table metadata, incremental tables and watermarks. Files and published tables are kept |

There is no command to modify a configuration: delete it and create it again.

## sync, sync[export], sync[publish]

| Command | Options | Does |
|---|---|---|
| `sync` | `--sync_id` (required), `--resume`, `--run_id`, `--fastbcp_dir_path` | Export, then publish the latest **completed** run if the configuration has a publish target and the export exited 0 or 40 |
| `'sync[export]'` | `--sync_id`, `--fastbcp_dir_path` | Export only (no `--resume`) |
| `'sync[publish]'` | `--sync_id`, `--run_id` (default: latest completed run) | Publish files already exported. Nothing to publish: exit 0. No completed run: exit 10 |

- `--resume` resumes the latest incomplete run of the configuration, or the `--run_id` given: pending and
  failed tables are exported again, completed ones are skipped. Errors (run not found, run of another
  sync, no incomplete run, run already running) exit 10.
- `--fastbcp_dir_path` on `sync` overrides the folder saved in the configuration.
- Only one run of a configuration at a time: a run left `running` by a crash blocks the next ones
  (`… is already running … use 'cleanup'`, exit 10).

## status

`--sync_id ID` (runs of a configuration: summary of the last 10, `--verbose` / `-v` lists the last 20)
or `--run_id ID` (one run, jobs by status). One of the two is required (exit 10 otherwise).

## cleanup

Removes runs of a configuration stuck in `running` or `failed`; completed runs and files are kept.

| Option | Description |
|---|---|
| `--sync_id ID` | Required |
| `--older-than DURATION` | `7d`, `24h`, `30m`, `60s` |
| `--status` | `running` or `failed` (default: both) |
| `--dry-run` | Show only |

Note the hyphens in `--older-than` and `--dry-run` (the only hyphenated options).

## run (legacy)

`run --config FILE.yml [--auth_file …] [--lxdb_auth_id …]` (`-c`). Each `sync` writes a copy of its
configuration as `lx_<date>_<run_id>.yml` next to its log; `run` creates a **new** configuration from
such a file and exports it (no publishing). Prefer `config create` + `sync`.
