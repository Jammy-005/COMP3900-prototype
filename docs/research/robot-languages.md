# Robot Languages (Export)

> **Status:** the RAPID notes from the team doc are mostly accurate (corrections
> inline). **Zone data, extrusion control and the whole KUKA KRL section were empty
> in the source.** This document lists what they need to cover.

Covers R6 in [brief.md](../brief.md). The internal toolpath representation has to be
exported to the high-level, proprietary languages that control specific robots.

## ABB RAPID

Used to control ABB industrial robots. Example motion instruction:

```
MoveL [[120.5,45.2,300.0],[0.707,0,0.707,0],[0,0,0,0],[9E9,9E9,9E9,9E9,9E9,9E9]], v100, z5, tExtruder;
```

It has five components.

### 1. Instruction

- `MoveL`: *linear interpolation* in Cartesian space. The tool centre point (TCP)
  moves at constant speed along a straight line. Used for almost all non-planar
  deposition segments: controlled, predictable tool motion along the surface.
- `MoveJ`: *joint interpolation*. Each axis moves to its target independently, and
  the TCP path is neither straight nor predictable. Used for non-print travel
  (retract/reposition), where path shape doesn't matter.
- `MoveC`: *circular motion* through a via-point to an end-point. Rarely needed
  unless the segmentation strategy emits arcs.

> [!NOTE]
> **Review:** "path shape doesn't matter" for `MoveJ` is true only if the path is
> collision-free. Near the part, an unpredictable TCP path can hit printed material.
> Collision checking (R4) must sample travel moves in joint space.

### 2. Pose: `robtarget`

```
[[x,y,z], [q1,q2,q3,q4], confdata, extax]
```

- `[x,y,z]`: *translation*, the TCP position in mm, relative to the work object
  frame.
- `[q1,q2,q3,q4]`: *rotation*, the TCP orientation as a quaternion in **w, x, y, z**
  order, relative to the work object frame.
- `confdata = [cf1,cf4,cf6,cfx]`: *configuration data*. A target pose can usually be
  reached with several joint solutions (elbow up/down, wrist flip, etc.), and RAPID
  needs to know which one.
  - `cf1`, `cf4`, `cf6`: integer quadrant numbers for axes 1, 4 and 6,
    `floor(joint_angle / 90°)`. These axes can rotate through more than one quadrant
    to reach the same pose, so RAPID needs to know which one the solution used.
  - `cfx`: axis configuration number (0–7), encoding the shoulder/elbow/wrist branch
    of the inverse kinematics (IK) solution as a single index.
  - In practice, the kinematics module has to compute actual joint angles for each
    waypoint, derive the quadrant numbers from them, and keep `cfx` consistent. The
    reach/joint-limit solver already produces these candidate configurations.
- `extax = [eax_a, …, eax_f]`: *external axes*, values for up to 6 of them
  (turntable, linear rail, etc.) in the order the controller configuration assigns.
  `9E9` means "not present / not used". With no external axes, all stay `9E9`. If an
  axis is later added to the kinematic model, its position goes here. That keeps the
  brief's "don't preclude other platforms/axes later" condition satisfiable.

