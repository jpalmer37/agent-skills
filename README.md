# agent-skills

A directory of agent skills following the [Agent Skills open standard](https://github.com/anthropics/skills).

## Skills

| Skill | Description |
|---|---|
| [py-plotting](skills/py-plotting/) | Intuitive and beautiful visualizations with Python Plotly + matplotlib |
| [py-dashboards](skills/py-dashboards/) | Clean and beautiful interactive dashboards using Python Plotly + Dash |
| [py-data](skills/py-data/) | Efficient data analysis using Python pandas and polars |

## Structure

Each skill lives in its own folder under `skills/` and follows the Agent Skills standard:

```
skills/
└── skill-name/
    ├── SKILL.md          # required — frontmatter metadata + instructions
    └── references/       # optional — detailed docs loaded by Claude as needed
```