# DEMREG Optimization Report

## Context and Problem Statement

- We recover **DEM(T)** from multi-channel counts using a temperature response matrix and **GSVD-regularized** inversion.
- Outputs per pixel: `DEM(T)`, uncertainty `edem`, log-temperature grid `elogt`, regularized counts `dn_reg`, and **χ²** misfit.
- The optimization focuses on the core pipeline implemented in the four modules:
  - `demmap_pos.py` (per-pixel solver / parallelization)
  - `dem_inv_gsvd.py` (GSVD + Tikhonov regularization core)
  - `dem_reg_map.py` (μ-grid search / discrepancy principle)

## Profiling & Analysis of the Baseline

**Method.**  
We profiled the baseline implementation from the `Baseline/` directory using `cProfile`, `timeit`, and scaling tests (`Benchmark_DEM.ipynb`). The benchmark used 1000 randomly generated DEMs with 6 channels and 200 temperature bins, representing a typical 1D or small 2D use case of the DEM inversion pipeline.

**Analyzed Modules:**
- `dem_inv_gsvd.py` – performs the Generalized Singular Value Decomposition (GSVD).  
- `dem_reg_map.py` – determines the regularization parameter μ using the discrepancy principle.  
- `demmap_pos.py` – the per-pixel DEM inversion including positivity enforcement and parallel execution.  
- `dn2dem_pos.py` – wrapper handling 0D, 1D, and 2D input data.

