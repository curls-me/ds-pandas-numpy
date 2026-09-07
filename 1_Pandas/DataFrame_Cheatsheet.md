# Pandas Cheat Sheet — All Pandas Notebooks

Quick reference for the commands covered across `1_Pandas/` (01–05) and `3_More_Pandas/` (01–03). Numpy notebooks are not included — arrays work differently from DataFrames and would need their own sheet.

## Setup

```python
import pandas as pd  # standard import, always alias as pd
```

## Creating a DataFrame

```python
# From a list of dictionaries (dict keys -> column names)
data_lst = [{"a": 1, "b": 2}, {"a": 3, "b": 4}]
df = pd.DataFrame(data_lst)          # each dict becomes one row

# From a list of lists + explicit column names
data_vals = [[1, 2], [3, 4]]         # each inner list is one row
data_cols = ["a", "b"]               # column names, must match value count per row
df = pd.DataFrame(data=data_vals, columns=data_cols)

# From a CSV file
df = pd.read_csv("path/to/file.csv")                                # assumes headers in row 1
df = pd.read_csv("path/to/file.csv", header=None)                   # no header row in the file
df = pd.read_csv("path/to/file.csv", header=None, names=["a","b"])  # assign column names yourself
df = pd.read_csv("path/to/file.csv", sep=";")                       # non-comma separator (e.g. semicolon)
```

## Exploring a DataFrame

| Command | What it does |
|---|---|
| `df.head()` / `df.head(n)` | First 5 (or n) rows |
| `df.tail()` / `df.tail(n)` | Last 5 (or n) rows |
| `df.shape` | `(rows, columns)` — attribute, no `()` |
| `df.columns` | Column names |
| `df.info()` | Column dtypes + non-null counts |
| `df.describe()` | Stats (count, mean, std, min, max, quartiles) for numeric columns only |
| `df.dtypes` | Just the dtype list (no non-null counts) |
| `df.describe(include='all')` | Stats for numeric AND categorical columns (adds `unique`, `top`, `freq`) |
| `df.isnull().sum()` | Count of missing values per column |
| `len(df)` | Row count (equivalent to `df.shape[0]`) |

**Continuous vs. categorical:** `float64`/`int64` dtype usually means continuous (or a count); `object` dtype usually means categorical. For a numeric column that might secretly be categorical (e.g. a 1–10 rating), check `df["col"].nunique()` (few unique values = likely categorical) and `df["col"].unique()` to see the actual values — dtype alone doesn't tell the full story.

```python
# display() renders MULTIPLE DataFrames/outputs from one cell — a plain expression
# only auto-prints the LAST one, so use display() for every earlier one you want to see
display(df.head())
display(df.tail(10))
```

## Descriptive Statistics (on a column / Series)

```python
df["col"].mean()      # average
df["col"].median()    # middle value (50th percentile) — robust to outliers, unlike mean
df["col"].mode()[0]   # most frequent value — mode() returns a Series (there can be ties), [0] takes the first
df["col"].min()       # smallest value
df["col"].max()       # largest value
df["col"].var()       # variance — average squared distance from the mean
df["col"].std()       # standard deviation — sqrt(variance), same unit as the data (easier to interpret)
df["col"].quantile(0.9)  # value at the 90th percentile
```

### Displaying mean, median & mode together

Comparing these three side by side is a quick, no-chart way to spot skew in a distribution (see the table below).

```python
col = df["petal_length"]

print("Mean:  ", col.mean())
print("Median:", col.median())
print("Mode:  ", col.mode()[0])   # mode() returns a Series (there can be ties), [0] takes the first

# Or as a tidy one-row summary table instead of separate print lines
pd.DataFrame({
    "mean": [col.mean()],
    "median": [col.median()],
    "mode": [col.mode()[0]]
})
```

| Comparison | Shape |
|---|---|
| Mean ≈ Median | Roughly symmetric |
| Mean > Median | Right-skewed (a tail of high values pulls the mean up) |
| Mean < Median | Left-skewed (a tail of low values pulls the mean down) |

