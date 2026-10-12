# Contributing

## Adding a product

1. Create `plugins/<product>/.claude-plugin/plugin.json` (copy `plugins/fastbcp/` as a template).
2. Add `plugins/<product>/skills/<skill>/SKILL.md` and `references/`.
3. Register the plugin in `.claude-plugin/marketplace.json`.
4. Add a row to the table in `README.md`.
5. Run `claude plugin validate .` and `claude plugin validate plugins/<product>`.

## Writing skills

- **Only documented facts.** Every flag, value and default must come from the product docs.
  When the docs contradict themselves, choose a safe rule and record it in `references/troubleshooting.md`.
- **SKILL.md stays short** (about 150 lines): when to use, workflow, decision table, hard rules, one example.
  Put details in `references/*.md` and link them from SKILL.md.
- **Frontmatter `description`** says what the skill does *and* when to trigger it (product name, typical
  user phrasings).
- **Self-contained.** A skill must not link to files in another plugin. Duplicate shared knowledge
  (e.g. connection types shared by FastBCP and FastTransfer) instead.
- **Long CLI flags** in generated commands. Give both bash and PowerShell forms where they differ.
- **No secrets.** Examples use `--trusted`, shell variables or doc placeholders, never real credentials.
- State the product version the skill reflects, and update it with each product release.

## Testing a skill

- Load the local marketplace in Claude Code: `/plugin marketplace add ./` then `/plugin install <name>@arpe-io`.
- Try typical prompts in a fresh session and check every flag in the generated command against the docs.
