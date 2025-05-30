# programs/dglpart.c

## Overview

This program is a specialized tool for partitioning graphs for use with DistDGL (Distributed Deep Graph Library). It reads graph data in a specific format (likely edges and node features/labels from separate files), partitions the graph using ParMETIS, physically redistributes the graph data (including metadata) according to this partition, and then writes out the partitioned subgraphs in a format suitable for DistDGL. A key aspect is its handling of node and edge metadata which needs to be moved along with the graph structure. It also employs a cyclic mapping for distributing vertices initially to balance memory usage during the reading and initial processing phases.

## Key Components

### Functions/Classes/Modules

*   **`main(int argc, char *argv[])`**:
    *   Entry point. Parses command-line arguments (input file stem, number of partitions per PE).
    *   Calls `DistDGL_GPart`.
*   **`DistDGL_GPart(char *fstem, idx_t nparts_per_pe, MPI_Comm comm)`**:
    *   Orchestrates the overall process:
        1.  Reads the graph data using `DistDGL_ReadGraph`.
        2.  Reports peak memory usage.
        3.  Partitions the graph using `ParMETIS_V3_PartKway`.
        4.  Moves the graph (including metadata) according to the partition using `DistDGL_MoveGraph`.
        5.  Writes the partitioned subgraphs to disk using `DistDGL_WriteGraphs`.
        6.  Reports peak memory usage again.
*   **`DistDGL_ReadGraph(char *fstem, MPI_Comm comm)`**:
    *   Reads graph data from multiple files: `<fstem>_stats.txt`, `<fstem>_edges.txt`, `<fstem>_nodes.txt`.
    *   **Stats file**: Provides global number of vertices, edges, and number of constraints (node features).
    *   **Edges file**: Each line is `dest_id source_id <metadata_string>`. Edges are read in chunks by PE 0 and distributed to other PEs based on a cyclic mapping of source/destination IDs to balance load during reading.
    *   **Nodes file**: Each line is `<constraint_values> <metadata_string>`. Node data is also read in chunks and distributed.
    *   Handles potentially large metadata strings associated with nodes and edges. Metadata is temporarily stored to disk by `DistDGL_MoveGraph` before being re-read by `DistDGL_WriteGraphs` (or, in this read function, saved to disk after reading before being moved).
    *   Constructs a local `graph_t` object on each PE. The global graph is implicitly represented by these distributed pieces.
*   **`DistDGL_MoveGraph(graph_t *ograph, idx_t *part, idx_t nparts_per_pe, MPI_Comm comm)`**:
    *   Redistributes the `ograph` (original graph) based on the computed `part` vector to create `mgraph` (moved graph).
    *   Similar to the standard `MoveGraph` in ParMETIS, but with added complexity to handle node/edge metadata (`vmptr`, `emptr`, `vmdata`, `emdata`) and vertex types (`vtype`).
    *   Calculates send/receive counts and displacements for graph structure and all associated metadata.
    *   Uses `gkMPI_Alltoall` for count exchange and `gkMPI_Isend`/`gkMPI_Irecv`/`gkMPI_Waitall` for data exchange.
*   **`DistDGL_WriteGraphs(char *fstem, graph_t *graph, idx_t nparts_per_pe, MPI_Comm comm)`**:
    *   Writes the local subgraphs held by each PE to output files.
    *   Each PE is responsible for `nparts_per_pe` partitions.
    *   Before writing, it renumbers the local vertices within each partition to have contiguous IDs starting from 0 for that partition, and globally unique IDs across all partitions using `CommSetup` and `CommInterfaceData` on a temporarily created graph structure.
    *   For each partition `p` assigned to the current PE, it creates:
        *   `p<global_part_id>-<fstem>_nodes.txt`: Node ID (new local) and its metadata string.
        *   `p<global_part_id>-<fstem>_edges.txt`: Source node ID (new local), Dest node ID (new local), and edge metadata string.
*   **`DistDGL_CheckMGraph(ctrl_t *ctrl, graph_t *graph, idx_t nparts_per_pe)`**:
    *   A debugging function to check the consistency of the moved graph, similar to `CheckMGraph` in `move.c`.
*   **`i2kvsorti(size_t n, i2kv_t *base)` / `i2kvsortii(size_t n, i2kv_t *base)`**:
    *   Sorting functions for `i2kv_t` structures (key1, key2, val). `i2kvsorti` sorts by key1, key2 (asc) then val (desc). `i2kvsortii` sorts by key1, key2, val (all asc).
*   **`DistDGL_mapFromCyclic(idx_t u, idx_t npes, idx_t *vtxdist)` / `DistDGL_mapToCyclic(idx_t u, idx_t npes, idx_t *vtxdist)`**:
    *   Helper functions for mapping global vertex ID `u` to/from a cyclic distribution target PE and local index. The cyclic distribution aims to balance memory when reading large graph files. `vtxdist` here is the standard ParMETIS vtxdist for the *target* cyclic distribution, not the final partition.

### mvinfo_t (struct)
*   A simple struct to hold counts of data to be moved: `nvtxs`, `nedges`, `nvmdata` (size of vertex metadata), `nemdata` (size of edge metadata).

## Important Variables/Constants

*   **`fstem`**: File stem for input and output graph files.
*   **`nparts_per_pe`**: Number of graph partitions each processor will manage/output.
*   **`CHUNKSIZE`**: Size of chunks for reading large edge/node files.
*   `graph->vmptr`, `graph->emptr`, `graph->vmdata`, `graph->emdata`: Store pointers and actual string data for vertex and edge metadata.
*   `graph->vtype`: Stores type information for each vertex.

## Usage Examples

```bash
# Command-line execution:
mpirun -np <num_procs> dglpart <graph_file_stem> <num_partitions_per_proc>

# Example:
mpirun -np 4 dglpart mygraph 2
# This would partition 'mygraph' into 4*2=8 total partitions.
# Each of the 4 PEs would write out 2 partition files.
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` and common headers.
    *   `parmetislib.h` (indirectly): For `ctrl_t`, `graph_t`, ParMETIS API functions (`ParMETIS_V3_PartKway`), MPI wrappers, memory management, sorting utilities.
    *   `io.c` (from `libparmetis`): Although this `dglpart.c` has its own I/O, it doesn't directly use `pio.c` from the library but rather implements custom file reading for the DistDGL format.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For all parallel operations.
    *   Standard C library: For file I/O (`stdio.h`), string manipulation (`string.h`), memory allocation (`stdlib.h`).
*   **Other Interactions**:
    *   Reads input graph data (structure, node features/types, edge features) from text files with a specific naming convention based on `fstem`.
    *   Outputs multiple files, one set per final partition, containing the subgraph structure and associated metadata.
    *   The cyclic mapping during the read phase is a strategy to deal with potentially skewed distributions in the input files and improve load balance during initial data loading and distribution.
    *   Metadata is temporarily written to disk (`emdata-<pid>-<mype>.bin`, `vmdata-<pid>-<mype>.bin`) by `DistDGL_ReadGraph` before being moved by `DistDGL_MoveGraph` and then presumably re-read by `DistDGL_WriteGraphs` to include in the final output files. This is a way to manage potentially large metadata that might not fit in memory if kept for all vertices/edges during all stages.

```
