# pandas Patterns

## Table of Contents
- [Indexing & Selection](#indexing--selection)
- [String Operations](#string-operations)
- [DateTime Operations](#datetime-operations)
- [Window & Rolling Functions](#window--rolling-functions)
- [Pivot & Reshape](#pivot--reshape)
- [Apply & Map](#apply--map)
- [Merging & Concatenation](#merging--concatenation)
- [Performance Tips](#performance-tips)

---

## Indexing & Selection

```python
# Single column → Series
df["col"]

# Multiple columns → DataFrame
df[["a", "b", "c"]]

# Boolean filter
mask = (df["value"] > 50) & (df["group"].isin(["A", "B"]))
df[mask]

# .query() — readable string syntax
df.query("value > 50 and group in ['A', 'B']")
df.query("date >= @start_date")   # reference Python variable with @

# .loc (label-based: rows, columns)
df.loc[df["flag"] == True, ["id", "value"]]

# .iloc (position-based: rows, columns)
df.iloc[0:5, 1:4]

# Set index for fast lookups
df = df.set_index("id")
df.loc["user_123"]              # O(1) lookup

# .at / .iat — fast single-value access
df.at[row_label, "col"]
df.iat[0, 3]
```

## String Operations

```python
df["name"].str.lower()
df["name"].str.strip()
df["name"].str.replace(r"\s+", " ", regex=True)
df["email"].str.extract(r"@(.+)$")             # capture group → column
df["text"].str.contains("keyword", case=False, na=False)
df["text"].str.split(",", expand=True)          # split into multiple columns
df["first"] = df["full_name"].str.split(" ").str[0]
```

## DateTime Operations

```python
df["date"] = pd.to_datetime(df["date"])

# Extract components
df["year"]  = df["date"].dt.year
df["month"] = df["date"].dt.month
df["dow"]   = df["date"].dt.day_name()

# Date arithmetic
df["days_since"] = (pd.Timestamp.today() - df["date"]).dt.days
df["next_month"]  = df["date"] + pd.DateOffset(months=1)

# Resample time series to period buckets
daily = df.set_index("date").resample("D")["value"].sum()    # daily
weekly = df.set_index("date").resample("W-MON")["value"].sum()
monthly = df.set_index("date").resample("ME")["value"].mean()
```

## Window & Rolling Functions

```python
# Rolling average (window = 7 periods)
df["rolling_7d"] = df["value"].rolling(window=7, min_periods=1).mean()

# Expanding (cumulative) statistics
df["cum_sum"]  = df["value"].expanding().sum()
df["cum_mean"] = df["value"].expanding().mean()

# Percent change
df["pct_change"] = df["value"].pct_change()

# GroupBy rolling (per-group)
df["rolling_group"] = (
    df.groupby("group")["value"]
    .transform(lambda x: x.rolling(7, min_periods=1).mean())
)

# Rank within group
df["rank"] = df.groupby("group")["value"].rank(method="dense", ascending=False)

# Lag / lead
df["prev_value"] = df.groupby("group")["value"].shift(1)
df["next_value"] = df.groupby("group")["value"].shift(-1)
```

## Pivot & Reshape

```python
# Wide format: one column per category value
pivot = df.pivot_table(index="date", columns="category",
                        values="amount", aggfunc="sum", fill_value=0)

# Melt: wide → long
long = pd.melt(df, id_vars=["id", "date"],
               value_vars=["col_a", "col_b", "col_c"],
               var_name="metric", value_name="value")

# Crosstab (frequency table)
pd.crosstab(df["group"], df["outcome"], normalize="index")

# Stack / unstack (multi-index)
stacked = pivot.stack()     # columns → rows
unstacked = stacked.unstack(level=1)
```

## Apply & Map

```python
# Element-wise map (fastest for simple transforms)
df["tier"] = df["score"].map({1: "Low", 2: "Mid", 3: "High"})
df["category_code"] = df["category"].map(category_to_int)

# Vectorized string / conditional (prefer over apply)
df["label"] = np.where(df["value"] > 100, "high", "low")

# apply (row-wise, slower — use only when vectorized is impossible)
df["result"] = df.apply(lambda row: complex_fn(row["a"], row["b"]), axis=1)

# .transform — returns same-shape result (useful in groupby context)
df["normalized"] = df.groupby("group")["value"].transform(
    lambda x: (x - x.mean()) / x.std()
)
```

## Merging & Concatenation

```python
# Merge on key
result = pd.merge(df_left, df_right, on="key", how="left")

# Merge on different column names
result = pd.merge(df_left, df_right,
                  left_on="user_id", right_on="id", how="inner")

# Merge on index
result = pd.merge(df_left, df_right,
                  left_index=True, right_index=True, how="outer")

# Stack DataFrames vertically
combined = pd.concat([df_a, df_b, df_c], ignore_index=True)

# Add suffix to distinguish overlapping columns
result = pd.merge(df_a, df_b, on="id", suffixes=("_old", "_new"))
```

## Performance Tips

```python
# Use categorical dtype for low-cardinality string columns (huge memory savings)
df["category"] = df["category"].astype("category")

# Use smaller numeric dtypes when range allows
df["flag"] = df["flag"].astype("int8")          # 0/1 flags
df["count"] = df["count"].astype("int32")

# Read only needed columns from large CSV
df = pd.read_csv("big.csv", usecols=["id", "date", "value"])

# Chunked reading for very large files
chunks = []
for chunk in pd.read_csv("big.csv", chunksize=100_000):
    chunks.append(chunk[chunk["value"] > 0])
df = pd.concat(chunks, ignore_index=True)

# Check memory usage
df.memory_usage(deep=True).sum() / 1024**2     # MB

# Avoid chained assignment — use .loc or .copy()
subset = df[df["group"] == "A"].copy()
subset["new_col"] = 1    # safe; no SettingWithCopyWarning
```
