# programs/io.c

## Overview

This file provides I/O utility functions specifically for the ParMETIS executable programs (drivers/tests like `parmetis`, `mtest`, `ptest`, `otest`). It includes functions to read distributed graphs from files where each processor reads its portion (or PE `npes-1` / PE 0 reads and distributes), read mesh data, read coordinate files, and write partition vectors or ordering vectors to files. It handles both standard METIS graph formats and potentially specialized formats for coordinates or meshes.

## Key Components

### Functions/Classes/Modules

*   **`ParallelReadGraph(graph_t *graph, char *filename, MPI_Comm comm)`**:
    *   Reads a distributed graph in METIS format.
    *   PE `npes-1` opens the file, reads header information (global number of vertices, edges, format code, number of constraints `ncon`, number of objectives `nobj`), and broadcasts this information.
    *   `vtxdist` (vertex distribution) is computed and broadcasted.
    *   PE `npes-1` then reads the graph line by line. For each vertex line:
        *   It determines which PE the vertex belongs to based on `vtxdist`.
        *   It sends the vertex data (degree, weights if any, adjacency list with edge weights if any) to the target PE.
    *   Other PEs receive their respective data and populate their local `graph_t` structure.
    *   This method involves significant inter-processor communication as one PE reads and distributes.
*   **`Mc_ParallelWriteGraph(ctrl_t *ctrl, graph_t *graph, char *filename, idx_t nparts, idx_t testset)`**:
    *   Writes a distributed graph to a single file in METIS format.
    *   PE 0 writes the header.
    *   Then, PEs from 0 to `npes-1` sequentially append their local graph data to the file. This requires careful coordination to ensure correct file access.
*   **`ReadTestGraph(graph_t *graph, char *filename, MPI_Comm comm)`**:
    *   Reads a graph in METIS format. PE 0 reads the entire graph using `ReadMetisGraph`.
    *   `vtxdist` is computed to distribute the graph.
    *   PE 0 then sends the appropriate portions of `xadj` and `adjncy` to other PEs. Vertex and edge weights are not explicitly handled in this simplified distribution shown (sets `vwgt` and `adjwgt` to `NULL`).
*   **`ReadTestCoordinates(graph_t *graph, char *filename, idx_t *r_ndims, MPI_Comm comm)`**:
    *   Reads vertex coordinates from a file.
    *   PE 0 opens the file, reads the first line to determine `ndims` (number of dimensions), and broadcasts `ndims`.
    *   PE 0 reads the entire coordinate file and sends the relevant coordinate sets to each PE based on `graph->vtxdist`.
*   **`ReadMetisGraph(char *filename, idx_t *r_nvtxs, idx_t **r_xadj, idx_t **r_adjncy)`**:
    *   A serial function to read an entire graph in METIS format (no weights).
    *   Allocates and returns `xadj` and `adjncy`.
*   **`Mc_SerialReadGraph(graph_t *graph, char *filename, idx_t *wgtflag, MPI_Comm comm)`**:
    *   PE 0 reads an entire graph using `Mc_SerialReadMetisGraph` (which supports weights).
    *   Graph metadata (`ncon`, `nobj`, `fmt`, `wgtflag`, `vtxdist`) is broadcast.
    *   PE 0 then distributes the graph data (`xadj`, `vwgt`, `adjncy`, `adjwgt`) to other PEs.
*   **`Mc_SerialReadMetisGraph(char *filename, idx_t *r_nvtxs, ..., idx_t *wgtflag)`**:
    *   A detailed serial function to read a METIS graph file, parsing the format line to understand if vertex weights (`readvw`) and/or edge weights (`readew`) are present, and also reads `ncon` and `nobj` if specified.
    *   Allocates and returns all graph arrays.
*   **`WritePVector(char *gname, idx_t *vtxdist, idx_t *part, MPI_Comm comm)`**:
    *   Writes a distributed partition vector `part` to a file named `<gname>.part`.
    *   PE 0 gathers the partition vector segments from all other PEs via `MPI_Recv` (others `MPI_Send`) and writes them sequentially.
*   **`WriteOVector(char *gname, idx_t *vtxdist, idx_t *order, MPI_Comm comm)`**:
    *   Writes a distributed ordering vector `order` to a file named `<gname>.order.<npes>`.
    *   Similar to `WritePVector`, PE 0 gathers and writes. It also includes a check on PE 0 to ensure the received global ordering is a valid permutation.
*   **`ParallelReadMesh(mesh_t *mesh, char *filename, MPI_Comm comm)`**:
    *   Reads a distributed mesh from a file.
    *   PE `npes-1` reads the header (number of elements, element type), broadcasts it.
    *   `elmdist` is computed and broadcast.
    *   PE `npes-1` reads element connectivity line by line and sends to the appropriate PE.
    *   Node IDs are normalized (global 0-based).

## Important Variables/Constants

*   **`MAXLINE`**: Defines a large buffer size (e.g., 64MB) for `fgets`, anticipating potentially very long lines if many adjacencies or weights are on one line in the METIS graph format.
*   Format variables (`fmt`, `readew`, `readvw`, `ncon`, `nobj`): Parsed from graph file headers to determine how to read weights and other parameters.

## Usage Examples

```c
// In a ParMETIS test program (e.g., parmetis.c):
// graph_t graph;
// ParallelReadGraph(&graph, filename, comm); // Reads the input graph
// ...
// idx_t *part = imalloc(graph.nvtxs, "partition_vector");
// ParMETIS_V3_PartKway(graph.vtxdist, graph.xadj, ..., part, ...);
// ...
// WritePVector(filename, graph.vtxdist, part, comm); // Writes the resulting partition
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` (for `graph_t`, `mesh_t`, `ctrl_t`, `idx_t`, MPI wrappers, GKlib utilities) and `programs/proto.h`.
    *   `parmetislib.h` (indirectly): Provides `ismalloc`, `imalloc`, `rmalloc`, `gk_cmalloc`, `gk_free`, `MAKECSR`, `icopy`, `strtoidx`, `strtoreal`, `GlobalSESum`, `errexit`, `imin`, `imax`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for communication in all parallel read/write functions.
    *   Standard C library: For file I/O (`fopen`, `fgets`, `sscanf`, `fprintf`, `fclose`), string manipulation (`strlen`), memory operations (`memcpy`).
*   **Other Interactions**:
    *   These I/O functions are crucial for the ParMETIS test programs to load graph/mesh data and save results.
    *   The parallel reading strategies vary:
        *   `ParallelReadGraph` and `ParallelReadMesh`: One PE reads and sends lines/data to others.
        *   `ReadTestGraph`, `ReadTestCoordinates`, `Mc_SerialReadGraph`: PE 0 reads everything, then scatters data.
    *   Writing is generally done by PE 0 gathering data from other PEs.

```
