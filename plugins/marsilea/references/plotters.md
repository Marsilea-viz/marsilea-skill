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
- Other (arc, area, range, sequence logo)

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
             edgecolor=None, linewidth=1, marker='o', palette=None,
             size_legend_kws=None, color_legend_kws=None)
# Dot matrix: dot size encodes `size` matrix, color encodes `color` matrix.
# Wrapped by ma.SizedHeatmap. Set color="none" + edgecolor for open circles.

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
```
