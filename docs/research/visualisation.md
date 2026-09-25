# Visualisation

> **Status:** library survey from the team doc, tidied. The client wants a
> **web-based** interface (see
> [client-meetings.md](../client-meetings.md#visualisation)), and that should drive
> the choice.

Covers R7 in [brief.md](../brief.md). All of these are open-source Python libraries.

| Library | Runs in | Notes from the source doc |
|---|---|---|
| **Viser** | Web (local server + browser client; usable over SSH; Jupyter-compatible) | Fast, simple 3D viewer for point clouds and meshes. API for 3D primitives; GUI building blocks (buttons, checkboxes, text inputs, sliders); scene interaction (clicks, selection, transform gizmos); programmatic camera control and rendering. |
| **Rerun** | Web and desktop | Timeline playback for stepping through states frame by frame. Can show orientation vectors, velocities, etc. "Great for toolpath simulation." |
| **PyVista** | Desktop (VTK-based) | Shows 3D meshes, stress points and contour maps. Integrates with NumPy and is efficient with numerical data. |
| **MeshLib** (spelled "MesbLib" in the source) | Library | Supports multi-axis workflows and manufacturing simulation. Geometry engine with additive-manufacturing, collision and mesh-slicing features. |

> [!NOTE]
> **Review:**
>
> - **Viser and Rerun match the client's "web-based" answer.** PyVista is
>   desktop-first; web support comes through extra backends.
> - **Rerun's timeline** covers the design's wish list (pause, play, speed-up, jump
>   to instruction) with little custom UI. **Viser** gives more control over custom
>   GUI, e.g. parameter sliders that trigger a re-slice. Prototype both on the same
>   small IR file before committing.
> - **MeshLib is a geometry library, not a viewer.** Evaluate it for segmentation and
>   collision instead. **Check its licence first:** the deliverable must be open
>   source, and not every freely downloadable geometry library permits that.
> - Whatever the viewer, it should **read the serialised IR** (see
>   [design.md](../design.md#review) item 2) rather than live Python objects. That
>   keeps it decoupled from the library, and lets anyone preview a toolpath without
>   re-running the slicer.
> - For verification (R7), show more than the path: the robot pose at each step, and
>   highlighted collision and reach failures (R4, R3).
