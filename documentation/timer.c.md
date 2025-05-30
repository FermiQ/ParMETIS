# libparmetis/timer.c

## Overview

This file provides utility functions for managing and printing timing information within ParMETIS. It allows different parts of the code to be profiled by starting, stopping, and clearing timers. A summary of these timings can then be printed, showing the maximum and sum of times across all MPI processes for each profiled section, along with a balance metric.

## Key Components

### Functions/Classes/Modules

*   **`InitTimers(ctrl_t *ctrl)`**:
    *   Initializes all predefined timers in the `ctrl_t` structure to zero.
    *   Calls `cleartimer()` for each timer field in `ctrl_t` (e.g., `ctrl->TotalTmr`, `ctrl->MatchTmr`, etc.).
*   **`PrintTimingInfo(ctrl_t *ctrl)`**:
    *   Prints a summary of all the timers defined in `ctrl_t`.
    *   For each timer, it calls `PrintTimer` to display its statistics.
*   **`PrintTimer(ctrl_t *ctrl, timer tmr, char *msg)`**:
    *   Calculates and prints statistics for a single timer value `tmr`.
    *   The `msg` string is a descriptive label for the timer.
    *   It performs MPI reductions to get the sum (`MPI_SUM`) and maximum (`MPI_MAX`) of the timer value `tmr` across all processors in `ctrl->comm`.
    *   If the sum of the timer is non-zero, it prints the maximum time, sum of times, and a balance metric (`max_time * npes / sum_time`) on PE 0.

## Important Variables/Constants

*   **`ctrl_t` (struct)**: This structure holds all the timer variables. Examples:
    *   `ctrl->TotalTmr`: Total time for the operation.
    *   `ctrl->InitPartTmr`: Time for initial partitioning.
    *   `ctrl->MatchTmr`: Time for matching.
    *   `ctrl->ContractTmr`: Time for graph contraction.
    *   `ctrl->CoarsenTmr`: (Commented out in `PrintTimingInfo` but present in `InitTimers`) Overall coarsening time.
    *   `ctrl->RefTmr`: (Commented out in `PrintTimingInfo` but present in `InitTimers`) Overall refinement time.
    *   `ctrl->SetupTmr`: Time for communication setup.
    *   `ctrl->ProjectTmr`: Time for partition projection.
    *   `ctrl->KWayInitTmr`: Time for k-way refinement initialization.
    *   `ctrl->KWayTmr`: Time for k-way refinement.
    *   `ctrl->MoveTmr`: Time for moving graph data.
    *   `ctrl->RemapTmr`: Time for remapping partitions.
    *   `ctrl->SerialTmr`: Time for serial operations.
    *   `ctrl->AuxTmr1` to `ctrl->AuxTmr6`: Auxiliary timers for custom profiling.
*   **`timer` (typedef for `double`)**: The data type used to store timer values.

## Usage Examples

```c
// How timers are typically used within ParMETIS code (using macros from macros.h):

// At the beginning of an operation (e.g., in SetupCtrl or a major function):
// InitTimers(ctrl);

// Around a specific code section to be timed:
// STARTTIMER(ctrl, ctrl->MatchTmr);
// // ... matching code ...
// STOPTIMER(ctrl, ctrl->MatchTmr);

// At the end of the main ParMETIS API function, if DBG_TIME is enabled:
// IFSET(ctrl->dbglvl, DBG_TIME, PrintTimingInfo(ctrl));
```
The timer functions themselves (`cleartimer`, `starttimer`, `stoptimer`, `gettimer`) are usually accessed via macros defined in `macros.h`.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t` definition, `timer` typedef, `idx_t`, `real_t` types, and MPI wrappers (`gkMPI_Reduce`, `gkMPI_Barrier`).
    *   `macros.h`: Defines the `cleartimer`, `starttimer`, `stoptimer`, `gettimer`, `STARTTIMER`, `STOPTIMER` macros which directly manipulate the timer values.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`:
        *   `MPI_Wtime()`: Used by the timer macros (via `starttimer`/`stoptimer`) to get wall-clock time.
        *   `gkMPI_Reduce`: Used in `PrintTimer` to compute sum and max of times across processors.
        *   `gkMPI_Barrier`: Used in `STARTTIMER`/`STOPTIMER` macros (if `DBG_TIME` is set) to synchronize before/after timing.
*   **Other Interactions**:
    *   Timer values are cumulative. `starttimer` subtracts current time, `stoptimer` adds current time.
    *   The output from `PrintTimingInfo` (via `PrintTimer`) is only produced on PE 0.
    *   The "Balance" metric printed indicates how well the workload for that timed section was distributed. A balance of 1.0 is perfect; higher values indicate imbalance.

```
