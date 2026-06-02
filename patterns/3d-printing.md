# 3D Printing — Diagnostics & Repair Patterns

Practical knowledge for printing, repairing existing parts with prints, and
diagnosing models that slicers or repair tools choke on.

## Table of contents

- [STL diagnostic decision tree](#stl-diagnostic-decision-tree)
- [Dual-silhouette / shadow-art models: don't "repair"](#dual-silhouette--shadow-art-models-dont-repair)
- [trimesh diagnostic snippet](#trimesh-diagnostic-snippet)
- [How STL repair tools actually work](#how-stl-repair-tools-actually-work)
- [Repairing broken parts: print a splint, don't reprint the whole thing](#repairing-broken-parts-print-a-splint-dont-reprint-the-whole-thing)
- [Captive-grommet sandwich pattern](#captive-grommet-sandwich-pattern)
- [Slicer settings for structural PETG](#slicer-settings-for-structural-petg)
- [Reusable OpenSCAD modules](#reusable-openscad-modules)

## STL diagnostic decision tree

When a slicer warns "model needs repair" or an online repair tool hangs, run
[the trimesh diagnostic](#trimesh-diagnostic-snippet) first. The counts tell
you what's actually wrong and which tool to reach for:

| Pattern | Likely cause | Right fix |
|---|---|---|
| Many **edges with 1 triangle** (open holes) | Truly missing surface | Hole-filling repair (3D Builder, Meshmixer auto-repair, ADMesh) |
| Many **edges with 3+ triangles** (non-manifold), 0 holes, 2+ bodies | Boolean union never performed, OR intentional intersecting shells | **First try slicing as-is** (see next two sections). If slicer fails, voxel remesh — but only if the geometry isn't intentional |
| `is_winding_consistent = False` | Some triangles flipped | Auto-orient normals (most tools) |
| Many duplicate / degenerate faces | Sloppy export | Vertex weld + degenerate cleanup |
| `body_count > 1` plus interpenetration | Multi-part assembly OR dual-silhouette art | Investigate before "fixing" |

**Slicers handle most non-manifold meshes fine** via even-odd ray casting per
layer — every point inside an odd number of shells gets filled. Many "needs
repair" warnings are false positives. Always try slicing as-is before
running a repair tool, especially for art / decorative pieces.

## Dual-silhouette / shadow-art models: don't "repair"

These models work by having two (or more) extrusions that interpenetrate at
right angles; the intersection casts a different correct silhouette from
each viewing axis. The interpenetration is **the entire design** —
removing it destroys the effect.

**Fingerprint of a dual-silhouette model:**
- 2+ separate bodies that overlap
- Many non-manifold edges (3+ triangles), zero holes
- Often comes from text + figure pairings (e.g. "Lord of the Rings" text + Fellowship silhouette)

**What kills these:**
- Voxel remesh → fuses everything into a single blob, both silhouettes ruined
- Online "repair" tools that try to resolve all intersections → either hang
  (O(n²) without acceleration) or fuse the shells
- 3D Builder's auto-repair → same

**What works:**
- Slice as-is in any modern slicer. Dismiss the repair warning. Check the
  layer preview — if walls and infill look right, you're done.
- If the slicer truly can't handle it, mark the bodies as "Union" or merge
  them as a *modifier* (not a remesh) so the slicer treats them as one
  filled volume.

## trimesh diagnostic snippet

```python
import trimesh, numpy as np

m = trimesh.load("model.stl")
if isinstance(m, trimesh.Scene):
    m = trimesh.util.concatenate(list(m.geometry.values()))

print(f"Vertices:           {len(m.vertices):,}")
print(f"Faces:              {len(m.faces):,}")
print(f"Bounds (mm):        {m.bounds[1] - m.bounds[0]}")
print(f"Watertight:         {m.is_watertight}")
print(f"Winding consistent: {m.is_winding_consistent}")
print(f"Body count:         {m.body_count}")
print(f"Degenerate faces:   {(~m.nondegenerate_faces()).sum()}")

edges_sorted = np.sort(m.edges, axis=1)
unique, counts = np.unique(edges_sorted, axis=0, return_counts=True)
print(f"Edges shared by 1 triangle (holes):           {(counts == 1).sum():,}")
print(f"Edges shared by 2 triangles (manifold/good):  {(counts == 2).sum():,}")
print(f"Edges shared by 3+ triangles (non-manifold):  {(counts >= 3).sum():,}")
```

Install: `pip install trimesh numpy`. Loads .stl, .3mf, .obj, .ply.

## How STL repair tools actually work

Standard operations, usually combined:

1. **Vertex welding** — merge vertices within ε of each other. Closes
   hairline cracks from per-triangle vertex storage.
2. **Hole filling** — after welding, edges with only one triangle form
   boundary loops; triangulate them closed.
3. **Normal/winding correction** — pick a known-outside face, propagate
   consistent winding to neighbors.
4. **Degenerate cleanup** — drop zero-area triangles and duplicates.
5. **Non-manifold cleanup** — split or merge edges shared by >2 triangles.
6. **Self-intersection resolution** — spatial-hash + triangle-triangle
   tests, then local remesh. **O(n²) worst case** — the reason online
   repairers hang on dual-silhouette models.
7. **Voxel remesh (nuclear option)** — rasterize to a voxel grid, marching
   cubes back to a surface. Always produces a watertight manifold, but
   loses sharp edges and fine detail, and **fuses intentionally separate
   shells**.

Most online repairers wrap open libraries: ADMesh, MeshLab filters, libigl,
PyMesh, OpenVDB (voxel path). Some use legacy NetFabb (now Autodesk),
which was what Windows 3D Builder's repair button shipped with.

## Repairing broken parts: print a splint, don't reprint the whole thing

For broken pressed-wood / MDF / hardboard parts (panels, brackets,
spacers), reprinting in PLA or PETG often gives **wrong stiffness** —
plastic flexes and creeps where MDF is rigid. A printed reinforcement
that holds the original pieces together is usually a better repair:

- **Original material stays where it belongs** — same stiffness, same
  thickness, same fit
- **3D print only solves what plastic is good at** — clamping the joint,
  providing wear surfaces for moving parts
- **Faster to print** — a 150mm splint vs a 440mm replacement
- **Easier to iterate** — fit-check prints are quick

Glue the broken pieces with wood glue (Titebond III) or epoxy first, then
bond a printed reinforcement. Superglue (cyanoacrylate) works on the
plastic-to-MDF bond if you want fast cure.

## Captive-grommet sandwich pattern

For any panel where a **strap or cable passes through a cutout** and tears
the substrate at the cutout edges (classic stress-concentration failure
mode for MDF/pressed wood):

```
        ┌──────────────┐  ← top plate (capture ring)
        │   ╔════╗     │     pill hole = strap opening
        │   ║    ║     │     (smaller than grommet OD)
   ═════╪═══╝    ╚═════╪═══  ← MDF panel (the original wood)
        │   ║    ║     │     grommet OD fills cutout
        │   ║    ║     │
        └───╨────╨─────┘  ← bottom plate (load-bearing)
            │    │           grommet sits on bottom plate,
            │    │           tube fills the MDF cutout,
            └────┘           strap rides on plastic not MDF
```

**Mechanical principle:**
- Strap loads → grommet wall (plastic, not MDF) → bottom plate → glue
- Top plate's hole is smaller than the grommet OD → grommet is
  *mechanically captured* and can't extract even if glue fails
- The two plates also sandwich the crack on both faces

**When to use over single-plate-with-grommet:**
- Long-life parts (kids' products, outdoor gear) where glue degrades
- Anywhere mechanical capture beats glue-only insurance
- Only viable if there's clearance to add ~1.5mm on both faces

**When single-plate is fine:**
- Tight fit on one face (no top clearance)
- Low-cycle parts where glue lifetime isn't a concern

Worked example with parameterized OpenSCAD at `~/Projects/wagon-splint/`.

## Slicer settings for structural PETG

Defaults for a structural part (load-bearing, glued, lives outdoors or in
a car):

| Setting | Value | Why |
|---|---|---|
| Layer height | 0.2mm | Standard; 0.16mm only for fine detail |
| Walls / perimeters | **4** | Doubles strength vs default 2. For thin features (1.5mm walls), 4 perimeters × ~0.42mm line width fills solid. |
| Top shells | 4–5 layers | Thin top plates become effectively solid |
| Bottom shells | 3 layers | First-layer adhesion + buffer |
| Infill pattern | **Gyroid** | Isotropic — strong in any direction. Loads from arbitrary angles handled equally. |
| Infill % | 40% | Plenty for shelled structural parts |
| Outer wall speed | **~30 mm/s** | PETG layer adhesion improves dramatically. Outer walls are usually the load-bearing perimeter. |
| Supports | None if possible | Reorient instead — supports leave weak interfaces |
| Brim | 3–5mm if small footprint | PETG corner-lift insurance |

**Filament prep:** PETG absorbs moisture fast. Wet filament = stringing
+ reduced layer adhesion. Dry at 65°C for 4 hours (oven) or overnight
(food dehydrator) before structural prints. Fit-check / draft prints in
PLA tolerate damp filament fine.

## Reusable OpenSCAD modules

Small enough to copy into any project:

```openscad
// Rounded rectangle, corner at origin
module rounded_rect(l, w, r) {
    hull() for (x = [r, l-r], y = [r, w-r])
        translate([x, y]) circle(r);
}

// Pill / stadium shape, centered at origin, long axis along Y
module pill_centered(long_axis, short_axis) {
    hull() for (y = [-(long_axis - short_axis)/2,
                      (long_axis - short_axis)/2])
        translate([0, y]) circle(short_axis/2);
}
```

Don't forget `$fn = 96;` (or higher) at the top for smooth curves on
exported STL.
