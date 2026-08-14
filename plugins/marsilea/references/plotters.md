# Marsilea plotter catalog

Every plotter lives in `marsilea.plotter` (imported as `mp`) and is attached to a
board with `add_left/right/top/bottom(plot, size=, pad=)` or overlaid with
`add_layer(plot)`. Sizes are in inches. Every plotter accepts trailing kwargs
`label=`, `label_loc=`, `label_props=` (`label` is the legend/colorbar title) and
passes unknown `**kwargs` down to the underlying matplotlib/seaborn call.

## Table of contents
- Mesh / matrix plotters (overlay on the main canvas)
- Text & label plotters
- Bar & numeric plotters
- Statistical (seaborn-backed) plotters
- Significance annotation (`annotate_stats`)
- Other (arc, area, range, sequence logo, image, emoji)

## Mesh / matrix plotters — `plotter/mesh.py`

Use these as the main layer (via `add_layer`) or as matrix-shaped side panels.

```python
mp.ColorMesh(data, cmap=None, norm=None, vmin=None, vmax=None, mask=None,
             center=None, alpha=None, linewidth=None, linecolor=None,
             annot=None, fmt=None, annot_kws=None, cbar_kws=None, label=None)
# Gradient heatmap cells. This is what ma.Heatmap wraps.

mp.Colors(data, palette=None, cmap=None, mask=None, linewidth=None,
          linecolor=None, label=None, legend_kws=None)
# Categorical color strip/block. The go-to annotation bar for row/column
# categories (cell type, cluster, condition). palette maps category -> color.

mp.SizedMesh(size, color=None, cmap=None, norm=None, vmin=None, vmax=None,
             alpha=None, center=None, sizes=(1, 200), size_norm=None,
             edgecolor=None, linewidth=1, frameon=True, grid=False,
             grid_color='.8', grid_linewidth=1, palette=None, marker='o',
             legend=True, size_legend_kws=None, color_legend_kws=None,
             edgecolor_legend_text=None, edgecolor_legend_kws=None, **kwargs)
# Dot matrix: dot size encodes `size` matrix, color encodes `color` matrix.
# Wrapped by ma.SizedHeatmap. Set color="none" + edgecolor for open circles.
#
# REACH FOR THIS whenever a cell carries TWO values — expression + % of cells,
# NES + FDR, logFC + p-value, mean + n. A heatmap can only show one of them.
# Also for sparse/zero-heavy matrices, where a heatmap makes "zero" and "low"
# the same color but dot area shrinks to nothing.
#
# Two knobs decide whether it reads correctly:
#   size_norm=Normalize(vmin, vmax)  pin the size scale to the meaningful range.
#       Omit it and sizes autoscale to the observed data, so the smallest value
#       present is always the smallest dot — 40 % on a 0-100 % scale draws as if
#       it were 0 %. This renders fine and is wrong; there is no warning.
#   sizes=(lo, hi)  marker AREA range in points^2. Default (1, 200); up to
#       (1, 600) on a roomy canvas, lower when cells are tight.
# size_legend_kws / color_legend_kws go to legendkit. ALWAYS pass a `fmt` to the
# size legend — its labels are interpolated data values, so the default prints
# things like "1.17641":
#     size_legend_kws=dict(title="% cells", fmt="{x:.0f}")
# `show_at=[0.25, 0.5, 1.0]` picks which entries appear; the values are
# PERCENTILES of the data range, not data values. `num_handle=4` sets how many
# entries to show when `show_at` is omitted.
# `color` categorical? Then `palette` is required to map category -> color.

mp.MarkerMesh(data, color='black', marker='*', size=35, frameon=False, label=None)
# Draws a marker at each True cell of a boolean matrix. Great for flagging
# significant / high cells over a heatmap (add via add_layer).

mp.TextMesh(texts, color='black', frameon=False, label=None)
# Prints a matrix of text strings in a grid.
```

## Text & label plotters — `plotter/text.py`

