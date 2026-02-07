# Unitree G1 Description (MJCF)

> [!IMPORTANT]
> Requires MuJoCo 2.3.4 or later.

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for a full history of changes.

## Overview

This package contains a simplified robot description (MJCF) of the [G1 Humanoid
Robot](https://www.unitree.com/g1/) developed by [Unitree
Robotics](https://www.unitree.com/). It is derived from the [publicly available
MJCF
description](https://github.com/unitreerobotics/unitree_ros/blob/master/robots/g1_description/g1_29dof_rev_1_0.xml).

<p float="left">
  <img src="g1.png" width="400">
  <img src="g1_with_hands.png" width="400">
</p>

## MJCF derivation steps

1. Copied the MJCF description from [g1_description](https://github.com/unitreerobotics/unitree_ros/blob/master/robots/g1_description/g1_29dof_rev_1_0.xml).
2. Manually edited the MJCF to extract common properties into the `<default>` section.
3. Added stand keyframe.
4. Added joint position actuators (needs tuning).
5. Applied similar edits to `g1_with_hands.xml`.

## MJX

A version of [g1.xml](g1.xml) for use in [MJX](https://mujoco.readthedocs.io/en/stable/mjx.html) is available in `scene_mjx.xml` with the following changes:

* Manually designed collision geoms and corresponding contact pairs (see figure below).
* Tuned solver and line search iterations.
* Used lower, more realistic PD gains.
* Added `home` and `knees_bent` keyframes.

The above model was successfully transfered to hardware in [MuJoCo Playground](https://playground.mujoco.org/).

<p float="left">
  <img src="g1_mjx_colliders.png" width="400">
</p>

## Unity3D import notes (OBJ conversion)

If parts look disconnected in Unity after converting meshes from STL to OBJ, the cause is almost always a mismatch in mesh local frames introduced during conversion. MuJoCo places mesh geoms directly using the mesh vertex coordinates. If the OBJ export changes the mesh origin, rotation, axis convention, or units, visuals no longer align with their bodies even though joints are correct.

What’s happening:
- STL meshes in the original MJCF are aligned to each link’s frame. Converting to OBJ can:
  - Change up/forward axes (e.g., Y-up vs Z-up).
  - Bake or lose object transforms.
  - Re-center pivots to bounding-box centers.
  - Switch units (mm vs m).
- Unity renders those shifted meshes, so the robot appears “exploded”. In simulation, bodies and joints still exist; only visuals are misaligned.

How to fix (Blender → OBJ recipe that preserves assembly):
1. Scene setup
   - Scene Units: Metric, Unit Scale = 1.0 (meters).
   - Import all STL files; don’t move vertices. Ensure each object’s Origin is the link frame (do not recenter to geometry).
2. Per-object cleanup
   - With the object selected: Apply Rotation & Scale only (Ctrl+A → Rotation & Scale). Do NOT Apply Location.
   - Verify transforms: rotation = (0,0,0), scale = (1,1,1). Location can be nonzero (that’s your link-frame origin).
3. Export OBJ (File → Export → Wavefront (.obj))
   - Selection Only: as needed.
   - Scale: 1.0.
   - Apply Transform: OFF (critical to avoid baking an axis conversion).
   - Forward/Up: leave defaults; irrelevant when Apply Transform is OFF.
   - Geometry: Triangulate Faces ON (or add a Triangulate modifier), Write Normals ON, Keep Vertex Order ON.
   - Include UVs/Materials if you need them, but .mtl isn’t required for MuJoCo.
   - Do not run any “recenter origin/pivot” operation before export.
4. Replace STL with OBJ in MJCF
   - Set `<compiler meshdir="assets/obj"/>` and update mesh file names to `.obj`. No geom-level pos/quat should be added; the OBJ must match STL frames.

If units were exported as mm, re-export in meters or set per-asset scale in MJCF (last resort):
- `<mesh name="..." file="...obj" scale="0.001 0.001 0.001"/>`

Unity troubleshooting checklist:
- Press Play (the MuJoCo Plugin builds physics at runtime).
- Enable drawing of frames/joints to verify bodies are connected; if visuals are offset but joints look correct, it’s an OBJ frame issue.
- If visuals are rotated 90° or mirrored, you likely enabled Apply Transform during export or changed axis conventions; re-export with the recipe above.

Notes:
- We already updated `meshdir` to `assets/obj` and switched asset files to `.obj` in `g1.xml` and `g1_mjx.xml`. No additional MJCF edits are needed if OBJ exports preserve the original STL frames.

### Blender STL import settings (impact on later OBJ export)

The STL importer options you see affect how object transforms are initialized in Blender. These transforms will change mesh local frames if you later bake them into the OBJ, which “explodes” the assembly in Unity/MuJoCo.

Defaults observed:
- Scale: 1.0
- Scene Unit: OFF
- Forward Axis: Y
- Up Axis: Z
- Facet Normals: OFF
- Validate Mesh: ON

Guidance:
- Scale: Keep 1.0 if your STL is in meters (MuJoCo expects meters). If your STL is in millimeters, import with Scale=0.001 or fix at export (preferred: re-export OBJs in meters).
- Scene Unit: Leave OFF to avoid implicit rescaling. Only enable if you know your STL unit differs and your Blender Scene Units are set correctly (Metric, Unit Scale=1.0).
- Forward/Up axes: Harmless on import. Critical only if you bake transforms on export. Our recipe keeps “Apply Transform” OFF on OBJ export, so axes won’t be baked and assembly frames are preserved.
- Facet Normals: Shading-only; no effect on assembly. Keep OFF (you’ll write normals on OBJ export).
- Validate Mesh: Keep ON; it fixes degenerate geometry without changing object origins.

Checklist after STL import:
- Do not move vertices or recenter origins/pivots.
- Apply Rotation & Scale only (Ctrl+A → Rotation & Scale). Do NOT Apply Location.
- Confirm per-object: rotation=(0,0,0), scale=(1,1,1), location can be nonzero (it encodes the link frame).

### OBJ export checklist (preserve assembly)

- File → Export → Wavefront (.obj)
  - Apply Transform: OFF (prevents axis/scale baking; preserves link-local frames).
  - Scale: 1.0.
  - Geometry: Triangulate Faces ON, Write Normals ON, Keep Vertex Order ON.
  - Do not recenter origins/pivots before export.
- In MJCF, keep geom mesh="..." names unchanged and only swap STL to OBJ with `<compiler meshdir="assets/obj"/>`.

Troubleshooting:
- Exploded or rotated visuals: you likely baked transforms (Apply Transform ON) or applied Location. Re-export with the checklist and re-import.
- Wrong size: re-export OBJs in meters, or as a last resort use `<mesh ... scale="0.001 0.001 0.001">` if your source is mm.
- Unity MuJoCo Plugin: press Play to build physics; enable frame/joint drawing to verify bodies are connected. Misalignment indicates visual frame issues, not missing joints.

### Blender export settings: impact on assembly

MuJoCo and the Unity plugin expect each mesh’s vertex coordinates to be expressed in the link-local frame used by the MJCF. If Blender export settings bake object transforms (location/axis/scale) into the mesh, visual geoms shift away from their bodies and the robot appears disconnected.

Your current OBJ export defaults:
- Include: Selection Only = OFF
- Scale = 1.0
- Forward Axis = -Z, Up Axis = Y
- UV Coordinates = ON, Normals = ON
- Triangulated Mesh = OFF
- Apply Modifiers = ON
- Apply Transform = ON

Guidance for OBJ:
- Apply Transform: OFF (critical; prevents baking axis and location into vertices; preserves link-local frames).
- Triangulated Mesh: ON (consistent triangle meshes).
- Normals: ON (visual quality; does not affect assembly).
- Scale: 1.0 (export in meters).
- Forward/Up: ignored when Apply Transform is OFF; leave defaults.
- Apply Modifiers: ON (safe).
- Selection Only: optional.
- If available, enable “Keep Vertex Order” and avoid any “recenter origin/pivot” operations.

Your current Blender STL export defaults:
- ASCII = OFF (binary)
- Selection Only = OFF
- Scale = 1.0
- Scene Unit = OFF
- Forward = Y, Up = Z
- Apply Modifiers = ON

Guidance for Blender STL:
- STL exporter bakes object transforms into geometry by design. If any object has nonzero Location, that translation is baked into vertices and changes the mesh’s local frame.
- To minimize damage, you would need per-object: rotation=(0,0,0), scale=(1,1,1), location=(0,0,0) before export—this is error-prone for assembled scenes.
- STL carries no material/vertex-order metadata and often has inconsistent normals; it’s less robust for Unity pipelines.

Recommendation (higher chance of correct assembly in Unity):
- Prefer OBJ export with Apply Transform OFF and Triangulated Mesh ON.
- Avoid re-exporting STL from Blender for Unity import; it tends to bake transforms and shift frames.

Quick OBJ export checklist:
1) After STL import: Apply Rotation & Scale only (Ctrl+A → Rotation & Scale). Do NOT Apply Location. Do not recenter origins/pivots.
2) Export OBJ:
   - Apply Transform: OFF
   - Triangulated Mesh: ON
   - Normals: ON
   - Scale: 1.0
   - Apply Modifiers: ON
3) In MJCF: only swap filenames to .obj and ensure `<compiler meshdir="assets/obj"/>`. Do not add geom-level pos/quat to “fix” visuals.

Why these properties matter:
- Apply Transform ON with Forward/Up remaps axes and location, rotating/offsetting vertices relative to link frames.
- Scene Unit and Scale rescale vertices; unit mismatch yields too-large/too-small parts.
- STL export bakes transforms; OBJ with Apply Transform OFF preserves object-local frames, matching MJCF expectations.

Troubleshooting:
- If visuals appear rotated or exploded, re-export OBJs with Apply Transform OFF and ensure you did not Apply Location.
- If size is wrong, fix units at export (meters) or use `<mesh ... scale="...">` as a last resort.

## Unity importer error: “convert to binary STL or OBJ”

The Unity MuJoCo plugin only supports OBJ and binary STL. The error
```
NotImplementedException: Type of mesh file not yet supported. Please convert to binary STL or OBJ.
```
means your source STL is ASCII. Re-export as binary STL or OBJ.

Blender → Binary STL export that preserves assembly:
- Before export (per object):
  - Apply Rotation & Scale (Ctrl+A → Rotation & Scale).
  - Do NOT Apply Location.
  - Rotation = (0,0,0), Scale = (1,1,1). Location can be nonzero in Blender; ensure it is not baked on export.
- Export STL (File → Export → STL):
  - ASCII: OFF (binary STL).
  - Scale: 1.0.
  - Scene Unit: OFF.
  - Forward: Y, Up: Z.
  - Apply Modifiers: ON.
  - Batch: optional if exporting many parts.

If you keep Location un-applied and export binary STL, vertices remain in the object’s local frame and visuals align with MJCF bodies in Unity.

### OBJ vs Blender STL for Unity (which is safer)

- OBJ (preferred):
  - Export with Apply Transform: OFF.
  - Triangulated Mesh: ON.
  - Normals: ON.
  - Scale: 1.0.
  - Best chance to preserve link-local frames without baking axes/location.
- Blender STL (fallback):
  - Binary STL (ASCII OFF).
  - No axis/location baking options; ensure you did NOT apply Location before export.
  - Robust for the Unity importer, but easier to accidentally bake offsets if transforms aren’t cleaned.

Recommendation:
- Use OBJ with Apply Transform OFF for highest chance of correct assembly.
- If OBJ still appears exploded, re-export as binary STL using the settings above and verify transforms were not baked.

### Mapping your Blender OBJ defaults to safe settings

Your OBJ defaults:
- Forward Axis = -Z, Up Axis = Y
- Triangulated Mesh = OFF
- Apply Transform = ON

Update to:
- Apply Transform: OFF (critical).
- Triangulated Mesh: ON.
- Normals: ON.
- Scale: 1.0.
- UVs: ON (optional).
- Forward/Up: leave defaults; ignored when Apply Transform is OFF.
- Selection Only: optional.
- Do not recenter origins/pivots.

### Export workflow: assembled scene vs per-part export

Goal: preserve each mesh’s link-local frame so Unity/MuJoCo assembles correctly.

Recommendation (highest probability of success):
- Prefer per-part export in a clean scene:
  1) Open a new Blender file.
  2) Import one original STL part.
  3) Per-object: Apply Rotation & Scale (Ctrl+A → Rotation & Scale). For STL export, set Location=(0,0,0). For OBJ export, keep Location as-is and do NOT bake it.
  4) Export that single part (OBJ or binary STL) with the checklists below.
  5) Repeat for all parts.

