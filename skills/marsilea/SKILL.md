---
name: marsilea
description: >-
  Build composable, publication-quality figures with the Marsilea Python library
  — a central plot (heatmap, dot matrix, bar chart) surrounded by aligned panels
  (dendrograms, category strips, bars, labels). Use when the user wants to
  create, edit or debug a Marsilea plot, or describes a multi-panel figure with
  aligned side annotations even if unnamed: clustered/grouped heatmaps,
  oncoprints, UpSet plots, sequence logos, arc diagrams, "a heatmap with a
  dendrogram on top and a bar chart on the side". Also for any matrix with TWO
  values per cell — dot/bubble plots where size encodes one and color the other:
  single-cell marker dot plots (expression × % cells), GSEA dot plots (NES ×
  FDR) — and for significance brackets on a composed figure. Prefer over
  matplotlib/seaborn for multi-panel or grouped figures. Do NOT use for a single
  standalone plot (a lone heatmap, bar/scatter, or boxplot needing p-value stars
  — statannotations' job), plotly/dashboards, or analysis producing no figure
  (DESeq2, a bare scipy dendrogram).
license: MIT
---

# Marsilea: Composable Visualization

Marsilea builds figures by **composition**: you start with one main plot, then
attach smaller aligned plots to its sides (or overlay them on top). Every added
plot shares the main plot's axis, so rows/columns stay aligned automatically.
This is what makes it ideal for annotated heatmaps, single-cell dot plots,
oncoprints, and similar multi-panel figures.

The whole library follows one mental model. Internalize this and most tasks
become mechanical:

> **Pick a board → add side/overlay plots → (optionally) group & cluster →
> (optionally) annotate significance → add legends & title → render/save.**

## Setup

Marsilea is on PyPI and conda-forge. If it is not importable, install it:

```bash
pip install marsilea          # or: conda install -c conda-forge marsilea
```

Significance annotation (Step 4) needs **marsilea ≥ 0.7** plus an optional
extra, which pulls in `statannotations` and `statsmodels`:

```bash
pip install "marsilea[stats]"
```

Only install the extra when the figure actually needs brackets — plain
`marsilea` covers everything else.

Standard imports — use these two aliases consistently, they appear everywhere in
the docs and users will expect them:

```python
import matplotlib
matplotlib.use("Agg")           # headless: render to a file without a display
import marsilea as ma          # boards + datasets: ma.Heatmap, ma.WhiteBoard, ma.load_data
import marsilea.plotter as mp  # the plots you attach: mp.Bar, mp.Colors, mp.Labels, ...
```

Set the `Agg` backend (before importing pyplot/marsilea, or via the `MPLBACKEND=Agg`
environment variable) whenever you're rendering to a file in a non-interactive or
sandboxed environment. Without it, `render()` can hang waiting on a GUI backend
that isn't there. Drop it if the user is working in a notebook or wants an
interactive window.

Data is plain numpy arrays or pandas DataFrames/Series. A `Heatmap` takes a 2D
matrix; side plots take a 1D array/Series (one value per row or column) or a
category list. Nothing needs to be pre-aligned — Marsilea keeps everything in
register as long as lengths match the main matrix.

## Step 1 — Pick a board

The board is the central plot and the canvas everything attaches to. Pick it
from **what the data is**, not from habit — a heatmap is not the default
answer, it is the answer for one colored value per cell.

| The data is | Encoding | Board |
|---|---|---|
| **Two values per cell** — magnitude *and* confidence / abundance / n | color + size | `ma.SizedHeatmap(size, color=...)` |
| Sparse, zero-heavy, or full of NaN | size (empty cells shrink away) | `ma.SizedHeatmap` |
| Enrichment / GSEA-style results | color = NES or effect, size = −log10 FDR or gene count | `ma.SizedHeatmap` |
| One dense continuous matrix; or values you want written in the cells; or diverging around a `center` | color | `ma.Heatmap` |
| A matrix of **categories** | discrete blocks | `ma.CatHeatmap(data, palette=...)` |
| Not a matrix at all — the center is a bar / stacked-bar / violin / arc plot | — | `ma.WhiteBoard(width, height)`, built with `add_layer` |

`ma.ClusterBoard(cluster_data)` is a blank canvas that still supports
clustering/grouping/dendrograms. Rarely needed directly — `Heatmap` subclasses it.

### Don't default to a heatmap

**Ask whether there is a second matrix.** Expression *and* percent-of-cells;
logFC *and* p-value; mean *and* sample count; correlation *and* n. A heatmap can
only show one of them, so choosing it silently throws the other away. When the
user has both, say so and build a dot plot — this is the single most common
place where the wrong figure gets made.

The same goes for **sparse data**: a heatmap paints "zero" and "low" as two
near-identical colors, and the eye cannot separate them. Dot area can go to
nothing, so the structure shows.

Keep a heatmap when the matrix is a **single dense continuous field** — a
correlation matrix, a big expression block (≳50 × 50, where dots would be too
small to read), or anything you want to overlay `annot=True` values on.

**`SizedHeatmap` vs `Heatmap` + `add_layer(SizedMesh)`.** Use `SizedHeatmap`
when the dots *are* the plot. Overlay a `SizedMesh` on a `Heatmap` when you want
a colored cell background *and* dots on top — the pbmc3k pattern below: filled
cells for expression, open circles for percentage.

**Three knobs make or break a dot plot:**

- `size_norm=Normalize(vmin, vmax)` — pin the size scale to the meaningful
  range. Without it sizes autoscale to the observed data, so a percentage
  matrix spanning only 40–60 % fills the entire size range and reads as if it
  spanned 0–100 %.
- `sizes=(lo, hi)` — the marker area range in points². Default `(1, 200)`;
  raise toward `(1, 600)` on a roomy canvas, lower it when cells are tight.
- `size_legend_kws=dict(title=..., fmt="{x:.0f}")` — always pass a `fmt`. The
  size legend labels are interpolated data values, so without one they print as
  `10.1902`, `24.7712`, …

### Heatmap

`Heatmap` key params: `data`, `cmap`, `vmin`/`vmax`, `center` (for diverging
maps), `linewidth`/`linecolor` (cell borders), `annot=True` (write cell values),
`label` (colorbar title), `width`/`height` (main canvas size in **inches**).

```python
import numpy as np
import marsilea as ma

data = np.random.randint(0, 10, (10, 10))
h = ma.Heatmap(data, cmap="viridis", label="Expression")
```

### Dot plot

`SizedHeatmap(size, color=...)` takes the *size* matrix first and the *color*
matrix second; both are the same shape. Everything else — sides, grouping,
dendrograms, legends — works exactly as on a `Heatmap`.

```python
import numpy as np
import marsilea as ma
import marsilea.plotter as mp
from matplotlib.colors import Normalize

terms = [f"Pathway {i}" for i in range(8)]
groups = ["Tumor", "Stroma", "Immune"]

nes = np.random.uniform(-2.5, 2.5, (len(terms), len(groups)))   # color
fdr = np.random.uniform(1e-6, 0.5, (len(terms), len(groups)))
sig = -np.log10(fdr)                                            # size

d = ma.SizedHeatmap(
    size=sig, color=nes,
    cmap="RdBu_r", center=0,                    # diverging on the color matrix
    sizes=(5, 250), size_norm=Normalize(0, 5),  # pin the size scale, don't autoscale
    width=1.6, height=3.2,
    color_legend_kws=dict(title="NES"),
    size_legend_kws=dict(title="-log10 FDR", fmt="{x:.1f}"),  # else labels read "1.17641"
)
d.add_left(mp.Labels(terms))
d.add_top(mp.Labels(groups))
d.add_legends("right")
d.render()
```

### Sizing the canvas (inches — compute it from the data shape)

Marsilea has no `figsize`. Everything is in **inches**: `width`/`height` set the
main canvas, each side plot adds its own `size` on top, and the figure grows to
fit. Because side plots stay aligned to the main axis, sizing the main canvas
sensibly is what makes or breaks the layout — so derive it from the matrix shape
rather than guessing a number.

A reliable rule for matrix boards (`Heatmap`/`SizedHeatmap`/`CatHeatmap`): pick a
per-cell size in inches, multiply by the number of columns (width) and rows
(height), then clamp to a readable range.

```python
def canvas_size(n_rows, n_cols, cell=0.3, lo=2.0, hi=12.0):
    # cell ≈ inches per cell; ~0.3 keeps cells near-square and labels legible
    width  = min(max(n_cols * cell, lo), hi)
    height = min(max(n_rows * cell, lo), hi)
    return width, height

w, h = canvas_size(*data.shape)          # data is (n_rows, n_cols)
h_map = ma.Heatmap(data, width=w, height=h)
```

Choosing the numbers:

- **Per-cell ≈ 0.25–0.4 in** keeps heatmap cells legible and roughly square. Drop
  toward ~0.15 for large matrices (more than ~40 cells on a side) so the figure
  stays a sane size; raise toward ~0.5 for a small matrix you want people to read
  cell by cell.
- **Clamp to ~2–12 in per side.** A 200-row matrix at 0.3 in/cell would be 60 in
  tall — cap it (rows compress) or the figure is unusable. The `lo`/`hi` bounds
  above do this.
- **Very lopsided shapes** (e.g. 500 × 10): don't force square cells. Use a
  smaller per-cell value on the long dimension, or just let the clamp cap it, so
  the figure isn't a thin ribbon.

Typical `size` (inches) for the side plots you attach:

- Category / annotation strips (`mp.Colors`, `mp.Chunk`): **0.2–0.3**.
- Dendrograms (`add_dendrogram`): **0.5–1.0** (default is 0.5).
- Bar / number / distribution panels (`mp.Numbers`, `mp.Bar`, `mp.Violin`,
  `mp.Box`): **0.6–1.2**, enough to read the values.
- Text labels (`mp.Labels`): leave `size=None` so they auto-size to the text.

If the whole figure still comes out too big or small after this, don't re-tune
every number individually — pass `render(scale=0.8)` (or `1.2`) to scale the
entire composition at once.

## Step 2 — Add side plots and overlays

Attach plots with `add_left`, `add_right`, `add_top`, `add_bottom` (or the
generic `add_plot(side, plot)`). To draw *on top of* the main canvas, use
`add_layer`. All of these return the board, so calls can be chained, though
writing them one per line reads more clearly.

Common signature: `add_left(plot, name=None, size=None, pad=0.0, legend=True)`
where `size` and `pad` are inches. `size=None` lets the plot auto-size.

```python
h.add_left(mp.Numbers(data.sum(axis=1), color="#F05454"))  # row-sum bars on the left
h.add_top(mp.Labels(["c%d" % i for i in range(10)]))        # column labels on top
h.add_layer(mp.MarkerMesh(data > 7, color="red", marker="*"))  # star the big cells
```

The plots you attach come from `marsilea.plotter` (`mp`). The most common:

| Plotter | Purpose |
|---|---|
| `mp.Colors(cats, palette=...)` | Categorical color strip — the workhorse row/column annotation bar. |
| `mp.Labels(labels)` | One text label per row/column (tick-like). |
| `mp.AnnoLabels(labels, mark=[...])` | Label only a few selected items, with leader lines. |
| `mp.Numbers(values)` | Single-series bar chart (row sums, counts). |
| `mp.Bar / mp.Box / mp.Violin / mp.Boxen / mp.Point / mp.Strip / mp.Swarm` | Statistical distributions per row/column (seaborn-backed). All carry `annotate_stats()` — Step 4. |
| `mp.StackBar(df)` | Stacked bars (df rows = segments). |
| `mp.Chunk(labels, fill_colors=...)` | Labels for grouped chunks — pairs with `group_rows/cols`. |
| `mp.ColorMesh / mp.SizedMesh / mp.MarkerMesh / mp.TextMesh` | Matrix layers to overlay via `add_layer`. |
| `mp.Title(text)` | A title band on a side. |

Full plotter catalog with every constructor argument is in
`references/plotters.md` — read it when the user needs a plotter or argument not
listed above.

## Step 3 — Group and cluster (heatmaps / cluster boards)

These only work on `Heatmap`/`ClusterBoard` because they need matrix data.

`add_dendrogram(side, method="ward", ...)` hierarchically clusters rows
(`"left"`/`"right"`) or columns (`"top"`/`"bottom"`) and draws the tree.

`group_rows(group, order=None, spacing=0.01)` / `group_cols(...)` partition the
matrix into labeled chunks and reorder to keep each chunk together. `order` fixes
chunk order; `spacing` is the gap (as a fraction). `cut_rows(cut)` / `cut_cols`
split at fixed index positions *without* reordering.

Grouping and clustering compose: after `group_rows`, `add_dendrogram` clusters
*within* each chunk and (by default) adds a "meta" dendrogram over the chunk
means. A board can only be split once per axis — don't call both `group_rows`
and `cut_rows`, or `group_rows` twice.

The idiomatic grouped-heatmap trio:

```python
groups = np.random.choice(["A", "B", "C"], 10)
h.group_rows(groups, order=["A", "B", "C"])
h.add_left(mp.Chunk(["A", "B", "C"], fill_colors=["#F05454", "#F0F0F0", "#54F0F0"]))
h.add_dendrogram("left", colors=["#F05454", "#F0F0F0", "#54F0F0"])
```

## Step 4 — Significance annotation (optional)

Every seaborn-backed plotter (`mp.Bar`, `mp.Box`, `mp.Boxen`, `mp.Violin`,
`mp.Point`, `mp.Strip`, `mp.Swarm`) has `annotate_stats()`: it runs a
statistical test on pairs of categories and draws the brackets on the panel.
Needs marsilea ≥ 0.7 and `pip install "marsilea[stats]"`.

Call it **on the plotter, before attaching it to the board** — it configures the
plotter and returns it; the brackets are drawn at `render()`.

```python
import numpy as np, pandas as pd
import marsilea as ma
import marsilea.plotter as mp

genes = [f"Gene {i}" for i in range(6)]
control = pd.DataFrame(np.random.normal(4, 1, (30, 6)), columns=genes)
treated = pd.DataFrame(np.random.normal(4, 1, (30, 6)) + [0, 1, -1, 0, 1.5, 0],
                       columns=genes)

box = mp.Box({"Control": control, "Treated": treated}, label="Expression")
box.annotate_stats(pairs="hue", test="Mann-Whitney", text_format="star",
                   comparisons_correction="Benjamini-Hochberg")

h = ma.Heatmap(treated.values - control.values, label="Difference")
h.add_top(box, size=2, pad=0.1)
h.add_bottom(mp.Labels(genes))
h.add_legends()
h.render()
```

Three things decide whether this works:

- **`pairs="hue"` needs dict input.** Pass `{"Control": df1, "Treated": df2}` to
  the plotter to get two conditions per category; `pairs="hue"` then compares
  them inside every category. `pairs="all"` compares the categories with each
  other instead, and an explicit list annotates only the comparisons you name.
- **Categories are named by the columns of the input DataFrame.** Pass a
  DataFrame with meaningful column labels and you can write
  `pairs=[(("Gene 1", "Control"), ("Gene 1", "Treated"))]`. A plain numpy array
  names them by position (`0, 1, 2, …`) instead, which is far harder to read.
- **The annotation follows the data.** Brackets are attached to categories, not
  positions, so `group_cols`/clustering move them with the data — and a pair
  whose two sides land in different groups is bracketed across both axes.

Already have p-values from elsewhere (a DE pipeline, a model)? Pass
`pvalues=[...]` with an explicit `pairs` list and no test is run.

Full option surface — every test name, the correction methods, bracket styling,
`ref=` shorthands — is in `references/plotters.md` § Significance annotation.

## Step 5 — Legends, title, render, save

`add_legends(side="right", order=None, ...)` collects the legend/colorbar from
every added plot into one packed box. Add it *after* everything else (and, when
composing multiple boards, after composition) so all legends merge. Pass a plot
`name=` when adding it and reference those names in `order=[...]` to control
legend order.

```python
h.add_legends("right")
h.add_title(top="My figure")   # or bottom=/left=/right=
h.render()                     # builds the layout and draws everything
h.save("figure.png", dpi=300)  # auto-renders if needed; defaults bbox_inches="tight"
```

After `render()`, reach into any subplot for matplotlib tweaks:
`h.figure` is the Figure, `h.get_main_ax()` the main axes, `h.get_ax("name")`
a named side plot. Names come from `name=` when adding, or `get_plot_names()`.

Always finish a generated script with `render()` (and `save()` if the user wants
a file) — without `render()` nothing is drawn.

## Complete worked example

A single-cell style dot plot: normalized expression heatmap, dot overlay for
percent-of-cells, a violin panel on top, count bars on the right, grouped rows
with a chunk strip and dendrogram, one merged legend box.

```python
import numpy as np
import marsilea as ma
import marsilea.plotter as mp
from matplotlib.colors import Normalize

genes = [f"Gene{i}" for i in range(12)]
cells = ["CD4 T", "CD8 T", "B", "NK", "Monocyte", "Dendritic"]
lineage = ["Lymphoid", "Lymphoid", "Lymphoid", "Lymphoid", "Myeloid", "Myeloid"]

expr = np.random.rand(len(cells), len(genes))
pct  = np.random.rand(len(cells), len(genes)) * 100
counts = np.random.randint(50, 500, len(cells))

h = ma.Heatmap(expr, cmap="Greens", label="Norm. expression", width=5, height=3.5)
h.add_layer(mp.SizedMesh(pct, color="none", edgecolor="#6E75A4",
                         size_norm=Normalize(0, 100), sizes=(1, 300),
                         size_legend_kws=dict(title="% cells", fmt="{x:.0f}")))
h.add_top(mp.Violin(expr, color="#ee6666", linewidth=0), size=0.8, name="dist")
h.add_right(mp.Numbers(counts, color="#fac858", label="Cell count"), size=0.7, pad=0.1)
h.add_left(mp.Labels(cells))
h.add_bottom(mp.Labels(genes))

h.group_rows(lineage, order=["Lymphoid", "Myeloid"])
h.add_left(mp.Chunk(["Lymphoid", "Myeloid"], ["#33A6B8", "#B481BB"]), pad=0.05)
h.add_dendrogram("left", colors=["#33A6B8", "#B481BB"])

h.add_legends("right")
h.add_title(top="PBMC marker expression")
h.render()
h.save("pbmc_dotplot.png", dpi=300)
```

## Specialized figures

Some figure types have dedicated builders. Read `references/advanced.md` for
full APIs and examples when the user asks for:

- **UpSet plots** — set-intersection visualization (`ma.Upset`, `ma.UpsetData`).
- **Oncoprints** — mutation landscapes across samples (the `oncoprinter` package).
- **Sequence logos** — `mp.SeqLogo` for motif/alignment logos.
- **Arc diagrams** — `mp.Arc` for node-link/chord-style relationships.
- **Composing multiple boards** — `board_a + board_b` (side by side) and
  `board_a / board_b` (stacked) build a `CompositeBoard`; add legends after.

## Working style

Prefer writing a complete, runnable script and running it to confirm the figure
renders, rather than describing code you haven't executed — Marsilea's layout can
surprise you (a mis-sized side plot, a legend that collides), and a quick render
catches it. When something looks off, render to a PNG and inspect it. Save the
final script to the user's working folder alongside the image so they can rerun
and tweak it.

Traps worth remembering. The first three render with no error but are subtly
wrong, so *look at the image*:

- `mp.AnnoLabels(labels, mark=[...])`: `mark` is the list of label **values** to
  show, not a boolean mask. A mask matches nothing and draws zero callouts. See
  `references/plotters.md` for the "top N" pattern.
- `mp.SizedMesh` / `ma.SizedHeatmap` without `size_norm`: sizes autoscale to the
  observed data, so the smallest value in the matrix always draws as the
  smallest dot — even when it is 40 % on a 0–100 % scale. Pass
  `size_norm=Normalize(vmin, vmax)`.
- `annotate_stats(pairs="hue")` on `mp.Strip`/`mp.Swarm`/`mp.Point` **without**
  `dodge=True`: seaborn draws the hue levels on top of each other, so those
  comparisons are skipped with a warning and no bracket appears. The script
  still exits 0.

These raise, and the message tells you the fix:

- `SplitTwice` — you split the same axis twice. Consolidate to a single
  `group_*`/`cut_*` call.
- `LayerConflict` — a seaborn plotter (`mp.Violin`, `mp.Box`, …) cannot share
  the main canvas with a mesh; `add_layer` is for mesh plotters only. Move it to
  a side with `add_top`/`add_bottom`/`add_left`/`add_right`.
- `DuplicatePlotter` — one plotter instance can only be added to a board once.
  Construct a second one instead of reusing the variable.
