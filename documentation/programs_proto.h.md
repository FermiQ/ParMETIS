# programs/proto.h

## Overview

This header file contains function prototypes for routines defined within the `programs` directory of ParMETIS. These routines are typically part of the test drivers, command-line executables, or I/O utilities specific to these programs, rather than being part of the core ParMETIS library itself. This file allows different `.c` files within the `programs` directory to call each other's functions.

## Key Components

This file consists almost entirely of function prototype declarations.

### Categories of Prototypes (based on typical ParMETIS program structure):

*   **I/O Functions (likely from `programs/io.c`)**:
    *   `ParallelReadGraph(graph_t *, char *, MPI_Comm)`: Reads a distributed graph.
    *   `Mc_ParallelWriteGraph(ctrl_t *, graph_t *, char *, idx_t, idx_t)`: Writes a distributed graph.
    *   `ReadTestGraph(graph_t *, char *, MPI_Comm)`: Reads a graph with PE 0 doing most work.
    *   `ReadTestCoordinates(graph_t *, char *, idx_t *, MPI_Comm)`: Reads vertex coordinates.
    *   `ReadMetisGraph(char *, idx_t *, idx_t **, idx_t **)`: Serial METIS graph reader.
    *   `Mc_SerialReadGraph(graph_t *, char *, idx_t *, MPI_Comm)`: PE 0 reads, then distributes.
    *   `Mc_SerialReadMetisGraph(char *, idx_t *, idx_t *, idx_t *, idx_t *, idx_t **, idx_t **, idx_t **, idx_t **, idx_t *)`: Detailed serial METIS graph reader.
    *   `WritePVector(char *gname, idx_t *vtxdist, idx_t *part, MPI_Comm comm)`: Writes a partition vector.
    *   `WriteOVector(char *gname, idx_t *vtxdist, idx_t *order, MPI_Comm comm)`: Writes an ordering vector.
*   **Graph Adaptation Functions (likely from `programs/adaptgraph.c`)**:
    *   `AdaptGraph(graph_t *, idx_t, MPI_Comm)`: Adapts graph vertex weights.
    *   `AdaptGraph2(graph_t *, idx_t, MPI_Comm)`: Another variant of graph adaptation.
    *   `Mc_AdaptGraph(graph_t *, idx_t *, idx_t, idx_t, MPI_Comm)`: Multi-constraint graph adaptation.
*   **Test Program Main Logic (e.g., from `ptest.c`, `otest.c`)**:
    *   `TestParMetis_GPart(char *filename, char *xyzfile, MPI_Comm comm)`: (Prototype might be slightly different in actual `ptest.c` or `otest.c` regarding `xyzfile`). A main test function. The `otest.c` version is `void TestParMetis(char *filename, MPI_Comm comm)`.
    *   `ComputeRealCut(idx_t *, idx_t *, char *, MPI_Comm)`: Computes edge cut by gathering partition and reading graph on PE 0.
    *   `ComputeRealCutFromMoved(idx_t *, idx_t *, idx_t *, idx_t *, char *, MPI_Comm)`: Computes cut involving a moved graph's partition. (The actual name in `otest.c` is `ComputeRealCut2`).
    *   `TestMoveGraph(graph_t *, graph_t *, idx_t *, MPI_Comm)`: Utility to test graph movement.
    *   `TestSetUpGraph(ctrl_t *, idx_t *, idx_t *, idx_t *, idx_t *, idx_t *, idx_t)`: (Likely an older or internal setup utility for tests, seen in `otest.c` as just `SetUpGraph`).
*   **Mesh I/O (likely from `programs/meshio.c` or similar, though not explicitly in current file list beyond `mtest.c` using `ParallelReadMesh`)**:
    *   `mienIO(mesh_t *, char *, idx_t, idx_t, MPI_Comm)`: (Purpose less clear without seeing `mienio.c`, possibly specialized mesh I/O).
    *   `ParallelReadMesh(mesh_t *, char *, MPI_Comm)`: Reads a distributed mesh.

## Important Variables/Constants

This file only contains function declarations. It relies on types like `graph_t`, `ctrl_t`, `mesh_t`, `idx_t`, `real_t`, `MPI_Comm` being defined in headers included via `parmetisbin.h` (which includes `parmetislib.h`).

## Usage Examples

```c
// This is a header file. It's included by parmetisbin.h, which is then
// included by .c files in the programs directory (e.g., parmetis.c, otest.c).

// Example: In parmetis.c
// #include <parmetisbin.h> // This will include programs/proto.h
// ...
// graph_t graph;
// ParallelReadGraph(&graph, argv[1], comm); // Call to function prototyped in programs/proto.h
// ...
```

## Dependencies and Interactions

*   **Internal Dependencies (programs directory)**:
    *   This file is included by `parmetisbin.h`.
    *   It declares functions that are implemented in other `.c` files within the `programs` directory (e.g., `io.c`, `adaptgraph.c`).
*   **Internal Dependencies (ParMETIS library)**:
    *   Many function signatures use types defined in the ParMETIS library (e.g., `graph_t`, `ctrl_t`, `mesh_t`, `idx_t`).
*   **External Libraries**:
    *   `MPI`: MPI types like `MPI_Comm` are used in function signatures.
*   **Other Interactions**:
    *   This file enables modularity within the `programs` directory, allowing different test programs or utilities to share common functions like I/O routines.
    *   It provides a quick reference to the helper functions available specifically for the ParMETIS executables.

```
