# Figure–Dash Integration Deep Dive

## dcc.Graph Config Options

Control the Plotly mode bar and interaction behavior:

```python
dcc.Graph(
    id="chart",
    figure=fig,
    config={
        "displayModeBar": True,         # show the toolbar
        "displaylogo": False,           # hide Plotly logo
        "modeBarButtonsToRemove": [
            "lasso2d", "select2d",      # remove selection tools
            "pan2d", "zoom2d",          # remove pan/zoom
            "autoScale2d",
            "hoverClosestCartesian",
            "hoverCompareCartesian",
        ],
        "modeBarButtonsToAdd": [        # add custom buttons
            "drawline", "drawrect", "eraseshape",
        ],
        "toImageButtonOptions": {
            "format": "svg",            # "png", "jpeg", "svg", "webp"
            "filename": "dashboard_export",
            "height": 600,
            "width": 1000,
            "scale": 2,
        },
        "scrollZoom": False,            # disable scroll-to-zoom
        "doubleClick": "reset",         # "reset", "autosize", False
        "responsive": True,             # resize with container
    },
    style={"height": "100%"},
)
```

## Responsive Figures in Dash

Ensure charts resize with their container:

```python
# 1. Set responsive config
dcc.Graph(id="chart", config={"responsive": True},
          style={"height": "100%", "minHeight": "350px"})

# 2. In the figure layout, use autosize
fig.update_layout(autosize=True, height=None)

# 3. Wrap in a fixed-height container
dbc.Card(
    dcc.Graph(id="chart", config={"responsive": True}),
    style={"height": "450px"},
    className="shadow-sm border-0",
)
```

## Working with customdata

`customdata` carries arbitrary row-level data through figure events without affecting the visual representation.

```python
import plotly.express as px

# Plotly Express — use custom_data parameter (list of column names)
fig = px.scatter(
    df, x="revenue", y="profit", color="segment",
    custom_data=["company_id", "company_name", "region"],
    template="plotly_white",
)

# Graph Objects — set customdata on each trace
import plotly.graph_objects as go
fig = go.Figure(go.Scatter(
    x=df["revenue"], y=df["profit"],
    customdata=df[["company_id", "company_name", "region"]].values,
    hovertemplate=(
        "<b>%{customdata[1]}</b><br>"
        "Revenue: $%{x:,.0f}<br>"
        "Profit: $%{y:,.0f}<br>"
        "Region: %{customdata[2]}"
        "<extra></extra>"
    ),
))
```

**Accessing customdata in callbacks:**
```python
@callback(Output("detail", "children"), Input("chart", "clickData"))
def show_detail(click_data):
    if not click_data:
        raise PreventUpdate
    pt = click_data["points"][0]
    company_id   = pt["customdata"][0]
    company_name = pt["customdata"][1]
    region       = pt["customdata"][2]
    return html.Div([
        html.H5(company_name),
        html.P(f"ID: {company_id} | Region: {region}"),
    ])
```

## Figure Events Reference

### clickData

Fires when a user clicks a data point.

```python
@callback(Output("output", "children"), Input("chart", "clickData"))
def on_click(data):
    # data = {"points": [{"curveNumber": 0, "pointNumber": 3, "pointIndex": 3,
    #          "x": ..., "y": ..., "customdata": [...]}]}
    ...
```

### hoverData

Fires when a user hovers over a data point. Useful for live detail panels.

```python
@callback(Output("hover-info", "children"), Input("chart", "hoverData"))
def on_hover(data):
    if not data:
        return "Hover over a point to see details"
    pt = data["points"][0]
    return f"Hovering: {pt['x']}, {pt['y']}"
```

### selectedData

Fires when the user uses box-select or lasso-select on the chart.

```python
@callback(Output("selection-count", "children"), Input("chart", "selectedData"))
def on_select(data):
    if not data or not data.get("points"):
        return "No selection"
    n = len(data["points"])
    return f"{n} points selected"
```

### relayoutData

Fires on zoom, pan, axis range change, or autorange reset.

```python
@callback(Output("zoom-info", "children"), Input("chart", "relayoutData"))
def on_zoom(relayout):
    if not relayout:
        raise PreventUpdate
    # Zoom event
    if "xaxis.range[0]" in relayout:
        return f"Zoomed: {relayout['xaxis.range[0]']} to {relayout['xaxis.range[1]']}"
    # Reset event
    if "xaxis.autorange" in relayout:
        return "Reset to full range"
    raise PreventUpdate
```

## Figure Animations in Dash

Animate transitions when figure data changes:

```python
fig.update_layout(
    transition={
        "duration": 500,
        "easing": "cubic-in-out",
    }
)
```

For smooth bar/line transitions, keep trace order and length consistent across updates so Plotly can interpolate between states.

**Animated scatter with slider (fully in Dash):**
```python
fig = px.scatter(
    df, x="gdp", y="life_exp",
    animation_frame="year", animation_group="country",
    size="population", color="continent",
    size_max=60, range_x=[100, 100000], range_y=[25, 90],
    log_x=True, template="plotly_white",
)
# Embed in Dash — the animation controls render automatically
dcc.Graph(id="animated-chart", figure=fig)
```

## Updating Figure Annotations from Callbacks

```python
@callback(Output("chart", "figure"), Input("annotation-input", "value"),
          State("chart", "figure"))
def add_annotation(text, current_fig):
    if not text:
        raise PreventUpdate
    fig = go.Figure(current_fig)
    fig.add_annotation(
        x=target_x, y=target_y, text=text,
        showarrow=True, arrowhead=2, ax=30, ay=-40,
        font=dict(size=12, color="#E45756"),
    )
    return fig
```

## Disabling Interactions Per Chart

```python
# Read-only chart (no hover, no click, no select)
dcc.Graph(
    id="static-kpi-sparkline",
    figure=fig,
    config={"staticPlot": True},   # completely non-interactive
    style={"height": "80px"},
)

# Interactive but no selection tools
dcc.Graph(
    id="browse-only-chart",
    figure=fig,
    config={
        "modeBarButtonsToRemove": ["select2d", "lasso2d"],
        "displayModeBar": "hover",
    },
)
```
