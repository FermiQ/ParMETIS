# libparmetis/match.c

## Overview

This file implements graph matching algorithms, which are crucial for the coarsening phase of multilevel graph partitioning. It provides two main matching strategies: `Match_Global` for matching that can span across processors, and `Match_Local` for matching restricted to vertices within the same processor (based on their `home` or `where` info). The goal of matching is to select pairs of vertices (or groups) that will be merged to form the vertices of the next coarser graph.

## Key Components

### Functions/Classes/Modules

*   **`Match_Global0(ctrl_t *ctrl, graph_t *graph)`**:
    *   (Older version, possibly for comparison or specific internal use, as `Match_Global` seems to be the primary one called elsewhere like `kmetis.c`).
    *   Finds a Heavy Edge Matching (HEM) that can involve both local and remote vertices.
    *   Iterates `NMATCH_PASSES`. In each pass:
        *   Processes unmatched vertices in a permuted order.
        *   For each vertex `i`, it tries to find a suitable match `k` (local or remote) that maximizes edge weight, considering `myhome[i] == myhome[k]` (vertices should belong to the same original partition/PE for some schemes).
        *   Local matches are granted immediately.
        *   Remote match requests are sent to the target PE. Target PEs grant requests if their vertex is unmatched, with some logic (`nkept`) to balance who "keeps" the matched pair for the coarse graph.
        *   Two-hop matching is considered in the last pass if `ctrl->twohop` is set and local matching fails.
    *   Vertices that remain unmatched are matched with themselves (they carry over to the coarser graph).
    *   Sets `graph->match_type = PARMETIS_MTYPE_GLOBAL`.
    *   Calls `CreateCoarseGraph_Global` and optionally `DropEdges`.
