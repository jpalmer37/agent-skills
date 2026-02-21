# Plotly Chart Recipes

## Scatter Plot

```python
import plotly.express as px

fig = px.scatter(
    df, x="x_col", y="y_col",
    color="category",           # color by category
    size="magnitude",           # bubble size
    hover_data=["label"],       # extra hover info
    trendline="ols",            # add regression line
    template="plotly_white",
    title="Scatter Plot",
)
fig.show()
```

## Line Chart

```python
fig = px.line(
    df, x="date", y="value",
    color="series",             # multiple lines
    line_dash="group",          # dashed for some series
    markers=True,               # show data points
    template="plotly_white",
    title="Time Series",
)
# Range slider for time series
fig.update_xaxes(rangeslider_visible=True)
```

## Bar Chart

```python
# Grouped
fig = px.bar(df, x="category", y="value", color="group",
             barmode="group", template="plotly_white")

# Stacked
fig = px.bar(df, x="category", y="value", color="group",
             barmode="stack", template="plotly_white")

# Horizontal
fig = px.bar(df, x="value", y="category", orientation="h",
             template="plotly_white")

# Sort bars by value
fig.update_layout(xaxis={"categoryorder": "total descending"})
```

## Histogram

```python
fig = px.histogram(
    df, x="col",
    nbins=30,
    color="group",              # overlay groups
    barmode="overlay",          # or "stack"
    opacity=0.7,
    marginal="box",             # add marginal box plot
    template="plotly_white",
)
```

## Box / Violin Plot

```python
# Box plot with jittered points
fig = px.box(df, x="group", y="value", points="all",
             template="plotly_white")

# Violin
fig = px.violin(df, x="group", y="value", box=True,
                points="all", template="plotly_white")
```

## Heatmap

```python
import plotly.graph_objects as go

fig = go.Figure(data=go.Heatmap(
    z=matrix,           # 2D array or DataFrame.values
    x=col_labels,
    y=row_labels,
    colorscale="RdBu_r",
    zmid=0,             # center colorscale at 0
    text=matrix,        # show values in cells
    texttemplate="%{text:.2f}",
))
fig.update_layout(title="Heatmap", template="plotly_white")
```

**Correlation matrix shortcut:**
```python
import plotly.express as px
fig = px.imshow(df.corr(), text_auto=".2f", aspect="auto",
                color_continuous_scale="RdBu_r", color_continuous_midpoint=0)
```

## Subplots

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go

fig = make_subplots(
    rows=2, cols=2,
    subplot_titles=("Plot A", "Plot B", "Plot C", "Plot D"),
    shared_xaxes=False,
    vertical_spacing=0.12,
)

fig.add_trace(go.Scatter(x=x1, y=y1, name="A"), row=1, col=1)
fig.add_trace(go.Bar(x=cats, y=vals, name="B"), row=1, col=2)
# ...

fig.update_layout(height=700, title="Dashboard Grid", template="plotly_white")
```

## Animated Chart

```python
fig = px.scatter(
    df, x="x", y="y",
    animation_frame="year",     # column that drives animation
    animation_group="country",  # track identity across frames
    size="population",
    color="continent",
    range_x=[0, 100], range_y=[0, 100],
    template="plotly_white",
)
fig.update_layout(transition_duration=500)
```

## Pie / Donut Chart

```python
fig = px.pie(df, names="label", values="count",
             hole=0.4,          # set >0 for donut
             template="plotly_white")
fig.update_traces(textposition="inside", textinfo="percent+label")
```

## Facet Grid

```python
fig = px.scatter(df, x="x", y="y",
                 facet_col="category",   # separate panel per column value
                 facet_row="group",      # separate panel per row value
                 template="plotly_white")
fig.for_each_annotation(lambda a: a.update(text=a.text.split("=")[-1]))
```

## Map (Choropleth)

```python
fig = px.choropleth(df, locations="iso_alpha", color="value",
                    hover_name="country",
                    color_continuous_scale="Viridis",
                    projection="natural earth")
```

## Custom Color Palette

```python
MY_COLORS = ["#4C78A8", "#F58518", "#E45756", "#72B7B2", "#54A24B"]

fig = px.bar(df, x="cat", y="val",
             color="cat",
             color_discrete_sequence=MY_COLORS,
             template="plotly_white")
```

## Annotations & Reference Lines

```python
# Horizontal reference line
fig.add_hline(y=threshold, line_dash="dash", line_color="red",
              annotation_text="Threshold", annotation_position="top right")

# Vertical reference line
fig.add_vline(x="2024-01-01", line_dash="dot", line_color="gray")

# Shaded region
fig.add_vrect(x0="2023-06-01", x1="2023-09-01",
              fillcolor="yellow", opacity=0.15, line_width=0)

# Text annotation
fig.add_annotation(x=x_pos, y=y_pos, text="Peak", showarrow=True,
                   arrowhead=2, ax=20, ay=-40)
```
