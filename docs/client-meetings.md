# Client Meetings

Questions the team put to the client, with the client's answers (**A:**).
Unanswered and not-yet-asked questions are collected at the end.

The answers are authoritative. The questions are the team's own framing.

## Meeting 1: requirements clarification

*(Date not recorded in the source doc.)*

### Problem framing

**Q:** Is this a decision problem ("can we print this, yes/no") or an optimisation
problem ("fastest / most efficient / fewest lifts")?

**A:** More of an initial MVP. Software for this doesn't really exist yet: there are
enterprise packages behind paywalls, and slicing done in Rhino (CAD) and
Grasshopper. The client is looking for something that can initially **supplement
their current workflows**. They have found some initial libraries with some early
results. The goal is a slicing tool that can generate more complex toolpaths than
fixed-Z (planar) slicing.

**Q:** Which print traits should we prioritise (surface quality, total print time,
inter-layer cooling), or should we balance them?

**A:** "We just want something out there." Balance traits in general. Prioritising
a particular parameter comes later.

### Technology

**Q:** We will at least prototype in Python. Will C be required?

**A:** Our choice of language.

**Q:** Can we use other open-source libraries?

**A:** Yes, whatever we think we need.

### Visualisation

**Q:** Where does visualisation happen: a separate web application, a separate
desktop application, or within the library?

**A:** A **web-based interface** for visualisation.

**Q:** What exactly should the visualisation do (e.g. pause / play / rewind)?

**A:** Our choice.

### API

**Q:** Do you want changes to our current API's inputs and outputs?

- `Load`: constructor; receives the print model, robot, and tool
- `Compute`: computes the toolpath; receives print parameters and strategy
- `Visualise`: runs visualisation with the existing toolpath, if there is one
- `Export`: returns commands; receives the output file type

**A:** No direct answer recorded. **The client will send ABB robot models.** (In the
source doc this note sits under the API question; it may really answer the
robot/nozzle question below.)

### Hardware

**Q:** If the project is successful, will the software be tested on actual hardware?

**A:** Definitely, hopefully pretty soon.

### Other notes from the meeting

- The client wants **the two groups on this brief to work on slightly different
  things**.
- We can assume the user has a decent understanding of the print parameters.

## Open questions

These were asked without a recorded answer, or will be needed and haven't been
asked yet.

| # | Question | Status | Why it matters |
|---|---|---|---|
| 1 | Exactly which nozzle/extruder and robot? | Asked; answer unclear (see API above) | Kinematics (R3), collision geometry (R4), extrusion mapping (R6) |
| 2 | Any preferred visualisation library or deployment target? | Asked; no answer | R7 |
| 3 | Any feedback on the Load / Compute / Visualise / Export API? | Asked; no answer | R8 |
| 4 | How is the extruder driven: digital/analog I/O, fieldbus, a separate controller? What material? | Not asked | R6: the export mapping can't be designed without this |
| 5 | Which ABB model and controller/RobotWare version? Is there a KUKA cell to test KRL on? | Not asked | R6: instruction availability, program-size limits |
| 6 | Cell layout: build plate / work object frame, fixtures, external axes (turntable, rail)? | Not asked | R3, R4 |
| 7 | Can they share their existing Grasshopper definitions and the "initial libraries" they mentioned? | Not asked | The brief says the project "builds on that existing logic" |
| 8 | Which open-source licence? | Not asked | D1 |
| 9 | Which part is the other group focusing on? | Not asked | The client wants the groups on different things |
| 10 | How do they load programs onto the controller? Any file-size limits? | Not asked | Non-planar prints can produce very long programs |
