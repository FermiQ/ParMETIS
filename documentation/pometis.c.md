# programs/pometis.c

## Overview

This file contains the `main` function for the `pometis` command-line executable. This program is specifically designed as a driver for ParMETIS's parallel graph ordering routines, primarily `ParMETIS_V3_NodeND` (and its tunable version `ParMETIS_V32_NodeND`) and `ParMETIS_SerialNodeND`. It allows users to compute fill-reducing orderings for sparse matrices from the command line.

## Key Components

### Functions/Classes/Modules

*   **`main(int argc, char *argv[])`**:
    *   Entry point for the `pometis` executable.
    *   Initializes MPI.
    *   **Command-line Argument Parsing**:
        *   Expects 9 arguments if `optype == 2`, fewer for other optypes.
            1.  `graph-file`: Name of the file containing the graph.
            2.  `op-type`: Integer specifying the ordering operation:
                *   `1`: `ParMETIS_V3_NodeND` (standard parallel nested dissection).
                *   `2`: `ParMETIS_V32_NodeND` (tunable parallel nested dissection).
                *   `3`: `ParMETIS_SerialNodeND` (serial nested dissection on PE 0 after graph assembly).
            3.  `seed`: Random seed.
            4.  `dbglvl`: Debug level.
            5.  `mtype` (for `optype == 2`): Matching type (e.g., `PARMETIS_MTYPE_LOCAL`, `PARMETIS_MTYPE_GLOBAL`).
            6.  `rtype` (for `optype == 2`): Refinement type (e.g., `PARMETIS_SRTYPE_GREEDY`, `PARMETIS_SRTYPE_2PHASE`).
            7.  `p_nseps` (for `optype == 2`): Number of separators in parallel.
            8.  `s_nseps` (for `optype == 2`): Number of separators in serial.
            9.  `ubfrac` (for `optype == 2`): Imbalance fraction for separators.
    *   Reads the distributed graph using `ParallelReadGraph`.
    *   Allocates `order` and `sizes` arrays.
    *   **Operation Dispatch (based on `optype`)**:
        *   `optype == 1`: Sets up basic `options` (seed, dbglvl) and calls `ParMETIS_V3_NodeND`.
        *   `optype == 2`: Parses detailed tuning parameters (`mtype`, `rtype`, etc.) and calls `ParMETIS_V32_NodeND`.
        *   `optype == 3`: Sets `options[0] = 0` (no options passed to ParMETIS internals, defaults used by SerialNodeND) and calls `ParMETIS_SerialNodeND`.
    *   Writes the computed ordering to a file using `WriteOVector`.
    *   Prints separator sizes and an "OPC" (Operations Per Cycle, a measure of factorization complexity) estimate based on separator sizes.
    *   Frees allocated memory and finalizes MPI.

## Important Variables/Constants

*   `optype`: Command-line argument determining which ordering function to call.
*   `options`: Array for basic options like seed and debug level for `ParMETIS_V3_NodeND`.
*   `seed`, `dbglvl`, `mtype`, `rtype`, `p_nseps`, `s_nseps`, `ubfrac`: Parameters for the tunable `ParMETIS_V32_NodeND`.
*   `graph` (graph_t): Stores the distributed graph.
*   `order` (idx_t *): Stores the computed fill-reducing ordering.
*   `sizes` (idx_t *): Stores the sizes of the separators and leaf domains from the nested dissection.

## Usage Examples

```bash
# Command-line execution for ParMETIS_V3_NodeND:
mpirun -np <num_procs> pometis <graphfile> 1 <seed> <dbglvl>

# Example: Order 'mygraph.graph' using default parallel ND with seed 0, dbglvl 0
mpirun -np 4 pometis mygraph.graph 1 0 0

# Command-line execution for tunable ParMETIS_V32_NodeND:
mpirun -np <num_procs> pometis <graphfile> 2 <seed> <dbglvl> <mtype> <rtype> <p_nseps> <s_nseps> <ubfrac>

# Example: Order 'mygraph.graph' using tunable parallel ND
mpirun -np 4 pometis mygraph.graph 2 0 0 1 2 1 1 1.05
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` and `programs/proto.h`.
    *   `programs/io.c`: For `ParallelReadGraph`, `WriteOVector`.
    *   `libparmetis/parmetis.h` (API): Calls `ParMETIS_V3_NodeND`, `ParMETIS_V32_NodeND`.
    *   `libparmetis/pspases.c`: `ParMETIS_SerialNodeND` is called for `optype == 3`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`.
    *   Standard C library (stdio, stdlib, string).
*   **Other Interactions**:
    *   This program is the primary command-line tool for accessing ParMETIS's parallel graph ordering capabilities.
    *   The `sizes` array output provides insight into the structure of the nested dissection separator tree, which is important for understanding the quality of the ordering for sparse factorization.
    *   The "OPC" calculation is a heuristic to estimate the computational cost of factorization using this order.

```