Why:
- Assembled scenes are easy to accidentally nudge or recenter, and STL export bakes Location into vertices. Per-part export ensures transforms are sanitized and avoids cross-object offsets.

OBJ export (preserve link-local frames):
- File → Export → Wavefront (.obj)
  - Apply Transform: OFF
  - Triangulated Mesh: ON
  - Normals: ON
  - Scale: 1.0
  - Apply Modifiers: ON
- Per-object before export:
  - Apply Rotation & Scale.
  - Do NOT Apply Location (keep link-frame offset unbaked).
  - Do not recenter origin/pivot.

Binary STL export (fallback; Unity importer supports binary STL, not ASCII):
- File → Export → STL
  - ASCII: OFF (binary)
  - Scale: 1.0
  - Scene Unit: OFF
  - Apply Modifiers: ON
- Per-object before export:
  - Apply Rotation & Scale.
  - Set Location=(0,0,0) (required: STL exporter bakes Location).
  - Do not recenter origin/pivot.

Notes:
- For OBJ, Location can be nonzero because we explicitly prevent baking transforms (Apply Transform OFF).
- For STL, Location must be zero to avoid baked offsets that “explode” the assembly in Unity.

Troubleshooting quick check:
- Exploded visuals with OBJ: you likely exported with Apply Transform ON or applied Location. Re-export per-part with the OBJ checklist.
- Exploded visuals with STL: Location wasn’t zeroed before export. Reset transforms and re-export binary STL per-part.
- Importer error about ASCII STL: re-export as binary STL or OBJ.

