# Marsilea skill for AI coding agents

An [Agent Skill](https://agentskills.io) that teaches a coding agent to build
composable, publication-quality figures with the
[Marsilea](https://marsilea.readthedocs.io) Python library — annotated and
clustered heatmaps, dot/bubble plots (single-cell markers, GSEA enrichment),
significance-annotated panels, oncoprints, UpSet plots, sequence logos, arc
diagrams, and more.

The skill follows the open [Agent Skills specification](https://agentskills.io/specification),
so it works in Claude Code and in the other agents that read `SKILL.md`.

## Install

### Claude Code / Cowork

```
/plugin marketplace add Marsilea-viz/marsilea-skill
/plugin install marsilea@marsilea-marketplace
```

Run `/reload-plugins` (or restart) afterwards. The skill then activates on its
own whenever you ask for a Marsilea or composable-visualization figure.

### Any other agent

Copy the skill directory into wherever your agent looks for skills:

```shell
git clone https://github.com/Marsilea-viz/marsilea-skill
cp -r marsilea-skill/skills/marsilea <your-agent-skills-dir>/
```

Common locations are `.agent/skills/`, `.cursor/skills/` or `~/.config/<agent>/skills/` —
check your agent's own documentation. Some agents can also install straight from
the repository:

```shell
npx skills add Marsilea-viz/marsilea-skill
```

## Requirements

Marsilea has to be installed in the Python environment the agent runs code in:

```shell
pip install marsilea   # or: conda install -c conda-forge marsilea
```

Significance annotation needs **marsilea >= 0.7** plus its optional extra:

```shell
pip install "marsilea[stats]"
```

## What's inside

```
.
├── .claude-plugin/
│   ├── marketplace.json     # Claude Code catalog entry (source: ".")
│   └── plugin.json          # Claude Code plugin manifest
└── skills/
    └── marsilea/
        ├── SKILL.md         # the skill: core workflow
        └── references/
            ├── plotters.md  # full plotter catalog
            └── advanced.md  # UpSet, oncoprint, seq logo, arc, composition
```

`skills/marsilea/` is the single source of truth. The Claude plugin loads it from
the repository root via `"source": "."`, so there is no second copy to keep in
sync.

## Developing

Validate against the Agent Skills spec before pushing:

```shell
uvx --from skills-ref agentskills validate ./skills/marsilea
```

Note the spec caps `description` at 1024 characters — that is the limit most
likely to be hit when editing the frontmatter.

## Updating

Push changes and bump `version` in both `.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json`. Claude Code users pull updates with
`/plugin marketplace update marsilea-marketplace`.

## Related

- [Marsilea documentation](https://marsilea.readthedocs.io)
- [`llms.txt`](https://marsilea.readthedocs.io/llms.txt) — machine-readable
  summary for agents that can fetch a URL instead of installing a skill
- [Use Marsilea with AI assistants](https://marsilea.readthedocs.io/en/stable/ai_assistants.html)

## License

MIT
