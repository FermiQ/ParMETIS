# libparmetis/ctrl.c

## Overview

This file is responsible for managing the `ctrl_t` control structure in ParMETIS. The `ctrl_t` structure holds various parameters, options, and state information that govern the behavior of the ParMETIS algorithms. Functions in this file handle the setup (allocation and initialization) and teardown (deallocation) of this critical structure.

## Key Components

### Functions/Classes/Modules

*   **`SetupCtrl(pmoptype_et optype, idx_t *options, idx_t ncon, idx_t nparts, real_t *tpwgts, real_t *ubvec, MPI_Comm comm)`**:
    *   Allocates and initializes a `ctrl_t` structure.
    *   **Parameters**:
        *   `optype`: The type of ParMETIS operation being performed (e.g., `PARMETIS_OP_KMETIS`, `PARMETIS_OP_AMETIS`).
        *   `options`: An array of options provided by the user. If `NULL` or `options[0] == 0`, default options are used.
        *   `ncon`: Number of balance constraints.
        *   `nparts`: Number of target partitions.
        *   `tpwgts`: Array of target partition weights for each constraint.
        *   `ubvec`: Array of imbalance tolerance for each constraint.
        *   `comm`: The MPI communicator to be used.
    *   **Functionality**:
        *   Duplicates the input MPI communicator (`ctrl->gcomm`, `ctrl->comm`).
        *   Sets `mype` (MPI rank) and `npes` (MPI size).
        *   Parses the `options` array or sets default values for:
            *   `partType`: Type of partitioning (STATIC, ADAPTIVE, REFINE).
            *   `ps_relation`: Coarsening relationship (COUPLED, UNCOUPLED), especially for adaptive/refine modes.
            *   `dbglvl`: Debug level, also used to set boolean flags like `dropedges`, `twohop`, `fast`, `ondisk`.
            *   `seed`: Random number seed.
        *   Initializes other common control parameters: `optype`, `ncon`, `nparts`, `redist_factor`, `redist_base`.
        *   Allocates and initializes `ctrl->tpwgts` (target partition weights) and `ctrl->ubvec` (imbalance tolerances).
        *   Initializes timers (`InitTimers`).
        *   Seeds the random number generator (`srand`).
*   **`SetupCtrl_invtvwgts(ctrl_t *ctrl, graph_t *graph)`**:
    *   Computes the inverse of the total vertex weights for each constraint and stores them in `ctrl->invtvwgts`.
    *   This is often used to normalize partition weights or calculate load balance ratios.
    *   Requires `graph->vwgt` (vertex weights) and `graph->ncon`.
*   **`FreeCtrl(ctrl_t **r_ctrl)`**:
    *   Deallocates all memory associated with the `ctrl_t` structure.
    *   Frees workspace allocated via `WSpace` calls.
    *   Frees the duplicated MPI communicator if `ctrl->free_comm` is set.
    *   Frees arrays like `invtvwgts`, `ubvec`, `tpwgts`, and MPI request arrays.
    *   Sets the input pointer `*r_ctrl` to `NULL`.

## Important Variables/Constants

*   **`ctrl_t` (struct)**: This is the central structure managed by this file. Key members initialized include:
    *   `comm`, `gcomm`: MPI communicators.
    *   `mype`, `npes`: MPI rank and size.
    *   `optype`: Type of ParMETIS operation.
    *   `partType`: Partitioning type (STATIC, ADAPTIVE, REFINE).
    *   `ps_relation`: Coarsening strategy (COUPLED, UNCOUPLED).
    *   `dbglvl`, `seed`: Debug level and random seed.
    *   `ncon`, `nparts`: Number of constraints and partitions.
    *   `tpwgts`: Target partition weights.
    *   `ubvec`: Imbalance tolerance vector.
    *   `invtvwgts`: Inverse of total vertex weights per constraint.
    *   `ipc_factor`, `redist_factor`, `redist_base`: Factors for balancing cut vs. redistribution.
    *   Arrays for MPI requests: `sreq`, `rreq`, `statuses`.
*   **`PMV3_OPTION_DBGLVL`, `PMV3_OPTION_SEED`, `PMV3_OPTION_PSR`**: Indices for accessing specific options in the user-provided `options` array.
*   **`UNBALANCE_FRACTION`**: Default value for imbalance if not specified.
*   **`GLOBAL_DBGLVL`, `GLOBAL_SEED`**: Default debug level and seed if `options` is not provided.

## Usage Examples

```
N/A
```
These functions are fundamental to the initialization and finalization phases of any ParMETIS routine. `SetupCtrl` is one of the first functions called to prepare the environment, and `FreeCtrl` is called at the end to release resources. End-users of the ParMETIS library indirectly cause these functions to be called when they invoke a ParMETIS API function (e.g., `ParMETIS_V3_PartKway`).

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `pmoptype_et`, `ctrl_t`, `graph_t` definitions, constants (e.g., `PARMETIS_OP_*`, `PARMETIS_PSR_*`, `UNBALANCE_FRACTION`), MPI wrappers (`gkMPI_*`), memory management (`gk_malloc`, `rmalloc`, `rsmalloc`, `rcopy`, `gk_free`), timer functions (`InitTimers`), and workspace management (`FreeWSpace`). It also includes `defs.h` which defines many default constants.
    *   `comm.c`: `GlobalSEMax` is used to synchronize the seed if needed.
    *   `graph.c`: `isum` (from `gklib.c` via `graph.c`'s includes) is used in `SetupCtrl_invtvwgts`.
    *   The `ctrl_t` structure created and managed here is passed to almost all other ParMETIS functions, controlling their behavior.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for `MPI_Comm_dup`, `MPI_Comm_rank`, `MPI_Comm_size`, `MPI_Comm_free`.
    *   Standard C library: `memset`, `srand`, `getpid`.
*   **Other Interactions**:
    *   The `options` array passed by the user directly influences the settings within the `ctrl_t` structure.
    *   The `dbglvl` set in `ctrl_t` controls debugging output throughout ParMETIS.

```
