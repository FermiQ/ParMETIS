# programs/parmetis.c

## Overview

This file contains the `main` function for the `parmetis` command-line executable. This program serves as a general-purpose driver for several ParMETIS V3 API functions, allowing users to partition graphs or compute vertex orderings from the command line by specifying the operation type and various parameters.

## Key Components

### Functions/Classes/Modules

*   **`main(int argc, char *argv[])`**:
    *   Entry point for the `parmetis` executable.
    *   Initializes MPI.
    *   **Command-line Argument Parsing**:
        *   Expects 7 arguments:
            1.  `graph-file`: The name of the file containing the graph.
            2.  `op-type`: Integer specifying the ParMETIS operation to perform (see below).
            3.  `nparts`: Number of partitions desired.
            4.  `adapt-factor`: Adaptation factor (used by `AdaptGraph` for op-type 3).
            5.  `ipc2redist`: IPC to redistribution factor (for op-type 3).
            6.  `dbglvl`: Debug level for ParMETIS options.
            7.  `seed`: Random seed for ParMETIS options.
    *   Reads the distributed graph using `ParallelReadGraph`.
    *   Initializes `tpwgts` (target partition weights) for equal distribution and `ubvec` (imbalance tolerance) to 1.05.
    *   Optionally reads coordinates if `optype >= 20` (geometric operations).
    *   Allocates `part` (for partition vector) and `sizes` (for ordering results).
    *   **Operation Dispatch (based on `optype`)**:
        *   `optype == 1`: Calls `ParMETIS_V3_PartKway` for k-way partitioning. Writes partition vector.
        *   `optype == 2`: Calls `ParMETIS_V3_RefineKway` to refine an existing partition (implicitly, `part` would need to be initialized, though it's initialized to `mype % nparts` here which might not be a meaningful existing partition unless `nparts` is related to `npes`). Writes partition vector.
        *   `optype == 3`: Calls `ParMETIS_V3_AdaptiveRepart`.
            *   If `npes > 1`, adapts graph using `AdaptGraph`.
            *   If `npes == 1`, first partitions with `PartKway` to get an initial `part` vector, then adapts vertex weights based on this partition and `adptf`.
        *   `optype == 4`: Calls `ParMETIS_V3_NodeND` for parallel nested dissection ordering. (Output writing commented out).
        *   `optype == 5`: Calls `ParMETIS_SerialNodeND` (assembles graph on PE0, then serial ND). Prints `sizes`. (Output writing commented out).
        *   `optype == 11`: (Commented out - `TestAdaptiveMETIS`).
        *   `optype == 20`: Calls `ParMETIS_V3_PartGeomKway` for geometric k-way partitioning.
        *   `optype == 21`: Calls `ParMETIS_V3_PartGeom` for coordinate-based partitioning.
    *   Frees allocated memory and finalizes MPI.
*   **`ChangeToFortranNumbering(idx_t *vtxdist, idx_t *xadj, idx_t *adjncy, idx_t mype, idx_t npes)`**:
    *   (Duplicate of the function in `libparmetis/renumber.c`). Converts graph arrays from 0-based to 1-based indexing.
    *   *(This is commented out in the `main` function, suggesting 0-based is default for this driver)*.

## Important Variables/Constants

*   `optype`: Command-line argument determining which ParMETIS API function to call.
*   `nparts`: Desired number of partitions.
*   `adptf`: Adaptation factor for `AdaptGraph`.
*   `ipc2redist`: IPC to redistribution ratio for adaptive repartitioning.
*   `options`: Array to set `dbglvl` and `seed` for ParMETIS calls.
*   `graph` (graph_t): Stores the distributed graph.
*   `part` (idx_t *): Stores the resulting partition vector.
*   `sizes` (idx_t *): Stores separator sizes for ordering.
*   `xyz` (real_t *): Stores vertex coordinates for geometric operations.
*   `MAXNCON`: Used for `ubvec` size.

## Usage Examples

```bash
# Command-line execution:
mpirun -np <num_procs> parmetis <graphfile> <optype> <nparts> <adaptfactor> <ipc2redist> <dbglvl> <seed>

# Example: Partition 'mygraph.graph' into 4 parts using k-way partitioning
mpirun -np 4 parmetis mygraph.graph 1 4 10 1.0 0 0

# Example: Order 'mygraph.graph' using parallel nested dissection
mpirun -np 4 parmetis mygraph.graph 4 4 10 1.0 0 0
# (nparts, adaptfactor, ipc2redist might be ignored for optype 4)
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` and `programs/proto.h`.
    *   `programs/io.c`: For `ParallelReadGraph`, `ReadTestCoordinates`, `WritePVector`, `WriteOVector`.
    *   `programs/adaptgraph.c`: For `AdaptGraph`.
    *   `libparmetis/parmetis.h` (API): Calls various `ParMETIS_V3_*` functions.
    *   `libparmetis/pspases.c`: `ParMETIS_SerialNodeND` is called for `optype == 5`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`.
    *   Standard C library (stdio, stdlib, string).
*   **Other Interactions**:
    *   This program acts as a command-line frontend to the ParMETIS library, allowing users to invoke different parallel graph algorithms.
    *   The specific combination of parameters and `optype` determines the exact workflow and ParMETIS functions called.
    *   Output (partition files, ordering files) is written for some operations.

```
