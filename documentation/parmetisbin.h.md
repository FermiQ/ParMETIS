# programs/parmetisbin.h

## Overview

This header file serves as a common include file for the executable programs (test drivers, command-line tools) in the ParMETIS `programs` directory, such as `parmetis`, `mtest`, `ptest`, `otest`, and `dglpart`. It bundles essential headers from both the ParMETIS library (`libparmetis`) and standard libraries, providing a convenient way to include all necessary definitions and declarations for these programs.

## Key Components

This file's main role is to include other header files.

### Included Library Headers:

*   **`GKlib.h`**: The main header for George Karypis's GKlib. This provides low-level utilities for memory management, sorting, data structures, etc., which are used by both ParMETIS library internals and potentially the test programs themselves.
*   **`parmetis.h`**: The public API header for the ParMETIS library. This defines the functions that users (and these test programs) call (e.g., `ParMETIS_V3_PartKway`), as well as user-visible constants and type definitions (like `idx_t`, `real_t`).

### Included Internal ParMETIS Headers (from `libparmetis`):

These headers provide access to internal data structures, constants, macros, and function prototypes of the ParMETIS library, which might be needed by more sophisticated test programs or drivers that interact closely with internal components (though direct use of internal functions is generally for testing/debugging).

*   **`gklib_defs.h`**: ParMETIS-specific header for GKlib data structures and renamed function prototypes.
*   **`rename.h`**: Macros to rename internal ParMETIS functions (e.g., `Match_Global` to `libparmetis__Match_Global`) to avoid symbol clashes. This implies the test programs might also be compiled with these renames active if they call internal functions directly.
*   **`defs.h`**: Global constant definitions used throughout ParMETIS (default parameters, algorithm thresholds, debug flags).
*   **`struct.h`**: Core internal ParMETIS data structures (`ctrl_t`, `graph_t`, `mesh_t`, etc.).
*   **`macros.h`**: Utility macros (timers, assertions, workspace management).
*   **`../libparmetis/proto.h`**: Function prototypes for internal C functions within the `libparmetis` directory. (Note the relative path `../libparmetis/` which is correct as this header is in `programs/`).

### Included Program-Specific Header:

*   **`proto.h`**: This refers to `programs/proto.h`, which would contain function prototypes for functions defined within the `programs` directory itself (e.g., I/O routines in `programs/io.c`, test utility functions).

### Constants
*   **`MAXNCON`**: Defines the maximum number of constraints to be 32. This is likely used for statically allocating arrays related to constraints in the test programs.

### Preprocessor Defines (Example):
```c
/*
#define DEBUG			1
#define DMALLOC			1
*/
```
These lines (commented out by default) suggest options for enabling general debugging flags or a specific dynamic memory allocation library (DMALLOC) during development of the test programs or the library.

## Important Variables/Constants

*   **`MAXNCON`**: Maximum number of constraints supported by arrays sized with this constant in the test programs.
*   Other constants and types are inherited from the included headers (e.g., `idx_t`, `real_t` from `parmetis.h`; various algorithm parameters from `libparmetis/defs.h`).

## Usage Examples

```c
// In any ParMETIS program .c file (e.g., parmetis.c, otest.c):
#include <parmetisbin.h> // Note: typically just "parmetisbin.h" if include paths are set up

// After this include, the .c file has access to:
// - ParMETIS API functions, types, and constants.
// - MPI functions (via parmetis.h or GKlib.h).
// - GKlib utility functions.
// - Internal ParMETIS structures, constants, and macros (use with caution, primarily for testing).
// - Prototypes for other functions within the programs directory (via programs/proto.h).
```

## Dependencies and Interactions

*   **Internal Dependencies (ParMETIS library)**:
    *   Relies heavily on the headers from the `libparmetis` directory.
    *   The test programs built using this header link against the compiled ParMETIS library.
*   **Internal Dependencies (programs directory)**:
    *   Includes `proto.h` from its own directory for inter-program-module function calls.
*   **External Libraries**:
    *   `MPI`: Included via `parmetis.h`.
    *   `GKlib`: Included via `GKlib.h`. The ParMETIS build system must provide these.
*   **Other Interactions**:
    *   This header file is a key part of building the executable programs provided with ParMETIS.
    *   It ensures that these programs have a consistent view of all necessary definitions from the ParMETIS library and its dependencies.

```
