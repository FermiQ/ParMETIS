# libparmetis/frename.c

## Overview

This file provides Fortran wrapper interfaces for several ParMETIS C API functions. It addresses the common issue of name mangling differences between C and Fortran compilers by defining multiple versions of the function name (e.g., all uppercase, all lowercase, with trailing underscore, with double trailing underscore). Each wrapper function converts the Fortran MPI communicator (`MPI_Fint`) to a C MPI communicator (`MPI_Comm`) before calling the actual C ParMETIS function.

## Key Components

### Functions/Classes/Modules

The file primarily consists of a macro `FRENAME` and its instantiations for various ParMETIS API functions.

*   **`FRENAME(name0, name1, name2, name3, name4, dargs, cargs)` (Macro)**:
    *   A macro used to generate the Fortran wrapper functions.
    *   `name0`: The actual C function name (e.g., `ParMETIS_V3_AdaptiveRepart`).
    *   `name1` to `name4`: The variously mangled Fortran names (e.g., `PARMETIS_V3_ADAPTIVEREPART`, `parmetis_v3_adaptiverepart`, `parmetis_v3_adaptiverepart_`, `parmetis_v3_adaptiverepart__`).
    *   `dargs`: The declaration of arguments for the wrapper functions, including `MPI_Fint *icomm`.
    *   `cargs`: The arguments to be passed to the C function `name0`, with `*icomm` being converted to `comm` via `MPI_Comm_f2c`.

The following ParMETIS V3 API functions are wrapped using this macro:
*   **`ParMETIS_V3_AdaptiveRepart`**: For adaptive repartitioning.
*   **`ParMETIS_V3_PartGeomKway`**: For geometric k-way partitioning using coordinates and graph structure.
*   **`ParMETIS_V3_PartGeom`**: For geometric partitioning based only on coordinates.
*   **`ParMETIS_V3_PartKway`**: For graph partitioning using k-way multilevel algorithm.
*   **`ParMETIS_V3_Mesh2Dual`**: For converting a mesh to its dual graph.
*   **`ParMETIS_V3_PartMeshKway`**: For partitioning a mesh.
*   **`ParMETIS_V3_NodeND`**: For computing fill-reducing orderings of sparse matrices.
*   **`ParMETIS_V3_RefineKway`**: For refining an existing k-way partition.

Each instantiation of `FRENAME` creates four wrapper functions. For example, for `ParMETIS_V3_AdaptiveRepart`, it creates:
*   `PARMETIS_V3_ADAPTIVEREPART(...)`
*   `parmetis_v3_adaptiverepart(...)`
*   `parmetis_v3_adaptiverepart_(...)`
*   `parmetis_v3_adaptiverepart__(...)`

All these wrappers internally call the C function `ParMETIS_V3_AdaptiveRepart` after converting the Fortran MPI communicator.

## Important Variables/Constants

*   **`MPI_Fint *icomm`**: Fortran MPI communicator handle passed to the wrapper.
*   **`MPI_Comm comm`**: C MPI communicator handle, converted from `*icomm` using `MPI_Comm_f2c`.

## Usage Examples

```fortran
! Example of how a Fortran program might call one of these wrappers
! (assuming appropriate ParMETIS library linking and USE MPI)

INTEGER(KIND=MPI_ADDRESS_KIND) :: vtxdist(npes+1)
INTEGER(KIND=MPI_ADDRESS_KIND) :: xadj(nvtxs+1)
INTEGER(KIND=MPI_ADDRESS_KIND) :: adjncy(nedges)
! ... other parameters ...
INTEGER :: part(nvtxs)
INTEGER :: edgecut
INTEGER :: options(PARMETIS_MAX_OPTIONS)
INTEGER :: icomm_f ! Fortran MPI communicator
! ... initialize parameters and icomm_f ...

CALL parmetis_v3_partkway(vtxdist, xadj, adjncy, vwgt, adjwgt, &
                         wgtflag, numflag, ncon, nparts, tpwgts, ubvec, &
                         options, edgecut, part, icomm_f)
```
The user links against the ParMETIS library, and the Fortran linker picks the appropriately named symbol based on the compiler's convention.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides the prototypes for the actual C ParMETIS API functions (e.g., `ParMETIS_V3_AdaptiveRepart`).
    *   The C functions defined elsewhere in the library (e.g., `ametis.c`, `ometis.c`, `kmetis.c`) are the ultimate targets of these wrappers.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Requires the MPI library for the `MPI_Comm_f2c` function to convert Fortran communicators to C communicators.
*   **Other Interactions**:
    *   This file is crucial for Fortran interoperability. Without it, Fortran programs would have a much harder time calling ParMETIS C functions due to name mangling and MPI communicator type differences.
    *   The specific set of mangled names provided (uppercase, lowercase, single underscore, double underscore) covers common conventions for Fortran compilers on various systems.

```