## License

This model is released under a [BSD-3-Clause License](LICENSE).

## Publications

If you use the MJX model in your work, please use the following citation:

```bibtex
@misc{zakka2025mujocoplayground,
  title={MuJoCo Playground},
  author={Kevin Zakka and Baruch Tabanpour and Qiayuan Liao and Mustafa Haiderbhai and Samuel Holt and Jing Yuan Luo and Arthur Allshire and Erik Frey and Koushil Sreenath and Lueder A. Kahrs and Carmelo Sferrazza and Yuval Tassa and Pieter Abbeel},
  year={2025},
  eprint={2502.08844},
  archivePrefix={arXiv},
  primaryClass={cs.RO},
  url={https://arxiv.org/abs/2502.08844},
}
```

### Blender hierarchy: Collection → pelvis (Object) → pelvis.001 (Mesh data)

Blender separates:
- Object: a scene node with Location/Rotation/Scale and a pointer to mesh data.
- Mesh data (datablock): the vertex/face geometry, often auto-named with a suffix (e.g., pelvis.001).

What you’re seeing:
- Scene Collection → Collection → pelvis → pelvis.001
  - “pelvis” is the Object (its transforms matter for export).
  - “pelvis.001” is the Mesh datablock used by that Object.

Which one to select for export:
- Select the Object (e.g., “pelvis”) and export.
  - OBJ (preferred): export with Apply Transform OFF to avoid baking axes/location; vertices remain in the Object’s local frame used by MJCF.
  - Binary STL (fallback): STL exporter bakes transforms; sanitize transforms first (Apply Rotation & Scale; set Location=(0,0,0)), then export with ASCII OFF.

