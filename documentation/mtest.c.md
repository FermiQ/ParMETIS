# programs/mtest.c

## Overview

This file contains a test program (`mtest`) for ParMETIS, specifically designed to test the mesh partitioning capabilities, primarily `ParMETIS_V3_PartMeshKway`. It reads a mesh, optionally allows specifying the number of common nodes to define element adjacency for the dual graph, partitions the mesh (which involves internal dual graph creation and partitioning), and then likely (though not explicitly shown in the provided snippet) would perform some checks or report statistics.

## Key Components

### Functions/Classes/Modules

*   **`main(int argc, char *argv[])`**:
    *   Entry point for the `mtest` program.
    *   Initializes MPI.
    *   Parses command-line arguments:
        *   `argv[1]`: Mesh filename.
        *   `argv[2]` (optional): Number of common nodes (`mgcnum`) to define adjacency in the dual graph. If not provided, it's derived from the mesh element type.
    *   Calls `ParallelReadMesh` to read the distributed mesh data into a `mesh_t` structure.
    *   Sets up parameters for `ParMETIS_V3_PartMeshKway`:
        *   `nparts`: Set to `npes` (number of processors).
        *   `tpwgts`: Target partition weights (equal for all parts).
        *   `ubvec`: Uniform imbalance tolerance (`UNBALANCE_FRACTION`).
        *   `options`: Basic ParMETIS options (enable options, set debug level, seed).
    *   The `eptr` array (element pointers for node lists) is artificially modified for the last element (`eptr[nelms]--`). This seems like a specific test case or a way to introduce a slight irregularity, but its exact purpose is unclear without further context. **Correction**: `MAKECSR(i, nelms, eptr)` converts `eptr` from element sizes to CSR format. The `eptr[nelms]--` then seems to be an intentional modification *after* CSR conversion, potentially to test robustness or a specific scenario. *Further Correction*: `ismalloc(nelms+1, esizes[mesh.etype], "main; eptr")` initializes `eptr` with `esizes[mesh.etype]` for all entries. Then `MAKECSR` creates the CSR structure. The `eptr[nelms]--` is indeed a peculiar modification to the last element's degree if `esizes` was uniform.
    *   Calls `ParMETIS_V3_PartMeshKway` to partition the mesh.
    *   Frees allocated memory and finalizes MPI.

## Important Variables/Constants

*   **`mesh` (mesh_t)**: Structure holding the distributed mesh data.
*   **`part` (idx_t *)**: Array to store the computed partition for each local element.
*   **`mgcnum` (idx_t)**: Number of common nodes to define an edge in the dual graph. Derived from `mesh.etype` or command line.
*   **`esizes` (array)**: Maps element type to number of nodes per element.
*   **`mgcnums` (array)**: Maps element type to default `ncommon` value.
*   **`UNBALANCE_FRACTION`**: Default imbalance tolerance.
*   `options`: Array to pass options to ParMETIS.

## Usage Examples

```bash
# Command-line execution:
mpirun -np <num_procs> mtest <mesh_file_name> [num_common_nodes]

# Example:
mpirun -np 4 mtest my_mesh.msh 3
# This would partition my_mesh.msh into 4 partitions, using 3 common nodes
# to determine element adjacency for the internal dual graph.
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` and other common headers.
    *   `io.c` (programs/io.c): For `ParallelReadMesh`.
    *   `parmetis.h` (ParMETIS library API): `ParMETIS_V3_PartMeshKway` is the main API call being tested.
    *   `parmetislib.h` (indirectly): For `mesh_t`, `idx_t`, `real_t`, MPI wrappers, memory utilities (`imalloc`, `ismalloc`, `rmalloc`, `gk_free`, `gk_malloc_init`, `gk_malloc_cleanup`, `MAKECSR`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For parallel execution.
    *   Standard C library: For `printf`, `exit`, `atoi`.
*   **Other Interactions**:
    *   This program serves as a driver to test the mesh partitioning functionality of ParMETIS.
    *   It reads a mesh, partitions it, and would typically be used in conjunction with result analysis (e.g., checking edge cuts, balance, or visualizing the partition) which is not shown in the snippet.
    *   The modification of `eptr[nelms]` is unusual and might be a specific test for robustness or a particular graph structure.

```
