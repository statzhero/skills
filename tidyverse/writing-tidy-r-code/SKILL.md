---
name: writing-tidy-r-code
description: |
  Modern tidyverse patterns, style guide, and migration guidance for R development. Use this skill when writing R code, reviewing tidyverse code, updating legacy R code to modern patterns, or enforcing consistent style. Covers native pipe usage, join_by() syntax, .by grouping, pick/across/reframe operations, tidy selection, stringr patterns, naming conventions, and migration from base R or older tidyverse APIs. Use the R (btw) MCP tools to resolve function documentation and library references automatically.
---

# Writing tidyverse R

This skill covers modern tidyverse patterns for R 4.5+ and tidyverse 2.0+, style guidelines, and migration from legacy patterns.

## Core Principles

1. **Use modern tidyverse patterns** - Prioritize dplyr 1.1+ features, native pipe, and current APIs
2. **Write readable code first** - Optimize only when necessary
3. **Follow tidyverse style guide** - Consistent naming, spacing, and structure
4. **Use R MCP tools** - Automatically resolve function documentation and library references without being asked (if available)

## Code Organization

**Use newspaper style: high-level logic first, helpers below**

- Main functions and primary logic should appear at the top of a file
- Don't define functions inside other functions unless they are very brief
- This makes code scannable; readers see the important parts first

## Pipe Usage

**Always use native pipe `|>` instead of magrittr `%>%`**

**Use `\(x)` for anonymous functions instead of `function(x)` or `~`**

```r
# Good
data |> map(\(x) x + 1)
data |> keep(\(x) !is.na(x))

# Avoid
data %>% map(function(x) x + 1)
data %>% map(~ .x + 1)
```

R 4.5+ provides all needed features.

## Error Handling

**Use `cli::cli_abort()` for errors following tidyverse conventions**

