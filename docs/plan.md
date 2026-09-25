# Plan

> **Status: unvetted draft.** Names and per-person assignments have been removed;
> only the workstreams are kept.

## Sprint 1 workstreams

| Workstream | Pipeline stage (see [design.md](design.md)) |
|---|---|
| Inverse kinematics (robot path) | Compute → `RobotPath` |
| Tool + print parameters (tool path) | Compute → `ToolPath` |
| Pipelines and integration tests | All |
| Visualisation | Visualise |
| Infrastructure with all input/output types | Load, data types |
| Tooling, design | All |

## Review

- **Nobody owns segmentation, export, or collision.** Segmentation is the core
  algorithm (R2), and export produces what the client will actually run (R6).
  Neither appears in Sprint 1.
- **Starting with IK is risky.** IK is hard to test without real toolpaths to feed
  it. Build a thin end-to-end skeleton first, then replace stages one at a time:
  1. Load a simple sample STL.
  2. Slice it planar, with a vertical nozzle.
  3. Write the IR to a file.
  4. Export RAPID from the IR (motion only, with a stub extrusion mapping).
  5. Show the IR in a web viewer.
  6. Add an integration test that runs steps 1–5 on the sample.

  Every later improvement (a non-planar strategy, orientation solving, IK,
  collision) then drops into a working pipeline, and the integration test catches
  regressions. It also puts an MVP in front of the client early, which is what they
  asked for.
- **"Infra with all in/out types" is the right early task.** Its output should be
  the IR schema ([design.md](design.md#review) item 2), because every other
  workstream depends on it.
- **No milestones yet.** The source doc has no dates, sprint goals, or definition of
  done. Add them once the course sprint schedule is fixed.
- **Coordinate with the other group.** The client wants the two groups working on
  slightly different things. Agree the split before Sprint 1 work starts.
