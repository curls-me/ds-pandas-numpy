# NumPy Cheat Sheet — All NumPy Notebooks

Quick reference for the commands covered across `2_Numpy/` (01–02). See `../1_Pandas/DataFrame_Cheatsheet.md` for the Pandas equivalent — arrays and DataFrames work differently, so they get separate sheets.

## Setup

```python
import numpy as np  # standard import, always alias as np
```

## Why NumPy Arrays Are Fast

A NumPy array looks like a list on the surface, but two things underneath make it much faster for numerical work:

1. **Contiguous memory** — all values sit in one memory block. Python lists scatter elements around memory, adding overhead when accessing them.
2. **Homogeneous type** — every element is the same dtype (e.g. all `float64`). Python lists can mix types, so Python has to check each element's type before operating on it.

Together these enable **vectorisation** — the CPU processes many values in one operation instead of looping through them one at a time.

```python
# Always use NumPy's own methods on NumPy arrays, not Python built-ins
np.sum(numpy_array)   # fast — vectorised
sum(numpy_array)       # slow — NumPy arrays aren't optimised for Python built-ins
```

`np.sum()` is roughly 10–50x faster than Python's `sum()` on the same data; the gap grows for more complex operations.

## Creating Arrays

```python
# From a list — dtype inferred automatically
arr = np.array([1, 2, 3, 4, 5])

# From a tuple with an explicit dtype
arr = np.array((1, 2, 3, 4, 5), dtype=np.int32)

np.zeros((3, 4))        # 3x4 matrix of zeros
np.ones((10, 5))        # 10x5 matrix of ones
np.identity(5)           # 5x5 identity matrix (1s on diagonal, 0s elsewhere)
np.random.rand(2, 2)     # 2x2 array of random floats between 0 and 1
np.arange(0, 20, 0.5)    # like range(), but supports float steps: start, stop (excl.), step
```

## Inspecting Arrays

```python
arr.shape   # (rows,) for 1D, (rows, cols) for 2D — attribute, no ()
arr.dtype   # the element type, e.g. int64, float64
```

**Homogeneity:** NumPy forces every element to share one dtype. Mixing strings and numbers casts everything to string (`'U...'` = Unicode) — the numbers stop behaving like numbers.

```python
mixed = np.array(["1", 2, 3, "10", 5])
mixed.dtype        # '<U21' — a string dtype
type(mixed[1])      # str, not int — 2 became "2"

# Numeric strings CAN be forced to a numeric dtype on creation
castable = np.array([1, 2, 3, "10", 5], dtype=np.int32)   # works, "10" -> 10

# A genuinely non-numeric string can't be cast this way
# np.array([1, 2, 3, "bozo", 5], dtype=np.int32)  # raises ValueError
```

## Array Arithmetic

All standard operators (`+`, `-`, `*`, `/`, `**`, `%`) apply **element-wise** — no loops needed.

```python
a = np.array([1, 2, 3, 4])
b = np.array([5, 6, 7, 8])
a + b   # element-wise addition, 1D

first_2d = np.array([[1, 2], [3, 4]])
second_2d = np.array([[5, 6], [7, 8]])
first_2d * second_2d   # element-wise multiplication, 2D
```

### Broadcasting

Broadcasting is what happens when you combine an array with a scalar, or a smaller array — NumPy "expands" the smaller one to match the larger one's shape, then applies the operation element-wise.

```python
arr = np.array([[1, 2], [3, 4]])

# Scalar broadcasting — applies to every element
arr - 4
arr * 5
arr % 3

# Broadcast a 1D array across COLUMNS — [4, 5] maps to [col0, col1]
arr - [4, 5]        # subtracts 4 from column 0, 5 from column 1

# Broadcast a column vector across ROWS — [[4], [5]] maps to [row0, row1]
arr - [[4], [5]]    # subtracts 4 from row 0, 5 from row 1
```

