# Technical Architecture & Pipeline Notes

This document details the mathematical algorithms and graphics pipelines powering **Third-Dimension (3D Logo Material Studio)**.

---

## 1. Sub-Pixel Iso-Contour Tracing Pipeline

Raster-to-mesh converters traditionally construct voxels or pixel-stepped heightfields, causing visible "staircase" artifacts along diagonal and curved edges. Third-Dimension circumvents this by evaluating a continuous 2D Signed Distance Field (SDF).

### Mathematical Steps:

1. **Gaussian SDF Blur**:
   A separable 5-tap normalized binomial filter kernel `[1/16, 4/16, 6/16, 4/16, 1/16]` is applied in horizontal and vertical passes across the binary mask. This transforms discrete binary boundaries `0 or 1` into a smooth scalar gradient field $f(x, y) \in [0, 1]$.

2. **Sub-Pixel Marching Squares**:
   For each cell $(x, y)$, corner field values $v_{TL}, v_{TR}, v_{BR}, v_{BL}$ are sampled against an iso-surface threshold $\tau = 0.45$. Edge intersections are computed via linear interpolation:
   $$t = \frac{\tau - v_1}{v_2 - v_1}$$
   This places vertex coordinates on continuous floating-point coordinates rather than integer grid centers.

3. **Taubin / Laplacian Zero-Shrinkage Filtering**:
   Raw polygonal rings undergo 5 passes of Taubin smoothing. Each pass computes:
   $$\vec{P}_i \leftarrow \vec{P}_i + \lambda \left(\frac{\vec{P}_{i-1} + \vec{P}_{i+1}}{2} - \vec{P}_i\right)$$
   Alternating positive $\lambda = +0.48$ with negative anti-shrinkage inflation $\mu = -0.46$ dampens high-frequency noise without diminishing the overall shape volume.

4. **Chaikin Corner-Cutting**:
   Points are iteratively subdivided into quadratic B-splines using Chaikin's corner-cutting algorithm:
   $$Q_i = \frac{3}{4}P_i + \frac{1}{4}P_{i+1}, \quad R_i = \frac{1}{4}P_i + \frac{3}{4}P_{i+1}$$

5. **Winding Order & Hole Assignment**:
   The signed Shoelace area determines outer loops (Counter-Clockwise, Area $> 0$) and inner holes (Clockwise, Area $< 0$). Holes are matched to their parent outer boundary via ray-casting point-in-polygon containment testing.

---

## 2. Procedural 24-Bit Tangent-Space Normal Map Generation

All micro-relief textures (Denim, Linen, Leather, Suede, Concrete) are synthesized algorithmically inside a 1024×1024 floating-point heightfield $H(x, y)$ using 2D Bicubic Hermite-interpolated gradient noise.

To eliminate 8-bit quantization banding, the heightfield is converted directly into a 24-bit tangent-space normal vector using a 3×3 Sobel filter:

$$\frac{\partial H}{\partial x} \approx \frac{(tr + 2mr + br) - (tl + 2ml + bl)}{8}$$
$$\frac{\partial H}{\partial y} \approx \frac{(bl + 2bc + br) - (tl + 2tc + tr)}{8}$$

The normalized 3D surface normal is:
$$\vec{N} = \text{normalize}\left(-\frac{\partial H}{\partial x} \cdot s, \; -\frac{\partial H}{\partial y} \cdot s, \; 1.0\right)$$
$$\text{Color}_{RGB} = \left(\frac{N_x + 1}{2}, \; \frac{N_y + 1}{2}, \; \frac{N_z + 1}{2}\right) \times 255$$

---

## 3. Triplanar UV Projection

Extruded meshes have vertical sidewalls and chamfered bevels where standard 2D UV mapping stretches severely. The engine re-projects vertex UVs based on face normal dominance:

$$\text{dominantAxis} = \max(|N_x|, |N_y|, |N_z|)$$

- Front / Back ($Z$-dominant): $UV = (X, Y) \times \text{scale}$
- Left / Right ($X$-dominant): $UV = (Z, Y) \times \text{scale}$
- Top / Bottom ($Y$-dominant): $UV = (X, Z) \times \text{scale}$

This delivers seamless tactile grain over all face orientations.

---

## 4. Drafts History

- `drafts/3d_logo_material_studio.v1.html`: Original single-file implementation exported from local experimentation.
- `index.html`: Polished production entrypoint with responsive layout, SVG dimension fallbacks, and multi-platform optimizations.
