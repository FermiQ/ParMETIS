# libparmetis/wave.c

## Overview

This file implements the `WavefrontDiffusion` function, a specialized diffusion algorithm used in ParMETIS, particularly for initial load balancing when the number of constraints is one (`ncon=1`). The method models the flow of workload (vertices) from overloaded partitions to underloaded ones. It iteratively solves a diffusion equation on the partition connectivity graph and then moves a fraction of vertices based on the computed "flow" or "potentials."

## Key Components

### Functions/Classes/Modules

*   **`WavefrontDiffusion(ctrl_t *ctrl, graph_t *graph, idx_t *home)`**:
    *   Performs k-way directed diffusion for single-constraint load balancing.
    *   **Parameters**:
        *   `ctrl`: Control structure.
        *   `graph`: The graph data, including `graph->where` (current partition) and `graph->nvwgt` (normalized vertex weights).
        *   `home`: Array indicating the original home partition/processor of each vertex.
    *   **Functionality**:
        1.  **Initialization**:
            *   Allocates memory for partition connectivity matrix (`matrix`), flow vectors (`transfer`), load imbalance (`load`), solution potentials (`solution`), etc.
            *   Ensures all partitions initially have at least one vertex by moving vertices from the most populated partition if necessary.
            *   Computes initial external degrees (`ed`) of vertices and initial partition weights (`npwgts`).
            *   Calculates initial load imbalance (`ComputeLoad`).
        2.  **Diffusion Passes (`npasses`, limited by `NGD_PASSES` or `nparts/2`)**:
            *   In each pass `l`:
                *   Sets up the partition connectivity graph (`matrix`) using `SetUpConnectGraph`.
                *   Checks for disconnected subdomains in the partition graph. If connected:
                    *   Solves the diffusion equation `Ax = b` (where `A` is `matrix`, `b` is `load`) using `ConjGrad2` to get `solution` potentials.
                    *   Computes `transfer` amounts between partitions based on `solution` potentials using `ComputeTransferVector`.
                *   **Vertex Movement**:
                    *   Identifies the top three most overloaded partitions (`first`, `second`, `third`).
                    *   Permutes vertices (`perm`), prioritizing "dirty" vertices (not in their `home` partition) in some passes.
                    *   Iterates through a fraction of vertices (e.g., `nvtxs/3`), prioritizing those with high external degree (`ed`).
                    *   For a selected vertex `i` in partition `from`:
                        *   If `from` is one of the overloaded partitions (or vertex `i` is not in its `home`), it's considered for a move.
                        *   It checks its neighbors: if a neighbor `adjncy[j]` is in partition `to`, and the `transfer` from `from` to `to` is significant (e.g., `> flowFactor * nvwgt[i]`), the vertex `i` is moved to `to`.
                        *   Updates `psize` (local count of vertices per partition), `npwgts`, `load`, `where[i]`, and external degrees (`ed`) of `i` and its affected neighbors.
                *   Checks for convergence: if load balance (`balance`) is good enough (`< ubfactor + 0.035`) or no swaps occurred in a pass, the diffusion may terminate early.
        3.  **Finalization**:
            *   Computes the final edge cut (`ComputeSerialEdgeCut`) and total vertices moved from home (`Mc_ComputeSerialTotalV`).
            *   Calculates a final `cost` based on cut and total moved volume.
    *   Returns the computed `cost`.

## Important Variables/Constants

*   **`matrix_t matrix`**: Represents the graph of connections between partitions.
*   **`load`**: Vector storing current load imbalance for each partition.
*   **`solution`**: Vector storing "potentials" or "pressures" from the CG solver.
*   **`transfer`**: Vector storing computed flow amounts along edges of the partition graph.
*   **`ed`**: Array storing the external degree of each vertex.
*   **`psize`**: Array storing the number of vertices in each partition locally.
*   **`flowFactor`**: A factor (0.35 to 1.0, can depend on `mype` in some versions) determining the threshold for moving a vertex based on `transfer` and its weight.
*   **`NGD_PASSES`**: Constant from `defs.h`, upper limit on diffusion passes.
*   `ubfactor`: Target unbalance factor from `ctrl->ubvec[0]`.

## Usage Examples

```
N/A
```
This is an internal ParMETIS function. `WavefrontDiffusion` is typically called by `Balance_Partition` (in `initbalance.c`) as one of the strategies to achieve an initial balanced partition, especially when `ncon == 1`.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants.
    *   `diffutil.c`: Uses `SetUpConnectGraph`, `ComputeLoad`, `ConjGrad2`, `ComputeTransferVector`, `Mc_ComputeSerialTotalV`.
    *   `serial.c`: Uses `ComputeSerialEdgeCut`.
    *   `util.c`: Uses `FastRandomPermute`, `RandomPermute`, `GetThreeMax`, `iargmax`.
    *   GKlib utilities: For memory allocation, array operations.
*   **External Libraries**:
    *   None directly, but underlying functions use MPI.
*   **Other Interactions**:
    *   Modifies `graph->where` to achieve better load balance.
    *   The `flowFactor` and number of passes (`npasses`, `NGD_PASSES`) control the aggressiveness and extent of diffusion.
    *   The function assumes a single constraint (`ncon=1`).
    *   The permutation of vertices (especially prioritizing dirty ones) tries to improve stability or convergence.

```