See the full [broadcasting rules](https://numpy.org/doc/stable/user/basics.broadcasting.html) for more detail.

## Indexing & Slicing

NumPy has no `.loc`/`.iloc` — index directly with `array[row_selector, column_selector]`. Use `:` as a wildcard meaning "all".

```python
range_arr = np.arange(0, 20, 1).reshape(5, 4)   # reshape a 1D range into 5 rows, 4 columns

range_arr[:, 2]        # all rows, column at index 2
range_arr[0:2]          # first 2 rows, all columns (omitted column selector = all)
range_arr[0:2, 1:3]     # rows 0-1, columns 1-2
range_arr[1, 0]         # single element, row 1 col 0
```

## Aggregation Methods

Same statistical methods as Pandas columns — `.sum()`, `.mean()`, `.std()`, `.min()`, `.max()` — but with an `axis` parameter controlling direction:

| `axis` | Direction | Result |
|---|---|---|
| `0` | down the rows | one value **per column** |
| `1` | across columns | one value **per row** |
| omitted | whole array | single scalar |

Mnemonic: **the axis you specify is the one that disappears.**

```python
range_arr.sum(axis=0)    # column totals
range_arr.sum(axis=1)    # row totals
range_arr.sum()           # grand total

range_arr.mean(axis=0)   # column means
range_arr.std(axis=0)     # column std deviations
range_arr.max(axis=0)     # column maximums
range_arr.min(axis=0)     # column minimums
```

### `argmin()` / `argmax()` — find the INDEX, not the value

```python
arr = np.array([7, 9, 4])
arr.max()      # 9 — the value
arr.argmax()   # 1 — the position of that value

range_arr.argmin(axis=0)   # row index of the minimum, per column
range_arr.argmax()          # flattened index of the overall maximum
```

### Other useful methods

```python
range_arr.cumsum(axis=0)   # cumulative sum down the rows

range_arr.flatten()   # 2D -> 1D, always returns a COPY
range_arr.ravel()      # 2D -> 1D, returns a VIEW if possible (faster, but can mutate original)
```

## Finding Elements: `np.where()`

`np.where(condition)` returns the **indices** where the condition is `True` — positions, not filtered values (that's what boolean masking `arr[arr > x]` gives you instead).

```python
my_arr = np.array([2, 4, 6, 8, 24, 3, 8, 9, 12])

np.where(my_arr <= 2)   # indices where value <= 2
np.where(my_arr == 8)    # indices where value == 8
np.where(my_arr > 6)     # indices where value > 6

my_arr[my_arr > 6]        # the MATCHING VALUES instead of their indices
```

`np.where()` also has a three-argument form, `np.where(condition, if_true, if_false)`, for vectorised if/else.

## NumPy ↔ Pandas Equivalents

Pandas DataFrames are built on NumPy arrays — most array methods have a direct Pandas counterpart:

| NumPy (array) | Pandas (Series) |
|---|---|
| `array.argmax()` | `series.idxmax()` |
| `array.cumsum()` | `series.cumsum()` |
| `np.where(cond)` | boolean masking: `series[cond]` |

## Binning + Pivot Tables (NumPy feeding Pandas)

`pd.cut()` turns a continuous column into labelled bins — pair `np.arange()` for the bin edges with `pd.pivot_table()` to summarise across two categorical dimensions.

```python
import pandas as pd

df = pd.read_csv("../data/winequality-red.csv", sep=";")

# Bin edges via NumPy, then cut the continuous column into those bins
bin_edges = np.arange(4, 17)
fa_bins = pd.cut(df["fixed acidity"], bins=bin_edges, labels=bin_edges[:-1])
fa_bins.name = "fa_bin"
df = pd.concat([df, fa_bins], axis=1)   # attach the bin labels as a new column

# Pivot table: mean residual sugar by quality (rows) x fixed acidity bin (columns)
pd.pivot_table(df, values="residual sugar", index="quality", columns="fa_bin")

# Swap the aggregation function — e.g. max instead of the default mean
pd.pivot_table(df, values="residual sugar", index="quality", columns="fa_bin", aggfunc=np.max)
```

## Key Takeaways

- NumPy arrays beat Python lists on speed because of **contiguous memory** + **homogeneous dtype**, which enable **vectorisation**.
- Always call NumPy's own methods (`np.sum()`, etc.) on NumPy arrays — not Python's built-ins.
- Arithmetic operators are **element-wise** by default; **broadcasting** lets a scalar or smaller array apply across a bigger one without writing a loop.
- Index with `array[row, col]` — no `.loc`/`.iloc`; `:` means "all".
- `axis=0` = down the rows (per-column result); `axis=1` = across columns (per-row result); the axis named is the one that collapses.
- `.max()`/`.min()` return the **value**; `.argmax()`/`.argmin()` return the **index** of that value.
- `np.where(condition)` returns **indices**; boolean masking (`arr[arr > x]`) returns the **values**.
- Most array methods have a Pandas equivalent — understanding NumPy explains what's running under a DataFrame's hood.
