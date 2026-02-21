---
name: py-plotting
description: Creating intuitive and beautiful data visualizations in Python using Plotly and matplotlib. Use when users ask to plot, chart, graph, or visualize data — including scatter plots, line charts, bar charts, histograms, heatmaps, box plots, and more. Applies to both interactive web-ready figures (Plotly) and publication-quality static images (matplotlib).
---

# Python Plotting

## Library Decision

| Use **Plotly** when | Use **matplotlib** when |
|---|---|
| Interactivity is needed (hover, zoom, pan) | Publication-quality static output (paper, PDF) |
| Output is a web page or notebook | Fine-grained control over every element |
| Working with Plotly Express (fast, idiomatic) | Using a seaborn-style statistical chart |
| Building charts for a Dash app | Complex multi-panel scientific figures |

## Quick Start

**Plotly Express (most common):**
```python
import plotly.express as px

fig = px.scatter(df, x="col_a", y="col_b", color="category",
                 title="My Chart", template="plotly_white")
fig.show()                          # interactive in notebook/browser
fig.write_html("chart.html")        # shareable standalone file
fig.write_image("chart.png", scale=2)  # high-res static (requires kaleido)
```

**matplotlib (static):**
```python
import matplotlib.pyplot as plt
import matplotlib as mpl

mpl.rcParams.update({"font.family": "sans-serif", "figure.dpi": 150})

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(x, y, color="#4C78A8", linewidth=2, label="Series A")
ax.set(title="My Chart", xlabel="X", ylabel="Y")
ax.legend(frameon=False)
ax.spines[["top", "right"]].set_visible(False)
plt.tight_layout()
plt.savefig("chart.png", dpi=300, bbox_inches="tight")
```

## Aesthetics Quick Reference

**Plotly templates** (set via `template=`):
- `"plotly_white"` — clean white background (general use)
- `"plotly_dark"` — dark theme (presentations)
- `"seaborn"` — seaborn-inspired style
- `"ggplot2"` — R ggplot2-inspired style
- `"simple_white"` — minimal, publication-friendly

**Plotly color sequences** (set via `color_discrete_sequence=`):
- `px.colors.qualitative.Safe` — colorblind-safe categorical
- `px.colors.qualitative.Plotly` — default Plotly palette
- `px.colors.sequential.Viridis` — sequential (maps, heatmaps)

**matplotlib style shortcuts:**
```python
plt.style.use("seaborn-v0_8-whitegrid")   # clean grid style
plt.style.use("bmh")                        # Bayesian Methods for Hackers style
```

## Plotly Layout Polish

```python
fig.update_layout(
    font_family="Inter, Arial, sans-serif",
    title_font_size=18,
    margin=dict(l=40, r=20, t=60, b=40),
    legend=dict(orientation="h", yanchor="bottom", y=1.02, xanchor="right", x=1),
    plot_bgcolor="white",
    paper_bgcolor="white",
)
fig.update_xaxes(showgrid=True, gridcolor="#EEEEEE", zeroline=False)
fig.update_yaxes(showgrid=True, gridcolor="#EEEEEE", zeroline=False)
```

## Hover Templates (Plotly)

```python
fig.update_traces(
    hovertemplate="<b>%{x}</b><br>Value: %{y:,.0f}<extra></extra>"
)
```

## Common Chart Patterns

See the reference files for detailed patterns and examples:
- **[references/plotly-charts.md](references/plotly-charts.md)** — Plotly chart recipes: scatter, line, bar, histogram, box, heatmap, subplots, animations
- **[references/matplotlib-charts.md](references/matplotlib-charts.md)** — matplotlib chart recipes: line, bar, histogram, heatmap, subplots, twin axes, annotations

## Output & Sharing

| Goal | Method |
|---|---|
| Interactive notebook display | `fig.show()` |
| Standalone shareable HTML | `fig.write_html("out.html")` |
| High-res PNG/SVG/PDF | `fig.write_image("out.png", scale=2)` (install `kaleido`) |
| matplotlib PNG | `plt.savefig("out.png", dpi=300, bbox_inches="tight")` |
| matplotlib PDF (vector) | `plt.savefig("out.pdf", bbox_inches="tight")` |

Install kaleido for Plotly static export: `pip install kaleido`
