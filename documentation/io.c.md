# programs/io.c

## Overview

This file contains I/O utility functions specifically for the ParMETIS executable programs (drivers/tests like `parmetis`, `mtest`, `ptest`, `otest`). It includes functions to read distributed graphs from files where each processor reads its portion, read mesh data, read coordinate files, and write partition vectors or ordering vectors to files. It handles both standard METIS graph formats and potentially specialized formats for coordinates or meshes.

## Key Components

### Functions/Classes/Modules

*   **`ParallelReadGraph(graph_t *graph, char *filename, MPI_Comm comm)`**:
    *   Reads a distributed graph in METIS format. PE `npes-1` reads the header (number of vertices, edges, format code), broadcasts this info, and then each PE reads its assigned portion of the graph file.
    *   The file is assumed to be a single global graph file. PE `npes-1` reads chunks of the file and sends relevant lines/data to other PEs. This is a somewhat complex way to read a serial file in parallel.
    *   Handles vertex weights (`vwgt`) and edge weights (`adjwgt`) based on the format code.
    *   Populates the local `graph_t` structure on each PE.
*   **`Mc_ParallelWriteGraph(ctrl_t *ctrl, graph_t *graph, char *filename, idx_t nparts, idx_t testset)`**:
    *   Writes a distributed graph to a file in METIS format.
    *   PE 0 writes the header. Then, each PE (in order) appends its local graph data to the file.
    *   The output filename is constructed using `filename`, `testset` (likely an identifier), `graph->ncon`, and `nparts`.
*   **`ReadTestGraph(graph_t *graph, char *filename, MPI_Comm comm)`**:
    *   Reads a graph in METIS format, but PE 0 reads the entire graph and then scatters it to other PEs.
    *   `vtxdist` is computed to distribute the graph.
    *   `xadj` and `adjncy` are sent from PE 0 to other PEs.
*   **`ReadTestCoordinates(graph_t *graph, char *filename, idx_t *r_ndims, MPI_Comm comm)`**:
    *   Reads vertex coordinates from a file.
    *   PE 0 reads the entire coordinate file, determines `ndims` from the first line.
    *   PE 0 then sends the relevant portions of the coordinate data to other PEs.
*   **`ReadMetisGraph(char *filename, idx_t *r_nvtxs, idx_t **r_xadj, idx_t **r_adjncy)`**:
    *   A serial function (callable by any PE, but typically PE 0) to read an entire graph in METIS format from a file.
    *   Allocates and returns `xadj` and `adjncy`.
*   **`Mc_SerialReadGraph(graph_t *graph, char *filename, idx_t *wgtflag, MPI_Comm comm)`**:
    *   PE 0 reads an entire graph in METIS format (using `Mc_SerialReadMetisGraph`) and then distributes it among all PEs.
    *   Handles vertex and edge weights based on `wgtflag` and format code.
*   **`Mc_SerialReadMetisGraph(char *filename, idx_t *r_nvtxs, idx_t *r_ncon, idx_t *r_nobj, idx_t *r_fmt, idx_t **r_xadj, idx_t **r_vwgt, idx_t **r_adjncy, idx_t **r_adjwgt, idx_t *wgtflag)`**:
    *   A serial function to read a METIS graph file, supporting multi-constraint and multi-objective information if present in the format line.
    *   Populates all relevant graph arrays.
*   **`WritePVector(char *gname, idx_t *vtxdist, idx_t *part, MPI_Comm comm)`**:
    *   Writes a distributed partition vector `part` to a file named `<gname>.part`.
    *   PE 0 gathers the partition vector segments from all other PEs and writes them sequentially.
*   **`WriteOVector(char *gname, idx_t *vtxdist, idx_t *order, MPI_Comm comm)`**:
    *   Writes a distributed ordering vector `order` to a file named `<gname>.order.<npes>`.
    *   PE 0 gathers the ordering vector segments and writes them. It also performs a check to ensure the global ordering is a valid permutation.
*   **`ParallelReadMesh(mesh_t *mesh, char *filename, MPI_Comm comm)`**:
    *   Reads a distributed mesh. PE `npes-1` reads header (number of elements, type), broadcasts it.
    *   Then, PE `npes-1` reads the element connectivity line by line and sends to appropriate PEs.
    *   Normalizes node numbering based on global min/max node IDs.

## Important Variables/Constants

*   **`MAXLINE`**: Defines a large buffer size (e.g., 64MB) for reading lines from graph files, anticipating potentially very long lines if many adjacencies are on one line.
*   Format variables (`fmt`, `readew`, `readvw`, `ncon`, `nobj`): Parsed from graph file headers to determine how to read weights.

## Usage Examples

```c
// In a ParMETIS test program (e.g., parmetis.c):
// graph_t graph;
// ParallelReadGraph(&graph, filename, comm); // Reads the input graph
// ...
// ParMETIS_V3_PartKway(graph.vtxdist, graph.xadj, ..., part, ...);
// ...
// WritePVector(filename, graph.vtxdist, part, comm); // Writes the resulting partition
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` (for `graph_t`, `mesh_t`, `ctrl_t`, `idx_t`, MPI wrappers, GKlib utilities) and likely other common headers for executables.
    *   `parmetislib.h` (indirectly): Provides `ismalloc`, `imalloc`, `rmalloc`, `gk_cmalloc`, `gk_free`, `MAKECSR`, `icopy`, `strtoidx`, `strtoreal`, `GlobalSESum`, `errexit`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for communication in parallel read/write functions (e.g., `MPI_Bcast`, `MPI_Send`, `MPI_Recv`).
    *   Standard C library: For file I/O (`fopen`, `fgets`, `sscanf`, `fprintf`, `fclose`), string manipulation (`strlen`).
*   **Other Interactions**:
    *   These I/O functions are specific to the driver programs and provide ways to load graphs from disk and save results.
    *   The `ParallelReadGraph` function's strategy of having one PE (npes-1) read and distribute lines can be a bottleneck for very large files and many PEs. Other approaches (like each PE reading a contiguous block of a pre-split file, or more advanced parallel I/O libraries) are not used here.
    *   `ReadTestGraph` and `Mc_SerialReadGraph` are simpler: PE 0 reads all, then scatters. This is easier to implement but less scalable.

```
