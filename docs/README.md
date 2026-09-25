# Documentation

These documents were reorganised from the team's working Google Doc. Team-member
details (skills, roles, project preferences, contact details, per-person task
allocations) have been removed. The original export was removed from the
repository because it contained personal information.

## How to read these docs

The source doc mixes three kinds of content, and they deserve different levels of
trust:

| Kind | Trust | Where |
|---|---|---|
| Client brief and client answers | **Authoritative**: this is what we are building | [brief.md](brief.md), [client-meetings.md](client-meetings.md) |
| Research on file formats, robot languages, libraries | **Mostly accurate**; corrections marked inline | [research/](research/) |
| Design ideas and plans | **Unvetted drafts**; several ideas have real problems | [design.md](design.md), [plan.md](plan.md) |

Review comments added during reorganisation look like this:

> [!NOTE]
> **Review:** …

Anything without that marker comes from the original doc (tidied, not changed in
meaning).

## Contents

- [brief.md](brief.md): client brief, with numbered requirements (R1–R8) and
  deliverables (D1–D5) so work can be traced back to them
- [client-meetings.md](client-meetings.md): questions asked, client answers, open
  questions
- [design.md](design.md): proposed pipeline and data flow, with review
- [plan.md](plan.md): Sprint 1 workstreams, with review
- research/
  - [input-formats.md](research/input-formats.md): STL, OBJ, NURBS via STEP/IGES
  - [robot-languages.md](research/robot-languages.md): ABB RAPID, KUKA KRL
  - [visualisation.md](research/visualisation.md): candidate viewer libraries
- [reference/132-project-brief.pdf](reference/132-project-brief.pdf): original
  brief

## Biggest gaps in the current thinking

1. **No segmentation strategy has been chosen.** Segmentation is the slicer's core
   algorithm (R2), and it appears in neither the design nor the Sprint 1 plan.
   See [design.md](design.md#review) item 1.
2. **Extrusion is not modelled anywhere.** R6 requires a configurable,
   per-controller extrusion mapping. Non-planar layers change thickness along a
   single path, so flow must vary per point. See [design.md](design.md#review)
   item 3.
3. **The intermediate representation is undefined.** R5 requires a robot- and
   CAD-agnostic toolpath structure. The design names `PrintPath` / `ToolPath` /
   `RobotPath` without defining them, and export reads from the robot-specific one.
   See [design.md](design.md#review) item 2.
4. **Nozzle orientation is confused with the part's surface normal.** The STL
   research spends effort smoothing mesh normals, but in curved-layer slicing the
   nozzle follows the *layer* surface, not the part surface. See
   [input-formats.md](research/input-formats.md#challenges-for-this-project).
5. **Picking "any possible" IK solution per point** will cause configuration flips
   mid-path. The choice has to be made across the whole path, and it should use the
   free rotation about the nozzle axis. See [design.md](design.md#review) item 5.
6. **The visualisation plan contradicts the client.** The design says "separate
   window… generate video", but the client asked for a web-based interface.
7. **Empty sections:** KUKA KRL, RAPID zone data, extrusion control, the proposal,
   and the class/API design.
