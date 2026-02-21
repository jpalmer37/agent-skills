# Dash Callback Patterns

## Basic Single Input → Single Output

```python
from dash import callback, Input, Output
from dash.exceptions import PreventUpdate

@callback(
    Output("chart", "figure"),
    Input("dropdown", "value"),
)
def update_chart(selected_value):
    if not selected_value:
        raise PreventUpdate
    fig = px.bar(df[df["category"] == selected_value], x="x", y="y",
                 template="plotly_white")
    return fig
```

## Multiple Inputs → Multiple Outputs

```python
@callback(
    Output("chart-a", "figure"),
    Output("chart-b", "figure"),
    Output("summary-text", "children"),
    Input("dropdown", "value"),
    Input("date-range", "start_date"),
    Input("date-range", "end_date"),
)
def update_all(category, start, end):
    filtered = df[
        (df["category"] == category) &
        (df["date"] >= start) &
        (df["date"] <= end)
    ]
    fig_a = px.line(filtered, x="date", y="value", template="plotly_white")
    fig_b = px.histogram(filtered, x="value", template="plotly_white")
    summary = f"{len(filtered):,} rows | avg: {filtered['value'].mean():.2f}"
    return fig_a, fig_b, summary
```

## Chained Callbacks (dependent dropdowns)

```python
# Callback 1: update sub-category options based on category
@callback(
    Output("sub-dropdown", "options"),
    Output("sub-dropdown", "value"),
    Input("category-dropdown", "value"),
)
def update_sub_options(category):
    if not category:
        raise PreventUpdate
    subs = df[df["category"] == category]["subcategory"].unique()
    options = [{"label": s, "value": s} for s in sorted(subs)]
    return options, options[0]["value"] if options else None

# Callback 2: use both to filter data
@callback(
    Output("chart", "figure"),
    Input("category-dropdown", "value"),
    Input("sub-dropdown", "value"),
)
def update_chart(category, sub):
    filtered = df[(df["category"] == category) & (df["subcategory"] == sub)]
    return px.bar(filtered, x="x", y="y", template="plotly_white")
```

## Loading State

```python
# Wrap any output in dcc.Loading to show a spinner
dcc.Loading(
    id="loading",
    type="circle",          # "circle", "dot", "cube", "default"
    children=dcc.Graph(id="chart"),
)
```

## State (read without triggering)

```python
from dash import State

@callback(
    Output("output", "children"),
    Input("submit-btn", "n_clicks"),     # triggers callback
    State("text-input", "value"),        # read but does NOT trigger
)
def on_submit(n_clicks, text_value):
    if not n_clicks:
        raise PreventUpdate
    return f"Submitted: {text_value}"
```

## Sharing Data Between Callbacks via dcc.Store

```python
# In layout
dcc.Store(id="shared-data", storage_type="session")   # or "memory" or "local"

# Callback writes to store
@callback(Output("shared-data", "data"), Input("dropdown", "value"))
def cache_data(value):
    filtered = df[df["category"] == value]
    return filtered.to_dict("records")   # must be JSON-serializable

# Multiple callbacks read from store (no repeated data fetching)
@callback(Output("chart-a", "figure"), Input("shared-data", "data"))
def chart_a(records):
    filtered = pd.DataFrame(records)
    return px.line(filtered, x="date", y="value")

@callback(Output("chart-b", "figure"), Input("shared-data", "data"))
def chart_b(records):
    filtered = pd.DataFrame(records)
    return px.histogram(filtered, x="value")
```

## Clientside Callback (JavaScript, no Python round-trip)

```python
# For lightweight UI operations (theme toggle, show/hide, etc.)
from dash import clientside_callback

clientside_callback(
    """
    function(n_clicks, current_style) {
        if (n_clicks % 2 === 0) {
            return {"display": "block"};
        } else {
            return {"display": "none"};
        }
    }
    """,
    Output("panel", "style"),
    Input("toggle-btn", "n_clicks"),
    State("panel", "style"),
)
```

## Triggered Input Detection

```python
from dash import ctx

@callback(
    Output("output", "children"),
    Input("btn-a", "n_clicks"),
    Input("btn-b", "n_clicks"),
)
def on_any_button(a, b):
    if not ctx.triggered_id:
        raise PreventUpdate
    if ctx.triggered_id == "btn-a":
        return "Button A clicked"
    return "Button B clicked"
```

## Pattern-Matching Callbacks (dynamic components)

```python
from dash import ALL, MATCH

# Layout creates components with dict ids:
# dcc.Graph(id={"type": "chart", "index": i})

@callback(
    Output({"type": "chart", "index": MATCH}, "figure"),
    Input({"type": "dropdown", "index": MATCH}, "value"),
)
def update_dynamic_chart(value):
    return px.bar(df[df["cat"] == value], x="x", y="y")

# ALL: fires once with a list of all matching values
@callback(
    Output("summary", "children"),
    Input({"type": "dropdown", "index": ALL}, "value"),
)
def summarize_all(values):
    return f"Selected: {', '.join(str(v) for v in values if v)}"
```