```python
mp.Labels(labels, align=None, padding=2, text_props=None, label=None)
# One label per row/column, like tick labels. Pass a list/array/Series.

mp.AnnoLabels(labels, mark=None, text_pad=0.5, text_gap=0.5, pointer_size=0.5,
              linewidth=None, connectionstyle=None)
# Label only selected items with leader lines. Use when there are too many rows
# to label them all.
#
# IMPORTANT — `mark` is the list of label VALUES to keep, NOT a boolean mask.
# Internally it does np.in1d(labels, mark), so pass the full `labels` array plus
# the subset of actual values you want annotated:
#     mp.AnnoLabels(all_names, mark=["Gene_02", "Gene_11"])   # correct
# A boolean mask (e.g. scores > 0.9) matches nothing and SILENTLY renders zero
# callouts — the script still exits 0, so you only catch it by viewing the image.
# To pick "top N" rows, first turn indices/scores into the actual names:
#     top = [all_names[i] for i in np.argsort(scores)[-5:]]
#     mp.AnnoLabels(all_names, mark=top)
# Style text with direct kwargs (e.g. fontsize=10), passed through to matplotlib
# text — there is no text_props dict argument here.

mp.Title(title, align='center', padding=10, fontsize=None, fill_color=None,
         bordercolor=None, borderwidth=None, borderstyle=None)
# Title band on a side. board.add_title(top=/bottom=/left=/right=) is a shortcut.

mp.Chunk(texts, fill_colors=None, *, align=None, props=None, padding=8,
         underline=False, bordercolor=None, borderwidth=None, label=None)
# One label (+ optional fill color) per group chunk. Pair with group_rows/cols:
# the number of texts must match the number of groups.

mp.FixedChunk(texts, fill_colors=None, *, ratio=None)
# Chunk labels with explicitly fixed size ratios.
```

## Bar & numeric plotters — `plotter/bar.py`

```python
mp.Numbers(data, width=0.8, color='C0', orient=None, show_value=True, fmt=None,
           value_pad=2.0, props=None, label=None)
# Single-series bar chart (one bar per row/column). Ideal for row sums, counts.

mp.StackBar(data, items=None, colors=None, orient=None, show_value=False,
            value_loc='center', width=0.8, fmt=None, legend_kws=None, label=None)
# Stacked bars; `data` is a DataFrame whose rows are the stacked segments.

mp.CenterBar(data, names=None, width=0.8, colors=None, orient=None, show_value=True)
# Diverging / center-aligned bars.
```

## Statistical plotters (seaborn-backed) — `plotter/_seaborn.py`

All share the same signature and pass `**kwargs` to the matching seaborn function:

```python
mp.Bar   / mp.Box   / mp.Boxen / mp.Violin /
mp.Point / mp.Strip / mp.Swarm(
    data, hue_order=None, palette=None, orient=None, legend_kws=None,
    group_kws=None, label=None, label_loc=None, label_props=None, **kwargs)
```

`data` is a 2D array/DataFrame (each column becomes one distribution aligned to a
row/column of the main plot), or a dict/mapping of name -> data for grouped hue.
Example: `mp.Violin(expr, color="#ee6666", linewidth=0, density_norm="count")`.

These cannot be `add_layer`'d onto a mesh canvas — a seaborn plot scales the
axes to the data while a mesh draws a fixed grid of cells, so one displaces the
other. Marsilea raises `LayerConflict`; attach them to a side instead.

## Significance annotation — `plotter/_stats_annot.py`

Marsilea ≥ 0.7. Every seaborn-backed plotter above carries `annotate_stats()`,
which tests pairs of categories and draws the brackets. Requires the optional
extra `pip install "marsilea[stats]"` (statannotations; `statsmodels` on top for
multiple-comparison correction). Marsilea draws the brackets itself, so a
comparison spanning two groups of a split canvas looks like any other.

```python
plot.annotate_stats(pairs, test="Mann-Whitney", ref=None, pvalues=None,
                    **configure_kws)   # returns the plotter
```

Call it on the plotter before attaching it to the board; brackets are drawn at
`render()`.

### `pairs` — what gets compared

Categories are named by the **columns of the input data**. A plain array names
them by position (`0, 1, 2, …`). Duplicated column labels raise.

| Form | Meaning |
|---|---|
| `"hue"` | Compare the hue levels inside every category. Needs dict input: `mp.Box({"WT": df1, "KO": df2})`. |
| `"all"` | Compare the categories with each other. On a split canvas this stays **inside each group**; warns above 8 categories (36 brackets is unreadable). |
| explicit list | Only the comparisons you name. |

In an explicit list each side of a pair is a bare category label,
`("Gene 1", "Gene 4")`, or a `(category, hue_level)` tuple when the data has
hue, `(("Gene 1", "WT"), ("Gene 1", "KO"))`. Mixing the two forms raises.

`ref=` reduces a shorthand to comparisons against one reference — a hue level
for `pairs="hue"`, a category label for `pairs="all"`. A category `ref` reaches
into every group, not just its own, which is how you compare one control
category against everything after `group_cols`.