Mode shows where the data **clusters** — if there are two distinct modes (bimodal), mean/median alone won't reveal that; you need a histogram to see the shape (e.g. `petal_length` in the Iris dataset has two separate clusters by species).

## Selecting Columns

```python
df["col"]              # single column -> Series (bracket notation, always works)
df.col                  # same, but breaks on spaces in the name or a clash with a DataFrame method
df[["col1", "col2"]]    # multiple columns -> DataFrame (note the double brackets)
```

## Selecting Rows

```python
df[:3]                  # slice notation, like a list — rows 0, 1, 2 (not including 3)
df.loc[0, "sex"]        # single value, by LABEL
df.loc[0:10, "sex"]     # row range (inclusive of 10!) + one column, by label
df.loc[10:15, ["sex","length"]]  # row range + multiple columns, by label

df.iloc[0, 0]           # single value, by INTEGER position
df.iloc[0:5, 0]         # first 5 rows, first column, by position (end excluded, like a list)
df.iloc[10:15, [0, 4]]  # rows 10-14, columns 0 & 4, by position

df["col"].iloc[-10:]    # last 10 rows of one column — negative slicing, position-only
df.iloc[-10:]           # last 10 rows of the whole DataFrame
```

`.loc` = **l**abels · `.iloc` = **i**ntegers. Mixing them up raises `KeyError` (`.loc`) or `ValueError` (`.iloc`).

Negative slicing (`-10:` = "last 10") only works with `.iloc` or plain brackets — `.loc` uses labels, so `-10` would look for a label literally named `-10` and fail unless your index happens to contain that.

## Filtering (Boolean Masks & `.query()`)

```python
mask = df["length"] <= 0.4      # comparison -> a Series of True/False, one per row
df[mask]                        # keep only rows where the mask is True

# Combine conditions — use & / | (not the Python keywords and/or, which don't work elementwise)
# Each condition MUST be wrapped in ( ) because & binds tighter than > and ==
df[(df["length"] <= 0.4) & (df["sex"] == "F")]

# Same filter written as a readable string — .query() uses and/or instead of &/|
df.query("length <= 0.4 and sex == 'F'")

# Chain of AND conditions
df[(df["pH"] > 3.0) & (df["pH"] < 3.5) & (df["residual sugar"] < 2.0)]

# Chain .head() onto a filter to preview just the first few matches
df[mask].head()

# Compare against a computed value instead of a hard-coded number, so the filter
# stays correct if the data changes
mask = df["chlorides"] <= df["chlorides"].mean()         # below average
mask = df["chlorides"] <= df["chlorides"].median()       # below median
mask = df["chlorides"] <= df["chlorides"].quantile(0.9)  # below 90th percentile

# Build on a previous filter's result (save it to a variable first, not just .head())
df7 = df[(df["pH"] > 3.0) & (df["pH"] < 3.5)]   # first filter, saved
df8 = df7[df7["residual sugar"] < 2.0]          # second filter, applied on top of df7
```

`&` = AND · `|` = OR · `~` = NOT (negate a mask)

**Watch out:** `.describe()` does NOT return one number — it returns a Series of stats (`mean`, `std`, `min`...). Comparing a column directly to `.describe()` (e.g. `df["col"] <= df["col"].describe(1)`) doesn't work — pick one specific stat instead (`.mean()`, `.median()`, `.quantile(x)`).

## Value Counts & Cross-tabulation

```python
df["col"].value_counts()               # count of rows per distinct value, sorted high to low by default
df["col"].value_counts().head(10)      # top 10 most frequent values
df["col"].value_counts().tail(10)      # 10 least frequent values
df["col"].value_counts(ascending=True).head(10)  # same as .tail(10) above, sorted the other way

# Cross-tab: count of rows for every combination of two columns
pd.crosstab(df["col_a"], df["col_b"])
pd.crosstab(df["col_a"], df["col_b"], margins=True)  # margins=True adds "All" row/column subtotals
```

## String Methods (`.str` accessor)

Vectorised string operations run across a whole column at once — much faster than a Python `for` loop, and the standard way to clean text columns.

