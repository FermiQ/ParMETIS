# libparmetis/msetup.c

## Overview

This file contains routines for setting up and initializing the `mesh_t` data structure in ParMETIS. The `mesh_t` structure is used to represent unstructured finite element meshes, which are inputs to mesh-based partitioning algorithms like `ParMETIS_V3_PartMeshKway`. The setup involves storing mesh element connectivity, element weights, and determining some global mesh properties like the range of node IDs.

## Key Components

### Functions/Classes/Modules

*   **`SetUpMesh(idx_t *etype, idx_t *ncon, idx_t *elmdist, idx_t *elements, idx_t *elmwgt, idx_t *wgtflag, MPI_Comm *comm)`**:
    *   This function creates and initializes a `mesh_t` structure from user-provided mesh data.
    *   **Parameters**:
        *   `etype`: Pointer to the element type (e.g., triangle, tetrahedron).
        *   `ncon`: Pointer to the number of weights per element (if weighted).
        *   `elmdist`: Array defining the distribution of elements across processors.
        *   `elements`: Array containing the node IDs for each element, concatenated.
        *   `elmwgt`: Array of element weights (if `wgtflag` indicates).
        *   `wgtflag`: Flag indicating if element weights are provided.
        *   `comm`: MPI communicator.
    *   **Functionality**:
        1.  Gets MPI rank (`mype`) and size (`npes`).
        2.  Creates a `mesh_t` structure (`CreateMesh`).
        3.  Sets basic mesh properties: `elmdist`, `gnelms` (global number of elements), `nelms` (local number of elements), `elements` array, `elmwgt` array, `etype`, `ncon`.
        4.  Determines `esize` (number of nodes per element) based on `etype`.
        5.  If element weights are not provided (`wgtflag&1 == 0`), allocates and initializes `mesh->elmwgt` to 1 for each local element and constraint.
        6.  Normalizes global node IDs:
            *   Finds the global minimum node ID (`gminnode`) across all processors.
            *   Subtracts `gminnode` from all node IDs in the local `elements` array. This makes node IDs start effectively from 0 globally for internal processing.
            *   Stores `gminnode` in `mesh->gminnode` (to restore original numbering later if needed).
        7.  Finds the global maximum of these normalized node IDs (`gmaxnode`) to determine `mesh->gnns` (global number of unique nodes).
    *   Returns the initialized `mesh_t` structure.
*   **`CreateMesh(void)`**:
    *   Allocates memory for a `mesh_t` structure.
    *   Calls `InitMesh` to initialize its fields to default null/invalid values.
    *   Returns the allocated `mesh_t` pointer.
*   **`InitMesh(mesh_t *mesh)`**:
    *   Sets all fields of a `mesh_t` structure to zero or default invalid values (e.g., -1, NULL). This ensures a clean state before population.

## Important Variables/Constants

*   **`mesh_t` (struct)**: The central data structure for representing distributed meshes. Key members include:
    *   `etype`: Type of mesh element (e.g., triangle, tetrahedron).
    *   `esize`: Number of nodes per element of `etype`.
    *   `ncon`: Number of weights associated with each element.
    *   `elmdist`: Array defining the distribution of elements to PEs.
    *   `nelms`, `gnelms`: Local and global number of elements.
    *   `elements`: Concatenated list of node IDs for each local element.
    *   `elmwgt`: Array of weights for local elements.
    *   `nns`, `gnns`: Local (not explicitly stored/computed here) and global number of unique nodes.
    *   `gminnode`: The minimum global node ID found in the input mesh, used for normalizing node IDs.
*   **`esizes[5]` (local array in `SetUpMesh`)**: Maps `etype` to the number of nodes per element (e.g., `esizes[1]=3` for triangles if `etype=1` means triangle).

## Usage Examples

```
N/A
```
These are internal ParMETIS functions. `SetUpMesh` is called at the beginning of ParMETIS API functions that operate on meshes, such as `ParMETIS_V3_PartMeshKway` (via `mmetis.c`) or `ParMETIS_V3_Mesh2Dual` (via `mesh.c`, though `mesh.c` has its own setup logic which is more complex as it directly prepares for dual graph construction). `SetUpMesh` primarily prepares the `mesh_t` structure for basic access to mesh elements and their properties.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `mesh_t` definition, `idx_t` type, MPI wrappers (`gkMPI_Comm_size`, `gkMPI_Comm_rank`, `gkMPI_Allreduce`), memory management (`gk_malloc`, `ismalloc`), and math utilities (`imin`, `imax`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for determining `gminnode` and `gmaxnode` via `gkMPI_Allreduce`.
*   **Other Interactions**:
    *   The `elements` array is modified in place by `SetUpMesh` to contain normalized node IDs (0-based with respect to the global minimum).
    *   The `mesh_t` structure populated by `SetUpMesh` is then used by other functions, for example, as a precursor to building a dual graph of the mesh.

```
