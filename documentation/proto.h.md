# libparmetis/proto.h

## Overview

This header file serves as a central repository for function prototypes of internal routines within the ParMETIS library. Its primary purpose is to declare the signatures of functions defined in various `.c` files in the `libparmetis` directory, allowing these functions to be called across different modules/files while ensuring type safety and enabling compilation.

## Key Components

This file consists almost entirely of function prototype declarations. These prototypes are organized by the source file in which the corresponding functions are implemented.

### Categories of Prototypes (based on source file comments):

*   **`ctrl.c`**: Functions for managing the `ctrl_t` control structure (e.g., `SetupCtrl`, `FreeCtrl`).
*   **`kmetis.c`**: Core k-way partitioning logic (e.g., `Global_Partition`).
*   **`mmetis.c`**: (No prototypes listed, likely API functions or static internal helpers).
*   **`gkmetis.c`**: (No prototypes listed, likely API functions or static internal helpers).
*   **`match.c`**: Graph matching algorithms for coarsening (e.g., `Match_Global`, `Match_Local`, `CreateCoarseGraph_Global`, `CreateCoarseGraph_Local`, `DropEdges`).
*   **`initpart.c`**: Initial partitioning routines (e.g., `InitPartition`, `KeepPart`).
*   **`kwayrefine.c`**: K-way refinement algorithms (e.g., `ProjectPartition`, `ComputePartitionParams`, `KWayFM`, `KWayBalance`).
*   **`remap.c`**: Functions for remapping graphs or partitions (e.g., `ParallelReMapGraph`).
*   **`move.c`**: Functions for moving graph data based on partitions (e.g., `MoveGraph`, `ProjectInfoBack`, `FindVtxPerm`).
*   **`wspace.c`**: Workspace memory management (e.g., `AllocateWSpace`, `wspacemalloc`).
*   **`ametis.c`**: Adaptive repartitioning (e.g., `Adaptive_Partition`).
*   **`rmetis.c`**: (No prototypes listed, likely API functions or static internal helpers).
*   **`wave.c`**: Wavefront diffusion algorithm (e.g., `WavefrontDiffusion`).
*   **`balancemylink.c`**: Balancing between two partitions (e.g., `BalanceMyLink`).
*   **`redomylink.c`**: Another routine for balancing/refining a link (e.g., `RedoMyLink`).
*   **`initbalance.c`**: Initial balancing routines (e.g., `Balance_Partition`, `AssembleAdaptiveGraph`).
*   **`mdiffusion.c`**: Multi-constraint diffusion (e.g., `Mc_Diffusion`, `ExtractGraph`).
*   **`diffutil.c`**: Utilities for diffusion (e.g., `SetUpConnectGraph`, `Mc_ComputeMoveStatistics`, `ConjGrad2`).
*   **`akwayfm.c`**: Adaptive k-way FM refinement (e.g., `KWayAdaptiveRefine`).
*   **`selectq.c`**: Queue selection utilities for refinement (e.g., `Mc_DynamicSelectQueue`, `Mc_HashVwgts`).
*   **`csrmatch.c`**: CSR-based matching (e.g., `CSR_Match_SHEM`).
*   **`serial.c`**: Serial graph operations used within parallel context (e.g., `Mc_ComputeSerialPartitionParams`, `Mc_SerialKWayAdaptRefine`).
*   **`weird.c`**: Input checking routines for API functions (e.g., `CheckInputsPartKway`, `CheckInputsNodeND`). Also `PartitionSmallGraph`.
*   **`mesh.c`**: (No prototypes listed, API function `ParMETIS_V3_Mesh2Dual` is declared in `parmetis.h`).
*   **`pspases.c`**: Routines for PSPASES compatibility (e.g., `AssembleEntireGraph`).
*   **`node_refine.c`**: Node-based separator refinement (e.g., `AllocateNodePartitionParams`, `KWayNodeRefine_Greedy`).
*   **`initmsection.c`**: Initial multisection routines (e.g., `InitMultisection`, `AssembleMultisectedGraph`).
*   **`ometis.c`**: Ordering routines (e.g., `MultilevelOrder`, `Order_Partition`, `LabelSeparators`, `CompactGraph`, `LocalNDOrder`).
*   **`xyzpart.c`**: Geometric partitioning based on coordinates (e.g., `Coordinate_Partition`).
*   **`stat.c`**: Statistics computation (e.g., `ComputeSerialBalance`, `ComputeParallelBalance`, `PrintPostPartInfo`).
*   **`debug.c`**: Debugging print functions (e.g., `PrintVector`, `PrintGraph`).
*   **`comm.c`**: Low-level communication utilities (e.g., `CommSetup`, `CommInterfaceData`, `GlobalSESum`).
*   **`util.c`**: General utility functions (e.g., `myprintf`, `BSearch`, `RandomPermute`).
*   **`graph.c`**: Graph data structure management (e.g., `SetupGraph`, `FreeGraph`).
*   **`renumber.c`**: Graph renumbering utilities (e.g., `ChangeNumbering`).
*   **`timer.c`**: Timer initialization and printing (e.g., `InitTimers`, `PrintTimingInfo`).
*   **`parmetis.c`**: (Likely refers to the file containing API entry points or Fortran interface helpers like `ChangeToFortranNumbering`).
*   **`msetup.c`**: Mesh setup routines (e.g., `SetUpMesh`).
*   **`gkmpi.c`**: Wrappers for MPI calls (e.g., `gkMPI_Comm_size`, `gkMPI_Allreduce`).

## Important Variables/Constants

This file primarily contains function declarations. It does not define variables or constants itself, but relies on types like `ctrl_t`, `graph_t`, `idx_t`, `real_t`, `ikv_t`, `matrix_t`, `mesh_t`, `MPI_Comm`, `pmoptype_et` which are defined in other headers included via `parmetislib.h`.

## Usage Examples

```c
// This is a header file. It's included by parmetislib.h, which is then
// included by most .c files in the ParMETIS library.

// Example: In kmetis.c
// #include <parmetislib.h> // This will include proto.h
// ...
// void Global_Partition(ctrl_t *ctrl, graph_t *graph) {
//   ...
//   Match_Global(ctrl, graph); // Call to function prototyped in proto.h (implemented in match.c)
//   ...
// }
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   This file is included by `parmetislib.h`.
    *   It depends on `ctrl_t`, `graph_t`, `idx_t`, `real_t`, `ikv_t`, `matrix_t`, `mesh_t`, `MPI_Comm`, `pmoptype_et` and other types being defined in headers like `struct.h` and `parmetis.h` (which are included before `proto.h` via `parmetislib.h`).
    *   The actual implementations of these prototyped functions are spread across numerous `.c` files in the `libparmetis` directory.
*   **External Libraries**:
    *   `MPI`: Some prototypes involve MPI types like `MPI_Comm`.
*   **Other Interactions**:
    *   This file is essential for modular compilation of the ParMETIS library, allowing different `.c` files to call functions defined in others.
    *   It provides a quick overview of the internal functions available within the library and their parameters.

```
