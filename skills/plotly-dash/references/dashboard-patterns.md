# Dashboard Patterns

## Cross-Filtering Dashboard

The most impactful interactive pattern: selecting data in one chart filters all other charts.

### Full Two-Chart Cross-Filter

```python
import dash
from dash import dcc, html, callback, Input, Output, State
import dash_bootstrap_components as dbc
import plotly.express as px
import pandas as pd

app = dash.Dash(__name__, external_stylesheets=[dbc.themes.FLATLY])

# Sample data
df = px.data.gapminder().query("year == 2007")

app.layout = dbc.Container([
    dcc.Store(id="store-data", data=df.to_dict("records")),
    dbc.Row([dbc.Col(html.H2("Cross-Filter Dashboard"), width=12)], className="mb-3 pt-3"),
    dbc.Row([
        dbc.Col(dbc.Card(dcc.Graph(id="scatter"), className="shadow-sm border-0 p-2"), width=7),
        dbc.Col(dbc.Card(dcc.Graph(id="bar"), className="shadow-sm border-0 p-2"), width=5),
    ]),
    dbc.Row([
        dbc.Col(html.Small(id="selection-info", className="text-muted"), width=12),
    ], className="mt-2"),
], fluid=True)


def _highlight_selection(fig, selected_ids, all_ids):
    """Dim unselected points by reducing opacity."""
    if selected_ids:
        colors = ["rgba(76,120,168,1.0)" if i in selected_ids
                  else "rgba(76,120,168,0.15)" for i in all_ids]
        fig.update_traces(marker=dict(color=colors))
    return fig


@callback(
    Output("scatter", "figure"),
    Output("bar", "figure"),
    Output("selection-info", "children"),
    Input("scatter", "selectedData"),
    Input("bar", "clickData"),
    State("store-data", "data"),
)
def cross_filter(scatter_sel, bar_click, records):
    dff = pd.DataFrame(records)
    selected_countries = set()

    # Determine which countries are selected
    if scatter_sel and scatter_sel.get("points"):
        selected_countries = {p["customdata"][0] for p in scatter_sel["points"]}
    elif bar_click and bar_click.get("points"):
        continent = bar_click["points"][0]["x"]
        selected_countries = set(dff[dff["continent"] == continent]["country"])

    # Build scatter
    scatter_fig = px.scatter(
        dff, x="gdpPercap", y="lifeExp", size="pop", color="continent",
        custom_data=["country"], log_x=True, template="plotly_white",
        hover_name="country", title="GDP vs Life Expectancy",
    )

    # Build bar — filter if selection exists
    if selected_countries:
        bar_df = dff[dff["country"].isin(selected_countries)]
        info = f"{len(selected_countries)} countries selected"
    else:
        bar_df = dff
        info = "Click or select points to cross-filter"

    bar_fig = px.bar(
        bar_df.groupby("continent", as_index=False)["pop"].sum(),
        x="continent", y="pop", template="plotly_white",
        title="Population by Continent",
    )

    return scatter_fig, bar_fig, info
```

### Highlighting vs. Filtering

Two approaches to cross-filter feedback:

| Approach | Pros | Cons |
|---|---|---|
| **Filter** — remove unselected data | Clean, focused view | Loses context of full dataset |
| **Highlight** — dim unselected points | Keeps full context visible | Can look busy with large datasets |

**Highlight pattern using opacity:**
```python
@callback(Output("chart-b", "figure"), Input("chart-a", "selectedData"))
def highlight_cross(selected):
    fig = px.scatter(df, x="x", y="y", template="plotly_white")
    if selected and selected.get("points"):
        sel_ids = {p["customdata"][0] for p in selected["points"]}
        fig.update_traces(
            selectedpoints=[i for i, row in enumerate(df.itertuples())
                            if row.id in sel_ids],
            selected=dict(marker=dict(opacity=1.0)),
            unselected=dict(marker=dict(opacity=0.15)),
        )
    return fig
```

## Master–Detail Pattern

Click a row or point to show a detailed view:

```python
app.layout = dbc.Container([
    dbc.Row([
        # Master: summary chart
        dbc.Col(dbc.Card(dcc.Graph(id="overview"), className="shadow-sm border-0 p-2"), width=5),
        # Detail: appears on click
        dbc.Col(dbc.Card(html.Div(id="detail-panel"), className="shadow-sm border-0 p-3"), width=7),
    ]),
], fluid=True)

@callback(
    Output("detail-panel", "children"),
    Input("overview", "clickData"),
)
def show_detail(click_data):
    if not click_data:
        return html.P("Click a data point to see details", className="text-muted")
    entity_id = click_data["points"][0]["customdata"][0]
    entity_df = df[df["id"] == entity_id]

    detail_fig = px.line(entity_df, x="date", y="value",
                         template="plotly_white", title=f"Detail: {entity_id}")
    return html.Div([
        html.H5(f"Entity: {entity_id}"),
        dcc.Graph(figure=detail_fig, config={"displayModeBar": False}),
        html.Hr(),
        html.Pre(entity_df.describe().to_string()),
    ])
```

## Sidebar Filter Panel

A clean sidebar with controls that drive all dashboard charts:

