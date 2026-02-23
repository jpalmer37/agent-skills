---
name: plotly-dash
description: Building impactful, clean, and elegant interactive dashboards by combining Plotly figures with Dash reactivity. Use when users want to create coordinated multi-chart dashboards, cross-filtered views, figure-driven interactions (click, hover, select), or need guidance on wiring Plotly visualizations into a Dash application. Complements the standalone plotly skill by focusing on how figures live inside Dash.
---

# Plotly + Dash Dashboard Integration

This skill focuses on the **interaction layer** between Plotly figures and Dash — how figures become reactive, how user interactions on one chart drive updates elsewhere, and how to compose polished, production-grade dashboards.

> **Prerequisite skills:** Use the **plotly** skill for chart creation recipes and the **py-dashboards** skill for core Dash layout/callback fundamentals.

## The dcc.Graph Component

`dcc.Graph` is the bridge between Plotly and Dash. It renders a Plotly figure and exposes interactive properties as callback inputs.

```python
dcc.Graph(
    id="main-chart",
    figure=fig,                     # any plotly.graph_objects.Figure
    config={
        "displayModeBar": True,
        "modeBarButtonsToRemove": ["lasso2d", "select2d"],
        "toImageButtonOptions": {"format": "svg", "filename": "chart"},
        "scrollZoom": False,
    },
    style={"height": "450px"},
    className="shadow-sm rounded-3",
)
```

**Key `dcc.Graph` properties for callbacks:**

| Property | Direction | Description |
|---|---|---|
| `figure` | Output | The Plotly figure object to render |
| `clickData` | Input | Data from a point click event |
| `hoverData` | Input | Data from a point hover event |
| `selectedData` | Input | Data from box-select or lasso-select |
| `relayoutData` | Input | Axis range changes, zoom, pan events |

## Figure Event Handling

Chart interactions are first-class Dash inputs. These enable drill-down, cross-filtering, and coordinated views.

```python
from dash import callback, Input, Output, State
from dash.exceptions import PreventUpdate
import plotly.express as px

@callback(
    Output("detail-panel", "children"),
    Input("main-chart", "clickData"),
)
def on_point_click(click_data):
    if not click_data:
        raise PreventUpdate
    point = click_data["points"][0]
    x_val, y_val = point["x"], point["y"]
    # Access custom data if set via customdata= in the figure
    custom = point.get("customdata", [])
    return f"Clicked: x={x_val}, y={y_val}"
```

**Event data shapes:**
```python
# clickData / hoverData
{"points": [{"curveNumber": 0, "pointNumber": 5, "x": "2024-03", "y": 42,
              "customdata": ["row_id_123"]}]}

# selectedData (box select / lasso)
{"points": [...], "range": {"x": [x0, x1], "y": [y0, y1]}}

# relayoutData (zoom / pan)
{"xaxis.range[0]": "2024-01-01", "xaxis.range[1]": "2024-06-01",
 "yaxis.range[0]": 0, "yaxis.range[1]": 100}
# or on autorange reset:
{"xaxis.autorange": True}
```

**Passing row identifiers through figures using `customdata`:**
```python
fig = px.scatter(
    df, x="x", y="y", color="category",
    custom_data=["id", "region"],       # columns carried through events
    template="plotly_white",
)
# In callback: point["customdata"][0] → id, point["customdata"][1] → region
```

## Cross-Filtering Pattern

The signature dashboard pattern: selecting data in one chart filters all others.

See **[references/dashboard-patterns.md](references/dashboard-patterns.md)** for the full cross-filtering recipe, including:
- Two-chart and multi-chart coordinated filtering
- Highlighting selected vs. unselected points with opacity
- Bidirectional cross-filter with `dcc.Store`
- Reset / clear selection behavior

**Minimal example:**
```python
@callback(
    Output("chart-b", "figure"),
    Input("chart-a", "selectedData"),
    State("shared-data", "data"),
)
def cross_filter(selected, records):
    dff = pd.DataFrame(records)
    if selected and selected.get("points"):
        ids = [p["customdata"][0] for p in selected["points"]]
        dff = dff[dff["id"].isin(ids)]
    return px.histogram(dff, x="value", template="plotly_white")
```

## Coordinated Zoom (Shared Axis Range)

Synchronize the x-axis across multiple time-series charts:

```python
@callback(
    Output("chart-b", "figure"),
    Input("chart-a", "relayoutData"),
    State("chart-b", "figure"),
)
def sync_zoom(relayout, current_fig):
    if not relayout or "xaxis.autorange" in relayout:
        raise PreventUpdate
    x0 = relayout.get("xaxis.range[0]")
    x1 = relayout.get("xaxis.range[1]")
    if x0 and x1:
        current_fig["layout"]["xaxis"]["range"] = [x0, x1]
        current_fig["layout"]["xaxis"]["autorange"] = False
    return current_fig
```

