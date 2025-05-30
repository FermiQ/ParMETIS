# libparmetis/xyzpart.c

## Overview

This file implements coordinate-based partitioning methods for ParMETIS. These methods use the geometric coordinates of vertices to partition the graph, often as an initial step before graph-based partitioning or refinement. The main idea is to sort vertices based on their coordinates (or derived values like Z-ordering or Hilbert curve ordering implicitly) and then divide the sorted list into `npes` (number of processors) contiguous blocks.

## Key Components

### Functions/Classes/Modules

*   **`Coordinate_Partition(ctrl_t *ctrl, graph_t *graph, idx_t ndims, real_t *xyz, idx_t setup)`**:
    *   The main function for coordinate-based partitioning.
    *   **Parameters**:
        *   `ctrl`: Control structure.
        *   `graph`: Graph structure (used for `vtxdist`, `nvtxs`, and to store the resulting partition in `graph->where`).
        *   `ndims`: Number of dimensions of the coordinates.
        *   `xyz`: Array of vertex coordinates (size `graph->nvtxs * ndims`).
        *   `setup`: Flag, if true, calls `CommSetup` (usually true for `PartGeomKway`, false for `PartGeom`).
    *   **Functionality**:
        1.  If `setup` is true, calls `CommSetup` to prepare graph communication structures.
        2.  Allocates `bxyz` to store binned coordinates and `cand (ikv_t *)` to store sort keys.
        3.  Calls `IRBinCoordinates` to discretize floating-point coordinates into integer bins.
        4.  **Z-ordering**: Computes a Z-order curve value (Morton code) for each vertex based on its binned coordinates (`bxyz`). This value interleaves the bits of the coordinates in different dimensions to create a 1D key that preserves spatial locality. The Z-order key is stored in `cand[i].key`, and the global vertex index in `cand[i].val`.
        5.  Calls `PseudoSampleSort` (a parallel sort variant) to sort the `cand` array based on the Z-order keys. This effectively sorts all global vertices by their Z-order.
        6.  The sorted list implicitly defines the partition: the first `gnvtxs/npes` vertices go to PE 0, the next to PE 1, and so on. `PseudoSampleSort` directly computes `graph->where` based on this sorted distribution.
*   **`IRBinCoordinates(ctrl_t *ctrl, graph_t *graph, idx_t ndims, real_t *xyz, idx_t nbins, idx_t *bxyz)`**:
    *   "Iterative Refinement Binning" of coordinates. Converts floating-point `xyz` coordinates into integer bin indices (`bxyz`) for each dimension.
    *   For each dimension `k`:
        1.  Sorts local vertices based on their coordinate in dimension `k`.
        2.  Determines global min/max coordinates in this dimension.
        3.  Initializes `nbins` `emarkers` (bin boundaries) uniformly spanning the global range.
        4.  Iteratively refines these bin boundaries for up to 5 iterations:
            *   Counts vertices in each bin globally (`gcounts`).
            *   If imbalance is high (e.g., `max(gcounts) > 4*gnvtxs/nbins`), adjusts `emarkers` to better distribute vertices, aiming for roughly `gnvtxs/nbins` per bin. This involves finding new split points within overpopulated bins.
        5.  Assigns each local vertex its bin number for dimension `k` based on the final `emarkers`.
*   **`RBBinCoordinates(ctrl_t *ctrl, graph_t *graph, idx_t ndims, real_t *xyz, idx_t nbins, idx_t *bxyz)`**:
    *   "Recursive Bisection Binning" of coordinates. Another method to discretize coordinates.
    *   For each dimension `k`:
        1.  Initializes 2 bins using the global center of mass as the separator.
        2.  Iteratively splits the most populated bins (based on `gcounts`) by their center of mass until `nbins` are created.
        3.  Assigns bin numbers to local vertices.
    *   *(Note: This function appears to be an alternative to `IRBinCoordinates` but `IRBinCoordinates` is called by `Coordinate_Partition`.)*
