# libparmetis/gklib_rename.h

## Overview

This header file is a critical part of ParMETIS's integration of GKlib (George Karypis Library). Its sole purpose is to rename the standard GKlib function names using C preprocessor macros. Each GKlib function (e.g., `iset`, `imalloc`, `ikvsorti`) is redefined with a `libparmetis__` prefix (e.g., `libparmetis__iset`, `libparmetis__imalloc`, `libparmetis__ikvsorti`).

This renaming strategy is employed to prevent symbol clashes that could occur if a user application links ParMETIS with another library that also uses GKlib, or if the user links against a system-installed version of GKlib. By prefixing these symbols, ParMETIS ensures that it uses its own embedded versions of these utility functions.

## Key Components

### Functions/Classes/Modules/Macros/Structs

This file consists entirely of `#define` preprocessor directives. There are no function, class, or struct definitions.

Examples of redefinitions:
*   `#define iAllocMatrix libparmetis__iAllocMatrix`
*   `#define iset libparmetis__iset`
*   `#define ikvsorti libparmetis__ikvsorti`
*   `#define imalloc libparmetis__imalloc`
*   `#define ipqCreate libparmetis__ipqCreate`
*   `#define rset libparmetis__rset`
*   `#define rmalloc libparmetis__rmalloc`
*   `#define rpqCreate libparmetis__rpqCreate`
*   `#define isorti libparmetis__isorti`
*   And many more, covering a wide range of GKlib functions for:
    *   Memory allocation and deallocation for various types (`idx_t`, `real_t`, `ikv_t`, `rkv_t`, matrices).
    *   Array manipulation (setting, copying, scaling, dot products, argmax, argmin, norm).
    *   Priority queue operations.
    *   Random number generation.
    *   Sorting for various types and criteria.
    *   CSR conversion utilities.

## Important Variables/Constants

Not applicable, as this file only contains macro definitions for renaming.

## Usage Examples

```c
// This header file is not directly used by end-users.
// It's included by parmetislib.h (typically via gklib_defs.h).

// In gklib_defs.h (or other ParMETIS internal files):
// #include "gklib_rename.h" // This applies the renaming
// ...
// // Now, when a ParMETIS internal file calls iset(...), the preprocessor
// // changes it to libparmetis__iset(...) before compilation.
// void libparmetis__iset(idx_t n, idx_t val, idx_t *array); // Prototype in gklib_defs.h
//
// // In gklib.c:
// void libparmetis__iset(idx_t n, idx_t val, idx_t *array) { // Implementation
//   // ...
// }

```
When ParMETIS source code is compiled, any call to a standard GKlib function name (like `iset`) is replaced by its `libparmetis__` prefixed version. The actual implementation of these functions (in `gklib.c`) also uses these prefixed names.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   This file is included by `gklib_defs.h`, which is part of the main internal header `parmetislib.h`.
    *   It directly affects how function calls to GKlib utilities are compiled within ParMETIS. The function implementations in `gklib.c` must define the `libparmetis__*` prefixed names. The function prototypes in `gklib_defs.h` must also declare these prefixed names.
*   **External Libraries**:
    *   None. This file only contains preprocessor definitions.
*   **Other Interactions**:
    *   This renaming is crucial for avoiding linker errors and ensuring ParMETIS uses its internal version of GKlib utilities, especially when integrated into larger projects that might also use GKlib or have conflicting symbols.
    *   The list of renamed symbols is generated from the `.o` files using a script (`./utils/listundescapedsumbols.csh`), indicating it's a somewhat automated process to keep the renames comprehensive.

```
