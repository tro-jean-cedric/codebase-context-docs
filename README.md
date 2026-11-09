# claude-skills

Claude Code skills, packaged as a plugin marketplace.

## codebase-context-docs

Analyzes a repository and writes a lean `CLAUDE.md` (under 150 lines) plus focused
`.claude/docs/` reference files — `architectural_patterns.md`, `testing_strategy.md`,
`build_and_deploy.md`, `data_model.md`, `api_reference.md` — using progressive disclosure,
so agents get oriented on turn one without paying a huge always-loaded context cost.

Triggers on requests like "generate a CLAUDE.md", "my CLAUDE.md is bloated", "/init",
or "get up to speed on this repo".

## Install

### Claude Code (recommended — gets updates)

```
/plugin marketplace add tro-jean-cedric/claude-skills
/plugin install codebase-context-docs@tro-skills
```

Update later with `/plugin marketplace update tro-skills`.

### Any agent, via the skills CLI

```bash
npx skills add tro-jean-cedric/claude-skills
```

### Manual

Copy `plugins/codebase-context-docs/skills/codebase-context-docs/` into `~/.claude/skills/`.

### claude.ai / Claude desktop

Zip the `codebase-context-docs` skill folder and upload it under
**Settings → Capabilities → Skills**.

## Layout

```
.claude-plugin/marketplace.json
plugins/codebase-context-docs/
├── .claude-plugin/plugin.json
└── skills/codebase-context-docs/
    ├── SKILL.md
    ├── references/templates.md
    └── evals/evals.json
```

## License

MIT
