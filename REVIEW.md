# Code Review: Yuuqq/interplot

## Overview
This review summarizes the findings of a code review for the `interplot` package. The findings are categorized into CRITICAL, HIGH, MEDIUM, and LOW severity based on their impact on security, maintainability, and reliability.

## Findings

### MEDIUM

#### 1. Arbitrary Code Execution (Injection) via `eval(parse(text = ...))`
- **Location**: Multiple files including `R/Interplot_mi.R` (e.g., lines 104, 110, 168), `R/Interplot_default.R` (e.g., lines 109, 110, 161, 257), `R/Interplot_mlm.R`, and `R/Interplot_mlmmi.R`.
- **Impact**: The package makes extensive use of `eval(parse(text = paste0("m$model$", var1)))` to extract data from the model object. While this pattern could theoretically lead to arbitrary R code execution, the real-world risk is low (threat model exaggeration) because this is a local plotting package and users already have full R execution capabilities. However, it still poses a robustness issue: special characters in variable names will break the function.
- **Fix Suggestion**: Avoid `eval(parse(...))`. Extract variables safely using list subsetting or environment extraction. For example, use `m$model[[var1]]` or `m.frame[[var1]]` where `m.frame <- model.frame(m)`. To get the number of levels for factors, you can use `length(levels(m$model[[var1]]))` instead of parsing text.

### HIGH

#### 2. Missing Automated Test Suite and CI/CD
- **Location**: Entire Repository (missing `tests/` directory and `.github/workflows/` directory).
- **Impact**: There is no standard R test infrastructure (like `testthat`) and no CI pipeline (like R-CMD-check via GitHub Actions). The scripts in the `dev/` folder (e.g., `dev/interplot_test.R`) appear to be manual testing scripts. This makes regressions highly likely during future development and refactoring.
- **Fix Suggestion**: Initialize the `testthat` framework by running `usethis::use_testthat()`. Convert the scripts in `dev/` into formal unit tests under `tests/testthat/`. Add a standard R-CMD-check GitHub Actions workflow (`usethis::use_github_action("check-standard")`).

#### 3. Severe Code Duplication Across S3 Methods
- **Location**: `R/Interplot_default.R`, `R/Interplot_mi.R`, `R/Interplot_mlm.R`, `R/Interplot_mlmmi.R`.
- **Impact**: There are thousands of lines of nearly identical code copied across the different S3 methods. For example, the simulation logic, data preparation, and histogram rendering logic are largely duplicated. This makes the code extremely hard to maintain; a bug fix in one place must be manually copied to several others.
- **Fix Suggestion**: Refactor the codebase by extracting common functionality into internal helper functions (e.g., `prepare_plot_data()`, `simulate_coefs()`). The S3 methods should only handle model-specific extraction (like extracting fixed/random effects or combining multiply imputed models) and then pass the standard data structure to a shared plotting or simulation function.

### MEDIUM

#### 4. Brittle S3 Dispatch and Class Checks
- **Location**: `R/Interplot.R` (lines 93-109)
- **Impact**: The main generic function `interplot()` manually checks `class(m)[1]` and mutates the class of the input object (`class(m) <- "mlmmi"`, etc.) before calling `UseMethod("interplot", m)`. This is generally an anti-pattern in R. It modifies the user's object under the hood and relies on strict class ordering, which can break if a model inherits from multiple classes or if `mitools` lists change structure.
- **Fix Suggestion**: Instead of mutating the class of `m`, define specific methods (e.g., `interplot.list`) and do the dispatch internally, or define a lightweight wrapper class (e.g., `structure(m, class = c("interplot_mi_list", class(m)))`) instead of overwriting the class entirely. Also, replace `class(m)[1] == "list"` with `inherits(m, "list")`.

#### 5. Unsafe Class Comparisons and Warning Conditions
- **Location**: `R/Interplot_mi.R` (line 88), `R/Interplot_default.R` (line 104)
- **Impact**: The checks `if(predPro == TRUE & class(m) == "lmmi")` and `if (predPro == TRUE & all(m.class == "lm"))` are somewhat unsafe. Note that `all(m.class == "lm")` will actually not produce a length > 1 warning, but suggesting `any(... == "lm")` could misidentify `glm` models and accidentally disable `predPro`. It is better to use `inherits(m, "lmmi")` to correctly and robustly identify object classes.
- **Fix Suggestion**: Use `inherits(m, "lmmi")` or `any(class(m) == "lmmi")` instead of direct equality comparisons.

### LOW

#### 6. Error Handling / Missing Context
- **Location**: `R/Interplot_default.R` (lines 105, 113, 116, 244, etc.)
- **Impact**: The functions use `stop("The function does not support interactions between two factors.")` directly. While clear, using `stop()` without a specific class or call context can be less developer-friendly.
- **Fix Suggestion**: Use `cli::cli_abort()` or `rlang::abort()` for more informative and visually clear error messages, or explicitly set `call. = FALSE` in `stop()` to hide the internal stack trace from the user.

#### 7. Outdated Global Variable Bindings in `interplot.plot`
- **Location**: `R/Interplot_plot.R` (Note: dummy variables like `ymin <- ymax <- vector()` are declared)
- **Impact**: The function declares dummy variables like `ymin <- ymax <- vector()` to avoid "no visible binding for global variable" issues in `ggplot2`. This clutters the environment.
- **Fix Suggestion**: A cleaner R package convention is to use `.data$ymin` within `aes()` (from `rlang`), or declare `utils::globalVariables(c("ymin", "ymax", "fake", "coef1", "lb", "ub"))` once in a separate `globals.R` file (note that the package already uses `globalVariables()` in some files).