*   **`SampleSort(ctrl_t *ctrl, graph_t *graph, ikv_t *elmnts)`**:
    *   A parallel sample sort algorithm to sort `elmnts (ikv_t *)` distributed across processors.
    *   Each PE selects local splitters. These are gathered, globally sorted, and global splitters are chosen.
    *   Elements are then sent to PEs based on these global splitters (`MPI_Alltoall`).
    *   Each PE sorts its received elements.
    *   A final step determines the global rank and assigns partition numbers (`elmnts[i].key = target_pe`).
    *   The sorted `elmnts` (with new keys as target PEs and values as original global vertex IDs) are sent back to original PEs.
    *   `graph->where` is populated from the received sorted elements.
    *   *(Note: Assumes `nvtxs > npes` on each PE. Described as "poorly implemented" with a TODO to fix in 4.0. `PseudoSampleSort` is used instead by `Coordinate_Partition`.)*
*   **`PseudoSampleSort(ctrl_t *ctrl, graph_t *graph, ikv_t *elmnts)`**:
    *   A variant of parallel sample sort, likely optimized or simplified compared to `SampleSort`. This is the one actually used by `Coordinate_Partition`.
    *   **Steps**:
        1.  Selects local samples (`mypicks`) from locally sorted `elmnts`. The number of local samples `nlsamples` is determined heuristicall (e.g., `gnvtxs/(npes*npes)` capped by `npes` and a minimum).
        2.  Gathers all samples (`allpicks`) to all PEs.
        3.  Sorts `allpicks` globally and selects `npes-1` global splitters, stored back in `mypicks`.
        4.  Each PE determines how many of its local `elmnts` fall into buckets defined by global splitters (`scounts`).
        5.  `MPI_Alltoall` communicates these counts (`rcounts`).
        6.  `MPI_Alltoallv` redistributes `elmnts` based on these counts. Received elements are in `relmnts`.
        7.  `relmnts` are sorted locally.
        8.  A global prefix sum (`MPI_Scan`) on `nrecv` (number of elements received by each PE) determines the global rank range for each PE's elements.
        9.  Each PE assigns a target PE index (`relmnts[j].key = target_pe`) to its `relmnts` based on `vtxdist` (global vertex distribution for final partitions) and the global rank.
        10. `MPI_Alltoallv` sends `relmnts` back to the PEs that originally owned the vertex data (using original `scounts`/`rcounts`).
        11. `graph->where` is populated from the final received `elmnts`.

## Important Variables/Constants

*   **`xyz`**: Input array of vertex coordinates.
*   **`ndims`**: Number of dimensions for coordinates.
*   **`bxyz`**: Temporary array storing integer bin indices for each vertex and dimension.
*   **`cand (ikv_t *)`**: Array storing Z-order keys (`key`) and global vertex indices (`val`) for sorting.
*   **`nbins` (in `IRBinCoordinates`)**: Number of bins to discretize coordinates into (e.g., `1<<9 = 512`).
*   **`emarkers` (in `IRBinCoordinates`)**: Array of bin boundaries.
*   **`mypicks`, `allpicks` (in sorting functions)**: Arrays to store local and global samples/splitters.
*   **`elmnts` (in sorting functions)**: The array of `ikv_t` (key/value pairs) being sorted. Key is sort criteria, value is original global ID. After sorting and partitioning, key becomes target PE.
*   **`graph->where`**: Output array where the partition assignment (target PE) for each local vertex is stored.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions. `Coordinate_Partition` is the main routine called by higher-level functions like `ParMETIS_V3_PartGeomKway` (in `gkmetis.c`) or `ParMETIS_V3_PartGeom`.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants, MPI wrappers, memory utilities.
    *   `comm.c`: `CommSetup`, `CommUpdateNnbrs` (used by `SampleSort`).
    *   GKlib utilities: Sorting (`ikvsorti`, `ikvsortii`, `rkvsorti`), memory allocation, math (`gk_min`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used extensively in `IRBinCoordinates` and the sorting routines for `MPI_Allreduce`, `MPI_Allgather`, `MPI_Alltoall`, `MPI_Alltoallv`, `MPI_Scan`.
*   **Other Interactions**:
    *   The quality of the coordinate-based partition depends heavily on the spatial distribution of vertices and the effectiveness of the Z-ordering and parallel sorting in preserving locality.
    *   `PseudoSampleSort` is a complex parallel sorting algorithm critical to this partitioning method.
    *   These routines are often used to provide a good initial data distribution for more computationally intensive graph-based partitioning algorithms.

```
