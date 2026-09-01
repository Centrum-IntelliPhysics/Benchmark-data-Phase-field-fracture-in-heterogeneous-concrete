# Benchmark dataset: Phase-field fracture in heterogeneous concrete with varying notch positions and aggregate distributions

Quasi-static mode-I fracture of a three-dimensional notched concrete specimen, simulated with FEniCS using a second-order (AT2) phase-field formulation.

This dataset contains **1100 finite-element simulations** of a 3D single-edge-notched plate containing four stiff, tough aggregate inclusions embedded in a softer cementitious matrix. Each simulation is fully described by five scalars — the notch height and the four aggregate positions — while the full displacement and damage histories are stored at 401 load increments on the original finite element mesh.

---

## Full Dataset

The complete collection comprises **1100** distinct mesostructure realizations. The full dataset is available in the JHU archive at:

<!-- TODO: paste the JHU Research Data Repository link / DOI here once minted -->
**[link to be added]**

---

## Problem setup

A plate of dimensions $1.0 \times 1.0 \times 0.1$ carries a through-thickness notch of length $a = 0.1$ at height $h$ on the left face. Four aggregates of radius $r_\mathrm{agg} = 0.02$ sit directly on the ligament ahead of the notch tip, so the propagating crack is forced to interact with them.

<p align="center">
  <img src="figures/geometry.pdf" alt="Geometry, modulus field, and sampled inputs" width="900"/>
</p>

**Boundary conditions.** The bottom face is a roller ($u_y = 0$, free to contract laterally); the single edge $x = y = 0$ is fully fixed to remove the remaining rigid-body translations; the top face carries a monotonically increasing displacement $u_y = u_\mathrm{max}\,t$ with $u_\mathrm{max} = 0.02$; the two faces normal to $z$ are traction-free.

**Material.** The aggregates are twice as stiff and 50% tougher than the matrix:

| Phase | $E$ [MPa] | $\nu$ [-] | $G_c$ [N/mm] | $\ell$ [mm] | $k$ [-] |
|---|---|---|---|---|---|
| Matrix | $210 \times 10^3$ | 0.30 | 2.7 | 0.02 | $10^{-8}$ |
| Aggregate | $420 \times 10^3$ | 0.27 | 4.05 | 0.02 | $10^{-8}$ |

**What varies.** Only the notch height $h \in [0.2, 0.8]$ and the four aggregate abscissae $x_i \sim \mathcal{U}(0.1, 1)$. Every material and loading parameter is held fixed, so the dataset isolates the effect of the mesostructure.

---

## Generation pipeline

<p align="center">
  <img src="figures/framework.pdf" alt="Dataset generation framework" width="900"/>
</p>

Meshes are generated with **Gmsh** (the surfaces above and below the notch are meshed separately and extruded, so the notch is a genuine geometric discontinuity rather than a pre-damaged band) and the coupled displacement/phase-field problem is solved in **FEniCS** with a staggered scheme, one pass per load increment. Because the mesh refinement box follows the sampled notch height, **the mesh is regenerated for every realization**: node counts range from 42,406 to 44,690 and element counts from 244,218 to 247,874.

---

## Repository / archive contents

```
processed_xyz_disp_joint1100_standalone.h5     269.5 GB   main data file
branch5_inputs_joint1100.npz                   106 KB     five branch scalars + trunk time
split880_220_seed42.npz                        118 KB     train / test partition
true_endpointclean_centerline_joint1100.npz    947 KB     extracted crack centerlines
```

<p align="center">
  <img src="figures/fs.pdf" alt="Structure of the released dataset" width="850"/>
</p>

The main archive holds 1100 groups, `job_0000` … `job_1099`, one per simulation:

| Dataset | Shape and type | Description |
|---|---|---|
| `coords` | $(N, 3)$, `float32` | reference (undeformed) nodal coordinates |
| `data` | $(N, 401, 6)$, `float32` | channels $[x, y, z, u_x, u_y, u_z]$ |
| `phi` | $(N, 401)$, `float32` | damage variable $d \in [0, 1]$ |
| `topology` | $(M, 4)$, `int32` | tetrahedral connectivity |
| `time_keys` | $(401,)$, `int32` | solver frame index of each load step |

`branch5_inputs_joint1100.npz` stores `branch_inputs` of shape $(1100, 5)$ in the order
`[aggregate_x_0, aggregate_x_1, aggregate_x_2, aggregate_x_3, initial_height]`,
together with `trunk_t` of shape $(401, 1)$ — the normalized pseudo-time $t \in [0, 1]$.

> **Two conventions worth knowing before you load anything.**
> 1. The first three channels of `data` are the **reference** coordinates and are constant in time; they are replicated along the time axis for convenience and are identical to `coords`. The deformed configuration is `coords + u`.
> 2. **Node counts differ between simulations** and node ordering carries no cross-simulation meaning, because the mesh is regenerated for every realization. Batching several simulations together requires padding and masking, or an architecture invariant to the number and ordering of points. The temporal discretization, by contrast, *is* shared: every simulation has the full sequence of 401 frames under an identical loading protocol.

