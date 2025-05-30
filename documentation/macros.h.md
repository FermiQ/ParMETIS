# libparmetis/macros.h

## Overview

This header file defines a collection of C preprocessor macros used throughout the ParMETIS library. These macros provide convenient shorthands for common operations, including bitwise operations, random number generation (though `HASHFCT` is more of a hashing aid), workspace management for memory, timer operations, and debugging assertions.

## Key Components

### Functions/Classes/Modules/Macros/Structs

This file primarily consists of `#define` directives for macros.

*   **Bitwise Operations**:
    *   `AND(a, b)`: Computes bitwise AND, handling potential negative `a` by operating on its absolute value.
    *   `OR(a, b)`: Computes bitwise OR, handling potential negative `a`.
    *   `XOR(a, b)`: Computes bitwise XOR, handling potential negative `a`.
    *   *(Note: The handling of negative numbers in these bitwise macros is unusual and might be specific to a particular need or could be problematic if standard two's complement behavior is expected for negative inputs directly in bitwise ops.)*

*   **Hashing Aid**:
    *   `HASHFCT(key, size)`: Computes `(key) % (size)`, a common way to map a key to an index in a hash table of `size`.

*   **Workspace Core Management (GKlib mcore)**:
    *   `WCOREPUSH`: Pushes the current state of the memory core/allocator (`ctrl->mcore`). Used to create a new allocation scope.
    *   `WCOREPOP`: Pops the memory core state, effectively freeing memory allocated since the corresponding `WCOREPUSH`.
    *   *(These rely on `ctrl->mcore` being a valid `gk_mcore_t*` from GKlib.)*

*   **Timer Macros**:
    *   `cleartimer(tmr)`: Sets timer `tmr` to `0.0`.
    *   `starttimer(tmr)`: Starts/resumes a timer by subtracting the current `MPI_Wtime()`.
    *   `stoptimer(tmr)`: Stops/pauses a timer by adding the current `MPI_Wtime()`. The duration is `tmr_after_stop - tmr_after_start`.
    *   `gettimer(tmr)`: Returns the current value of timer `tmr`.
    *   `STARTTIMER(ctrl, tmr)`: A wrapper that, if `DBG_TIME` is set in `ctrl->dbglvl`, optionally barriers and then calls `starttimer(tmr)`.
    *   `STOPTIMER(ctrl, tmr)`: A wrapper that, if `DBG_TIME` is set, optionally barriers and then calls `stoptimer(tmr)`.

*   **Debugging Assertion Macros**:
    *   `PASSERT(ctrl, expr)`: If `NDEBUG` is not defined, this macro checks if `expr` is true. If false, it prints an assertion failure message (including file and line number, and the expression itself) using `myprintf` (a ParMETIS utility, likely printing from PE 0) and then calls `assert(expr)` which would typically terminate the program. If `NDEBUG` is defined, the assertion is compiled out to nothing.
    *   `PASSERTP(ctrl, expr, msg)`: Similar to `PASSERT`, but allows for an additional custom message `msg` (which should be in `myprintf` format, e.g., `(ctrl, "Value was %d", val)`) to be printed upon assertion failure.

*   **Boundary List Macros (for vertex boundary management)**:
    *   `BNDInsert(nbnd, bndind, bndptr, vtx)`: Inserts vertex `vtx` into a boundary list.
        *   `bndind`: An array storing the vertex indices in the boundary list.
        *   `bndptr`: An array mapping a vertex index to its position in `bndind`.
        *   `nbnd`: The current number of elements in the boundary list (incremented by the macro).
    *   `BNDDelete(nbnd, bndind, bndptr, vtx)`: Deletes vertex `vtx` from a boundary list. It does this by swapping the target vertex with the last vertex in `bndind`, decrementing `nbnd`, and updating `bndptr`.

## Important Variables/Constants

*   `ctrl->mcore`: Used by `WCOREPUSH`/`WCOREPOP`, expected to be a GKlib memory core object.
*   `ctrl->dbglvl`: Used by `STARTTIMER`/`STOPTIMER` and assertion macros to control behavior.
*   `DBG_TIME`: A debug flag constant (defined in `defs.h` or `parmetis.h`).
*   `NDEBUG`: Standard C preprocessor macro; if defined, assertions are disabled.

## Usage Examples

```c
// Timer usage:
// cleartimer(ctrl->SomeTimer);
// STARTTIMER(ctrl, ctrl->SomeTimer);
// /* ... code to time ... */
// STOPTIMER(ctrl, ctrl->SomeTimer);
// printf("Time taken: %f\n", gettimer(ctrl->SomeTimer));

// Assertion usage:
// PASSERT(ctrl, nvtxs > 0);
// PASSERTP(ctrl, balance[0] < 1.5, (ctrl, "Imbalance too high: %.2f", balance[0]));

// Workspace usage:
// WCOREPUSH;
// my_temp_array = iwspacemalloc(ctrl, 100);
// /* ... use my_temp_array ... */
// WCOREPOP; // my_temp_array is effectively freed here

// Boundary list (conceptual):
// idx_t nbnd = 0, *bndind, *bndptr;
// /* ... allocate bndind and bndptr ... */
// BNDInsert(nbnd, bndind, bndptr, new_boundary_vertex);
// BNDDelete(nbnd, bndind, bndptr, old_boundary_vertex);
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Likely includes this file. It would provide `ctrl_t`, `idx_t`, `myprintf`, MPI wrappers (`gkMPI_Barrier`, `MPI_Wtime`), and GKlib memory core types/functions if `WCOREPUSH`/`POP` are used with GKlib's `gk_mcore`.
    *   `defs.h` or `parmetis.h`: For debug flag constants like `DBG_TIME`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: `MPI_Wtime()` is used by the timer macros. `gkMPI_Barrier` is also used.
    *   Standard C `assert.h`: For the `assert()` call within `PASSERT`/`PASSERTP`.
*   **Other Interactions**:
    *   The assertion macros are critical for debugging and ensuring preconditions or invariants hold during execution. They are typically compiled out in release builds (when `NDEBUG` is defined) to avoid performance overhead.
    *   Timer macros help in performance profiling.
    *   Workspace macros simplify scoped memory allocation if using GKlib's mcore facility.

```
