# libparmetis/defs.h

## Overview

This header file contains a collection of global constant definitions used throughout the ParMETIS library. These constants define default behaviors, algorithm parameters, limits, specific modes of operation, and debug levels. Centralizing these definitions allows for easier configuration and tuning of the library's internal workings.

## Key Components

### Functions/Classes/Modules/Macros/Structs

This file primarily consists of `#define` directives for constants.

### Important Variables/Constants

A wide range of constants are defined, categorized as follows:

*   **Default Global Settings**:
    *   `GLOBAL_DBGLVL`: Default debug level if not specified by the user.
    *   `GLOBAL_SEED`: Default random number seed if not specified.
*   **Algorithm Parameters & Thresholds**:
    *   `NUM_INIT_MSECTIONS`: Number of initial multi-sections.
    *   `MC_FLOW_BALANCE_THRESHOLD`: Threshold for flow balance in multi-constraint context.
    *   `MOC_GD_GRANULARITY_FACTOR`: Granularity factor for multi-constraint gradient descent.
    *   `RIP_SPLIT_FACTOR`: Factor for Recursive Inertial Partitioning splits.
    *   `MAX_NPARTS_MULTIPLIER`: Multiplier for maximum number of parts.
    *   `REDIST_WGT`: A weight factor, possibly for redistribution cost.
    *   `MAXNVWGT_FACTOR`: Factor for maximum node weight.
    *   `N_MOC_REDO_PASSES`, `N_MOC_GR_PASSES`, `NREMAP_PASSES`, `N_MOC_GD_PASSES`, `N_MOC_BAL_PASSES`, `NMATCH_PASSES`: Number of passes for various internal algorithms (Multi-constraint operations, Remapping, Matching).
    *   `NGD_PASSES`: Number of gradient descent passes.
    *   `NGR_PASSES`: Number of greedy refinement passes.
    *   `NIPARTS`: Number of random initial partitions.
    *   `NLGR_PASSES`: Number of greedy refinement passes during initial partitioning.
    *   `SMALLFLOAT`: A small floating-point value for comparisons.
    *   `COARSEN_FRACTION`, `COARSEN_FRACTION2`: Node reduction ratios between successive coarsening levels.
    *   `UNBALANCE_FRACTION`, `ORDER_UNBALANCE_FRACTION`: Default imbalance tolerance for partitioning and ordering.
    *   `MAXVWGT_FACTOR`: Maximum vertex weight factor, possibly used in load balancing constraints.
*   **Operation Modes & Types**:
    *   `STATIC_PARTITION`, `ORDER_PARTITION`, `ADAPTIVE_PARTITION`, `REFINE_PARTITION`: Identifiers for different types of partitioning operations.
    *   `PMV3_OPTION_DBGLVL`, `PMV3_OPTION_SEED`, `PMV3_OPTION_IPART`, `PMV3_OPTION_PSR`: Indices for accessing options in the `options` array for ParMETIS V3 API.
    *   `XYZ_XCOORD`, `XYZ_SPFILL`: Options possibly related to geometric partitioning or sparse fill.
    *   `ISEP_EDGE`, `ISEP_NODE`: Types of initial vertex separator algorithms.
*   **Special Values & Markers**:
    *   `UNMATCHED`, `MAYBE_MATCHED`, `TOO_HEAVY`: Markers used in matching algorithms.
    *   `HTABLE_EMPTY`: Marker for an empty slot in a hash table.
    *   `LTERM`: List terminator for `GKfree` variadic function (`(void **) 0`).
*   **Limits & Constraints**:
    *   `MAX_NCON_FOR_DIFFUSION`: Maximum number of constraints for which diffusion is attempted.
    *   `SMALLGRAPH`: Threshold defining a "small" graph, possibly triggering different algorithmic paths.
*   **Debug Levels (Bitmasks)**:
    *   `DBG_TIME` (`PARMETIS_DBGLVL_TIME`): Enables timing information.
    *   `DBG_INFO` (`PARMETIS_DBGLVL_INFO`): Enables general informational output.
    *   `DBG_PROGRESS` (`PARMETIS_DBGLVL_PROGRESS`): Enables progress output during execution.
    *   `DBG_REFINEINFO` (`PARMETIS_DBGLVL_REFINEINFO`): Enables detailed refinement information.
    *   `DBG_MATCHINFO` (`PARMETIS_DBGLVL_MATCHINFO`): Enables detailed matching information.
    *   `DBG_RMOVEINFO` (`PARMETIS_DBGLVL_RMOVEINFO`): Enables information about refinement moves.
    *   `DBG_REMAP` (`PARMETIS_DBGLVL_REMAP`): Enables remapping information.
    *   (Note: `PARMETIS_DBGLVL_*` constants are themselves defined in `parmetis.h`, which is typically included before `defs.h` via `parmetislib.h`)

## Usage Examples

```c
// Example of how a constant from defs.h might be used:
// if (graph->nvtxs < SMALLGRAPH) {
//   // Use a simpler algorithm for small graphs
// }
//
// for (pass=0; pass<NGR_PASSES; pass++) {
//   // Perform a greedy refinement pass
// }
```
These constants are used directly in C code throughout ParMETIS to control behavior, loop counts, array sizes, and conditional compilation or execution paths.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   This file is typically included via `parmetislib.h`, which itself includes `parmetis.h`. Some constants defined here (like `DBG_*`) are aliases to `PARMETIS_DBGLVL_*` constants defined in `parmetis.h`.
    *   These constants are used by virtually all C files within the `libparmetis` directory to govern algorithmic choices, parameters, and debugging outputs.
*   **External Libraries**:
    *   None. This file only contains preprocessor definitions.
*   **Other Interactions**:
    *   Changing values in this file can significantly alter the behavior, performance, and output quality of ParMETIS algorithms. It's a central place for tuning and configuring default settings.

```
