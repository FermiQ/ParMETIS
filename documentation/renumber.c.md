# libparmetis/renumber.c

## Overview

This file provides utility functions to switch graph and mesh numbering schemes between 0-based and 1-based indexing. ParMETIS internally uses 0-based indexing, but some users or external tools might provide or expect 1-based indexing. These functions handle the necessary conversions for various graph and mesh data arrays.

## Key Components

### Functions/Classes/Modules

*   **`ChangeNumbering(idx_t *vtxdist, idx_t *xadj, idx_t *adjncy, idx_t *part, idx_t npes, idx_t mype, idx_t from)`**:
    *   Converts graph numbering for standard CSR graph representation.
    *   **Parameters**:
        *   `vtxdist`: Vertex distribution array.
        *   `xadj`, `adjncy`: CSR graph structure arrays.
        *   `part`: Partition assignment array for vertices.
        *   `npes`: Number of processors.
        *   `mype`: Current processor's rank.
        *   `from`: If 1, converts from 1-based to 0-based. If 0, converts from 0-based to 1-based.
    *   **Functionality**:
        *   If `from == 1` (1-based to 0-based):
            *   Decrements elements of `vtxdist`.
            *   Decrements elements of `xadj`.
            *   Decrements elements of `adjncy`.
            *   The `part` array is NOT modified by this function (though it's a parameter, it's typically handled separately or assumed to be 0-based if `numflag` applies to it in API calls).
        *   If `from == 0` (0-based to 1-based):
            *   Increments elements of `vtxdist`.
            *   Increments elements of `adjncy`.
            *   Increments elements of `xadj`.
            *   Increments elements of `part`.
*   **`ChangeNumberingMesh(idx_t *elmdist, idx_t *eptr, idx_t *eind, idx_t *xadj, idx_t *adjncy, idx_t *part, idx_t npes, idx_t mype, idx_t from)`**:
    *   Converts numbering for a mesh and its optional dual graph.
    *   **Parameters**:
        *   `elmdist`: Element distribution array.
        *   `eptr`, `eind`: Mesh element connectivity arrays (elements to nodes).
        *   `xadj`, `adjncy`: CSR structure of the dual graph (if computed and passed).
        *   `part`: Partition assignment array for elements (or dual graph vertices).
        *   `npes`: Number of processors.
        *   `mype`: Current processor's rank.
        *   `from`: If 1, converts from 1-based to 0-based. If 0, converts from 0-based to 1-based.
    *   **Functionality**:
        *   If `from == 1` (1-based to 0-based):
            *   Decrements `elmdist`.
            *   Decrements `eptr`.
            *   Decrements `eind` (node indices in the mesh).
        *   If `from == 0` (0-based to 1-based):
            *   Increments `elmdist`.
            *   Increments `eind`.
            *   Increments `eptr`.
            *   If `xadj` and `adjncy` (dual graph) are provided:
                *   Increments `adjncy` of the dual graph.
                *   Increments `xadj` of the dual graph.
            *   If `part` (element partition) is provided:
                *   Increments `part`.

## Important Variables/Constants

*   **`from` (parameter)**: Determines the direction of conversion (1 for 1-to-0, 0 for 0-to-1).

## Usage Examples

```c
// Conceptual usage within an API function:
// if (*numflag > 0) { // User specified 1-based numbering
//   ChangeNumbering(vtxdist, xadj, adjncy, part, npes, mype, 1); // Convert to 0-based
// }
//
// /* ... ParMETIS internal logic using 0-based numbering ... */
//
// if (*numflag > 0) { // If original was 1-based
//   ChangeNumbering(vtxdist, xadj, adjncy, part, npes, mype, 0); // Convert back to 1-based
// }
```
These functions are typically called at the beginning and end of ParMETIS API functions if the user specifies `numflag = 1`.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `idx_t` type definition.
    *   These functions directly modify the input arrays (`vtxdist`, `xadj`, `adjncy`, `part`, `elmdist`, `eptr`, `eind`).
*   **External Libraries**:
    *   None.
*   **Other Interactions**:
    *   Crucial for user convenience, allowing users to work with 1-based indexing if that's their native format, while ParMETIS internals operate with 0-based indexing.
    *   Incorrectly managing the `from` flag or calling these functions inappropriately can lead to severe errors due to incorrect graph/mesh representation.

```
