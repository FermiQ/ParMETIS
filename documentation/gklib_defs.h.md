# libparmetis/gklib_defs.h

## Overview

This header file serves as a central point for defining data structures and declaring function prototypes related to the embedded GKlib (George Karypis Library) functionality within ParMETIS. It leverages GKlib's template macros to define custom key-value and priority queue types specific to ParMETIS's needs (using `idx_t` and `real_t`). It also lists the prototypes for the various BLAS-like, memory allocation, sorting, random number, and priority queue functions that are implemented in `gklib.c`.

## Key Components

### Functions/Classes/Modules/Macros/Structs

*   **Structure Definitions**:
    *   `uvw_t`: A structure to store a weighted edge, with fields `u`, `v` (vertex indices) and `w` (weight). All are of type `idx_t`.
    *   `GK_MKKEYVALUE_T(ikv_t, idx_t, idx_t)`: Macro instantiation defining `ikv_t` as a key-value pair structure where both key and value are `idx_t`.
    *   `GK_MKKEYVALUE_T(rkv_t, real_t, idx_t)`: Macro instantiation defining `rkv_t` as a key-value pair structure where the key is `real_t` and the value is `idx_t`.
    *   `GK_MKPQUEUE_T(ipq_t, ikv_t)`: Macro instantiation defining `ipq_t` as the type for a priority queue holding `ikv_t` elements.
    *   `GK_MKPQUEUE_T(rpq_t, rkv_t)`: Macro instantiation defining `rpq_t` as the type for a priority queue holding `rkv_t` elements.

*   **Function Prototypes**:
    *   The file lists prototypes for a comprehensive set of utility functions. These functions are implemented in `gklib.c` using GKlib's template mechanism. The prototypes here use the renamed versions (e.g., `libparmetis__iargmax` via `gklib_rename.h`).
    *   **BLAS-like functions**:
        *   For `idx_t`: `iargmax`, `iargmin`, `iaxpy`, `icopy`, `idot`, `iincset`, `imax`, `imin`, `inorm2`, `iscale`, `iset`, `isum`. (Prototypes via `GK_MKBLAS_PROTO(i, idx_t, idx_t)`).
        *   For `real_t`: `rargmax`, `rargmin`, `raxpy`, `rcopy`, `rdot`, `rincset`, `rmax`, `rmin`, `rnorm2`, `rscale`, `rset`, `rsum`. (Prototypes via `GK_MKBLAS_PROTO(r, real_t, real_t)`).
    *   **Memory Allocation functions**:
        *   For `idx_t`: `imalloc`, `ismalloc`, `irealloc`, `iAllocMatrix`, `iFreeMatrix`, `iSetMatrix`. (Prototypes via `GK_MKALLOC_PROTO(i, idx_t)`).
        *   For `real_t`: `rmalloc`, `rsmalloc`, `rrealloc`, etc. (Prototypes via `GK_MKALLOC_PROTO(r, real_t)`).
        *   For `ikv_t`: `ikvmalloc`, `ikvsmalloc`, etc. (Prototypes via `GK_MKALLOC_PROTO(ikv, ikv_t)`).
        *   For `rkv_t`: `rkvmalloc`, `rkvsmalloc`, etc. (Prototypes via `GK_MKALLOC_PROTO(rkv, rkv_t)`).
    *   **Priority Queue functions**:
        *   For `ipq_t` (holding `ikv_t`): `ipqCreate`, `ipqDestroy`, `ipqInit`, `ipqInsert`, `ipqDelete`, `ipqUpdate`, `ipqGetTop`, `ipqLength`, etc. (Prototypes via `GK_MKPQUEUE_PROTO(ipq, ipq_t, idx_t, idx_t)`).
        *   For `rpq_t` (holding `rkv_t`): `rpqCreate`, `rpqDestroy`, etc. (Prototypes via `GK_MKPQUEUE_PROTO(rpq, rpq_t, real_t, idx_t)`).
    *   **Random Number functions**:
        *   For `idx_t`: `isrand`, `irand`, `irandArrayPermute`, `irandInRange`. (Prototypes via `GK_MKRANDOM_PROTO(i, idx_t, idx_t)`).
    *   **Utility functions**:
        *   `iarray2csr`. (Prototype via `GK_MKARRAY2CSR_PROTO(i, idx_t)`).
    *   **Sorting functions**:
        *   `isorti`, `isortd` (for `idx_t`).
        *   `rsorti`, `rsortd` (for `real_t`).
        *   `ikvsorti`, `ikvsortii`, `ikvsortd` (for `ikv_t`).
        *   `rkvsorti`, `rkvsortd` (for `rkv_t`).
        *   `uvwsorti` (for `uvw_t`).

## Important Variables/Constants

This header file itself does not define many constants, but it relies on `idx_t` and `real_t` which are fundamental types defined elsewhere (typically in `parmetislib.h` or its included headers like `metis.h`). The GKlib macros like `GK_MKKEYVALUE_T` and `GK_MKPQUEUE_T` are used to declare new types.

## Usage Examples

```c
// This is a header file. It's included by other .c files in ParMETIS.
// Example of how it's used (in another .c file):

#include "parmetislib.h" // This would include gklib_defs.h

// ... later in the code ...
// ikv_t *my_pairs = ikvmalloc(10, "my_pairs_array"); // Uses prototype from gklib_defs.h
// iset(params->n, (idx_t)-1, where); // Uses prototype from gklib_defs.h for iset
// ...
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `gklib_rename.h`: This header is included first. It contains `#define` directives that rename all the GKlib function names to have a `libparmetis__` prefix (e.g., `iset` becomes `libparmetis__iset`). The prototypes in `gklib_defs.h` then declare these renamed functions.
    *   The actual implementations of these prototyped functions are found in `gklib.c`.
    *   This header is a core part of `parmetislib.h`, which is the main internal header for the library.
    *   Relies on `idx_t` and `real_t` type definitions from `metis.h` (included via `parmetislib.h`).
    *   The GKlib template macros (`GK_MKKEYVALUE_T`, `GK_MKPQUEUE_T`, `GK_MKBLAS_PROTO`, etc.) are defined externally in the GKlib system, but their usage here is to generate ParMETIS-specific code.
*   **External Libraries**:
    *   None directly.
*   **Other Interactions**:
    *   This file is crucial for the internal organization of ParMETIS, providing a clear interface to the embedded GKlib utility functions.
    *   The renaming strategy (via `gklib_rename.h`) is essential for preventing symbol clashes when ParMETIS is linked into larger applications that might also use GKlib independently.

```
