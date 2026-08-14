# Marsilea skill for Claude

A Claude Code / Cowork plugin that teaches Claude to build composable,
publication-quality figures with the [Marsilea](https://marsilea.readthedocs.io)
Python library — annotated and clustered heatmaps, dot/bubble plots (single-cell
markers, GSEA enrichment), significance-annotated panels, oncoprints, UpSet
plots, sequence logos, arc diagrams, and more.

## Install

In Claude Code (or Cowork), add this repository as a marketplace, then install
the plugin:

```
/plugin marketplace add Marsilea-viz/marsilea-skill
/plugin install marsilea@marsilea-marketplace
```

Replace `Marsilea-viz/marsilea-skill` with the `owner/repo` you push this to.
After installing, run `/reload-plugins` (or restart) and the skill activates
automatically whenever you ask Claude for a Marsilea / composable-visualization
figure.

Requires the `marsilea` package in your Python environment:

```
pip install marsilea   # or: conda install -c conda-forge marsilea
```

Significance annotation needs **marsilea >= 0.7** plus its optional extra
(`statannotations` + `statsmodels`):

```
pip install "marsilea[stats]"
```

## What's inside

```
.
├── .claude-plugin/
│   └── marketplace.json        # catalog listing the plugin
├── plugins/
│   └── marsilea/
│       ├── .claude-plugin/
│       │   └── plugin.json      # plugin manifest
│       ├── SKILL.md             # the skill: core workflow
│       └── references/
│           ├── plotters.md      # full plotter catalog
│           └── advanced.md      # UpSet, oncoprint, seq logo, arc, composition
```

## Updating

Push changes and bump `version` in both `plugin.json` and `marketplace.json`.
Users pull updates with `/plugin marketplace update marsilea-marketplace`.

## License

MIT
