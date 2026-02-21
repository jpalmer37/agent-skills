---
name: py-data
description: Conducting efficient data analysis in Python using pandas and polars. Use when users need to load, clean, transform, aggregate, or summarize tabular data — including CSV/Parquet/Excel ingestion, filtering, groupby, joins, window functions, and exploratory data analysis (EDA). Covers choosing between pandas and polars based on dataset size and performance requirements.
---

# Python Data Analysis

## Library Decision

| Use **pandas** when | Use **polars** when |
|---|---|
| Dataset fits comfortably in memory (<1 GB) | Dataset is large (>1 GB) or performance matters |
| Exploratory / interactive analysis | Production pipelines, batch jobs |
| Heavy use of ecosystem (sklearn, statsmodels) | Multi-core parallelism is needed |
| Familiar API is more important than speed | Lazy evaluation / query optimization wanted |

Both can read the same file formats. Switching is low-risk for most operations.

## Data Loading

```python
# pandas
import pandas as pd
df = pd.read_csv("data.csv")
df = pd.read_parquet("data.parquet")
df = pd.read_excel("data.xlsx", sheet_name="Sheet1")
df = pd.read_json("data.json", orient="records")

# polars
import polars as pl
df = pl.read_csv("data.csv")
df = pl.read_parquet("data.parquet")
df = pl.read_excel("data.xlsx")                  # requires fastexcel
lf = pl.scan_csv("data.csv")                     # lazy (preferred for large files)
lf = pl.scan_parquet("data/*.parquet")           # glob for multiple files
```

**Loading options:**
```python
# pandas — control dtypes and parsing
df = pd.read_csv("data.csv",
    dtype={"id": str, "amount": float},
    parse_dates=["date"],
    usecols=["id", "date", "amount"],
    na_values=["N/A", "missing"],
    low_memory=False)

# polars — infer schema then override
df = pl.read_csv("data.csv",
    dtypes={"id": pl.Utf8, "amount": pl.Float64},
    try_parse_dates=True,
    null_values=["N/A", "missing"])
```

## Quick Data Exploration

```python
# pandas
df.shape              # (rows, cols)
df.dtypes             # column types
df.head(10)
df.describe()         # numeric summary stats
df.isnull().sum()     # missing value counts
df["col"].value_counts(normalize=True)

# polars
df.shape
df.schema             # {name: dtype}
df.head(10)
df.describe()
df.null_count()
df["col"].value_counts(sort=True)
```

## Selecting & Filtering

See **[references/pandas-patterns.md](references/pandas-patterns.md)** and **[references/polars-patterns.md](references/polars-patterns.md)** for detailed examples. Quick reference:

```python
# pandas
df[["a", "b"]]                                  # select columns
df[df["value"] > 100]                           # boolean filter
df.query("value > 100 and group == 'A'")        # query string
df.loc[mask, ["a", "b"]]                        # loc: label-based
df.iloc[0:10, 1:3]                              # iloc: position-based

# polars
df.select(["a", "b"])                           # select columns
df.filter(pl.col("value") > 100)                # boolean filter
df.filter((pl.col("value") > 100) & (pl.col("group") == "A"))
```

## Cleaning

```python
# pandas
df = df.drop_duplicates()
df = df.dropna(subset=["key_col"])              # drop rows missing key_col
df["col"] = df["col"].fillna(df["col"].median())
df.columns = df.columns.str.lower().str.replace(" ", "_")
df["date"] = pd.to_datetime(df["date"], format="%Y-%m-%d")
df["amount"] = df["amount"].str.replace(",", "").astype(float)

# polars
df = df.unique()
df = df.drop_nulls(subset=["key_col"])
df = df.with_columns(
    pl.col("col").fill_null(pl.col("col").median()),
    pl.col("date").str.to_date("%Y-%m-%d"),
    pl.col("amount").str.replace(",", "").cast(pl.Float64),
)
```

## Aggregation & GroupBy

```python
# pandas
result = (df
    .groupby("group", as_index=False)
    .agg(
        count=("id", "count"),
        total=("amount", "sum"),
        avg=("amount", "mean"),
        p95=("amount", lambda x: x.quantile(0.95)),
    )
    .sort_values("total", ascending=False)
)

# polars
result = (df
    .group_by("group")
    .agg([
        pl.len().alias("count"),
        pl.col("amount").sum().alias("total"),
        pl.col("amount").mean().alias("avg"),
        pl.col("amount").quantile(0.95).alias("p95"),
    ])
    .sort("total", descending=True)
)
```

## Joins

```python
# pandas (merge)
merged = df_a.merge(df_b, on="id", how="left")
merged = df_a.merge(df_b, left_on="user_id", right_on="id", how="inner")

# polars (join)
merged = df_a.join(df_b, on="id", how="left")
merged = df_a.join(df_b, left_on="user_id", right_on="id", how="inner")
```

## Column Transforms

```python
# pandas
df["new_col"] = df["a"] * df["b"]
df["bucket"] = pd.cut(df["score"], bins=[0, 25, 50, 75, 100],
                       labels=["D", "C", "B", "A"])
df["rank"] = df.groupby("group")["value"].rank(method="dense", ascending=False)

# polars
df = df.with_columns([
    (pl.col("a") * pl.col("b")).alias("new_col"),
    pl.col("score").cut([25, 50, 75, 100], labels=["D", "C", "B", "A"]).alias("bucket"),
    pl.col("value").rank(method="dense", descending=True).over("group").alias("rank"),
])
```

## Saving Results

```python
# pandas
df.to_csv("output.csv", index=False)
df.to_parquet("output.parquet", index=False)    # fast, columnar, smaller files

# polars
df.write_csv("output.csv")
df.write_parquet("output.parquet")
df.write_excel("output.xlsx")                   # requires xlsxwriter
```

**Prefer Parquet over CSV** for intermediate/large data — faster I/O, smaller files, preserves dtypes.
