# Structural Simulation Of Printed Parts
Modeling for FEA of 3D printed parts (FFF) with polymer materials.

![image](https://github.com/user-attachments/assets/94bc0af2-2ae0-4a5f-97ce-af8327fe36fb)


## 1. Overview

This document describes a meshing method that builds a finite element mesh directly from slicer-style data (per-layer toolpaths) rather than from the ideal CAD surface. [web:15]  
The goal is to capture walls, infill, and geometric changes across layers, producing a mesh that reflects how the part is actually printed. [web:15]

---

## 2. Data Inputs and Definitions

### 2.1 Inputs

- 3D part model (e.g., STL, CAD mesh), used only as source geometry for slicing. [web:15]  
- Slicer output per layer, or an equivalent internal representation, containing:  
 - Layer height and Z position.  
 - Exterior–interior boundary (outer contour).  
 - Infill–wall boundary (separating perimeter walls from infill). [web:15]

### 2.2 Key Regions

For each layer: [web:15]

- **Exterior region**: outside the part; no material.  
- **Wall region**: between the exterior–interior boundary and the infill–wall boundary; usually printed as solid perimeters.  
- **Infill region**: interior to the infill–wall boundary; printed with a lattice / pattern.

---

## 3. Critical-Layer Detection

The patents introduce **critical layers** to avoid building a full complex mesh at every layer. [web:15]  
Critical layers are layers where the outer shape changes significantly in Z.

### 3.1 Layer Comparison

For layer \(L_i\): [web:15]

1. Select a **second layer** \(L_j\) at least a minimum number of layers above \(L_i\).  
2. Select a **third layer** \(L_k\) adjacent to and above \(L_j\).  
3. Compute a **perimeter dimension** for these layers (e.g., area, max width, bounding-box diagonal).  
4. If the difference in this dimension between these layers exceeds a threshold, the layers are marked **critical**. [web:15]

### 3.2 Result

- A subset of layers is tagged as critical: \( \{L_{c_1}, L_{c_2}, ...\} \). [web:15]  
- These will be used to define cross-sections for the voxel mesh; non-critical layers are interpolated between them. [web:15]

---

## 4. 2D Reference Grid on Critical Layers

On each critical layer, the method overlays a **common 2D reference grid** in the XY plane. [web:15]

### 4.1 Grid Definition

- Choose a uniform cell size \(\Delta x, \Delta y\).  
- Define a grid covering the part’s XY bounding box. [web:15]

Each grid cell \(C_{m,n}\) is then classified by its position relative to the slice boundaries: [web:15]

- If center of \(C_{m,n}\) lies outside the exterior–interior boundary → **Exterior**.  
- If inside the exterior–interior boundary but outside the infill–wall boundary → **Wall**.  
- If inside the infill–wall boundary → **Infill**.

### 4.2 Output of This Step

For every critical layer \(L_{c_i}\): [web:15]

- A 2D raster map:  
 - `cell_type[m,n] ∈ {Exterior, Wall, Infill}`.

---

## 5. Interior Voxel Mesh Construction

The aligned 2D grids across critical layers form the basis of an **interior voxel mesh**. [web:15]

### 5.1 Voxel Columns

For each grid index \((m,n)\): [web:15]

1. Look at its classification on each critical layer.  
2. For segments along Z where the cell belongs to the part (Wall or Infill), create voxel **columns**:  
  - Base at the Z of one critical layer.  
  - Top at the Z of the next critical layer (or actual layer height if you refine). [web:15]

Each voxel \(V_{m,n,k}\) represents a small prismatic volume of the part, with attributes:

- Position (bounding box in X, Y, Z).  
- Region type (Wall or Infill). [web:15]

### 5.2 Regular Core

This voxel mesh is regular in X and Y, and layered in Z, providing: [web:15]

- Simple connectivity.  
- Easy mapping from print regions (wall/infill) to material properties.

---

## 6. Augmented Mesh Generation

The interior voxel mesh is then **augmented** to better approximate geometric details and stress-critical features. [web:15]

### 6.1 Boundary Augmentation

Near exterior–interior boundaries and other geometrically complex regions: [web:15]

- Add finer voxels or “cut” voxels to better follow curved boundaries.  
- Modify voxel shapes to more closely match the external contour.  
- Optionally refine locally where large gradient in geometry or expected stress occurs.

### 6.2 Augmented Mesh

The result is an **augmented voxel mesh** which: [web:15]

- Retains a regular core interior.  
- Has a more detailed representation of outer surfaces and critical internal features.  

---

## 7. Protomesh Construction

From the augmented voxel mesh, a **protomesh** is built as an intermediate finite-element-friendly structure. [web:15]

### 7.1 Element Creation

- Convert each voxel into one or more finite elements (e.g., 8-node hexahedra). [web:15]  
- Create nodes at voxel corners and faces, ensuring consistent node sharing between adjacent voxels.  

### 7.2 Tagging and Grouping

- Group elements into sets according to their **region**:  
 - Wall elements.  
 - Infill elements.  
 - Potentially others (supports, special internal structures). [web:15]  
- Maintain mapping from elements to their originating slice regions and layers.

The protomesh is thus a finite element mesh that still closely follows the layered voxel structure.

---

## 8. Final Mesh via Extrusion

The **final mesh** is obtained by “extruding” or interpolating the protomesh across non-critical layers and refining surfaces. [web:15]

### 8.1 Z-Interpolation

Between two critical layers \(L_{c_i}\) and \(L_{c_{i+1}}\): [web:15]

- Interpolate geometry along Z to represent intermediate layers.  
- Adjust element heights or create additional element layers if finer resolution is needed.  

### 8.2 Surface Conformance

- Optionally project boundary nodes onto the exact exterior surface defined by the exterior–interior profile, improving shape fidelity. [web:15]  
- The final mesh now:  
 - Represents the full 3D printed object including internal structure.  
 - Distinguishes wall vs infill regions for different material models. [web:15]

---

## 9. Algorithm Implementation Plan (High-Level)

### 9.1 Technology Choices

- **Languages**: Python, C++, or C#.  
- **Libraries** (example stack):  
 - Geometry & rasterization: CGAL / Shapely / Clipper.  
 - Meshing: Gmsh API / custom voxel mesher.  
 - Data: NumPy for grids and voxel arrays.

### 9.2 Step-by-Step Algorithm

1. **Slice the CAD model**  
  - Input: STL/CAD model.  
  - Use an existing slicer (e.g., CuraEngine, PrusaSlicer, or custom) to generate per-layer contours.  
  - Extract, for each layer: exterior contour(s) and infill–wall boundary.  

2. **Compute layer metrics and detect critical layers**  
  - For each layer:  
    - Compute perimeter metrics (e.g., outer contour area, bounding box, maximum span).  
  - For layer indices \(i\):  
    - Choose \(j, k\) as in the patent, compare metrics, and mark layers where changes exceed a threshold as critical. [web:15]  

3. **Build common 2D grid for each critical layer**  
  - Compute XY bounding box of all contours.  
  - Define grid resolution \(\Delta x, \Delta y\).  
  - For each critical layer:  
    - For each grid cell center:  
      - Test point-in-polygon against exterior contour.  
      - If inside, test against infill–wall boundary to separate Wall vs Infill.  
    - Store a 2D array `cell_type[m,n]`. [web:15]  

4. **Generate interior voxel mesh**  
  - For each grid index \((m,n)\):  
    - For critical layers in ascending Z:  
      - Where `cell_type[m,n]` ∈ {Wall, Infill}, create voxel columns between successive critical layer Z positions.  
  - Store voxel attributes: region type, indices, physical bounds. [web:15]  

5. **Augment voxels near boundaries**  
  - Identify voxels whose centers are close to the exterior boundary.  
  - Split these voxels or adjust their shapes to better match the contour (e.g., cut with polygonal boundary).  
  - Optionally refine voxel resolution locally based on geometric curvature or expected load paths. [web:15]  

6. **Convert augmented voxels to protomesh elements**  
  - Enumerate nodes: assign coordinates to voxel corners, merging duplicates.  
  - Create hexahedral elements per voxel with node connectivity.  
  - Tag elements with region type (Wall/Infill). [web:15]  

7. **Extrude / interpolate to final mesh**  
  - Between critical layers, subdivide in Z if element aspect ratios are too large.  
  - Optionally project boundary nodes to smoother, CAD-derived surfaces to reduce stair-stepping.  
  - Output mesh in an FEA-readable format (e.g., Abaqus .inp, Ansys .cdb, or a generic .msh). [web:15]  

8. **Assign material properties and run FEA**  
  - Assign different material models to element sets:  
    - Wall elements: near-solid properties.  
    - Infill elements: homogenized properties derived from infill pattern tests.  
  - Apply loads and boundary conditions, solve, and post-process.  

---

## 10. Implementation Notes and Extensions

- **Resolution control**:  
 - Choose grid spacing and critical-layer distance based on target accuracy vs. computational cost.  
- **Infill modeling**:  
 - Either treat infill as a homogenized continuum with effective stiffness or explicitly mesh infill trusses in selected regions.  
- **Bidirectional coupling to slicer**:  
 - Use simulation results to iteratively adjust wall thickness, infill density, or fiber placement, then regenerate the mesh for a new analysis loop.




