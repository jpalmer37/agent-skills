# Performance & Theming

## Partial Figure Updates with Patch

`dash.Patch()` modifies specific parts of a figure without reconstructing it. Critical for dashboards with large datasets or many traces.

### Basic Patch Operations

```python
from dash import Patch, callback, Input, Output

@callback(Output("chart", "figure"), Input("color-picker", "value"))
def change_trace_color(color):
    patched = Patch()
    patched["data"][0]["marker"]["color"] = color
    return patched
```

### Common Patch Recipes

```python
# Update chart title
patched["layout"]["title"]["text"] = "New Title"

# Update axis range
patched["layout"]["xaxis"]["range"] = [0, 100]

# Toggle grid visibility
patched["layout"]["xaxis"]["showgrid"] = False

# Update trace data (e.g., extend a line chart)
patched["data"][0]["x"].append(new_x)
patched["data"][0]["y"].append(new_y)

# Change bar colors by index
patched["data"][0]["marker"]["color"] = ["red" if i == selected else "blue"
                                          for i in range(n_bars)]

# Add an annotation
patched["layout"]["annotations"].append({
    "x": x_pos, "y": y_pos, "text": "Note",
    "showarrow": True, "arrowhead": 2,
})

# Clear all annotations
patched["layout"]["annotations"] = []
```

### Highlight Selected Bar with Patch

```python
@callback(Output("bar-chart", "figure"), Input("bar-chart", "clickData"))
def highlight_bar(click_data):
    patched = Patch()
    if not click_data:
        patched["data"][0]["marker"]["color"] = default_colors
        return patched
    idx = click_data["points"][0]["pointIndex"]
    colors = ["#4C78A8" if i != idx else "#E45756" for i in range(n_bars)]
    patched["data"][0]["marker"]["color"] = colors
    return patched
```

## Background Callbacks

For long-running computations that would otherwise freeze the UI:

```python
from dash import callback, Input, Output
import dash

@callback(
    Output("result", "children"),
    Input("run-btn", "n_clicks"),
    background=True,                     # runs in a background worker
    running=[
        (Output("run-btn", "disabled"), True, False),         # disable button while running
        (Output("progress-bar", "style"), {"visibility": "visible"},
                                          {"visibility": "hidden"}),
    ],
    progress=[Output("progress-bar", "value"), Output("progress-bar", "max")],
    prevent_initial_call=True,
)
def long_computation(set_progress, n_clicks):
    total = 100
    for i in range(total):
        # ... expensive computation step ...
        set_progress((i + 1, total))
    return "Done!"
```

**Requires a background callback manager:**
```python
from dash import DiskcacheManager
import diskcache

cache = diskcache.Cache("./cache")
background_callback_manager = DiskcacheManager(cache)

app = dash.Dash(__name__, background_callback_manager=background_callback_manager)
```

Install: `pip install diskcache`

## Server-Side Caching

Cache expensive data transformations to avoid recomputation:

```python
from flask_caching import Cache

cache = Cache(config={"CACHE_TYPE": "SimpleCache", "CACHE_DEFAULT_TIMEOUT": 300})
cache.init_app(app.server)

@cache.memoize(timeout=600)
def get_filtered_data(category, start_date, end_date):
    """Cached data query — only recomputes when arguments change."""
    dff = df[(df["category"] == category) &
             (df["date"] >= start_date) &
             (df["date"] <= end_date)]
    return dff.to_dict("records")

@callback(
    Output("chart", "figure"),
    Input("category", "value"),
    Input("dates", "start_date"),
    Input("dates", "end_date"),
)
def update_chart(category, start, end):
    records = get_filtered_data(category, start, end)  # cached
    dff = pd.DataFrame(records)
    return px.line(dff, x="date", y="value", template="plotly_white")
```

Install: `pip install Flask-Caching`

## Efficient Data Flow with dcc.Store

Avoid re-querying data by caching filtered results in `dcc.Store`:

```python
# Step 1: One callback filters data and writes to Store
@callback(Output("filtered-store", "data"), Input("dropdown", "value"))
def filter_data(value):
    return df[df["cat"] == value].to_dict("records")

# Step 2: Multiple chart callbacks read from Store (no repeated queries)
@callback(Output("chart-a", "figure"), Input("filtered-store", "data"))
def chart_a(records):
    return px.line(pd.DataFrame(records), x="date", y="value",
                   template="plotly_white")

@callback(Output("chart-b", "figure"), Input("filtered-store", "data"))
def chart_b(records):
    return px.histogram(pd.DataFrame(records), x="value",
                        template="plotly_white")

@callback(Output("chart-c", "figure"), Input("filtered-store", "data"))
def chart_c(records):
    return px.box(pd.DataFrame(records), y="value",
                  template="plotly_white")
```

**Storage types:**
| Type | Persistence | Size Limit | Use Case |
|---|---|---|---|
| `"memory"` | Cleared on page refresh | Browser memory | Default, most common |
| `"session"` | Survives refresh, cleared on tab close | ~5 MB | Multi-step workflows |
| `"local"` | Persists across sessions | ~5 MB | User preferences |

## Theming: Dash Bootstrap ↔ Plotly Consistency

### Using dash-bootstrap-templates

The easiest way to sync Plotly figure styling with your Bootstrap theme:

```python
# pip install dash-bootstrap-templates
from dash_bootstrap_templates import load_figure_template

# Load one or more themes — figures auto-match the Dash theme
load_figure_template(["flatly", "darkly"])

app = dash.Dash(__name__, external_stylesheets=[dbc.themes.FLATLY])
```

Once loaded, all `px.*` and `go.*` figures automatically pick up the matching template.

### Manual Theme Map

For finer control without the extra package:

```python
DARK_THEMES = {"DARKLY", "CYBORG", "SLATE", "VAPOR", "SOLAR"}

def figure_theme(bootstrap_theme_name):
    if bootstrap_theme_name in DARK_THEMES:
        return dict(
            template="plotly_dark",
            paper_bgcolor="rgba(0,0,0,0)",
            plot_bgcolor="rgba(0,0,0,0)",
            font_color="#EEEEEE",
        )
    return dict(
        template="plotly_white",
        paper_bgcolor="white",
        plot_bgcolor="white",
        font_color="#2C3E50",
    )

# Usage
fig.update_layout(**figure_theme("FLATLY"))
```

### Dynamic Theme Switching

Allow users to toggle light/dark mode:

```python
app.layout = html.Div([
    dbc.Switch(id="theme-switch", label="Dark Mode", value=False),
    dcc.Graph(id="themed-chart"),
])

@callback(
    Output("themed-chart", "figure"),
    Input("theme-switch", "value"),
)
def toggle_theme(is_dark):
    template = "plotly_dark" if is_dark else "plotly_white"
    bg = "rgba(0,0,0,0)" if is_dark else "white"
    fig = px.bar(df, x="cat", y="val", template=template)
    fig.update_layout(paper_bgcolor=bg, plot_bgcolor=bg)
    return fig
```

**Full app-wide dark mode** (switches both Bootstrap theme and Plotly template):
```python
# Requires clientside callback for instant Bootstrap theme swap
from dash import clientside_callback

clientside_callback(
    """
    function(isDark) {
        const link = document.querySelector('link[rel="stylesheet"]');
        link.href = isDark
            ? 'https://cdn.jsdelivr.net/npm/bootswatch@5/dist/darkly/bootstrap.min.css'
            : 'https://cdn.jsdelivr.net/npm/bootswatch@5/dist/flatly/bootstrap.min.css';
        return window.dash_clientside.no_update;
    }
    """,
    Output("theme-switch", "id"),   # dummy output
    Input("theme-switch", "value"),
)
```

## Consistent Color Palette Across Charts

Define once, use everywhere:

```python
PALETTE = {
    "primary":   "#4C78A8",
    "secondary": "#F58518",
    "danger":    "#E45756",
    "success":   "#54A24B",
    "accent1":   "#72B7B2",
    "accent2":   "#EECA3B",
    "accent3":   "#B279A2",
    "accent4":   "#FF9DA6",
}

COLORWAY = list(PALETTE.values())

# Apply to all figures
def apply_palette(fig):
    fig.update_layout(colorway=COLORWAY)
    return fig

# Or set globally
import plotly.io as pio
pio.templates["dashboard"] = go.layout.Template(
    layout=dict(
        colorway=COLORWAY,
        font=dict(family="Inter, Arial, sans-serif", size=12),
        paper_bgcolor="white",
        plot_bgcolor="white",
        title=dict(font=dict(size=15)),
        xaxis=dict(showgrid=True, gridcolor="#EEEEEE"),
        yaxis=dict(showgrid=True, gridcolor="#EEEEEE"),
    )
)
pio.templates.default = "plotly_white+dashboard"  # layer on top of plotly_white
```

## Responsive Grid Breakpoints

Ensure dashboard looks good on all screen sizes:

```python
# Use dbc.Col responsive breakpoints
dbc.Row([
    dbc.Col(dcc.Graph(id="chart-a"), xs=12, md=6, xl=4),  # full → half → third
    dbc.Col(dcc.Graph(id="chart-b"), xs=12, md=6, xl=4),
    dbc.Col(dcc.Graph(id="chart-c"), xs=12, md=12, xl=4),
])

# KPI cards that stack on mobile
dbc.Row([
    dbc.Col(kpi_card, xs=6, sm=6, md=3) for kpi_card in kpi_cards
])
```

**CSS for chart containers:**
```css
/* assets/responsive.css */
.dash-graph {
    min-height: 300px;
}

@media (max-width: 768px) {
    .dash-graph {
        min-height: 250px;
    }
    h2 { font-size: 1.4rem; }
}
```

## Reducing Callback Overhead

```python
# 1. Use prevent_initial_call to skip startup execution
@callback(..., prevent_initial_call=True)

# 2. Guard against empty inputs
from dash.exceptions import PreventUpdate
if not value:
    raise PreventUpdate

# 3. Use ctx.triggered_id to skip irrelevant triggers
from dash import ctx
if ctx.triggered_id != "the-input-that-matters":
    raise PreventUpdate

# 4. Debounce rapid-fire inputs (e.g., text input)
dcc.Input(id="search", debounce=True)   # only fires on Enter or blur

# 5. Clientside callbacks for zero-latency UI operations
from dash import clientside_callback
clientside_callback(
    "function(n) { return n % 2 === 0 ? {'display':'block'} : {'display':'none'}; }",
    Output("panel", "style"),
    Input("toggle", "n_clicks"),
)
```