```python
sidebar = dbc.Card([
    dbc.CardBody([
        html.H5("Filters", className="fw-bold mb-3"),

        html.Label("Category", className="fw-semibold small"),
        dcc.Dropdown(id="f-category", options=category_options,
                     value=None, multi=True, placeholder="All categories"),
        html.Hr(className="my-3"),

        html.Label("Date Range", className="fw-semibold small"),
        dcc.DatePickerRange(id="f-dates", start_date="2024-01-01",
                            end_date="2024-12-31",
                            display_format="MMM D, YYYY"),
        html.Hr(className="my-3"),

        html.Label("Metric", className="fw-semibold small"),
        dbc.RadioItems(id="f-metric",
                       options=[{"label": "Revenue", "value": "revenue"},
                                {"label": "Units", "value": "units"}],
                       value="revenue", inline=True),
        html.Hr(className="my-3"),

        dbc.Button("Reset Filters", id="btn-reset", color="secondary",
                   outline=True, size="sm", className="w-100"),
    ])
], className="shadow-sm border-0 rounded-3", style={"position": "sticky", "top": "1rem"})

# Main layout
app.layout = dbc.Container([
    dbc.Row([
        dbc.Col(sidebar, width=3),
        dbc.Col(html.Div(id="dashboard-content"), width=9),
    ]),
], fluid=True)
```

**Single callback to apply all filters:**
```python
@callback(
    Output("chart-1", "figure"),
    Output("chart-2", "figure"),
    Output("kpi-row", "children"),
    Input("f-category", "value"),
    Input("f-dates", "start_date"),
    Input("f-dates", "end_date"),
    Input("f-metric", "value"),
)
def apply_filters(categories, start, end, metric):
    dff = df.copy()
    if categories:
        dff = dff[dff["category"].isin(categories)]
    if start and end:
        dff = dff[(dff["date"] >= start) & (dff["date"] <= end)]

    fig1 = px.line(dff, x="date", y=metric, color="category",
                   template="plotly_white", title=f"{metric.title()} Over Time")
    fig2 = px.bar(dff.groupby("category", as_index=False)[metric].sum(),
                  x="category", y=metric, template="plotly_white",
                  title=f"Total {metric.title()} by Category")

    kpis = build_kpi_cards(dff, metric)
    return fig1, fig2, kpis
```

## KPI Cards Driven by Figures

Dynamic KPI cards that update alongside chart filters:

```python
def build_kpi_cards(dff, metric):
    total = dff[metric].sum()
    avg = dff[metric].mean()
    peak = dff.loc[dff[metric].idxmax()]

    cards = [
        _kpi_card("Total", f"${total:,.0f}", icon="bi-cash-stack"),
        _kpi_card("Average", f"${avg:,.0f}", icon="bi-bar-chart"),
        _kpi_card("Peak", f"{peak['date']:%b %d}", subtitle=f"${peak[metric]:,.0f}",
                  icon="bi-graph-up-arrow"),
        _kpi_card("Records", f"{len(dff):,}", icon="bi-database"),
    ]
    return [dbc.Col(c, width=3) for c in cards]


def _kpi_card(title, value, subtitle=None, icon=None):
    header = []
    if icon:
        header.append(html.I(className=f"bi {icon} me-2 text-primary"))
    header.append(html.Span(title, className="text-muted small"))

    body = [html.Div(header), html.H3(value, className="fw-bold mb-0")]
    if subtitle:
        body.append(html.Small(subtitle, className="text-muted"))
    return dbc.Card(dbc.CardBody(body), className="shadow-sm border-0 rounded-3")
```

## Tabbed Multi-View Dashboard

Organize related visualizations into tabs within a single page:

```python
app.layout = dbc.Container([
    html.H2("Analytics Dashboard", className="fw-bold pt-3 mb-3"),
    dbc.Tabs([
        dbc.Tab(label="Overview", tab_id="tab-overview", children=[
            dbc.Row([
                dbc.Col(dcc.Graph(id="overview-line"), width=8),
                dbc.Col(dcc.Graph(id="overview-pie"), width=4),
            ], className="mt-3"),
        ]),
        dbc.Tab(label="Comparison", tab_id="tab-compare", children=[
            dbc.Row([
                dbc.Col(dcc.Graph(id="compare-grouped-bar"), width=12),
            ], className="mt-3"),
        ]),
        dbc.Tab(label="Distribution", tab_id="tab-dist", children=[
            dbc.Row([
                dbc.Col(dcc.Graph(id="dist-histogram"), width=6),
                dbc.Col(dcc.Graph(id="dist-box"), width=6),
            ], className="mt-3"),
        ]),
    ], id="tabs", active_tab="tab-overview"),
], fluid=True)
```

## Drill-Down Navigation

Click a bar chart segment to navigate to a filtered detail page (multi-page app):

```python
from dash import ctx
import urllib.parse

# On overview page
@callback(
    Output("url", "pathname"),    # dcc.Location component
    Input("bar-chart", "clickData"),
    prevent_initial_call=True,
)
def drill_down(click_data):
    if not click_data:
        raise PreventUpdate
    category = click_data["points"][0]["x"]
    return f"/detail/{urllib.parse.quote(category)}"

# On detail page (pages/detail.py)
import dash
dash.register_page(__name__, path_template="/detail/<category>")

def layout(category=None):
    if not category:
        return html.P("No category selected")
    filtered = df[df["category"] == category]
    fig = px.line(filtered, x="date", y="value", template="plotly_white",
                  title=f"{category} Detail")
    return html.Div([
        dcc.Link("← Back to Overview", href="/"),
        dcc.Graph(figure=fig),
    ])
```
