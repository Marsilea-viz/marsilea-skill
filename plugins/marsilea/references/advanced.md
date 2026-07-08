# Marsilea advanced / specialized figures

Read this when the user needs a figure type beyond the standard
board-plus-annotations workflow.

## Table of contents
- UpSet plots
- Composing multiple boards (concatenation)
- Sequence logos
- Arc diagrams
- Oncoprints
- Built-in example datasets

## UpSet plots

For visualizing intersections among many sets (an alternative to Venn diagrams).

```python
import marsilea as ma

# From explicit sets:
data = ma.UpsetData.from_sets(
    [{"a", "b", "c"}, {"b", "c", "d"}, {"c", "d", "e"}],
    sets_names=["Set1", "Set2", "Set3"],
)
# Or from a membership table (items x sets boolean/0-1 DataFrame):
# data = ma.UpsetData.from_memberships(items, items_names=...)

us = ma.Upset(
    data,
    orient="h",                 # "h" or "v"
    sort_subsets="cardinality", # or "degree"
    min_degree=None, max_degree=None,
    add_intersections=True, add_sets_size=True,
)
us.render()
us.save("upset.png", dpi=300)
```

`ma.Upset` is itself a composed board, so you can `add_title`, `add_legends`, and
reach into axes after `render()` just like a Heatmap.

## Composing multiple boards (concatenation)

Boards compose with operators into a `CompositeBoard`:

- `board_a + board_b` → place side by side (horizontally).
- `board_a / board_b` → stack vertically.

Add legends and titles **after** composing so legends from all boards merge into
one box. This is how you build, e.g., two heatmaps sharing a row axis.

```python
h1 = ma.Heatmap(mat1, cmap="Reds", label="A")
h2 = ma.Heatmap(mat2, cmap="Blues", label="B")
combined = h1 + h2
combined.add_legends()
combined.render()
```

## Sequence logos

```python
import marsilea.plotter as mp
# matrix: pandas DataFrame, index = positions, columns = letters (A/C/G/T or amino acids),
# values = height/frequency.
logo = mp.SeqLogo(matrix, color_encode=None, stack="descending")
```

`SeqLogo` is usually attached to a board via `add_top`/`add_bottom` to annotate a
sequence-aligned heatmap, or overlaid with `add_layer`. See
`docs/source/examples/Gallery/plot_seqalign.py` and `.../Plotters/plot_seq_logo.py`
in the Marsilea repo for full runnable examples.

## Arc diagrams

```python
import marsilea.plotter as mp
# anchors: list of node labels/positions along the axis.
# links: list of (i, j) index pairs into anchors.
arc = mp.Arc(anchors, links, weights=None, colors=None, labels=None)
```

Attach with `add_top`/`add_bottom` to draw relationships above/below a linear
track. See `docs/source/examples/Gallery/plot_arc_diagram.py`.

## Oncoprints

Mutation-landscape plots (genes x samples, with mutation-type color coding and
marginal frequency bars) come from the companion `oncoprinter` package that ships
with Marsilea:

```python
from oncoprinter import OncoPrint
# op = OncoPrint(mutation_dataframe)  # samples x genes with mutation-type labels
# op.render(); op.save("oncoprint.png")
```

See `docs/source/examples/Gallery/plot_oncoprint.py` in the repo for the exact
input format and a full example, and `src/oncoprinter/` for the API.

## Built-in example datasets

`ma.load_data(name)` fetches bundled datasets for demos and testing:

- `"pbmc3k"` — single-cell PBMC expression (dict with `exp`, `pct_cells`, `count`).
- `"oncoprint"` — mutation data for oncoprints.
- `"mouse_embryo"` — spatial single-cell coordinates + types.
- `"cooking"`, `"seq_align"`, and others used across the gallery.

Use these when the user wants a demonstration and hasn't supplied their own data.
