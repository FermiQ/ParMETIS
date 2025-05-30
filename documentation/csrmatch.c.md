# libparmetis/csrmatch.c

## Overview

This file contains code for computing matchings in a graph represented in Compressed Sparse Row (CSR) format. The specific matching algorithm implemented is the Sorted Heavy Edge Matching (SHEM) heuristic. This is often used in multilevel graph partitioning during the coarsening phase to select pairs of vertices to collapse together.

## Key Components

### Functions/Classes/Modules

*   **`CSR_Match_SHEM(matrix_t *matrix, idx_t *match, idx_t *mlist, idx_t *skip, idx_t ncon)`**:
    *   Computes a matching in the graph/matrix using the Sorted Heavy-Edge Matching (SHEM) heuristic.
    *   **Parameters**:
        *   `matrix`: A `matrix_t` structure representing the graph (or matrix). It's expected to have `nrows`, `rowptr` (CSR row pointers), `colind` (CSR column indices), and `transfer` (analogous to edge weights, possibly representing flow or connectivity strength across `ncon` constraints).
        *   `match`: An array of size `nrows` that will store the matching information. `match[i] = j` means vertex `i` is matched with vertex `j`. Unmatched vertices are marked `UNMATCHED`.
        *   `mlist`: An array that stores the pairs of matched vertices. For each match `(i, j)`, it stores `max(i,j)` then `min(i,j)`.
        *   `skip`: An array indicating edges that should be ignored (skipped) during matching. `skip[j_idx] == 0` means the edge can be considered.
        *   `ncon`: The number of constraints or components associated with edge weights (from `matrix->transfer`). The matching considers the maximum absolute value across these constraints as the edge weight.
    *   **Functionality**:
        1.  Initializes all vertices as `UNMATCHED`.
        2.  Creates a list of `links` (edges), where each link's weight is the maximum absolute value of `transfer` over `ncon` constraints for that edge. Associates each link with its source vertex.
        3.  Sorts the vertices based on the heaviest link incident to them in descending order (`rkvsortd(nrows, links)` on `links[i].key`).
        4.  Iterates through the sorted vertices:
            *   If a vertex `i` is `UNMATCHED`:
                *   It searches for an `UNMATCHED` neighbor `edge` connected by a non-skipped edge.
                *   Among eligible neighbors, it selects the one that forms the heaviest edge with `i` (based on `matrix->transfer` values).
                *   If such a neighbor `maxidx` is found (and `maxidx != i`), it matches `i` with `maxidx` (`match[i] = maxidx`, `match[maxidx] = i`) and records the pair in `mlist`.
        5.  Frees the temporary `links` array.

## Important Variables/Constants

*   **`matrix->rowptr`, `matrix->colind`**: CSR representation of the graph structure.
*   **`matrix->transfer`**: Array storing edge weights, potentially multi-constraint. The matching heuristic uses the max absolute value over `ncon` as the effective edge weight.
*   **`match[i] = UNMATCHED`**: Indicates vertex `i` is not yet matched.
*   **`skip[j_idx]`**: If non-zero, the edge at index `j_idx` in the CSR structure is ignored.
*   **`links (rkv_t *)`**: Temporary array of key-value pairs used to sort vertices by their heaviest incident edge weight. `key` stores the weight, `val` stores the vertex index.

## Usage Examples

```
N/A
```
This function is an internal component of ParMETIS, typically used during the graph coarsening phase of multilevel partitioning algorithms. It's not intended for direct use by end-users. A higher-level coarsening function would call `CSR_Match_SHEM` to find a matching, which then guides how vertices are merged to create a coarser graph.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides the `matrix_t` structure definition, `rkv_t` type, memory management (`rkvmalloc`, `gk_free`), sorting (`rkvsortd`), mathematical utilities (`fabs`, `gk_max`, `gk_min`), and the `UNMATCHED` constant.
    *   This function is a key part of the coarsening process. The output `match` array is used by other functions to construct the next coarser graph.
*   **External Libraries**:
    *   None explicitly mentioned beyond standard C libraries implicitly used by helpers (e.g., `math.h` for `fabs`).
*   **Other Interactions**:
    *   The quality of the matching produced by `CSR_Match_SHEM` can significantly impact the overall quality of the partitioning produced by ParMETIS.
    *   The `skip` array allows other parts of the algorithm to prevent certain edges from being part of the matching.

```
