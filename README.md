# XL1 Skills

Claude Code plugin marketplace for XL1 blockchain and XYO protocol development.

## Quick Install

```shell
# Add the marketplace
/plugin marketplace add XYOracleNetwork/xl1-skills

# Install the XL1 skill stack
/plugin install xl1-skills
```

## Plugins

### [xl1-skills](plugins/xl1-skills/)

Full-stack skills for building dApps on XL1. Five cascading layers covering design patterns, chain operations, XYO primitives, build tooling, and development conventions.

## Team Setup

Add to your project's `.claude/settings.json` for automatic marketplace availability:

```json
{
  "extraKnownMarketplaces": {
    "xl1-skills": {
      "source": {
        "source": "github",
        "repo": "XYOracleNetwork/xl1-skills"
      }
    }
  }
}
```

## Developing Skills Locally

### Setup

Clone the repo and load the plugin directly — no marketplace install needed:

```shell
git clone git@github.com:XYOracleNetwork/xl1-skills.git
cd xl1-skills
claude --plugin-dir ./plugins/xl1-skills
```

Alternatively, install from the local marketplace so the plugin loads automatically in every session:

```shell
cd xl1-skills
# Inside a Claude Code session:
/plugin marketplace add ./
/plugin install xl1-skills
```

### Edit-Reload Cycle

Claude Code loads skill content at startup. After editing any skill file, you must reload for changes to take effect:

1. Edit a `SKILL.md` or sub-file in `plugins/xl1-skills/skills/`
2. Run `/reload-plugins` in your Claude Code session
3. Changes are active for the rest of the session

There is no file watcher — `/reload-plugins` is required after every edit.

### Skill File Structure

Each skill is a directory under `plugins/xl1-skills/skills/` containing a `SKILL.md` router and topic sub-files:

```
plugins/xl1-skills/skills/
├── development/
│   ├── SKILL.md          ← router (frontmatter + table of contents)
│   ├── typescript.md
│   ├── git.md
│   ├── testing.md
│   └── workflow.md
├── xy-toolchain/
│   ├── SKILL.md
│   └── ...
└── ...
```

`SKILL.md` files require YAML frontmatter with a `description` field. Claude uses this to decide when to activate the skill:

```yaml
---
description: When and why Claude should activate this skill.
---
```

The body is a table of contents linking to sub-files with guidance on when to read each one. Claude loads sub-files on demand, not all at once.

### Verifying Skills Load

After starting Claude Code or running `/reload-plugins`:

- Run `/help` — skills appear as `/xl1-skills:<name>` (e.g., `/xl1-skills:development`)
- Check the reload output for the skill count: `Reloaded: 1 plugins · 5 skills · ...`
- Invoke a skill directly: `/xl1-skills:xl1-patterns`

### Validating Plugin Structure

The CI workflow validates marketplace and plugin manifests. Run locally:

```shell
jq empty .claude-plugin/marketplace.json
jq empty plugins/xl1-skills/.claude-plugin/plugin.json
```

### Branching

Follow Gitflow. Feature branches off `develop`:

```shell
git checkout -b feature/improve-commit-reveal-skill develop
```

## Evaluation Prompt

This repo also serves as a test bed for evaluating the skill stack. The target prompt:

> Build me a two-player rock paper scissors game on XL1. Use commit-reveal so neither player can see the other's move before both have committed. Record moves and outcomes on-chain. Include a UI where anyone can browse past games and results without connecting a wallet, and connected players can start and play games.

## License

MIT
