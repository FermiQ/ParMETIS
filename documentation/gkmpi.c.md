# libparmetis/gkmpi.c

## Overview

This file provides a set of wrapper functions around standard MPI (Message Passing Interface) calls. The primary purpose of these wrappers is to allow ParMETIS to use its own `idx_t` type for counts and displacements in MPI calls, rather than being restricted to `int`. This is important for handling very large graphs where the number of vertices or edges might exceed the maximum value representable by a standard `int`.

The wrappers conditionally compile to use either the standard MPI functions (if `MPI_VERSION < 4` and `IDXTYPEWIDTH == 32`) or the newer MPI 4.0 `_c` suffixed functions that accept `MPI_Count` (which can be larger than `int`). For cases where `MPI_VERSION < 4` but `IDXTYPEWIDTH == 64` (meaning `idx_t` is 64-bit), these wrappers include logic to check for potential overflow if `idx_t` values are too large for `int`, and will call `errexit` if such an overflow is detected for collective operations that pass arrays of counts/displacements.

## Key Components

### Functions/Classes/Modules

Each function is a wrapper around a standard MPI function. The wrappers handle the potential type mismatch between `idx_t` and `int` or `MPI_Count`.

*   **`gkMPI_Comm_size(MPI_Comm comm, idx_t *size)`** wraps `MPI_Comm_size`.
*   **`gkMPI_Comm_rank(MPI_Comm comm, idx_t *rank)`** wraps `MPI_Comm_rank`.
*   **`gkMPI_Get_count(MPI_Status *status, MPI_Datatype datatype, idx_t *count)`** wraps `MPI_Get_count` or `MPI_Get_count_c`.
*   **`gkMPI_Send(void *buf, idx_t count, ..., MPI_Comm comm)`** wraps `MPI_Send` or `MPI_Send_c`.
*   **`gkMPI_Recv(void *buf, idx_t count, ..., MPI_Comm comm, MPI_Status *status)`** wraps `MPI_Recv` or `MPI_Recv_c`.
*   **`gkMPI_Isend(void *buf, idx_t count, ..., MPI_Request *request)`** wraps `MPI_Isend` or `MPI_Isend_c`.
*   **`gkMPI_Irecv(void *buf, idx_t count, ..., MPI_Request *request)`** wraps `MPI_Irecv` or `MPI_Irecv_c`.
*   **`gkMPI_Wait(MPI_Request *request, MPI_Status *status)`** wraps `MPI_Wait`.
*   **`gkMPI_Waitall(idx_t count, MPI_Request *array_of_requests, ...)`** wraps `MPI_Waitall`.
*   **`gkMPI_Barrier(MPI_Comm comm)`** wraps `MPI_Barrier`.
*   **`gkMPI_Bcast(void *buffer, idx_t count, ..., MPI_Comm comm)`** wraps `MPI_Bcast` or `MPI_Bcast_c`.
*   **`gkMPI_Reduce(void *sendbuf, void *recvbuf, idx_t count, ..., MPI_Comm comm)`** wraps `MPI_Reduce` or `MPI_Reduce_c`.
*   **`gkMPI_Allreduce(void *sendbuf, void *recvbuf, idx_t count, ..., MPI_Comm comm)`** wraps `MPI_Allreduce` or `MPI_Allreduce_c`.
*   **`gkMPI_Scan(void *sendbuf, void *recvbuf, idx_t count, ..., MPI_Comm comm)`** wraps `MPI_Scan` or `MPI_Scan_c`.
*   **`gkMPI_Allgather(void *sendbuf, idx_t sendcount, ..., MPI_Comm comm)`** wraps `MPI_Allgather` or `MPI_Allgather_c`.
*   **`gkMPI_Alltoall(void *sendbuf, idx_t sendcount, ..., MPI_Comm comm)`** wraps `MPI_Alltoall` or `MPI_Alltoall_c`.
*   **`gkMPI_Alltoallv(void *sendbuf, idx_t *sendcounts, ..., MPI_Comm comm)`**:
    *   Wraps `MPI_Alltoallv` or `MPI_Alltoallv_c`.
    *   If `MPI_VERSION < 4` and `IDXTYPEWIDTH == 64`, it dynamically allocates temporary `int` arrays for counts/displacements, copies `idx_t` values, checks for overflow, calls `MPI_Alltoallv`, and copies results back.
*   **`gkMPI_Allgatherv(void *sendbuf, idx_t sendcount, ..., MPI_Comm comm)`**:
    *   Wraps `MPI_Allgatherv` or `MPI_Allgatherv_c`.
    *   Similar overflow checks and temporary array usage as `gkMPI_Alltoallv` for the 64-bit `idx_t` / MPI 3.x case.
*   **`gkMPI_Scatterv(void *sendbuf, idx_t *sendcounts, ..., MPI_Comm comm)`**:
    *   Wraps `MPI_Scatterv` or `MPI_Scatterv_c`.
    *   Similar overflow checks and temporary array usage.
*   **`gkMPI_Gatherv(void *sendbuf, idx_t sendcount, ..., MPI_Comm comm)`**:
    *   Wraps `MPI_Gatherv` or `MPI_Gatherv_c`.
    *   Similar overflow checks and temporary array usage.
*   **`gkMPI_Comm_split(MPI_Comm comm, idx_t color, idx_t key, MPI_Comm *newcomm)`** wraps `MPI_Comm_split`.
*   **`gkMPI_Comm_free(MPI_Comm *comm)`** wraps `MPI_Comm_free`.
*   **`gkMPI_Finalize()`** wraps `MPI_Finalize`.

## Important Variables/Constants

*   **`MPI_VERSION`**: Preprocessor macro used to check the version of the MPI standard.
*   **`IDXTYPEWIDTH`**: Preprocessor macro (likely 32 or 64) indicating the bit width of `idx_t`.
*   **`INT_MAX`**: Used for overflow checking when `IDXTYPEWIDTH == 64` and `MPI_VERSION < 4`.

## Usage Examples

```c
// Internal ParMETIS code would call these gkMPI_* functions instead of direct MPI calls.
// Example:
// idx_t nsend = graph->sendptr[i+1] - graph->sendptr[i];
// gkMPI_Isend((void *)(sendbuffer + graph->sendptr[i]), nsend, IDX_T,
//             neighbor_pe, tag, ctrl->comm, ctrl->sreq + i);

// idx_t global_sum;
// idx_t local_val = 10;
// gkMPI_Allreduce((void *)&local_val, (void *)&global_sum, 1, IDX_T, MPI_SUM, ctrl->comm);
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `idx_t` type definition, `errexit` function for error handling, and memory management like `gk_imalloc`, `gk_free` (used for temporary buffers in the 64-bit `idx_t` / MPI 3.x case).
    *   These wrappers are used by other ParMETIS modules that perform MPI communication (e.g., `comm.c`, `diffutil.c`, `initbalance.c`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: This entire file consists of wrappers around MPI library calls. The behavior depends on the MPI version available at compile time.
*   **Other Interactions**:
    *   The conditional compilation based on `MPI_VERSION` and `IDXTYPEWIDTH` ensures that ParMETIS can be compiled with different MPI libraries and for different `idx_t` sizes while maintaining correctness for large datasets.
    *   The overflow checks and error exits are crucial for safety when using 64-bit `idx_t` with older MPI versions that expect `int` for counts/displacements in some vector collective operations.

```