> [!NOTE]
> **Review:**
>
> - The frame is whichever work object is passed with `\WObj:=…`. If it's omitted,
>   `wobj0` (the world frame) is used. So the exporter should always emit an explicit
>   work object, and the IR must say which frame its coordinates are in.
> - RAPID requires a **normalised** quaternion. Normalise before writing, and use
>   enough decimal places (`0.707` is a rounded value).
> - `ConfL \On` (the default) makes the controller enforce `confdata` on linear moves
>   and stop if it can't. `ConfL \Off` lets it pick the nearest configuration.
>   `SingArea \Wrist` lets it deviate orientation slightly near wrist singularities.
>   These should be exporter settings. They interact with how IK solutions are chosen
>   ([design.md](../design.md#review) item 5).

### 3. Speed: `speeddata`

- `v100`: a predefined record, 100 mm/s TCP speed with default orientation and
  external-axis caps.
- For finer control, declare the record explicitly as
  `[v_tcp, v_ori, v_leax, v_reax]`:
  - `v_tcp`: TCP linear speed in mm/s. This is the actual print speed, mapped
    directly from the toolpath's speed field.
  - `v_ori`: reorientation speed in °/s, which caps how fast the tool can rotate
    while translating. On non-planar paths the orientation varies continuously
    (unlike planar paths, where it's fixed per layer), so expose this as a
    configurable parameter rather than hardcoding a default.
  - `v_leax`, `v_reax`: linear and rotational external-axis speeds.

### 4. Zone: `zonedata`

*(Empty in the source doc.)*

> [!NOTE]
> **Review, to fill in:** zone data sets how close the TCP must get to a target
> before blending into the next move (`z5` is roughly a 5 mm radius). `fine` means an
> exact stop. For printing, `fine` on every point causes stop-start blobs, and large
> zones cut corners. With dense toolpaths the zone has to be smaller than the point
> spacing. Make it a profile parameter.

### 5. Tool: `tooldata`

*(Not described in the source doc beyond the example.)*

> [!NOTE]
> **Review:** `tExtruder` is a `tooldata` record (TCP offset and orientation, mass,
> centre of gravity). It must either be declared in the exported program or already
> exist on the controller. The tool/TCP definition belongs in the tool model, not
> hardcoded in the exporter.

### Extrusion control

*(Missing from the source doc, and it is the central part of R6.)*

> [!NOTE]
> **Review, to fill in:** first find out how the client's extruder is driven (open
> question 4 in [client-meetings.md](../client-meetings.md#open-questions)). Common
> RAPID mechanisms:
>
> - `SetDO` / `SetAO` between moves: simple, but not synchronised with motion.
> - `TriggIO` + `TriggL`: set an output at a given distance or time along a linear
>   move, synchronised with the path.
> - `TriggSpeed`: an analog output proportional to the actual TCP speed. It's used
>   for dispensing, and suits keeping the bead size constant as speed changes.
>
> The exporter should take a per-controller **profile** that maps the IR's
> extrusion events and values onto instructions like these. R6 asks for a mapping,
> not fixed syntax.

### Program size

> [!NOTE]
> **Review:** non-planar prints can produce tens or hundreds of thousands of targets.
> Controller memory limits program size, so large jobs may have to be split into
> several modules loaded in sequence. Confirm the limit and the loading method with
> the client (open question 10).

## KUKA KRL

Used to control KUKA industrial robots. *(No other content in the source doc.)*

> [!NOTE]
> **Review, what this section needs to cover:**
>
> - **Motions:** `PTP` (joint), `LIN` (linear), `CIRC` (circular), plus spline
>   motions (`SPL`, `SLIN`) on newer controllers.
> - **Pose:** `E6POS {X, Y, Z, A, B, C, S, T, E1…E6}`. Orientation is Euler angles
>   in degrees: A about Z, then B about Y, then C about X. It is **not** a
>   quaternion, so the exporter must convert, and handle the gimbal-lock case near
>   B = ±90°. `S` (status) and `T` (turn) play the role of RAPID's `confdata`.
>   `E1…E6` are external axes.
> - **Frames:** `$BASE` (work object) and `$TOOL` (TCP), usually selected through
>   `BASE_DATA[n]` / `TOOL_DATA[n]`.
> - **Speed:** `$VEL.CP` is in **m/s**, whereas RAPID uses mm/s. This is an easy unit
>   bug.
> - **Blending:** `C_DIS` with `$APO.CDIS` (mm), the equivalent of RAPID zones.
> - **Extrusion:** `$OUT[n]` / `$ANOUT[n]`, or `TRIGGER WHEN DISTANCE=… DO …` for
>   outputs synchronised with the path.
> - **Files:** a program is normally a `.src` (code) plus `.dat` (data) pair.
>
> These differences are why one IR plus a per-controller profile matters. Both
> exporters read the same IR, and differ only in formatting and mapping.
