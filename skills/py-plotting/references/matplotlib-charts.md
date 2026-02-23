# matplotlib Chart Recipes

## Global Style Setup

```python
import matplotlib.pyplot as plt
import matplotlib as mpl
import numpy as np

# Apply a clean style first, then override specifics
plt.style.use("seaborn-v0_8-whitegrid")

mpl.rcParams.update({
    "font.family": "sans-serif",
    "font.size": 12,
    "axes.titlesize": 14,
    "axes.titleweight": "bold",
    "figure.dpi": 150,
    "axes.spines.top": False,
    "axes.spines.right": False,
})
```

## Line Chart

```python
fig, ax = plt.subplots(figsize=(9, 5))

colors = ["#4C78A8", "#F58518", "#E45756"]
for i, (label, series) in enumerate(data.items()):
    ax.plot(x, series, color=colors[i], linewidth=2.5,
            marker="o", markersize=4, label=label)

ax.set(title="Line Chart", xlabel="Date", ylabel="Value")
ax.legend(frameon=False, loc="upper left")
plt.tight_layout()
plt.savefig("line.png", dpi=300, bbox_inches="tight")
```

## Bar Chart

```python
fig, ax = plt.subplots(figsize=(8, 5))

bars = ax.bar(categories, values, color="#4C78A8", edgecolor="white",
              linewidth=0.8)

# Add value labels on top of bars
for bar, val in zip(bars, values):
    ax.text(bar.get_x() + bar.get_width() / 2, bar.get_height() + 0.5,
            f"{val:,.0f}", ha="center", va="bottom", fontsize=10)

ax.set(title="Bar Chart", xlabel="Category", ylabel="Count")
ax.yaxis.set_major_formatter(mpl.ticker.FuncFormatter(
    lambda x, _: f"{x:,.0f}"))
plt.tight_layout()
```

**Horizontal bar chart:**
```python
ax.barh(categories, values, color="#4C78A8")
ax.invert_yaxis()   # largest bar at top
```

**Grouped bar chart:**
```python
x = np.arange(len(categories))
width = 0.35
ax.bar(x - width/2, vals_a, width, label="Group A", color="#4C78A8")
ax.bar(x + width/2, vals_b, width, label="Group B", color="#F58518")
ax.set_xticks(x)
ax.set_xticklabels(categories)
ax.legend(frameon=False)
```

## Histogram

```python
fig, ax = plt.subplots(figsize=(8, 5))

ax.hist(data, bins=30, color="#4C78A8", edgecolor="white",
        linewidth=0.6, alpha=0.85, density=False)

# Overlay a KDE (requires scipy)
from scipy.stats import gaussian_kde
kde = gaussian_kde(data)
xs = np.linspace(data.min(), data.max(), 200)
ax2 = ax.twinx()
ax2.plot(xs, kde(xs), color="#E45756", linewidth=2)
ax2.set_yticks([])
ax2.spines[["top", "right", "left"]].set_visible(False)

ax.set(title="Distribution", xlabel="Value", ylabel="Count")
plt.tight_layout()
```

## Scatter Plot

```python
fig, ax = plt.subplots(figsize=(7, 6))

scatter = ax.scatter(x, y, c=color_vals, s=size_vals,
                     cmap="viridis", alpha=0.7, edgecolors="white",
                     linewidths=0.5)

cbar = plt.colorbar(scatter, ax=ax, shrink=0.8)
cbar.set_label("Intensity")

ax.set(title="Scatter Plot", xlabel="X", ylabel="Y")
plt.tight_layout()
```

## Heatmap / Confusion Matrix

```python
import matplotlib.pyplot as plt
import seaborn as sns  # easiest heatmap

fig, ax = plt.subplots(figsize=(8, 6))
sns.heatmap(matrix_df, annot=True, fmt=".2f", cmap="RdBu_r",
            center=0, linewidths=0.5, ax=ax)
ax.set_title("Correlation Matrix")
plt.tight_layout()
```

**Pure matplotlib heatmap:**
```python
im = ax.imshow(matrix, cmap="RdBu_r", vmin=-1, vmax=1, aspect="auto")
plt.colorbar(im, ax=ax, shrink=0.8)
ax.set_xticks(range(len(cols)))
ax.set_xticklabels(cols, rotation=45, ha="right")
ax.set_yticks(range(len(rows)))
ax.set_yticklabels(rows)
```

## Subplots

```python
fig, axes = plt.subplots(2, 3, figsize=(14, 8), constrained_layout=True)
fig.suptitle("Overview Dashboard", fontsize=16, fontweight="bold")

axes[0, 0].plot(x, y, color="#4C78A8")
axes[0, 0].set_title("Panel A")

# Flatten for easy iteration
for i, ax in enumerate(axes.flat):
    ax.set_title(f"Panel {i+1}")
    # ...

# Hide unused axes
axes[1, 2].set_visible(False)
```

## Twin Axes (dual y-axis)

```python
fig, ax1 = plt.subplots(figsize=(9, 5))

color_a, color_b = "#4C78A8", "#E45756"
ax1.plot(x, y1, color=color_a, linewidth=2, label="Metric A")
ax1.set_ylabel("Metric A", color=color_a)
ax1.tick_params(axis="y", labelcolor=color_a)

ax2 = ax1.twinx()
ax2.bar(x, y2, color=color_b, alpha=0.3, label="Metric B")
ax2.set_ylabel("Metric B", color=color_b)
ax2.tick_params(axis="y", labelcolor=color_b)

# Combined legend
lines1, labels1 = ax1.get_legend_handles_labels()
lines2, labels2 = ax2.get_legend_handles_labels()
ax1.legend(lines1 + lines2, labels1 + labels2, frameon=False)
plt.tight_layout()
```

## Annotations

```python
# Arrow annotation
ax.annotate("Peak", xy=(x_peak, y_peak), xytext=(x_peak + 2, y_peak + 5),
            arrowprops=dict(arrowstyle="->", color="gray"),
            fontsize=10, color="gray")

# Horizontal reference line
ax.axhline(y=threshold, color="red", linestyle="--", linewidth=1.5,
           label=f"Threshold ({threshold})")

# Shaded region
ax.axvspan(x_start, x_end, alpha=0.1, color="yellow", label="Period")
```

## Box / Violin Plot

```python
# Box plot
fig, ax = plt.subplots(figsize=(8, 5))
ax.boxplot([group_a, group_b, group_c], labels=["A", "B", "C"],
           patch_artist=True,
           boxprops=dict(facecolor="#4C78A8", alpha=0.6),
           medianprops=dict(color="white", linewidth=2))
ax.set(title="Distribution Comparison", ylabel="Value")

# Violin (seaborn is easier)
sns.violinplot(data=df, x="group", y="value", palette="muted", ax=ax)
```

## Saving High-Quality Output

```python
# PNG for screen/web
plt.savefig("chart.png", dpi=300, bbox_inches="tight", facecolor="white")

# PDF/SVG for print (vector)
plt.savefig("chart.pdf", bbox_inches="tight")
plt.savefig("chart.svg", bbox_inches="tight")

# Transparent background
plt.savefig("chart.png", dpi=300, bbox_inches="tight", transparent=True)
```
