# Soli skills for Claude Code

Two [Claude Code](https://claude.com/claude-code) skills:

- **soli-lang** — writing, reading and debugging [Soli](https://github.com/solisoft/soli_lang) code: syntax, idiomatic style, the framework (routes, controllers, models, views, specs) and the traps where Ruby or JS habits break.
- **solidb** — SoliDB: SDBQL queries, the `Solidb` client, Soli model queries that need raw SDBQL, and the HTTP API.

Every claim in them was checked against a running `soli` / `solidb`.

## Install

In Claude Code:

```
/plugin marketplace add solisoft/soli-skills
/plugin install soli@soli-skills
```

`/plugin marketplace update soli-skills` pulls later versions.

Or as plain user skills, without the plugin system:

```bash
git clone https://github.com/solisoft/soli-skills ~/src/soli-skills
mkdir -p ~/.claude/skills
ln -s ~/src/soli-skills/plugins/soli/skills/soli-lang ~/.claude/skills/soli-lang
ln -s ~/src/soli-skills/plugins/soli/skills/solidb ~/.claude/skills/solidb
```

`git pull` in the clone then updates both.

## Layout

```
.claude-plugin/marketplace.json     the marketplace (one plugin)
plugins/soli/.claude-plugin/plugin.json
plugins/soli/skills/soli-lang/      SKILL.md + references/
plugins/soli/skills/solidb/         SKILL.md + references/ + scripts/
```

`soli new` also writes a copy of the solidb skill into every app
(`.claude/skills/solidb/`); soli_lang's `scripts/sync-solidb-skill.sh` keeps
that copy in step with this one.
