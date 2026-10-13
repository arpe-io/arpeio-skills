# arpe.io agent skills

Agent skills for using [arpe.io](https://www.arpe.io)'s high-performance data tools with Claude Code,
Codex, Cursor, GitHub Copilot and other AI agents.

A skill is a folder with a `SKILL.md` file (plus optional reference files) that teaches an agent how to
use a product correctly: which parameters exist, which options fit together, and which mistakes to avoid.

## Skills

| Product | Skill | What it does | Status |
|---|---|---|---|
| [FastBCP](https://fastbcp-docs.arpe.io) | [`fastbcp`](plugins/fastbcp/skills/fastbcp/SKILL.md) | Builds, explains and troubleshoots FastBCP commands: parallel bulk export from SQL Server, PostgreSQL, Oracle, MySQL and more to CSV / Parquet / JSON / Excel, locally or to S3, Azure, GCS, OneLake | Available |
| [FastTransfer](https://fasttransfer-docs.arpe.io) | [`fasttransfer`](plugins/fasttransfer/skills/fasttransfer/SKILL.md) | Builds, explains and troubleshoots FastTransfer commands: parallel bulk transfer between SQL Server, PostgreSQL, Oracle, MySQL, ClickHouse, SAP HANA, Teradata, Netezza, DuckDB, and CSV / Parquet / JSON file imports | Available |
| [MigratorXpress](https://migratorxpress.arpe.io) | [`migratorxpress`](plugins/migratorxpress/skills/migratorxpress/SKILL.md) | Builds, explains and troubleshoots MigratorXpress migrations: credentials file, task lists, resume, schema and table selection, Oracle / PostgreSQL / SQL Server / Netezza → SQL Server or PostgreSQL | Available |
| [LakeXpress](https://lakexpress-docs.arpe.io) | [`lakexpress`](plugins/lakexpress/skills/lakexpress/SKILL.md) | Builds, explains and troubleshoots LakeXpress pipelines: credentials file, sync configurations, incremental syncs, resume; Oracle / PostgreSQL / SQL Server / MySQL / MariaDB / SAP HANA / Teradata → Parquet on local disk, S3, GCS, Azure, OneLake → Snowflake, Databricks, Fabric, BigQuery, Redshift, MotherDuck, Glue, DuckLake | Available |
| [arpe.io ADBC drivers](https://adbc-drivers-docs.arpe.io) | [`arpeio-adbc-drivers`](plugins/arpeio-adbc-drivers/skills/arpeio-adbc-drivers/SKILL.md) | Installs, configures and troubleshoots the ArpeMSSQL, ArpePGSQL, ArpeOracle and ArpeNetezza ADBC drivers: installer, licence, URIs and connection strings, authentication, TLS, Arrow type mappings, bulk ingest, from Python, C, C#, Go, Java, R or Rust | Available |

## Install

### Claude Code

```
/plugin install fastbcp --marketplace arpe-io/arpeio-skills
/plugin install fasttransfer --marketplace arpe-io/arpeio-skills
/plugin install migratorxpress --marketplace arpe-io/arpeio-skills
/plugin install lakexpress --marketplace arpe-io/arpeio-skills
/plugin install arpeio-adbc-drivers --marketplace arpe-io/arpeio-skills
```

Or in two steps:

```
/plugin marketplace add arpe-io/arpeio-skills
/plugin install fastbcp@arpe-io
/plugin install fasttransfer@arpe-io
/plugin install migratorxpress@arpe-io
/plugin install lakexpress@arpe-io
/plugin install arpeio-adbc-drivers@arpe-io
```

Then just ask, e.g. *"Export dbo.Orders from SQL Server to Parquet on S3 with FastBCP, in parallel"* or
*"Copy the PostgreSQL table public.orders to SQL Server with FastTransfer"*.

### Other agents (Codex, Cursor, Copilot, Gemini CLI…)

Copy the skill folder into the directory your agent reads skills from:

```bash
git clone https://github.com/arpe-io/arpeio-skills.git
cp -r arpeio-skills/plugins/fastbcp/skills/fastbcp <your-agent-skills-dir>/
cp -r arpeio-skills/plugins/fasttransfer/skills/fasttransfer <your-agent-skills-dir>/
cp -r arpeio-skills/plugins/migratorxpress/skills/migratorxpress <your-agent-skills-dir>/
cp -r arpeio-skills/plugins/lakexpress/skills/lakexpress <your-agent-skills-dir>/
cp -r arpeio-skills/plugins/arpeio-adbc-drivers/skills/arpeio-adbc-drivers <your-agent-skills-dir>/
```

For agents without skill support, paste the content of `SKILL.md` as custom instructions. The files in
`references/` can be attached or pasted when needed.

## Repository layout

```
.claude-plugin/marketplace.json     # Claude Code marketplace listing every plugin
plugins/
  <product>/
    .claude-plugin/plugin.json      # one plugin per product
    skills/<skill>/
      SKILL.md                      # concise workflow + rules (always loaded)
      references/*.md               # detailed reference, loaded on demand
```

Each skill is self-contained, so it can be installed on its own.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Issues and pull requests are welcome, especially reports of
commands an agent got wrong.

## License

[MIT](LICENSE). The skills describe arpe.io products. The products themselves are commercial
software and need a license from arpe.io (free trial available).