### Quick start

```python
import h5py
import numpy as np

BASE = "."  # directory holding the four released files

# five-scalar input description and the shared trunk time
meta = np.load(f"{BASE}/branch5_inputs_joint1100.npz", allow_pickle=True)
job_names   = meta["job_names"].astype(str)        # (1100,)
branch      = meta["branch_inputs"]                # (1100, 5)
trunk_t     = meta["trunk_t"]                      # (401, 1), t in [0, 1]
print(meta["branch_order"])                        # channel order of `branch`

# train / test partition (by simulation, never by time frame)
split = np.load(f"{BASE}/split880_220_seed42.npz", allow_pickle=True)
train, test = split["train_job_names"].astype(str), split["test_job_names"].astype(str)

# one simulation
with h5py.File(f"{BASE}/processed_xyz_disp_joint1100_standalone.h5", "r") as f:
    g = f["job_0000"]
    coords = g["coords"][:]          # (N, 3)
    disp   = g["data"][:, :, 3:6]    # (N, 401, 3) -> ux, uy, uz
    damage = g["phi"][:]             # (N, 401)
    tets   = g["topology"][:]        # (M, 4)

deformed_final = coords + disp[:, -1, :]
```

---

## Visualizations

**Crack evolution for contrasting mesostructures.** Four realizations (rows) at load steps chosen so that the crack tip advances by roughly equal increments (columns). Aggregate outlines are overlaid in white. Depending on whether the aggregates sit on or off the crack path, the crack runs straight through, deflects around an inclusion, or is temporarily arrested before resuming.

<p align="center">
  <img src="figures/different_cases.pdf" alt="Damage evolution for four mesostructures" width="950"/>
</p>

**Displacement history.** The three components for one realization on the mid-thickness plane, at the same load steps. The dominant response is the mode-I opening in $u_y$; $u_x$ concentrates around the advancing crack tip and $u_z$ reflects the through-thickness Poisson contraction. The thin white band marks fully damaged elements, which are removed rather than interpolated through because the displacement is discontinuous across an open crack.

<p align="center">
  <img src="figures/displacement_components.pdf" alt="Displacement components over the loading history" width="950"/>
</p>

---

## Applications of this dataset

### 1. Operator learning on unstructured, simulation-specific meshes

The natural task is to map the compact mesostructure descriptor and the load parameter to the full space–time response:

$$(x_1, x_2, x_3, x_4,\, h) \times t \;\longmapsto\; (u_x, u_y, u_z,\, d)(\mathbf{x}, t).$$

- **DeepONet / neural operators** — the five branch scalars and the shared 401-point trunk map directly onto a branch/trunk architecture.
- **Point-cloud and graph architectures** — because the mesh is regenerated per realization, this dataset is a realistic test of architectures that must handle variable node counts, rather than the fixed uniform grids most fracture benchmarks provide.
- **Latent-space surrogates** — compressing each field snapshot with an autoencoder and learning the low-dimensional dynamics is a natural two-stage baseline.

### 2. Crack-path prediction and mesostructure–response coupling

- Predict the final crack centerline (provided in `true_endpointclean_centerline_joint1100.npz`, resampled onto a fixed grid of 201 abscissae) directly from the aggregate configuration.
- Study when a crack deflects around versus penetrates a tough inclusion, and how far a temporarily arrested crack is delayed — behaviors that are all present in the dataset and controlled by only five scalars.
- Classify or regress the onset of propagation: initiation load varies noticeably between realizations even under an identical loading protocol.

### 3. Inverse problems

- **Mesostructure identification** — recover the aggregate positions from an observed crack path or from a partially observed displacement field, an idealized version of inferring internal heterogeneity from surface measurements.
- **Sparse-observation reconstruction** — reconstruct the full 3D field history from a handful of virtual sensors, using the complete ground truth for evaluation.

### 4. Uncertainty quantification and generalization

The five-dimensional input space is sampled densely enough (1100 realizations) to study how surrogate accuracy degrades away from the training distribution — for example, holding out aggregate configurations that cluster near the notch tip and testing on well-separated ones.

---

## Cite our dataset

If you use this dataset in your work, please cite it as follows:

```
@data{CONCRETE_PF_2026,
  author    = {Liu, Zhan and Goswami, Somdatta},
  publisher = {Johns Hopkins Research Data Repository},
  title     = {{Data and code associated with: A phase-field dataset for fracture in heterogeneous concrete under varying notch positions and aggregate distributions}},
  year      = {2026},
  version   = {V1},
  doi       = {},
  url       = {}
}
```

<!-- TODO: fill in doi and url once the JHU archive deposit is published -->

---

## Contact

In case you need more information, please feel free to contact Zhan Liu (zliu274@jhu.edu) or Prof. Somdatta Goswami (somdatta@jhu.edu).
