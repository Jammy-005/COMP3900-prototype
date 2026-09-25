# Design (draft)

> **Status: unvetted draft** from the team doc. The overall shape
> (load → compute → visualise → export) is reasonable. The details have significant
> gaps, listed under [Review](#review).

## Pipeline

Input (surface-defining file: STL, OBJ, or NURBS via CAD interchange; plus robot and
tool parameters) → file processing → toolpath generation (segmentation, deposition,
collision checking, etc.) → toolpath export (RAPID, KRL)

## Data flow

1. **Load**
   1. Receive inputs for the print, robot and tool.
   2. `PrintInputStrategy`: print → `PrintModel`
   3. `RobotInputStrategy`: robot → `RobotModel`
   4. `ToolInputStrategy`: tool → `ToolModel`
2. **Compute**
   1. Receive `PrintParameters` and a `PrintStrategy`.
   2. The `PrintStrategy` determines the `PrintPath` (where material gets
      deposited).
   3. A `CollisionScene` holds any existing `CollisionModel`s, and initialises the
      print's own `CollisionModel` as empty.
   4. The `ToolPath` (all positions the tool can be in) is computed by tracing the
      `PrintPath`, taking `PrintParameters` and the `CollisionScene` into account.
   5. The `RobotPath` is computed by tracing the `ToolPath`, taking the
      `CollisionScene` into account and selecting any possible solution from the
      `ToolPath`.
3. **Visualise**
   1. Open a separate program/window.
   2. Visualisation takes `RobotPath` + `ToolPath` + `PrintPath` to generate a
      video.
   3. Supports pause, play, speed-up, jump to a specific instruction, rotate, pan,
      and zoom.
4. **Export**
   1. Export the `RobotPath` using `ExportStrategies`.

The public API proposed to the client is `Load`, `Compute`, `Visualise`, `Export`
(see [client-meetings.md](client-meetings.md#api)).

## Not yet written

The source doc has headings but no content for **Classes/Objects** and **API
Documentation** (struck through). The **Proposal** section is also empty.

## Review

### What's good

- The staged pipeline maps well onto the module boundaries the brief asks for (R8).
- **Starting the print's collision model empty and growing it** is the right idea.
  The nozzle has to clear what has *already been printed* at each point, not the
  finished part. Keep this.
- Separate loaders per input type make it easy to add formats later (R1).

### Problems

**1. Segmentation is missing (R2).** "`PrintStrategy` determines the `PrintPath`" is
the whole slicer, and nothing says which strategy. The brief asks us to adapt known
approaches, not research new ones. Candidates, roughly easiest first:

- *Planar (fixed Z)* as a baseline. It isn't the goal, but it gives a known-good
  input for testing every downstream stage.
- *Conformal / tween:* interpolate curved layers between a bottom and a top surface,
  or offset a top surface downwards.
- *Scalar-field isolines:* define a field over the model (e.g. interpolated height,
  or geodesic distance from the base). Its iso-surfaces are the layers, and
  iso-curves on each layer are the paths.
- *Stress-aligned:* needs FEA input. Probably beyond an MVP.

Survey existing open-source work before designing from scratch. For example,
`compas_slicer` (Python, curved-layer slicing, grew out of Grasshopper workflows) is
worth evaluating. Also ask the client which "initial libraries" they have tried
(open question 7).

**2. No intermediate representation (R5).** `PrintPath` / `ToolPath` / `RobotPath`
have no defined fields, and Export reads the `RobotPath`, which is robot-specific.
The brief needs one robot- and CAD-agnostic structure that both export and
visualisation read. A minimal version:

- per point: position, tool orientation, speed, extrusion/flow value
- per segment: type (print or travel)
- per path/layer: layer index, layer height, bead width
- per file: units, reference frame, parameters used

Serialise it (JSON is fine for an MVP), so the viewer and exporters can run from a
file without re-slicing.

**3. Extrusion is not modelled (R6).** Nothing in the data flow carries flow rate or
extruder on/off. On non-planar layers the thickness varies *along a single path*, so
flow or speed has to vary per point (flow ∝ local layer height × bead width ×
speed). The IR should carry an abstract quantity (e.g. volumetric flow, or a
normalised 0–1 value plus on/off events). A per-controller profile maps it to
concrete I/O commands. See
[research/robot-languages.md](research/robot-languages.md#extrusion-control).

**4. Orientation solving needs to be its own stage (R3).** The flow doesn't include:

- where the nominal nozzle direction comes from. For curved layers it's the normal
  of the *layer* surface (for a scalar field, its gradient), not the part's surface
  normal. See
  [research/input-formats.md](research/input-formats.md#challenges-for-this-project).
- a maximum tilt from the build direction, exposed as a parameter
- smoothing along the path, so orientation doesn't jump between neighbouring points

**5. "Selecting any possible solution" for IK will fail in practice.** A 6-axis arm
usually has several IK solutions per pose (up to 8 for a typical spherical-wrist
arm). Choosing one per point independently leads to configuration flips mid-path.
The controller can't switch arm configuration during a linear move, so at best the
program stops with an error. Solutions have to be chosen **across the whole path**:
enumerate candidates at each point, then pick a continuous sequence (e.g. a
shortest path through a layered graph, where edge cost is joint motion and edges
that jump or hit limits are infeasible).

The design also misses a big lever. An axisymmetric nozzle is indifferent to
rotation about its own axis, so the task is effectively 5-DOF on a 6-DOF robot. That
free angle should be searched to stay clear of joint limits, singularities and
collisions.

**6. The robot model has to be data, not code (R3).** `RobotInputStrategy` needs a
concrete input format: the kinematic chain (DH parameters or URDF), joint limits,
and link collision meshes. URDF covers all three and is widely used. Confirm what
format the client's "ABB models" arrive in.

**7. Collision checking is under-specified (R4).** Beyond the growing print model,
the scene needs:

- the tool/nozzle geometry
- the robot links at each solved configuration
- the build plate and any fixtures
- travel moves, sampled in joint space. A joint-interpolated move (`MoveJ` / `PTP`)
  does not follow a straight TCP line.

R4 says "detect **and flag**", so results should be recorded per point/segment for
the viewer to highlight, not just raised as a pass/fail.

**8. The visualisation step contradicts the client.** The client asked for a
**web-based** interface; the design says "open a separate program/window" and
"generate video". A video isn't an interactive preview. The viewer should load the
IR plus the robot model, scrub along a timeline, and highlight flagged collisions
and reach failures. See [research/visualisation.md](research/visualisation.md).

**9. The proposed API couples things too early.** `Load` takes the robot and tool in
the constructor, although segmentation needs neither. `Export` takes only a file
type, although it also needs a controller profile (extrusion mapping, speeds, zones,
frames). For R8, make each stage a separately callable, testable step with explicit
inputs and outputs, and add a convenience facade on top if wanted:

```
load_model(path, units)                  -> Mesh
slice(mesh, strategy, params)            -> PrintPath        # layers of curves
orient(print_path, orientation_params)   -> ToolPath         # the IR
solve(tool_path, robot, tool)            -> RobotPath        # joint configs + reach flags
check_collisions(robot_path, scene)      -> CollisionReport  # per-point flags
export(tool_path, robot_path, profile)   -> program text     # RAPID / KRL
```

**10. Coordinate frames are missing.** Define once, and carry through every stage:
model units and origin, the build plate / work object frame, and the tool (TCP)
frame. RAPID and KRL both express targets relative to a work-object/base frame and a
tool frame.
