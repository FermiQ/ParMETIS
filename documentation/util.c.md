# libparmetis/util.c

## Overview

This file provides a collection of general-purpose utility functions used throughout the ParMETIS library. These include custom printing functions, a binary search implementation, random permutation generators, mathematical helper functions (like `ispow2`, `log2Int`), array utility functions (argmax, average), and specialized balance checking routines for partitioning.

## Key Components

### Functions/Classes/Modules

*   **`myprintf(ctrl_t *ctrl, char *f_str, ...)`**:
    *   Prints a formatted string to `stdout`, prepended with the MPI rank of the calling processor (`[mype]`).
    *   Ensures a newline is printed and flushes `stdout`.
*   **`rprintf(ctrl_t *ctrl, char *f_str, ...)`**:
    *   Prints a formatted string to `stdout` only from PE 0.
    *   Includes an `MPI_Barrier` after printing to synchronize output or ensure all PEs have passed that point.
*   **`BSearch(idx_t n, idx_t *array, idx_t key)`**:
    *   Performs a binary search for `key` in a sorted `array` of size `n`.
    *   Returns the index of `key` if found, otherwise calls `errexit`.
    *   It has a small optimization: for the last 8 elements, it switches to a linear scan.
*   **`RandomPermute(idx_t n, idx_t *p, idx_t flag)`**:
    *   Randomly permutes the array `p` of size `n`.
    *   If `flag == 1`, initializes `p` to `0, 1, ..., n-1` before permuting.
    *   Uses `RandomInRange` and swaps elements multiple times (number of swaps is `n`).
*   **`FastRandomPermute(idx_t n, idx_t *p, idx_t flag)`**:
    *   A potentially faster version of `RandomPermute` for `n >= 25`.
    *   If `n < 25`, calls `RandomPermute`.
    *   Otherwise, permutes elements in chunks of 4 in each iteration.
*   **`ispow2(idx_t a)`**:
    *   Returns 1 if `a` is a power of 2, 0 otherwise.
*   **`log2Int(idx_t a)`**:
    *   Returns floor(log2(`a`)).
*   **`rargmax_strd(size_t n, real_t *x, size_t incx)`**:
    *   Returns the index of the maximum element in a `real_t` array `x` with stride `incx`.
*   **`rargmin_strd(size_t n, real_t *x, size_t incx)`**:
    *   Returns the index of the minimum element in a `real_t` array `x` with stride `incx`.
*   **`rargmax2(size_t n, real_t *x)`**:
    *   Returns the index of the second largest element in a `real_t` array `x`.
*   **`ravg(size_t n, real_t *x)`**:
    *   Computes the average of elements in a `real_t` array `x`.
*   **`rfavg(size_t n, real_t *x)`**:
    *   Computes the average of the absolute values of elements in a `real_t` array `x`.
*   **`BetterVBalance(idx_t ncon, real_t *vwgt, real_t *u1wgt, real_t *u2wgt)`**:
    *   Checks if collapsing vertex `v` (with weights `vwgt`) with vertex `u2` (weights `u2wgt`) results in a better "vector balance" for the combined vertex than collapsing `v` with `u1` (weights `u1wgt`).
    *   "Better balance" here means a smaller sum of absolute differences from the mean of combined component weights. Returns `diff1 - diff2`.
*   **`IsHBalanceBetterFT(idx_t ncon, real_t *pfrom, real_t *pto, real_t *nvwgt, real_t *ubvec)`**:
    *   Checks if moving a vertex with weights `nvwgt` from partition `pfrom` to partition `pto` improves overall balance.
    *   Compares the two largest scaled imbalances (`max_weight_component / ub_for_component`) before and after the hypothetical move. If the new primary imbalance is smaller, or if primary is same and secondary is smaller, or if both are same and sum of imbalances is smaller, returns 1.
*   **`IsHBalanceBetterTT(idx_t ncon, real_t *pt1, real_t *pt2, real_t *nvwgt, real_t *ubvec)`**:
    *   Compares two potential target partitions, `pt1` and `pt2`, for a vertex with weights `nvwgt`.
    *   Returns 1 if moving to `pt2` would result in a better balance state than moving to `pt1`. Similar comparison logic to `IsHBalanceBetterFT`.
*   **`GetThreeMax(idx_t n, real_t *x, idx_t *first, idx_t *second, idx_t *third)`**:
    *   Finds the indices of the three largest elements in a `real_t` array `x`. Stores them in `*first`, `*second`, `*third`.

## Important Variables/Constants

No global variables are defined in this file. Constants are typically from included headers like `defs.h`.

## Usage Examples

```c
// myprintf(ctrl, "Debug message: value is %"PRIDX"\n", my_value);
//
// idx_t perm[100];
// RandomPermute(100, perm, 1); // Initialize and shuffle perm
//
// if (ispow2(npes)) { /* ... */ }
//
// real_t weights[3] = {0.2, 0.5, 0.3};
// idx_t max_idx = rargmax_strd(3, weights, 1);
```
These functions are called internally by many different ParMETIS modules for diverse purposes like debugging output, randomization, index searching, and balance heuristic checks.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `idx_t`, `real_t` types, `errexit`, `RandomInRange` (from GKlib), `gk_SWAP` (from GKlib), `gkMPI_Barrier`.
    *   The balance checking functions (`BetterVBalance`, `IsHBalanceBetterFT`, `IsHBalanceBetterTT`) are crucial for refinement algorithms.
*   **External Libraries**:
    *   Standard C library: `stdio.h` (for `fprintf`, `vfprintf`, `fflush`), `string.h` (for `strlen`), `stdarg.h` (for variadic functions), `math.h` (for `fabs`).
    *   `MPI (Message Passing Interface)`: `gkMPI_Barrier` is used in `rprintf`.
*   **Other Interactions**:
    *   `myprintf` and `rprintf` are the standard ways of producing formatted output in ParMETIS, with control over which PEs print.
    *   The random permutation functions are essential for introducing randomness in various algorithms (e.g., initial partitioning, refinement).

```
