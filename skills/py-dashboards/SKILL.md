---
name: py-dashboards
description: Building clean and beautiful interactive dashboards using Python Plotly Dash. Use when users want to create a web-based dashboard, data app, or interactive analytics interface in Python — including multi-page apps, reactive callbacks, KPI cards, filters, and data tables.
---

# Python Dashboards with Plotly Dash

## App Structure

A Dash app has three parts: **layout** (what to show), **callbacks** (reactivity), and **styling**.

```python
import dash
from dash import dcc, html, Input, Output, callback
import dash_bootstrap_components as dbc
import plotly.express as px
import pandas as pd

app = dash.Dash(__name__, external_stylesheets=[dbc.themes.BOOTSTRAP])

app.layout = dbc.Container([
    dbc.Row([
        dbc.Col(html.H1("My Dashboard"), width=12)
    ]),
    dbc.Row([
        dbc.Col([dcc.Dropdown(id="dropdown", options=[], value=None)], width=3),
        dbc.Col([dcc.Graph(id="main-chart")], width=9),
    ]),
], fluid=True)

@callback(Output("main-chart", "figure"), Input("dropdown", "value"))
def update_chart(selected):
    fig = px.bar(df[df["group"] == selected], x="x", y="y",
                 template="plotly_white")
    return fig

if __name__ == "__main__":
    app.run(debug=True)
```

Install: `pip install dash dash-bootstrap-components plotly pandas`

## Layout with dash-bootstrap-components

Use `dbc` for responsive grid layout — it maps directly to Bootstrap 12-column grid.

```python
dbc.Container([
    # Header row
    dbc.Row([dbc.Col(html.H2("Dashboard Title"), width=12)], className="mb-3"),

    # KPI card row
    dbc.Row([
        dbc.Col(make_kpi_card("Total Users", "12,345"), width=3),
        dbc.Col(make_kpi_card("Revenue", "$98,200"), width=3),
        dbc.Col(make_kpi_card("Conversion", "4.2%"), width=3),
        dbc.Col(make_kpi_card("Churn", "1.8%"), width=3),
    ], className="mb-4"),

    # Charts row
    dbc.Row([
        dbc.Col(dcc.Graph(id="chart-a"), width=8),
        dbc.Col(dcc.Graph(id="chart-b"), width=4),
    ]),
], fluid=True)
```

**KPI card helper:**
```python
def make_kpi_card(title, value, delta=None):
    children = [html.H6(title, className="text-muted mb-1"),
                html.H3(value, className="mb-0")]
    if delta:
        children.append(html.Small(delta, className="text-success"))
    return dbc.Card(dbc.CardBody(children), className="shadow-sm")
```

## Common Input Controls

```python
# Dropdown (single or multi)
dcc.Dropdown(id="dd", options=[{"label": k, "value": v} for k, v in pairs],
             value="default", multi=False, clearable=False)

# Date range picker
dcc.DatePickerRange(id="dates", start_date="2024-01-01", end_date="2024-12-31")

# Slider
dcc.Slider(id="slider", min=0, max=100, step=5, value=50,
           marks={i: str(i) for i in range(0, 101, 25)})

# Radio buttons
dbc.RadioItems(id="radio", options=[...], value="a", inline=True)

# Checklist
dbc.Checklist(id="checks", options=[...], value=["a", "b"], inline=True)
```

## Callbacks

See **[references/dash-callbacks.md](references/dash-callbacks.md)** for patterns including:
- Multi-input/multi-output callbacks
- Chained callbacks
- Loading states with `dcc.Loading`
- `PreventUpdate` and no-op guards
- Clientside callbacks for performance
- `dcc.Store` for sharing state across callbacks

## Theming & Styling

**Bootstrap themes** (set via `external_stylesheets`):
| Theme | Character |
|---|---|
| `dbc.themes.BOOTSTRAP` | Default Bootstrap |
| `dbc.themes.FLATLY` | Clean, flat, professional |
| `dbc.themes.MINTY` | Soft green tones |
| `dbc.themes.LUX` | Elegant serif headers |
| `dbc.themes.DARKLY` | Dark mode |
| `dbc.themes.CYBORG` | High-contrast dark |

Browse all at [bootswatch.com](https://bootswatch.com).

**Inline style overrides:**
```python
html.Div("Text", style={"color": "#4C78A8", "fontWeight": "bold"})

# dbc className utilities (Bootstrap)
dbc.Card(..., className="shadow-sm border-0 rounded-3")
html.H2("Title", className="fw-bold text-primary mb-2")
```

**Custom CSS** — create `assets/custom.css` (Dash auto-loads files in `assets/`):
```css
body { font-family: 'Inter', sans-serif; background-color: #F8F9FA; }
.card { border-radius: 12px; }
h1, h2, h3 { font-weight: 700; }
```

## Multi-Page Apps

```python
# File: app.py
app = dash.Dash(__name__, use_pages=True,
                external_stylesheets=[dbc.themes.BOOTSTRAP])
app.layout = dbc.Container([
    dbc.NavbarSimple(brand="My App", color="primary", dark=True,
                     children=[dbc.NavItem(dbc.NavLink(p["name"],
                                href=p["path"])) for p in dash.page_registry.values()]),
    dash.page_container,
], fluid=True)
```

```python
# File: pages/overview.py
import dash
dash.register_page(__name__, path="/", name="Overview")
layout = html.Div("Overview page content")
```

## Data Table

```python
from dash import dash_table

dash_table.DataTable(
    id="table",
    data=df.to_dict("records"),
    columns=[{"name": c, "id": c} for c in df.columns],
    page_size=15,
    sort_action="native",
    filter_action="native",
    style_header={"fontWeight": "bold", "backgroundColor": "#F0F2F5"},
    style_data_conditionals=[
        {"if": {"row_index": "odd"}, "backgroundColor": "#FAFBFC"}
    ],
    style_table={"overflowX": "auto"},
)
```

## Running & Deployment

```bash
# Development
python app.py                        # runs on http://127.0.0.1:8050

# Production (gunicorn)
pip install gunicorn
gunicorn app:server -b 0.0.0.0:8050

# Note: expose `server = app.server` in app.py for gunicorn
```