**Profiling Environment:**
- CPU: Intel/AMD, 1 thread (NumPy BLAS)
- Python: 3.11.14 / NumPy: 2.3.3  
- Command:  
  ```bash
  python bench_demreg_sxs.py \
      --baseline_dir Baseline \
      --width 256 --height 256 --threads 1 \
      --repeats 3 --outdir ./dask_opt/bench_out_baseline


**Environment (baseline run):**

- CPU: {CPU_MODEL}, threads: 1
- Python/NumPy: 3.11.14 / 2.3.3
- Command:
  ```bash
  python bench_demreg_sxs.py --baseline_dir Baseline --improved_dir CPU_Vectorization --width 256 --height 256 --threads 1 --repeats 3 --outdir {OUTDIR}
  ```

**Key Observations**

| Category | Observation |
| --- | --- |
| **Hotspots** | >80% of wall time spent inside `demmap_pos → dem_pix` (GSVD + μ search). |
| **GSVD** | `numpy.linalg.svd` is the dominant cost; each pixel triggers a full GSVD decomposition. |
| **μ-grid search** | The loop over 42--500 μ samples scales linearly in runtime; unnecessarily long grids waste time. |
| **Parallelization** | Using `ProcessPoolExecutor` introduces non-trivial overhead; combined with BLAS multithreading, this can lead to oversubscription and reduced throughput. |
| **Memory** | High number of temporary array allocations in `dem_pix()` leads to heavy memory traffic and cache misses. |
| **Scalability** | Runtime grows nearly linearly with the number of pixels; the solver's structure remains fundamentally sequential per pixel. |


**Function-Level Summary (cProfile)**
| Function                                 | Time Share              | Comment                                 |
| ---------------------------------------- | ----------------------- | --------------------------------------- |
| `demmap_pos (Baseline/demmap_pos.py:10)` | ~99% total runtime      | Main computation routine.               |
| `np.linalg.svd` (inside `dem_inv_gsvd`)  | Major share of CPU time | Matrix decomposition per pixel.         |
| `threadpoolctl`, `ProcessPoolExecutor`   | Minor but measurable    | Thread and process management overhead. |

**Behavior Analysis**

-   The GSVD + μ-search dominates both CPU and memory usage.

-   The parallelization approach, while conceptually sound, does not scale efficiently because each worker still performs an independent GSVD for every pixel.

-   The balance between `n_par` batch size and `nmu` (number of μ samples) determines total runtime.

-   Small data sets perform worse due to parallel initialization costs, while larger batches achieve near-linear scaling but are still bound by Python-level iteration overhead.

**Conclusion**

The baseline version provides a correct and stable reference implementation of the DEMREG algorithm but is constrained by several structural bottlenecks:

-   Expensive **per-pixel GSVD** computations dominate total runtime.

-   Inefficient **Python-level loops** and temporary array allocations slow down execution.

-   **Parallel processing** is limited by high inter-process communication and BLAS oversubscription.

These insights directly motivated the subsequent optimization strategies implemented in the project:

1.  **CPU Vectorization (NumPy broadcasting)** -- reduce Python-level loops and exploit SIMD operations.

2.  **GPU Acceleration (CuPy)** -- offload GSVD and μ-search to GPU hardware for parallel execution.

3.  **Parallel Computing with Dask** -- distribute large-scale computations efficiently across multiple CPU cores or nodes.

## Implemented Strategies

We used the following optimizationg strategies.

- CPU Vectorization using numpy
- Utilizing GPU rescoursec with cupy
- Parallel Computing with Dask

### CPU Vectorization

> TODO SHANE

### GPU with cupy

> TODO GIDEON

A second major optimization effort focused on accelerating the DEMREG solver
using GPU computing. The initial approach was straightforward: we ported the
vectorized CPU implementation to run on the GPU by replacing NumPy with CuPy.
In particular, the GSVD computation and the discrepancy principle μ-grid search
were executed on the GPU without changing the underlying algorithm.

This first attempt successfully offloaded the heavy linear algebra operations
to the GPU, but performance gains were disappointing. The main reason was
architectural: the solver still processed each pixel sequentially and performed
one GSVD per pixel. Even though each SVD was faster on the GPU, the high number
of small kernel launches, memory transfers, and repeated allocations led to
significant overhead. In practice, the GPU implementation was slower than
the optimized CPU vectorized version for typical image sizes.

To address this, we explored a second strategy: re-designing the algorithm to
better match the GPU’s strengths. Instead of calling GSVD individually for
every pixel, the idea was to implement a fully vectorized GPU kernel that
processes batches of pixels simultaneously. In theory, this approach would have
significantly reduced kernel launch overhead and improved parallel occupancy.
However, we were unable to get this version to work within the project
timeframe, so it remained a concept rather than a working implementation.

We also experimented with an alternative solver based on a simplified L²
discrepancy formulation (`demmap_pos_gpu_l2`). This method avoids the GSVD per
pixel and instead computes the regularization parameter and solution using
purely vectorized linear algebra across all pixels. The result is an algorithm
that runs orders of magnitude faster on the GPU because it performs fewer small
decompositions and benefits from massive parallelism. However, this comes at a
cost: the simplified formulation produces slightly less accurate DEM
reconstructions and higher χ² residuals compared to the GSVD-based reference
implementation.

In summary:

- ✅ Approach 1 – Direct CuPy Port: Minimal code changes and mathematically
  identical to the CPU version. It offloads the GSVD and μ-search to the GPU, but
  still processes each pixel individually, leading to significant overhead and in
  many cases slower performance than the CPU vectorized version.

- ⚠️ Approach 2 – Batched GPU Solver (Concept Only): A redesigned solver
  intended to process many pixels in parallel in a fully vectorized GPU kernel.
  This approach would have drastically reduced per-pixel overhead and improved
  parallel efficiency, but we were unable to get a stable implementation working
  within the project timeframe.

- ✅ Approach 3 – Simplified L² Discrepancy Solver: A new algorithm that avoids
  per-pixel GSVD entirely and solves the regularization problem using vectorized
  linear algebra. This runs orders of magnitude faster on the GPU by leveraging
  large-scale parallelism, but at the cost of slightly lower accuracy and higher
  χ² residuals.

> 💡 **Key Insight:** This trade-off highlights a key lesson in GPU
> acceleration: simply offloading existing CPU code rarely achieves good
> performance. To fully leverage GPU capabilities, algorithmic redesign is
> often required.

Both the simple and L² solution can be found in the `./pyhon/gpu` directory.

### Parallel Computing with Dask

> TODO STEFAN

## Benchmark Design

> TODO SHANE

## Results & Comparisons

> TODO GIDEON

## Reflections

> TODO ALL