## Partial Figure Updates with Patch

Use `dash.Patch()` for surgical figure updates without rebuilding the entire figure — dramatically faster for large datasets.

```python
from dash import Patch

@callback(
    Output("main-chart", "figure"),
    Input("highlight-dropdown", "value"),
)
def highlight_trace(selected_trace):
    patched = Patch()
    # Update opacity of all traces
    for i in range(num_traces):
        patched["data"][i]["opacity"] = 1.0 if i == selected_trace else 0.2
    return patched

# Common Patch operations:
# patched["layout"]["title"]["text"] = "New Title"
# patched["data"][0]["marker"]["color"] = "red"
# patched["layout"]["annotations"].append(new_annotation)
```

## Dashboard Layout Composition

**Standard analytical dashboard structure:**
```python
import dash_bootstrap_components as dbc
from dash import dcc, html

def make_dashboard_layout(title):
    return dbc.Container([
        # ── Header ──
        dbc.Row([
            dbc.Col(html.H2(title, className="fw-bold"), width="auto"),
            dbc.Col(filter_controls(), width="auto", className="ms-auto"),
        ], align="center", className="mb-3 pt-3"),

        # ── KPI Row ──
        dbc.Row(id="kpi-row", className="mb-4"),

        # ── Primary Chart ──
        dbc.Row([
            dbc.Col(dbc.Card(dcc.Graph(id="primary-chart"),
                             className="shadow-sm border-0 p-2"), width=12),
        ], className="mb-4"),

        # ── Secondary Charts ──
        dbc.Row([
            dbc.Col(dbc.Card(dcc.Graph(id="chart-left"),
                             className="shadow-sm border-0 p-2"), width=6),
            dbc.Col(dbc.Card(dcc.Graph(id="chart-right"),
                             className="shadow-sm border-0 p-2"), width=6),
        ], className="mb-4"),

        # ── Data Table ──
        dbc.Row([
            dbc.Col(dbc.Card(html.Div(id="data-table"),
                             className="shadow-sm border-0 p-3"), width=12),
        ]),
    ], fluid=True)
```

## Consistent Figure Styling

Create a reusable figure factory to enforce visual consistency across all dashboard charts:

```python
DASH_COLORS = ["#4C78A8", "#F58518", "#E45756", "#72B7B2",
               "#54A24B", "#EECA3B", "#B279A2", "#FF9DA6"]

FIGURE_DEFAULTS = dict(
    template="plotly_white",
    font_family="Inter, Arial, sans-serif",
    font_size=12,
    title_font_size=15,
    margin=dict(l=40, r=20, t=50, b=40),
    plot_bgcolor="white",
    paper_bgcolor="white",
    colorway=DASH_COLORS,
)

def styled_figure(fig, height=400):
    """Apply dashboard-wide styling to any Plotly figure."""
    fig.update_layout(**FIGURE_DEFAULTS, height=height)
    fig.update_xaxes(showgrid=True, gridcolor="#EEEEEE", zeroline=False)
    fig.update_yaxes(showgrid=True, gridcolor="#EEEEEE", zeroline=False)
    fig.update_traces(
        hovertemplate=fig.data[0].hovertemplate or None,
    )
    return fig
```

## Theme Synchronization (Dash Bootstrap ↔ Plotly)

Match Plotly figure themes to the active Dash Bootstrap theme:

```python
# Map Bootstrap themes to Plotly templates
THEME_MAP = {
    "BOOTSTRAP": "plotly_white",
    "FLATLY":    "plotly_white",
    "LUX":       "plotly_white",
    "DARKLY":    "plotly_dark",
    "CYBORG":    "plotly_dark",
    "SLATE":     "plotly_dark",
}

def get_plotly_template(bootstrap_theme="BOOTSTRAP"):
    return THEME_MAP.get(bootstrap_theme, "plotly_white")
```

For dynamic theme switching, use `dash-bootstrap-templates`:
```python
# pip install dash-bootstrap-templates
from dash_bootstrap_templates import load_figure_template
load_figure_template("flatly")  # applies matching Plotly theme globally
```

## Reference Files

See the reference files for detailed patterns and complete examples:
- **[references/figure-dash-integration.md](references/figure-dash-integration.md)** — dcc.Graph deep dive, figure events, customdata, config options, animation in Dash
- **[references/dashboard-patterns.md](references/dashboard-patterns.md)** — Cross-filtering, coordinated views, master-detail, KPI dashboards, sidebar filter panels
- **[references/performance-theming.md](references/performance-theming.md)** — Patch updates, background callbacks, caching, theme sync, responsive design