*   **`Match_Global(ctrl_t *ctrl, graph_t *graph)`**:
    *   This appears to be the primary global matching function. Structurally very similar to `Match_Global0`.
    *   It performs multiple passes (`NMATCH_PASSES`) of attempting to match vertices.
    *   **Matching Logic (per pass)**:
        1.  Vertices are processed in a random order (`perm`).
        2.  Vertices marked `TOO_HEAVY` (normalized weight `nvwgt > maxnvwgt`) are not matched.
        3.  For an unmatched vertex `i`:
            *   If it's an island (no edges), it tries to match with another local island vertex.
            *   Otherwise, it seeks a heavy-edge match `k` where `myhome[i] == myhome[k]`. Preference is given to heavier edges, and for multi-constraint, `BetterVBalance` is used as a tie-breaker.
            *   If `k` is local, the match `(i, k)` is made.
            *   If `k` is remote, `i` is marked `MAYBE_MATCHED`, and a request is sent to the PE owning `k`. A "wside" logic (alternating preference based on global vertex ID) determines which PE initiates the request.
        4.  **Request Resolution**: MPI communication exchanges match requests. A PE receiving a request grants it if its local vertex is available. A heuristic (`nkept`) attempts to balance which PE gets to "keep" the resulting coarse vertex.
        5.  Granted matches are communicated back, and local `match` arrays are updated.
        6.  A final two-hop matching pass (`ctrl->twohop`) can be performed to match remaining vertices by looking at neighbors of neighbors (only if the intermediate vertex is local, based on the `Match_Global0` two-hop logic, though `Match_Global`'s two-hop part is more complex and involves creating a temporary transpose adjacency).
    *   Unmatched or `TOO_HEAVY` vertices are matched with themselves.
    *   Calls `CreateCoarseGraph_Global` and optionally `DropEdges`.
*   **`Match_Local(ctrl_t *ctrl, graph_t *graph)`**:
    *   Finds a Heavy Edge Matching (HEM) involving only local vertices.
    *   Vertices are matched only if `myhome[vertex1] == myhome[vertex2]` and both are local.
    *   Heavy vertices (weight `> maxnvwgt`) are not matched.
    *   Iterates through permuted local vertices. If a vertex `i` is unmatched and not too heavy, it finds the best local match `k` (also not too heavy) based on edge weight and (for multi-constraint) `BetterVBalance`.
    *   Matched pairs are recorded in `graph->match`. Unmatched vertices are matched with themselves.
    *   Sets `graph->match_type = PARMETIS_MTYPE_LOCAL`.
    *   Calls `CreateCoarseGraph_Local`.
*   **`CreateCoarseGraph_Global(ctrl_t *ctrl, graph_t *graph, idx_t cnvtxs)`**:
    *   Constructs the coarser graph (`cgraph`) based on the `match` array from `Match_Global`.
    *   Calculates `cgraph->vtxdist` (coarse graph vertex distribution).
    *   Populates `graph->cmap` which maps fine vertices to coarse vertices. This involves communication for remote matches.
    *   Exchanges adjacency lists of fine vertices that are matched with remote vertices. `scand`/`rcand` and `sgraph`/`rgraph` are used to manage this data transfer.
    *   Collapses matched fine vertices into coarse vertices:
        *   Sums vertex weights (`cvwgt`, `cvsize`).
        *   Sets `chome` for the coarse vertex (usually from the "keeper" of the match).
        *   Merges adjacency lists, summing edge weights for common neighbors, using a hash table (`htable`) to detect common neighbors efficiently.
    *   Initializes `cgraph->nvwgt`.
*   **`CreateCoarseGraph_Local(ctrl_t *ctrl, graph_t *graph, idx_t cnvtxs)`**:
    *   Constructs the coarser graph based on a local matching.
    *   Similar to `CreateCoarseGraph_Global` but simpler as all matches are local.
    *   Calculates `cgraph->vtxdist`.
    *   Constructs `graph->cmap` (local operation, then `CommInterfaceData` to share with neighbors).
    *   Collapses locally matched fine vertices into coarse vertices, summing weights and merging adjacency lists using a hash table.
*   **`DropEdges(ctrl_t *ctrl, graph_t *graph)`**:
    *   (Optional, if `ctrl->dropedges` is set) A heuristic to reduce the number of edges in the coarser graph.
    *   For each vertex, it determines a median weight of its incident edges (considering some random noise).
    *   Edges with weight less than the minimum of the median weights of their endpoints are dropped.
    *   This aims to remove less significant edges to speed up coarsening and potentially improve quality. Requires `CommSetup` on the coarse graph first.

## Important Variables/Constants

*   **`graph->match`**: Array storing matching information. `match[i] = global_idx_of_partner + KEEP_BIT` if local PE keeps the coarse vertex, or `global_idx_of_partner` if remote PE keeps it. `UNMATCHED`, `MAYBE_MATCHED`, `TOO_HEAVY` are special states.
*   **`graph->cmap`**: Maps fine graph vertex indices to coarse graph vertex indices.
*   **`graph->home` / `myhome`**: Array indicating the original partition/PE of vertices, used to guide matching (e.g., match only within the same home).
*   **`maxnvwgt`**: Threshold for marking vertices as `TOO_HEAVY`. `0.75 / ctrl->CoarsenTo`.
*   **`NMATCH_PASSES`**: Number of matching iterations.
*   **`KEEP_BIT`**: A bit flag (e.g., `1<<30`) added to `match[i]` to indicate that the current PE is responsible for the coarse vertex formed by this match.
*   `htable` (in CreateCoarseGraph*): Hash table used for efficient merging of adjacency lists.
*   `LHTSIZE`, `MASK`: Constants for the local hash table in `CreateCoarseGraph_Local`.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions used within the multilevel coarsening process (e.g., called by `Global_Partition` in `kmetis.c` or `Adaptive_Partition` in `ametis.c`).

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants.
    *   `graph.c`: For `CreateGraph`, `FreeCommSetupFields`.
    *   `comm.c`: For `CommInterfaceData`, `CommChangedInterfaceData`, `CommSetup`.
    *   `util.c`: For `FastRandomPermute`, `RandomPermute`, `RandomInRange`, `BetterVBalance`.
    *   `wspace.c`: For workspace allocation.
    *   `gklib.c` (indirectly): For GKlib utilities like `icopy`, `iset`, `imalloc`, `ikvsorti`, `isortd`, `isum`, `gk_max`, `gk_min`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For all inter-processor communication during global matching and coarse graph construction.
*   **Other Interactions**:
    *   The output of these matching functions (`graph->match`, `graph->cmap`) and the resulting coarse graph (`graph->coarser`) are fundamental inputs to the next level of the multilevel algorithm.
    *   The `ctrl->partType` (STATIC, ADAPTIVE, ORDER) influences how `home` information is used.
    *   `ctrl->twohop` and `ctrl->dropedges` enable optional heuristics.

```
