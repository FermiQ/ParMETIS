# libparmetis/rename.h

## Overview

This header file is part of ParMETIS's strategy to avoid symbol clashes with other libraries or user code, similar to `gklib_rename.h` but for ParMETIS's own internal functions. It defines macros that prepend `libparmetis__` to a comprehensive list of ParMETIS internal function names. When the ParMETIS library is compiled, these macros ensure that the compiled symbols have this prefix.

This mechanism is primarily for internal use and allows ParMETIS to be robustly linked into larger applications that might, for example, use different versions of METIS/ParMETIS or have functions with coincidentally similar names.

## Key Components

### Functions/Classes/Modules/Macros/Structs

This file consists entirely of `#define` preprocessor directives. Each directive renames an internal ParMETIS function.

Examples of redefinitions:
*   `#define KWayAdaptiveRefine libparmetis__KWayAdaptiveRefine`
*   `#define CommSetup libparmetis__CommSetup`
*   `#define Match_Global libparmetis__Match_Global`
*   `#define InitPartition libparmetis__InitPartition`
*   `#define KWayFM libparmetis__KWayFM`
*   `#define MoveGraph libparmetis__MoveGraph`
*   `#define Balance_Partition libparmetis__Balance_Partition`
*   `#define Mc_Diffusion libparmetis__Mc_Diffusion`
*   `#define MultilevelOrder libparmetis__MultilevelOrder`
*   ... and many more, covering functions from most `.c` files in `libparmetis`.

The list includes functions related to:
*   Adaptive refinement (`KWayAdaptiveRefine`)
*   Partitioning core logic (`Adaptive_Partition`, `Global_Partition`)
*   Balancing (`BalanceMyLink`, `RedoMyLink`, `Balance_Partition`)
*   Communication (`CommSetup`, `CommInterfaceData`)
*   Matching and Coarsening (`CSR_Match_SHEM`, `Match_Global`, `CreateCoarseGraph_Global`)
*   Control structure management (`SetupCtrl`, `FreeCtrl`)
*   Debugging (`PrintGraph`, `PrintVector`)
*   Diffusion utilities (`ComputeLoad`, `ConjGrad2`)
*   MPI wrappers (`gkMPI_Allreduce`, etc. - though these are often GKlib renames themselves, this file might list them for completeness or if ParMETIS has another layer of wrappers).
*   Graph structure management (`CreateGraph`, `SetupGraph`)
*   Initial partitioning/multisection (`InitPartition`, `InitMultisection`)
*   Refinement (`KWayFM`, `KWayBalance`, `ProjectPartition`)
*   Remapping and Moving graphs (`ParallelReMapGraph`, `MoveGraph`)
*   Mesh operations and setup (`SetUpMesh`) - *Self-correction: `SetUpMesh` is listed, but `ParMETIS_V3_Mesh2Dual` (the API function from `mesh.c`) is not, as API functions are not renamed.*
*   Node refinement for ordering (`KWayNodeRefine_Greedy`)
*   Ordering (`MultilevelOrder`, `CompactGraph`)
*   Serial utilities used in parallel context (`ComputeSerialEdgeCut`)
*   Statistics and timing (`ComputeParallelBalance`, `PrintTimingInfo`)
*   General utilities (`BSearch`, `RandomPermute`)
*   Workspace management (`AllocateWSpace`)
*   Geometric partitioning (`Coordinate_Partition`)

## Important Variables/Constants

Not applicable, as this file only contains macro definitions for renaming functions.

## Usage Examples

```c
// This header file is not directly used by end-users.
// It's included by parmetislib.h, which is the main internal header.

// In a ParMETIS internal .c file (e.g., kmetis.c):
// #include <parmetislib.h> // This will include rename.h (likely before proto.h)
// ...
// // When kmetis.c calls Match_Global(...), the preprocessor
// // changes it to libparmetis__Match_Global(...) before compilation.

// In proto.h (function prototypes):
// void libparmetis__Match_Global(ctrl_t *, graph_t *);
//
// In match.c (function implementation):
// void libparmetis__Match_Global(ctrl_t *ctrl, graph_t *graph) {
//   // ...
// }
```
When ParMETIS source code is compiled, any call to an internal ParMETIS function listed in `rename.h` is replaced by its `libparmetis__` prefixed version. The actual implementations of these functions also use these prefixed names.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   This file is included by `parmetislib.h`.
    *   It directly affects how internal function calls are compiled. The function implementations in their respective `.c` files must define the `libparmetis__*` prefixed names. The function prototypes in `proto.h` must also declare these prefixed names.
    *   It's a companion to `gklib_rename.h`, which does the same for embedded GKlib functions. This `rename.h` handles ParMETIS's own internal functions.
*   **External Libraries**:
    *   None. This file only contains preprocessor definitions.
*   **Other Interactions**:
    *   This renaming strategy is crucial for creating a robust ParMETIS library that avoids symbol name conflicts when linked into larger applications or with other libraries. It encapsulates ParMETIS's internal symbols.

```
