# Modern Join Syntax Examples (dplyr 1.1+)

## Use join_by() instead of character vectors

### Good - Modern join syntax

```r
transactions |>
  inner_join(companies, by = join_by(company == id))
```

### Good - Inequality joins

```r
transactions |>
  inner_join(companies, join_by(company == id, year >= since))
```

### Good - Rolling joins (closest match)

```r
transactions |>
  inner_join(companies, join_by(company == id, closest(year >= since)))
```

### Avoid - Old character vector syntax

```r
transactions |>
  inner_join(companies, by = c("company" = "id"))
```

## Multiple match handling

### Expect 1:1 matches, error on multiple

```r
inner_join(x, y, by = join_by(id), multiple = "error")
```

### Allow multiple matches explicitly

```r
inner_join(x, y, by = join_by(id), multiple = "all")
```

### Ensure all rows match

```r
inner_join(x, y, by = join_by(id), unmatched = "error")
```

### Prevent NA matching (recommended)

```r
# By default, NA matches NA in joins - usually not desired
left_join(x, y, by = join_by(id), na_matches = "never")
```

## Logging joins with tidylog

**Use tidylog:: prefix for joins to verify expected behavior**

### Use tidylog:: prefix (don't load the package)

```r
# Good - explicit tidylog call for verification
result <- transactions |>
  tidylog::left_join(companies, by = join_by(company == id))

# tidylog output:
# left_join: added 2 columns (name, region)
#            > rows only in x      12
#            > rows only in y     (3)
#            > matched rows       988
#            > rows total        1000
```

### Interpreting join output

| Output | Meaning |
|--------|---------|
| `rows only in x` | Rows in left table with no match (kept as NA in left joins) |
| `rows only in y` | Rows in right table with no match (in parentheses, dropped in left joins) |
| `matched rows` | Rows that matched between tables |
| `rows total` | Final row count after join |

