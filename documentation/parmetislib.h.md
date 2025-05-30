# libparmetis/parmetislib.h

## Overview

This is the primary internal header file for the ParMETIS library. It acts as a central include point that brings together all necessary definitions, structures, macros, and function prototypes required by the various modules within ParMETIS. By including `parmetislib.h`, a ParMETIS source file gains access to MPI, GKlib (via its ParMETIS-specific integration headers), the public ParMETIS API (`parmetis.h`), internal constant definitions, data structure definitions, utility macros, and prototypes for almost all internal ParMETIS functions.

## Key Components

This file's main role is to include other header files in the correct order.

### Included Headers:

*   **`mpi.h`**: Standard MPI header for parallel communication.
*   **`GKlib.h`**: The main header for George Karypis's GKlib. In ParMETIS, this is typically a modified or embedded version. It provides low-level utilities for memory management, sorting, data structures, etc.
*   **`parmetis.h`**: The public API header for ParMETIS. This defines the functions that users of the ParMETIS library call (e.g., `ParMETIS_V3_PartKway`), as well as user-visible constants and type definitions (like `idx_t`, `real_t`).
*   **`gklib_defs.h`**: ParMETIS-specific header that defines certain GKlib data structures (like `ikv_t`, `rpq_t`) using GKlib macros and declares prototypes for the ParMETIS-embedded GKlib functions (which are renamed, e.g., `libparmetis__iset`).
*   **`rename.h`**: Contains macros to rename internal ParMETIS functions, often to avoid conflicts with other libraries or to provide different versions (e.g., Fortran wrappers are handled differently, but this `rename.h` might be for internal C function aliasing or versioning if used beyond just GKlib renames). *Self-correction: The primary `rename.h` seen so far (`gklib_rename.h`) is for GKlib symbols. This `rename.h` might be more general or related to other internal aliasing if it's distinct.* Based on `proto.h` also including `rename.h`, it's likely for broader internal use or a legacy aspect.
*   **`defs.h`**: Contains global constant definitions used throughout ParMETIS (e.g., default parameters, algorithm thresholds, debug flags).
*   **`struct.h`**: Defines the core internal data structures used by ParMETIS, most notably `ctrl_t` (control structure) and `graph_t` (graph structure), `NRInfoType`, `cnbr_t`, `matrix_t`, `mesh_t`.
*   **`macros.h`**: Defines utility macros for timers, assertions, workspace management, etc.
*   **`proto.h`**: Contains function prototypes for most of the internal C functions within the `libparmetis` directory. This ensures that functions can call each other across different `.c` files.

### Preprocessor Defines (Example):
```c
/*
#define DEBUG			1
#define DMALLOC			1
*/
```
These lines (commented out by default) suggest options for enabling general debugging flags or a specific dynamic memory allocation library (DMALLOC) during development.

## Important Variables/Constants

This file itself doesn't define variables or constants directly but includes headers that do:
*   `parmetis.h`: User-level constants (e.g., `PARMETIS_OK`, option keys).
*   `defs.h`: Internal algorithm constants and debug flags.
*   `struct.h`: Definitions of `idx_t`, `real_t` (often through `metis.h` included by `parmetis.h`).

## Usage Examples

```c
// In any ParMETIS internal .c file (e.g., kmetis.c, match.c):
#include <parmetislib.h>

// After this include, the .c file has access to:
// - MPI functions
// - GKlib utility functions (renamed, e.g., libparmetis__iset)
// - ParMETIS API definitions (though internal files usually implement, not call, these)
// - Internal data structures like ctrl_t, graph_t
// - Internal constants from defs.h
// - Utility macros from macros.h
// - Prototypes for other internal ParMETIS functions from proto.h
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   This header is the root of most internal includes. It ensures that all necessary definitions are available and in a consistent order.
    *   It relies on all the headers it includes (`GKlib.h`, `parmetis.h`, `gklib_defs.h`, `rename.h`, `defs.h`, `struct.h`, `macros.h`, `proto.h`).
*   **External Libraries**:
    *   `MPI`: Directly includes `mpi.h`.
    *   `GKlib`: Includes `GKlib.h`. The ParMETIS build system must provide access to GKlib's headers and potentially its source (if compiled as part of ParMETIS).
*   **Other Interactions**:
    *   This file is crucial for compiling the ParMETIS library. Any changes to the set of included files or their order could impact the build.
    *   The commented-out `DEBUG` and `DMALLOC` macros suggest build-time configurations for development and debugging.

```