```python
# Clean up column names: lowercase, spaces -> underscores, '#' -> 'num'
df.columns = df.columns.str.lower().str.replace(" ", "_").str.replace("#", "num")

# Same idea applied to a data column, not just column names
df["station"] = df["station"].str.lower().str.replace(" ", "_")
```

## Groupby (split → apply → combine)

```python
df["sex"].unique()          # distinct values in a column
df["sex"].nunique()         # count of distinct values

g = df.groupby("sex")       # store the GroupBy object when running multiple aggregations on it
g.mean(numeric_only=True)   # mean per group (numeric_only avoids a warning on non-numeric columns)
g.max(numeric_only=True)    # max per group
g.count()                   # non-null count per column, per group
g.size()                    # row count per group (simpler than .count() when you just want totals)

df.groupby("sex").count()["length"]        # pull one column out of the grouped result
df.groupby(["sex", "n_rings"]).count()["weight_whole"]  # group by multiple columns at once

df.groupby("species")[["sepal_length", "petal_length"]].mean()  # limit aggregation to specific columns

# Group on a filtered subset (filter first, then group)
df[(df["pH"] > 3.0) & (df["pH"] < 4.0)].groupby("pH")["alcohol"].mean()

# Aggregate methods work the same way: .mean(), .max(), .min(), .sum(), .count()
df.groupby("quality")["chlorides"].mean()

# .max()/.min() on a single filtered column returns one scalar value, not a Series
df[(df["alcohol"] >= 9.25) & (df["alcohol"] <= 9.5)]["residual sugar"].max()

# .describe() also works per group — full stats breakdown for each category
df.groupby("subscription_type")["duration"].describe()
```

**Note:** a column can be numeric in *dtype* but behave like a category in practice — e.g. `quality` is an integer score (3–8), but it's really a categorical rating rather than a continuous measurement. Dtype alone doesn't always tell the full story; consider what the number actually represents.

## Sorting

```python
df.sort_values("length")                               # ascending (default)
df.sort_values("length", ascending=False)              # descending
df.sort_values(["length", "diameter"], ascending=False) # sort by multiple columns, 2nd breaks ties in the 1st

# Top / bottom N rows by a column — shortcut for sort_values + head
df.sort_values("density", ascending=False).head(5)  # 5 highest, written out
df.nlargest(5, "density")                            # same result on a DataFrame, more efficient
df.nsmallest(10, "sulphates")                        # 10 lowest, DataFrame version

# Simpler: call directly on a single column (Series) when you don't need the other columns
df["density"].nlargest(5)      # top 5 values of just this column
df["sulphates"].nsmallest(10)  # bottom 10 values of just this column
```

## Creating / Dropping / Renaming Columns

```python
df.rename(columns={"old": "new"}, inplace=True)  # rename one or more columns in place

df = df.eval("age = n_rings + 1.5")   # create a column from a string expression
df["age"] = df["n_rings"] + 1.5       # equivalent, direct assignment (usually clearer)

# Column names with spaces need backticks inside eval()
df = df.eval("total_acidity = `fixed acidity` + `volatile acidity`")
df["total_acidity"] = df["fixed acidity"] + df["volatile acidity"]  # equivalent, no backticks needed

df.drop("col", axis=1)                # returns a NEW DataFrame with the column removed, original unchanged
df.drop("col", axis=1, inplace=True)  # modifies df directly, returns None

# When a name matches MULTIPLE columns (duplicates), drop removes all of them in one call
df.drop(columns="fa_bin")             # drops every column named "fa_bin", however many there are
df.loc[:, ~df.columns.duplicated()]   # keep only the FIRST occurrence of each duplicated column name
```

`axis=1` = columns · `axis=0` = rows

## Rounding

```python
df["col"].round(2)                # round ONE column to 2 decimal places -> new Series
df["col"] = df["col"].round(2)    # assign back to actually update the column

df.round(2)                        # round ALL numeric columns to 2 decimal places
df.round({"col1": 2, "col2": 0})   # different number of decimals per column

# Round in a specific direction instead of to nearest
import numpy as np
np.floor(df["col"])    # always round DOWN
np.ceil(df["col"])     # always round UP
df["col"].astype(int)   # truncate the decimal entirely (cuts, doesn't round)
```

