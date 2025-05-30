# libparmetis/gklib.c

## Overview

This file is an integration point for GKlib (George Karypis Library) utilities within ParMETIS. GKlib is a separate library containing a rich set of low-level routines for memory allocation, array operations (BLAS-like), sorting, random number generation, priority queues, and more. Instead of directly including GKlib source or linking against it as a separate entity, ParMETIS incorporates a customized, templated version of these utilities directly into its codebase. This file uses preprocessor macros (GK_MK*) defined in GKlib's template system to generate concrete implementations of these utilities for specific data types used in ParMETIS (like `idx_t`, `real_t`, `ikv_t`, `rkv_t`).

## Key Components

### Functions/Classes/Modules

This file primarily consists of instantiations of GKlib macros that generate functions.

*   **BLAS Routines**:
    *   `GK_MKBLAS(i, idx_t, idx_t)`: Generates BLAS-like functions for `idx_t` arrays (e.g., `icopy`, `iset`, `isum`, `imax`, `imin`, `iaxpy`, `idot`, `iscale`, `inorm2`, `iargmax`, `iargmin`).
    *   `GK_MKBLAS(r, real_t, real_t)`: Generates BLAS-like functions for `real_t` arrays (e.g., `rcopy`, `rset`, `rsum`, `rmax`, `rmin`, `raxpy`, `rdot`, `rscale`, `rnorm2`, `rargmax`, `rargmin`).
*   **Memory Allocation Routines**:
    *   `GK_MKALLOC(i, idx_t)`: Generates memory allocation/deallocation functions for `idx_t` (e.g., `imalloc`, `ismalloc`, `irealloc`, `iAllocMatrix`, `iFreeMatrix`, `iSetMatrix`).
    *   `GK_MKALLOC(r, real_t)`: Generates memory allocation functions for `real_t` (e.g., `rmalloc`, `rsmalloc`, `rrealloc`).
    *   `GK_MKALLOC(ikv, ikv_t)`: Generates memory allocation functions for `ikv_t` (key-value pairs with `idx_t`).
    *   `GK_MKALLOC(rkv, rkv_t)`: Generates memory allocation functions for `rkv_t` (key-value pairs with `real_t` key, `idx_t` value).
*   **Priority Queue Routines**:
    *   `GK_MKPQUEUE(ipq, ipq_t, ikv_t, idx_t, idx_t, ikvmalloc, IDX_MAX, key_gt)`: Generates functions for managing priority queues (`ipq_t`) storing `ikv_t` items, ordered by `idx_t` keys. Examples: `ipqCreate`, `ipqInsert`, `ipqGetTop`, `ipqDelete`, `ipqUpdate`, `ipqDestroy`.
    *   `GK_MKPQUEUE(rpq, rpq_t, rkv_t, real_t, idx_t, rkvmalloc, REAL_MAX, key_gt)`: Generates functions for priority queues (`rpq_t`) storing `rkv_t` items, ordered by `real_t` keys. Examples: `rpqCreate`, `rpqInsert`, `rpqGetTop`, etc.
*   **Random Number Generation Routines**:
    *   `GK_MKRANDOM(i, idx_t, idx_t)`: Generates random number functions for `idx_t` (e.g., `isrand`, `irand`, `irandArrayPermute`, `irandInRange`).
*   **Utility Routines**:
    *   `GK_MKARRAY2CSR(i, idx_t)`: Generates `iarray2csr` for converting an array representation to CSR format for `idx_t`.
*   **Sorting Routines**:
    *   Explicitly defined qsort-based functions (not using GK_MKQSORT macro directly in this file, but the macro's underlying logic is used):
        *   `isorti`, `isortd`: Sort `idx_t` arrays (increasing, decreasing).
        *   `rsorti`, `rsortd`: Sort `real_t` arrays (increasing, decreasing).
        *   `ikvsorti`, `ikvsortii`, `ikvsortd`: Sort `ikv_t` arrays by key (increasing, increasing by key then value, decreasing).
        *   `rkvsorti`, `rkvsortd`: Sort `rkv_t` arrays by key (increasing, decreasing).
        *   `uvwsorti`: Sort `uvw_t` arrays (custom struct, by u then v).

## Important Variables/Constants

*   `IDX_MAX`, `REAL_MAX`: Used as initial maximum values in priority queue implementations.
*   Comparison macros like `key_gt`, `i_lt`, `r_lt`, `ikey_lt`, `ikeyval_lt`, `rkey_lt`, `uvwkey_lt` are defined locally for use with the sorting and priority queue generation.

## Usage Examples

```c
// These functions are used extensively throughout ParMETIS.
// Example (conceptual):
// idx_t *myarray = imalloc(100, "myarray");
// iset(100, 0, myarray); // Set all elements to 0
// idx_t sum_val = isum(100, myarray, 1); // Sum elements with stride 1
// isorti(100, myarray); // Sort the array
// gk_free((void **)&myarray, LTERM); // Free memory (gk_free is the GKlib memory wrapper)
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: This is the primary include. It, in turn, includes:
        *   `gklib_defs.h`: Contains typedefs for `ikv_t`, `rkv_t`, `uvw_t`, `ipq_t`, `rpq_t` and prototypes for the functions generated/defined in this `gklib.c` file. These prototypes use the renamed versions (e.g., `libparmetis__iset`).
        *   `gklib_rename.h`: Contains macros that rename the GKlib functions (e.g., `iset` becomes `libparmetis__iset`) to avoid symbol clashes if ParMETIS is linked with another library that also uses GKlib. The implementations in `gklib.c` define the `libparmetis__*` versions.
        *   Standard C headers like `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<stdarg.h>`, `<math.h>` are included via `parmetislib.h` (often through GKlib's own header structure which is partially replicated/adapted in ParMETIS).
*   **External Libraries**:
    *   None directly. The necessary GKlib functionality is compiled as part of ParMETIS itself from these templated definitions.
*   **Other Interactions**:
    *   This file provides the fundamental building blocks (memory management, array operations, sorting, random numbers, priority queues) that are used by almost all other modules in ParMETIS.
    *   The renaming scheme (`libparmetis__*`) is crucial for robust linking and avoiding symbol collisions in larger applications.

```
