# polars Patterns

## Table of Contents
- [Expressions & Column Operations](#expressions--column-operations)
- [String Operations](#string-operations)
- [DateTime Operations](#datetime-operations)
- [Window & Rolling Functions](#window--rolling-functions)
- [Pivot & Reshape](#pivot--reshape)
- [Lazy API](#lazy-api)
- [Joins & Concatenation](#joins--concatenation)
- [Performance Tips](#performance-tips)

---

## Expressions & Column Operations

Polars uses **expressions** (`pl.col(...)`) that compose cleanly and execute in parallel.

```python
import polars as pl

# Select columns
df.select(["a", "b"])
df.select(pl.col("^col_.*$"))          # regex column selection
df.select(pl.all().exclude("id"))      # all except "id"

# Add / overwrite columns
df = df.with_columns([
    (pl.col("a") + pl.col("b")).alias("sum_ab"),
    pl.col("value").log(base=10).alias("log_value"),
    pl.when(pl.col("score") > 90).then(pl.lit("A"))
      .when(pl.col("score") > 75).then(pl.lit("B"))
      .otherwise(pl.lit("C")).alias("grade"),
])

# Drop columns
df = df.drop(["tmp_col", "unused"])

# Rename columns
df = df.rename({"old_name": "new_name"})

# Cast types
df = df.with_columns([
    pl.col("id").cast(pl.Utf8),
    pl.col("amount").cast(pl.Float64),
    pl.col("flag").cast(pl.Boolean),
])
```

## String Operations

```python
df = df.with_columns([
    pl.col("name").str.to_lowercase().alias("name_lower"),
    pl.col("name").str.strip_chars().alias("name_stripped"),
    pl.col("email").str.extract(r"@(.+)$", group_index=1).alias("domain"),
    pl.col("text").str.contains("keyword").alias("has_keyword"),
    pl.col("csv_field").str.split(",").alias("items_list"),
    pl.col("full_name").str.splitn(" ", n=2).struct.field("field_0").alias("first"),
    pl.col("amount_str").str.replace(",", "").cast(pl.Float64).alias("amount"),
])
```

## DateTime Operations

```python
# Parse dates from strings
df = df.with_columns(
    pl.col("date_str").str.to_date("%Y-%m-%d").alias("date"),
    pl.col("datetime_str").str.to_datetime("%Y-%m-%d %H:%M:%S").alias("ts"),
)

# Extract components
df = df.with_columns([
    pl.col("date").dt.year().alias("year"),
    pl.col("date").dt.month().alias("month"),
    pl.col("date").dt.weekday().alias("dow"),        # 0=Monday
    pl.col("date").dt.truncate("1mo").alias("month_start"),
])

# Date arithmetic
df = df.with_columns([
    (pl.lit(pl.Series([pl.date(2025, 1, 1)])) - pl.col("date"))
        .dt.total_days().alias("days_until"),
])
```

## Window & Rolling Functions

polars uses `.over()` for group-aware window functions (no separate groupby step needed).

```python
# Rolling mean within each group (sorted by date)
df = df.sort(["group", "date"]).with_columns(
    pl.col("value")
      .rolling_mean(window_size=7, min_periods=1)
      .over("group")
      .alias("rolling_7d_avg"),
)

# Cumulative sum within group
df = df.with_columns(
    pl.col("value").cum_sum().over("group").alias("cum_sum"),
)

# Rank within group (dense, descending)
df = df.with_columns(
    pl.col("value").rank(method="dense", descending=True)
      .over("group").alias("rank"),
)

# Lag / lead within group
df = df.with_columns([
    pl.col("value").shift(1).over("group").alias("prev_value"),
    pl.col("value").shift(-1).over("group").alias("next_value"),
])

# Percent change within group
df = df.with_columns(
    (pl.col("value") / pl.col("value").shift(1).over("group") - 1)
      .alias("pct_change"),
)
```

## Pivot & Reshape

```python
# Wide format (pivot): one column per category value
pivoted = df.pivot(index="date", columns="category",
                   values="amount", aggregate_function="sum")

# Long format (unpivot / melt)
long = df.unpivot(
    on=["col_a", "col_b", "col_c"],     # columns to melt
    index=["id", "date"],
    variable_name="metric",
    value_name="value",
)
```

## Lazy API

Prefer the **lazy API** for large data — polars optimizes the query plan before execution.

```python
# Lazy scan (no data loaded yet)
lf = pl.scan_csv("data.csv")
lf = pl.scan_parquet("data/*.parquet")   # multiple files

# Build a query lazily
result = (
    lf
    .filter(pl.col("value") > 0)
    .with_columns(pl.col("amount").cast(pl.Float64))
    .group_by("group")
    .agg([
        pl.len().alias("count"),
        pl.col("amount").sum().alias("total"),
    ])
    .sort("total", descending=True)
    .limit(20)
    .collect()                   # execute everything here
)

# Show the optimized query plan
lf.explain(optimized=True)

# Streaming collect for very large data (memory-efficient)
result = lf.collect(streaming=True)
```

## Joins & Concatenation

```python
# Inner / left / outer join
result = df_a.join(df_b, on="id", how="inner")
result = df_a.join(df_b, on="id", how="left")
result = df_a.join(df_b, on="id", how="full", coalesce=True)

# Join on different column names
result = df_a.join(df_b, left_on="user_id", right_on="id", how="left")

# Anti-join (rows in df_a not in df_b)
result = df_a.join(df_b, on="id", how="anti")

# Stack DataFrames vertically (same schema)
combined = pl.concat([df_a, df_b, df_c], rechunk=True)

# Diagonal concat (union columns, fill missing with null)
combined = pl.concat([df_a, df_b], how="diagonal_relaxed")
```

## Performance Tips

```python
# 1. Use the lazy API for any pipeline with >100k rows
lf = pl.scan_parquet("big.parquet").filter(...).group_by(...).collect()

# 2. Read only needed columns
df = pl.read_csv("data.csv", columns=["id", "date", "value"])

# 3. Use categoricals for low-cardinality strings
df = df.with_columns(pl.col("category").cast(pl.Categorical))

# 4. Prefer Parquet format — polars reads it ~10x faster than CSV
df.write_parquet("output.parquet")
df = pl.scan_parquet("output.parquet").collect()

# 5. Batch transforms in one with_columns call (runs in parallel)
# Good:
df = df.with_columns([expr_a, expr_b, expr_c])
# Avoid chaining many separate with_columns calls

# 6. Check query plan for bottlenecks
pl.scan_csv("big.csv").filter(...).explain()

# 7. Set thread count explicitly if needed (default = all cores)
import os
os.environ["POLARS_MAX_THREADS"] = "8"

# 8. Null handling
df.null_count()                              # per-column null counts
df.drop_nulls(subset=["key"])               # drop rows with nulls in key columns
df.with_columns(pl.col("col").fill_null(0)) # fill nulls with 0
df.with_columns(pl.col("col").fill_null(strategy="forward"))  # forward-fill
```