```python
box.annotate_stats(pairs="hue", ref="Control")           # each level vs Control
bar.annotate_stats(pairs="all", ref="0", text_format="star")   # each dose vs 0 mg
box.annotate_stats(pairs=[(("Gene 1", "WT"), ("Gene 1", "KO"))])
```

### `test` — statannotations' catalogue

`Mann-Whitney` (default), `Mann-Whitney-gt`, `Mann-Whitney-ls`, `t-test_ind`,
`t-test_welch`, `t-test_paired`, `Wilcoxon`, `Kruskal`, `Levene`,
`Brunner-Munzel`.

### `pvalues` — annotate values you already have

```python
box.annotate_stats(pairs=[...], pvalues=[0.3, 1e-5], text_format="star")
```

One value per pair, in the order the pairs were listed. Needs an explicit
`pairs` list (a shorthand has no fixed order to match against), and no test is
run. This is the hook for a DE pipeline's adjusted p-values.

### `configure_kws` — everything else

An unknown name **raises** rather than being silently ignored.

| Kwarg | Effect |
|---|---|
| `comparisons_correction` | `Bonferroni`, `Holm-Bonferroni`, `Benjamini-Hochberg`, `Benjamini-Yekutieli` (aliases `bonf`, `HB`, `BH`/`fdr_bh`, `BY`/`fdr_by`). Needs `statsmodels`. |
| `alpha` | Significance level used by the correction. Default `0.05`. |
| `text_format` | `star` (`***`), `simple` (`p ≤ 0.001`), `full` (test name + p). |
| `pvalue_thresholds` | Custom cutoff → symbol table, e.g. `[[1e-3, "***"], [1e-2, "**"], [0.05, "*"], [1, "ns"]]`. |
| `color`, `line_width`, `text_offset`, `fontsize` | Bracket and label styling. |

The remaining names come straight from statannotations' `PValueFormat`:
`show_test_name`, `pvalue_format_string`, `simple_format_string`,
`correction_format`, `p_capitalized`, `p_separators`.

The correction covers **every bracket drawn on the plot as one family** — a
split canvas is one family of tests, not one per group. With `pairs="hue"` that
is one test per category, so correcting matters.

With a correction applied, a label reading `* (ns)` is not a bug: the raw
p-value was significant and the corrected one is not. `correction_format`
controls that suffix.

### Behaviour worth knowing

- Brackets are attached to categories, not positions, so `group_cols`/`group_rows`
  and clustering carry them along with the data.
- A pair whose sides land in **different groups** is bracketed across both axes,
  drawn in figure coordinates above the within-group brackets it passes over.
- A plotter on the left or right is drawn horizontally and the brackets follow;
  on the left they sit on the outer side, away from the main canvas.
- `mp.Strip`, `mp.Swarm` and `mp.Point` draw hue levels **on top of each other**
  unless given `dodge=True`. Comparisons between overlaid levels have nothing to
  point at, so they are skipped with a warning — pass `dodge=True`.
- A pair naming a category that is not in the data is skipped with a warning.

## Other common plotters

```python
mp.Arc(anchors, links, weights=None, width=None, colors=None, labels=None,
       legend_kws=None, label=None)                      # plotter/arc.py
# Arc / chord diagram: `links` are index pairs among `anchors`.

mp.Area(data, color=None, add_outline=True, alpha=0.4, linecolor=None,
        linewidth=1, group_kws=None, label=None)          # plotter/area.py
# Filled area plot aligned to the main axis.

mp.Range(data, items=None, marker='o', markersize=50, color1='#F75940',
         color2='#3DC7BE', linecolor='black', linewidth=1, label=None)  # plotter/range.py
# Dumbbell / range plot (two endpoints per item).

mp.SeqLogo(matrix, width=0.9, color_encode=None, stack='descending', **kwargs)  # plotter/bio.py
# Sequence logo; `matrix` is a pandas DataFrame (positions x letters).

mp.Image(images, align='center', scale=1, spacing=0.1, resize=None)  # plotter/images.py
# One image per row/column, as an aligned strip. `images` are file paths, URLs,
# or numpy arrays. `spacing` is 0-1, relative to the image container.

mp.Emoji(codes, lang='en', scale=1, spacing=0.1, **kwargs)          # plotter/images.py
# One twemoji per row/column, from unicode ("😆😆🤣") or short codes.
```