Will binary STL match the original ASCII STL:
- Yes, if you do not bake unintended offsets:
  - Before export: Apply Rotation & Scale; set Location=(0,0,0).
  - Export settings: ASCII OFF (binary), Scale=1.0, Scene Unit OFF, Apply Modifiers ON.
- Result: same geometry and local frame, just in binary STL format supported by the Unity plugin.

Tips:
- Avoid recentering origins/pivots; do not move vertices.
- If Blender auto-creates multiple mesh datablocks (pelvis.002, etc.), ensure you export the intended Object and keep consistent filenames on disk for MJCF.

### Does “Selection Only” cause exploded parts?

Short answer: No. “Selection Only” only filters which objects are exported. It does not alter frames or bake transforms by itself.

Use it safely:
- Select the Object (e.g., “pelvis”), not just the mesh datablock (e.g., “pelvis.001”).
- OBJ export:
  - Keep “Selection Only” ON to export exactly the intended part(s).
  - Critical: Apply Transform = OFF. This prevents baking axes/location into vertices.
- Binary STL export:
  - “Selection Only” ON is fine.
  - Critical: zero Location (0,0,0) before export because STL exporter bakes Location.
  - Apply Rotation & Scale; ASCII OFF.

If parts still appear exploded, the cause is almost certainly:
- Apply Transform ON (OBJ) or nonzero Location baked (STL).
- Axis/unit mismatches or recentered origins/pivots.
