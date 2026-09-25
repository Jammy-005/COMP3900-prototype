# Input Formats

> **Status:** research notes from the team doc, tidied. The file-format facts check
> out. The conclusion about normals needs care (see the review note under STL).

Covers R1 in [brief.md](../brief.md).

## Mesh vs parametric surfaces

- **Mesh:** the surface is broken into flat triangles (or polygons). Continuous
  surface normals and curvature are only approximated.
- **Parametric:** the surface is defined by mathematical curves/patches.

## STL (stereolithography)

**Type:** mesh.
**What it is:** a flat list of triangles, each with 3 vertices and a normal vector.
It has no colour, units, metadata, or topology: nothing says which triangles are
connected.
**Parsing:** `numpy-stl`, `trimesh`, `assimp` (Python/C++).

### Binary layout

```
[80 bytes]   header (arbitrary, usually ignored)
[4 bytes]    triangle count N (uint32)
[50 bytes] × N triangles:
               face normal (ni, nj, nk)   3 × float32
               vertex 1, 2, 3 (x, y, z)   9 × float32
               attribute byte count       uint16
```

**Vertex ordering (right-hand rule):** each triangle's vertices are listed
**counter-clockwise when viewed from outside the model**, i.e. from the direction
the normal points.

> [!NOTE]
> **Review:** STL also has an ASCII variant (`solid … facet normal … endsolid`).
> Some binary files also begin with the word `solid` in their header, so don't
> detect the format from header text. Check whether the file size equals
> `84 + 50 × N`. `trimesh` handles both variants.

### Challenges for this project

- **No connectivity between triangles.** Adjacency (which triangles share edges)
  has to be reconstructed before segmentation can follow the surface continuously.
- **Faceted normals make orientation discontinuous.** Every point on a triangle
  (centre, edges, corners) has exactly the same normal, because a triangle is flat.
  The real surface may be curving smoothly across that region. This matters in three
  ways:
  - *Discontinuous orientation commands:* when a toolpath crosses from one triangle
    to the next, the ideal nozzle angle can jump instantly, even over a tiny
    distance. The controller then has to slow sharply at every boundary (lower print
    speed), or it moves jerkily (worse print quality, bad for the hardware).
  - *Triangulation-dependent artefacts:* the same curved surface can be meshed many
    valid ways, depending on CAD export settings. So the normal at a point depends on
    how the STL was triangulated, not on the true shape. Different export
    resolutions give inconsistent orientations.
  - *Coarse tessellation makes it worse:* low-poly exports cause bigger jumps.
    High-resolution exports approximate the smooth normal field better, but mean
    huge files and more triangles to process in segmentation and collision checking.
- **No guarantee of watertightness.** Real STLs often have gaps, non-manifold edges
  and duplicate vertices, so run a repair/validation pass before segmentation.

**Proposed solution: vertex normal interpolation.** At each vertex, average the face
normals of the triangles sharing it (weighted by triangle area or by the angle at
that vertex). Then interpolate across each triangle (Phong-style) to get a smoothly
varying normal at any point.

> [!NOTE]
> **Review:** this is a useful *geometry* utility, with two cautions:
>
> 1. **Sharp edges.** Averaging across a deliberate crease (a box edge, say) smears
>    it. Only average across edges whose dihedral angle is below a threshold (a
>    "crease angle" parameter).
> 2. **The part's surface normal usually isn't the nozzle direction.** In
>    curved-layer slicing the nozzle follows the normal of the *layer* surface being
>    printed. That comes from the segmentation strategy (for a scalar field, its
>    gradient), not from the part mesh. The part normal matters mainly for the top
>    layers of conformal prints. On a side wall it points horizontally, so using it
>    as the nozzle direction would point the nozzle sideways. Keeping orientation
>    smooth belongs in the orientation stage: a tilt limit, smoothing along the path,
>    and a reorientation-speed cap. See [design.md](../design.md#review) items 4–5.

## OBJ (Wavefront)

**Type:** mesh.
**What it is:** indexed geometry. Vertex positions, texture coordinates and normals
are declared once in shared lists, and faces reference them by index. Shared edges
point at the *same* vertex entry (rather than repeating coordinates as STL does), so
connectivity is explicit.
**Parsing:** the same libraries as STL.
**Advantage over STL:** it can carry proper vertex normals and grouped sub-objects,
so less reconstruction is needed.

### Format

Each line starts with a tag saying what kind of data it holds:

```
# comment line
mtllib materials.mtl    # companion material file
o ObjectName            # named object/group
v 0.0 0.0 0.0           # vertex position
v 1.0 0.0 0.0
v 0.0 1.0 0.0
vt 0.0 0.0              # texture coordinate
vn 0.0 0.0 1.0          # vertex normal
usemtl MaterialName     # material for subsequent faces
f 1//1 2//1 3//1        # face: v/vt/vn indices (vt omitted here)
```

- `v x y z [w]`: vertex position. `w` is an optional homogeneous coordinate (default
  1.0, rarely used).
- `vt u v [w]`: texture coordinate (how a 2D image wraps onto the surface).
  Irrelevant for printing, but the parser must accept it.
- `vn nx ny nz`: a normal, declared separately from positions. This is the main
  practical difference from STL. Normals attach to *vertices*, not faces, so a
  well-formed OBJ can encode a smooth normal field and avoid the faceted-normal
  problem.
- `f …`: a face definition.

> [!NOTE]
> **Review:** parser details to handle:
>
> - Indices are **1-based**, and can be **negative** (counting back from the end of
>   the list so far).
> - Faces can have more than 3 vertices, and need triangulating.
> - Like STL, OBJ has **no units**. Take units as an explicit input parameter for
>   every mesh format.
> - Don't trust `vn` blindly: many exporters write faceted normals anyway.

## NURBS via CAD interchange

**Type:** parametric surface.
**What it is:** true parametric surfaces defined by *control points, knot vectors,
and degree*. They describe smooth curved patches exactly, with no approximation.
CAD interchange formats such as STEP (`.stp`) and IGES (`.igs`) are how NURBS models
are exported from tools like Rhino.
**Parsing:** OpenCASCADE (OCCT), the standard open-source CAD geometry kernel, with
Python bindings through `pythonocc-core`.

> [!NOTE]
> **Review:**
>
> - `pythonocc-core` is distributed mainly through conda, which is awkward for a
>   pip-installable library. Keep OCCT an **optional** dependency behind the NURBS
>   loader. `cadquery-ocp` is a pip-installable binding worth checking.
> - The client works in Rhino. The `rhino3dm` package reads `.3dm` files without
>   Rhino installed, and may be a more convenient route than STEP/IGES. Check what
>   geometry it can evaluate.
> - Segmentation and collision will almost certainly run on a tessellated mesh
>   anyway. NURBS helps only where exact normals/curvature are evaluated.
>   Recommendation: build the MVP on meshes (STL/OBJ) and add NURBS as a later
>   loader.