**Watch out:** `.round()` uses "round half to even" (banker's rounding) for exact ties — `2.5` rounds to `2`, not `3`. This matches Python's built-in `round()` and rarely matters in real data, but can surprise you if you're checking exact `.5` values.

## Missing Values

```python
df.fillna(-1, inplace=True)   # replace every NaN with a specific value
df.dropna(inplace=True)       # remove any row that contains at least one NaN
```

## Visualization (`.plot()`)

Pandas' `.plot()` is built on Matplotlib — good for quick exploratory charts without writing separate Matplotlib code.

```python
import matplotlib.pyplot as plt

# Histogram — distribution of ONE numeric column
df["quality"].plot(kind="hist", title="Distribution of Wine Quality Scores")
plt.xlabel("Quality Score")   # matplotlib call, styles the currently-open plot
plt.show()                    # renders the plot

# Alternative histogram syntax: .hist() directly on a Series (same result as kind="hist")
df["petal_length"].hist(bins=20)
plt.xlabel("petal_length (cm)")
plt.show()

# bins= controls the buckets: an integer -> that many EQUAL-WIDTH buckets across the range
df["quality"].plot(kind="hist", bins=10)

# bins= as a list -> CUSTOM edges you choose yourself; n edges give n-1 buckets
df["quality"].plot(kind="hist", bins=[0, 2, 4, 6, 8, 10])

# Scatter — relationship between TWO numeric columns
df.plot(kind="scatter", x="total sulfur dioxide", y="free sulfur dioxide",
        title="Free vs Total Sulfur Dioxide", alpha=0.4)  # alpha = point transparency
plt.show()

# Alternative scatter syntax: .plot.scatter() instead of .plot(kind="scatter") — same result
df.plot.scatter(x="petal_length", y="petal_width")
plt.show()

# Box plot — spread + outliers for one or more columns (see whisker math below)
df[["fixed acidity", "alcohol"]].plot(kind="box", title="Fixed Acidity vs Alcohol")
plt.show()

# Bar chart — aggregated value per category (pair with groupby)
df.groupby("quality")["alcohol"].mean().plot(   # groupby first, THEN plot the resulting Series
    kind="bar", title="Average Alcohol Content by Quality Score",
    color="steelblue", rot=0   # rot=0 keeps the category labels horizontal (not rotated)
)
plt.xlabel("Quality Score")
plt.ylabel("Average Alcohol (%)")
plt.show()

# Pie chart — share of each category (best with just a few categories)
counts = df["subscription_type"].value_counts()   # count rows per category first
counts.plot(kind="pie", autopct="%1.1f%%", ylabel="")  # autopct adds % labels on each slice
plt.title("Trips by Subscription Type")
plt.show()

# All pairwise scatter plots at once — quick way to eyeball every relationship
pd.plotting.scatter_matrix(df, figsize=(10, 10))
plt.show()
```

| Chart | `kind=` | Answers |
|---|---|---|
| Histogram | `"hist"` | What's the spread/shape of one variable? |
| Scatter | `"scatter"` (+ `x=`, `y=`) | How do two variables relate? |
| Box plot | `"box"` | How spread out is the data, and are there outliers? |
| Bar chart | `"bar"` / `"barh"` | What's the aggregated value per category? (pair with `.groupby()`) |
| Pie chart | `"pie"` | What share does each category hold? (few categories only) |

Other useful args: `alpha=` (point transparency, 0=invisible, 1=solid — helps see density when points overlap), `title=`, `color=`, `rot=` (label rotation), `bins=` (histogram bar count), `plt.xticks(rotation=45)` + `plt.tight_layout()` (long tick labels that would otherwise overlap).

**Watch out:** plotting many columns together only works well if they're on similar scales — one column with a much bigger range will dominate the y-axis and squash the rest flat. Select a subset of similarly-scaled columns first (e.g. `df[["fixed acidity", "alcohol"]]`). Same idea for histograms of a skewed column with extreme outliers: filter to a meaningful range first (e.g. `df[df["duration"] < 3600]`) and re-plot to see the real shape.

### Box plot whisker calculation

The box = middle 50% of the data (Q1 to Q3), the line inside = median. Whiskers and outliers are calculated as:

```python
q1 = df["col"].quantile(0.25)   # 25th percentile
q3 = df["col"].quantile(0.75)   # 75th percentile
iqr = q3 - q1                   # interquartile range = box height

lower_bound = q1 - 1.5 * iqr    # anything below this is an outlier
upper_bound = q3 + 1.5 * iqr    # anything above this is an outlier

# Whiskers extend to the actual data point closest to (but still inside) each bound —
# not necessarily exactly at the bound itself
lower_whisker = df[df["col"] >= lower_bound]["col"].min()
upper_whisker = df[df["col"] <= upper_bound]["col"].max()

# Anything beyond the bounds is plotted as an individual outlier dot
outliers = df[(df["col"] < lower_bound) | (df["col"] > upper_bound)]
```

`1.5` is a convention (Tukey's rule), not a hard rule — it works reasonably well for roughly bell-shaped data. There's no fixed guaranteed percentage "inside the whiskers" (95% is a common myth borrowed from standard-deviation rules); it depends on your data's actual shape and skew.

## Series (single column)

A single column selected from a DataFrame is a **Series** — a 1D labelled array. Most DataFrame methods also work on a Series (`.unique()`, `.mean()`, comparisons, etc.).

```python
type(df["length"])   # pandas.core.series.Series
```

## Datetime

Dates usually load from CSV as plain **strings** (`object` dtype). Convert them before doing any date math, sorting, or feature extraction.

```python
from datetime import datetime, date, time, timedelta

# --- Python's built-in datetime module (single values) ---
d1 = date(2021, 5, 14)          # a real date object, not just text
dt1 = datetime(2021, 5, 14, 11, 20, 30)  # date + time combined

# strptime: parse a STRING into a datetime, using format codes
d1 = datetime.strptime("2021-05-14", "%Y-%m-%d")

# strftime: format a datetime back into a STRING
formatted = datetime.now().strftime("%d/%m/%Y %H:%M")

# timedelta: the duration between two datetimes
duration = datetime(2021, 4, 23) - datetime(2020, 4, 23)  # -> a timedelta object
duration.days                          # whole days in the duration
duration / timedelta(hours=1)          # convert the whole duration to hours

# Add/subtract time from a date
today = date.today()
today + timedelta(days=2)              # 2 days from today
today + timedelta(weeks=2)             # 2 weeks from today

# --- Pandas equivalents (whole columns) ---
df["date"] = pd.to_datetime(df["date"], format="%Y/%m/%d")  # string column -> datetime64 dtype

# .dt accessor extracts a date component from EVERY row at once (column must be datetime64 first)
df["day"] = df["date"].dt.day
df["month"] = df["date"].dt.month
df["year"] = df["date"].dt.year
df["hour"] = df["date"].dt.hour
df["minute"] = df["date"].dt.minute
df["weekday"] = df["date"].dt.weekday   # Monday=0 ... Sunday=6

# Week of year — the older .dt.week is deprecated, use isocalendar() instead
df["week_of_year"] = df["date"].dt.isocalendar().week   # ISO week number, 1-52/53
df["iso_weekday"] = df["date"].dt.isocalendar().day     # ISO weekday, Monday=1 ... Sunday=7

# Generate a sequence of evenly-spaced dates
pd.date_range(start="2020-04-24", end="2020-05-24", freq="D")   # one per day
pd.date_range(start=today, periods=10, freq="min")               # 10 consecutive minutes from today
```

| Format code | Meaning | Example |
|---|---|---|
| `%Y` | 4-digit year | `2021` |
| `%m` | Zero-padded month | `05` |
| `%d` | Zero-padded day | `14` |
| `%H` / `%M` / `%S` | Hour / minute / second | `13` / `20` / `30` |
| `%B` | Full month name | `April` |
| `%A` | Full weekday name | `Tuesday` |

**Watch out:** calling `.dt` on a column that's still `object` dtype raises `AttributeError` — run `pd.to_datetime()` first. Always assign with bracket notation (`df["date"] = ...`), not dot notation (`df.date = ...`) — dot assignment can silently fail to update the real column. `isocalendar()` returns a table with columns `year`/`week`/`day` — there's no `.weekday` on it, so grab `.day` for the ISO weekday or use the separate `.dt.weekday` for pandas' own (non-ISO) numbering.

## Combining DataFrames

Three tools, different defaults — picking the right one saves debugging time later.

```python
# pd.get_dummies() — turn ONE categorical column into 0/1 indicator columns (one per category)
dummies = pd.get_dummies(df["quality"], prefix="quality")   # -> columns like quality_3, quality_4...

# df.join() — attach another DataFrame/Series, aligned by INDEX (default how='left')
joined = df.join(dummies)

# pd.concat() — stack a LIST of DataFrames together
pd.concat([df1, df2], axis=0)              # axis=0 (default): stack ROWS (append below)
pd.concat([df1, df2], axis=0, ignore_index=True)  # also reset the index (avoids duplicate labels)
pd.concat([df, dummies], axis=1)           # axis=1: stack COLUMNS side by side, aligned by index

# pd.merge() — join on a SHARED COLUMN (like a SQL JOIN), not the index
merged = pd.merge(df_a, df_b, on="id")                        # default how='inner'
merged = pd.merge(df_a, df_b, on="id", how="outer")           # keep every row from both sides
merged = pd.merge(df_a, df_b, on="quality", suffixes=[" red", " white"])  # disambiguate same-named columns
```

| `how` value | SQL equivalent | Keeps... |
|---|---|---|
| `'left'` (join default) | LEFT OUTER JOIN | All rows from the left DataFrame |
| `'right'` | RIGHT OUTER JOIN | All rows from the right DataFrame |
| `'outer'` | FULL OUTER JOIN | All rows from both DataFrames |
| `'inner'` (merge default) | INNER JOIN | Only rows present in **both** DataFrames |

- Use **`.join()`** when both DataFrames already share an index.
- Use **`pd.merge()`** when joining on a regular column (an ID, a key), especially with differing column names via `left_on`/`right_on`.
- An `inner` merge silently drops any row whose key isn't present on both sides — check row counts before/after if that surprises you.
- `pd.concat()` **keeps each DataFrame's original index** — pass `ignore_index=True` to avoid ending up with duplicate index labels.

### Binning with `pd.cut()`

Splits a continuous column into labelled, equal-**width** intervals (not equal-count — that's `pd.qcut()`).

```python
bins = pd.cut(df["fixed acidity"], bins=5)     # 5 equal-width bins across the column's range
bins.value_counts().sort_index()               # how many rows fall in each bin

# Attach the bin labels back onto the DataFrame
df = pd.concat([df, bins.rename("acidity_bin")], axis=1)

# The bin categories are ordered — grab the highest one to filter it out
top_bin = df["acidity_bin"].cat.categories[-1]
df_not_top = df[df["acidity_bin"] != top_bin]
```

### Pivot Tables with `pd.pivot_table()`

Like `groupby`, but produces a 2D table instead of a flat Series — good for summarising by two dimensions at once.

```python
pd.pivot_table(
    df,
    values="alcohol",        # column to aggregate
    index="quality",         # becomes the row labels
    columns="acidity_bin",   # becomes the column headers (omit for a 1D pivot)
    aggfunc="mean",          # how to aggregate: 'mean', 'sum', 'count', etc.
    observed=False,          # keep every bin category as a column even if some combos have no rows
)
```

`NaN` in a pivot table cell means no rows matched that row/column combination — it's not a data-quality problem, the absence itself is the signal. Use `fill_value=0` only if zero genuinely means "no observations" for your use case.
