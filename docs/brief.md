# Project Brief: 132 Multi-axis and Non-planar Slicer for Robotic 3D Printing

Transcribed from [reference/132-project-brief.pdf](reference/132-project-brief.pdf).
The requirement and deliverable IDs (R1…, D1…) are added here for traceability.
Otherwise the wording follows the brief.

- **Client:** Louis Lamont, Richard Blackwell, Max Melamed
- **Allowed groups:** 2
- **Disciplines:** Software Development; Web Application Development; Robotics;
  Additive Manufacturing
- **Resource:** https://archmanu.com/cpd/talk-series-48/

## Background and goals

Non-planar slicing for robotic printing generates paths where nozzle orientation and
layer height vary continuously across a surface, instead of fixed Z-stacked layers.
It requires:

1. decomposing a surface into deposition paths,
2. solving tool orientation within reach/joint limits,
3. checking collisions across the full path, and
4. outputting motion the controller can execute.

No open-source slicer solves this end to end. Capable tools are either
closed-source and cell-specific, or ad hoc Grasshopper definitions that don't
generalise. This project builds on that existing logic to develop an open, modular
slicer that is independent of CAD platform and robot type. The core algorithms are
built as a standalone library that supports export to common robot formats.

## Scope

**In scope:** a standalone, open-source non-planar slicing **library (not a CAD
plugin)** that takes a 3D model plus print parameters and outputs multi-axis
toolpaths:

- surface segmentation into deposition paths
- tool orientation solving
- path-level collision checking
- an intermediate toolpath representation independent of robot/CAD
- export to ABB RAPID and KUKA KRL with configurable extrusion command logic

**Out of scope:**

- slicing-algorithm research beyond adapting known approaches
- robot platforms beyond RAPID/KRL. The architecture should not preclude adding
  others later.

**Stretch:** physical print validation at Composite Sydney's studio if development
reaches a satisfactory level.

## Requirements

| ID | Requirement |
|---|---|
| R1 | Accept common mesh/surface input formats (e.g. STL, OBJ, or NURBS via a CAD interchange format). |
| R2 | Segment/slice input geometry into deposition paths using a defined strategy (e.g. isoline / stress-aligned / conformal / tween), with parameters **exposed rather than hardcoded**. |
| R3 | Compute tool orientation per path point, constrained by a **configurable** robot reach/joint-limit model (not tied to one physical arm). |
| R4 | Detect and flag collisions between **tool, part, and robot body** across the toolpath. |
| R5 | Represent toolpaths in an intermediate, robot- and CAD-agnostic data structure. |
| R6 | Export toolpaths to ABB RAPID and KUKA KRL, with extrusion/flow commands defined as an **adjustable, per-controller mapping** rather than fixed syntax. |
| R7 | Provide a way to visualise generated toolpaths (simulation/preview) for verification without requiring physical hardware. |
| R8 | Structure the codebase as a library with clear module boundaries (segmentation, kinematics/orientation, collision, export), not a monolithic script, so it can be extended or swapped independently. |

## Deliverables

| ID | Deliverable |
|---|---|
| D1 | Open-source source code (library and any CLI/visualisation tooling) in a public repository |
| D2 | Documentation covering architecture, module interfaces, and how to extend segmentation/orientation/export strategies |
| D3 | User guide for running the slicer on sample geometry and generating RAPID/KRL output |
| D4 | A small set of test cases/sample models demonstrating the pipeline end to end in simulation |
| D5 | Short final report or write-up summarising design decisions and known limitations |

## Expected knowledge and skills

Solid programming fundamentals (Python and/or C); 3D geometry and linear algebra
(vectors, transforms, meshes); some robotics (forward/inverse kinematics, joint
limits), which can be learned during the project; interest in computational
geometry and/or robotics; version control and multi-module codebases.
Grasshopper/Rhino or RAPID/KRL experience is a bonus, not a requirement.

## Generative AI policy

The client is happy for teams to make responsible use of generative AI tools
(Copilot, ChatGPT, Claude, etc.). They would rather students ship a better slicer
faster than avoid the tools on principle. **Students must still understand, and be
able to explain, every part of the codebase they submit.** AI-assisted code is not a
substitute for learning the geometry, kinematics, and robotics concepts behind the
project.