Error messages should follow the [tidyverse error style guide](https://style.tidyverse.org/errors.html):

```r
# Good
if (length(x) == 0) {
  cli::cli_abort(c(
    "x" = "{.arg x} must have at least one element.",
    "i" = "You provided an empty vector."
  ))
}

# Avoid
if (length(x) == 0) {
  stop("x must have at least one element")
}
```

## R Idioms

**Prefer R-native approaches over patterns from other languages**

- Use `message()` for informational output, not `cat()`
- Use `warning()` for warnings, `cli::cli_warn()` for styled warnings
- Use `TRUE`/`FALSE`, not `T`/`F`
- Use purrr `map_*()` over `sapply()` for type stability
- Use tidyna to avoid repetitive `na.rm = TRUE`

## NA Handling with tidyna

**Load tidyna to make functions ignore NA by default**

tidyna masks base functions (`mean`, `sum`, `sd`, etc.) so they automatically use `na.rm = TRUE`:

```r
library(tidyna)

# Good - tidyna masks mean/sum/etc. to ignore NA by default
data |>
  summarise(
    avg = mean(value),
    total = sum(value),
    med = median(value)
  )

# Avoid - repetitive na.rm = TRUE
data |>
  summarise(
    avg = mean(value, na.rm = TRUE),
    total = sum(value, na.rm = TRUE),
    med = median(value, na.rm = TRUE)
  )
```

## Join Syntax (dplyr 1.1+)

**Use `join_by()` instead of character vectors for joins**

Modern join syntax supports:
- Equality joins: `join_by(company == id)`
- Inequality joins: `join_by(company == id, year >= since)`
- Rolling joins: `join_by(company == id, closest(year >= since))`

See [join-examples.md](references/join-examples.md) for complete patterns.

## Multiple Match Handling

Use `multiple`, `unmatched`, and `na_matches` arguments for quality control:
- `multiple = "error"` - Expect 1:1 matches
- `multiple = "all"` - Allow multiple matches explicitly
- `unmatched = "error"` - Ensure all rows match
- `na_matches = "never"` - Don't match NA to NA (recommended default)

## Distinct Rows

**Use `.keep_all = TRUE` with `distinct()` to preserve all columns**

```r
# Good - keeps all columns
data |> distinct(id, .keep_all = TRUE)

# Avoid - drops all other columns
data |> distinct(id)
```

## Operation Logging with tidylog

**Use tidylog:: prefix to verify dplyr operations, especially joins**

Call tidylog functions directly instead of loading the package:

```r
# Use tidylog:: prefix for join verification
result <- transactions |>
 tidylog::left_join(companies, by = join_by(company == id))

# tidylog output:
# left_join: added 2 columns (name, value)
#            > rows only in x    5
#            > rows only in y  (1)
#            > matched rows     95
```

When to use tidylog:
- **Always for joins** - See how many rows matched, duplicated, or were dropped
- **Critical filters** - `tidylog::filter()` to verify expected row counts
- **Critical mutates** - `tidylog::mutate()` to verify expected changes
- **Any operation where silent data loss is a risk**

**Don't use tidylog** in production code, inside functions, or loops where output would be too verbose. It's for interactive verification only.

## Data Masking vs Tidy Selection

Understand the difference:
- **Data masking functions**: `arrange()`, `filter()`, `mutate()`, `summarise()`
- **Tidy selection functions**: `select()`, `relocate()`, `across()`

**Function arguments - embrace with `{{}}`**

```r
my_summary <- function(data, summary_var) {
  data |>
    summarise(mean_val = mean({{ summary_var }}))
}
```

**Character vectors - use `.data[[]]`**

```r
for (var in names(mtcars)) {
  mtcars |> count(.data[[var]]) |> print()
}
```

**Multiple columns - use `across()`**

```r
data |>
  summarise(across({{ summary_vars }}, \(x) mean(x)))
```

## Modern Grouping and Column Operations

**Use `.by` for per-operation grouping (dplyr 1.1+)**

This replaces the old `group_by() |> ... |> ungroup()` pattern.

Additional modern operations:
- `pick()` - Column selection inside data-masking functions
- `across()` - Apply functions to multiple columns
- `reframe()` - Multi-row summaries

See [grouping-examples.md](references/grouping-examples.md) for complete examples.

## String Manipulation with stringr

**Use stringr over base R string functions**

Benefits:
- Consistent `str_` prefix
- String-first argument order
- Pipe-friendly and vectorized

See [stringr-examples.md](references/stringr-examples.md) for common patterns and base R equivalents.

## Style Guide Essentials

### Object Names

- **Use snake_case for all names**
- **Variable names = nouns, function names = verbs**
- **Avoid dots except for S3 methods**

Good: `day_one`, `calculate_mean`, `user_data`
Avoid: `DayOne`, `calculate.mean`, `userData`

### Naming and Arguments

- Use snake_case for variables and functions
- Prefix non-standard arguments with `.` (e.g., `.data`, `.by`)

### Comments

- Comments should explain **why**, not **what**
- Don't restate what the code already says clearly
- Use comments for non-obvious decisions, workarounds, or very concise code

```r
# Good: explains why
# Skip NA values because downstream analysis requires complete cases
data <- data |> filter(!is.na(value))

# Avoid: restates what code does
# Filter out NA values
data <- data |> filter(!is.na(value))
```

### Coding style

See [tidyverse-style.md](references/tidyverse-style.md) for full style guide summary.

## Anti-Patterns to Avoid

### Legacy Patterns

| Avoid | Use Instead |
|-------|-------------|
| `%>%` | `|>` |
| `function(x)` or `~` | `\(x)` |
| `by = c("a" = "b")` | `by = join_by(a == b)` |
| `sapply()` | `map_*()` (type-stable) |
| `group_by() |> ... |> ungroup()` | `.by` argument |
| `cat()` for messages | `message()` or `cli::cli_inform()` |
| `stop()` for errors | `cli::cli_abort()` |
| `distinct(id)` | `distinct(id, .keep_all = TRUE)` |
| `mean(x, na.rm = TRUE)` | `mean(x)` with tidyna loaded |
| `!!paste0("new_", var)` | `across()` for dynamic columns |

### Performance Anti-Patterns

- **Don't grow objects in loops** - Pre-allocate or use `map()`
- **Don't use `sapply()`** - Type-unstable; use `map_dbl()`, `map_chr()`, etc.
- **Use vectorized operations** - `x + y` instead of element-wise loops

## Migration Reference

### Base R to Modern Tidyverse

| Base R | Modern Tidyverse |
|--------|------------------|
| `subset(data, condition)` | `filter(data, condition)` |
| `data[order(data$x), ]` | `arrange(data, x)` |
| `aggregate(x ~ y, data, mean)` | `summarise(data, mean(x), .by = y)` |
| `sapply(x, f)` | `map(x, f)` |
| `grepl("pattern", text)` | `str_detect(text, "pattern")` |
| `gsub("old", "new", text)` | `str_replace_all(text, "old", "new")` |

### Old to New Tidyverse Patterns

| Old Pattern | New Pattern |
|-------------|-------------|
| `data %>% function()` | `data |> function()` |
| `group_by(x) |> summarise() |> ungroup()` | `summarise(..., .by = x)` |
| `by = c("a" = "b")` | `by = join_by(a == b)` |
| `gather()/spread()` | `pivot_longer()/pivot_wider()` |
| `map_dfr(x, f)` | `map(x, f) |> list_rbind()` |
| `separate(col, into = ...)` | `separate_wider_delim()` |

See [migration-examples.md](references/migration-examples.md) for complete migration patterns.

