---
name: mechfaber
description: Design a real machine - you own the topology and the geometry, the solvers own every number. Use for robots, arms, mechanisms, anything with joints and loads.
---

# Designing machines

**You design the machine. The tools own the numbers.**

The engine owns duty and torque, reach and link lengths, gait and speed,
packing, statics, simulation, and which real components exist at what
rating. Never write one of those from memory - call for it.

You own the machine: its topology, its proportions, its geometry. You draw
the parts. Nothing in this engine will draw them for you and nothing should
- a solver that picks a shape is a solver guessing, and the engine has no
idea what kinds of machine exist. You do.

## What runs in parallel, and what cannot

A design session is slow because of how many exchanges it takes. So the
question at every step is not "how long does this compute" - almost nothing
here computes for long - but "does this have to wait for the thing before
it".

**Fan out over independent items.** Sourcing is N candidates that know
nothing about each other; STEP inspection is N files that know nothing about
each other. Both are context-heavy in and small out, which is exactly what a
subagent is for. Two are shipped with this plugin:

- **`part-reader`** - one candidate's datasheet, read and stored. One per
  candidate.
- **`step-inspector`** - one vendor STEP, downloaded and measured into bolt
  circles and an envelope. One per sourced module.

**Start sourcing BEFORE you need it.** It is the long pole and nothing about
it depends on your geometry. Dispatch the whole bill - actuators AND
electronics - the moment you know roughly what the machine is, then do step 1
against assumed proportions while they read. Their answers land before you
need the actuator masses to close the duty loop. Sourcing at step 2 in
sequence, then drawing, is a serial chain that had no reason to be one.

**Batch independent tool calls into one turn.** Not everything needs an
agent. `mech_joint_duty` and `mech_simulate` take the same graph and do not
depend on each other - issue both in one turn, and `mech_link_stress` after,
because it takes duty's output. Same for a pose sweep: each pose is its own
independent call. A turn spent asking one question that could have carried
four is a minute.

**Do NOT fan out the geometry.** The parts have to mate: shared datums, a
bolt circle one part cuts and another matches, a joint frame both sides
agree on. Two agents drawing an upper arm and a forearm in isolation produce
two parts that do not connect, and you will not find out until the build.
The topology, the proportions and every solid are yours, in one head, in one
script. This is the same reason nothing in this engine draws for you.

**Do not fan out a feedback loop.** Draw, gate, read the verdict, repair -
each step needs the one before it. Spawning an agent to "review the build
result" adds a round trip to read a structured result that is already on
your screen.

## The loop

**1. Numbers are not yours to choose.** `mech_joint_duty`, `mech_simulate`,
`mech_link_stress` and `mech_power_budget` judge any dimension you give them.

Nothing here PROPOSES a dimension - there is no layout solver, and a link
length, a plate thickness or a bolt circle is yours to pick and theirs to
refuse. If a proportion has to be assumed to get
started, **state it as an assumption in your answer and gate it immediately**
- an assumed number that passes a gate is evidence; one that never reaches a
gate is a guess wearing a result.

An INFEASIBLE result names the violated constraint - change the spec, never
the number.

**2. Get every sourced component's real numbers - from the sourcing tools,
not the open web.** That means the actuators AND the electronics. A machine
sources its whole bill: motors and gearmotors, but also the MCU, the
regulators/BECs, the battery pack, and any driver or ESC between them -
every number the wiring layout and the firmware run will later demand
(`min_voltage_v`, `output_a`, `capacity_wh`, Kt, winding R) is a sourcing
number exactly like rated torque, and typing one from memory is the same
mistake at a different voltage. Source them in the same pass you source the
actuators, while the shortlist is in front of you - not at step 5, where a
missing rating turns into a guess because the loom is half-drawn.

`mech_part_search` asks every catalogue at once: this
installation's own measured library, the step.parts mirror (16k+ parts with
STEP files), Mouser and EasyEDA - the latter two are WHERE THE ELECTRONICS
ARE, so a search that only ever names motors is using half the index. Ask it
for the MCU board and the regulator like you ask it for the shoulder module.
Read `trust` on each
hit - `measured` numbers were transcribed here from a cited page with the
evidence kept; `stated` numbers are the vendor's claim. Shortlist first,
then measure. Every record lands in the library with its citation, so the
next design starts richer than this one did. `mech_part_cad` fetches the
hit's STEP into your project and measures it on the way in.

### Reading candidates: one `part-reader` per part, all at once

**YOU are the reader now.** The engine does not run a model over vendor
pages any more - it fetches the documentation and hands the text back. That
made reading fast and free; what makes it FAST IN WALL CLOCK is doing all of
it at once.

**Dispatch one `part-reader` subagent per candidate, in a single turn** -
the actuator shortlist and the MCU and the regulator and the pack, together.
Each one calls `mech_part_docs`, reads its own document, calls
`mech_part_measure` itself, and reports one line:

> `robstride_03 - selectable:true | rated 2.4 A / peak 11 A / 0.92 N.m | envelope [46,46,47]`

Six candidates then cost roughly the slowest one, and six datasheets - tens
of thousands of characters each - never enter your context. Read serially in
your own window instead, that is most of an hour and most of a context spent
on pages you are about to reject.

The two tools, when you do need them directly (a single part, or a page a
subagent could not resolve):

- `mech_part_docs(part_id)` - the product page and its datasheet PDF as
  text, the field names this engine calculates with, and the record it
  already holds, if any.
- `mech_part_measure(part_id, specs=[...])` - what you read, stored.

Each spec is `{field, value, unit, evidence}`. Copy the value and unit
**exactly as printed** - `9.4` and `kgf.cm`, never `0.92` and `N.m`. The
engine converts; a number you converted yourself is a number nobody can
check. The `evidence` span is copied **verbatim** and is matched character by
character against the page the engine re-fetches, so a tidied or paraphrased
quote is discarded and that rating is lost.

**A subagent transcribes and reports. It does not choose the part.** You
hold the joint torques, the mass budget and the envelope; the choice and
the `why` you write into `mech_actuators` are yours, made when you make
them - not reconstructed afterwards from a subagent's recommendation.

If a reader comes back "already measured, selectable, same page", that part
cost nothing and there is nothing to submit.

### Then the STEPs: one `step-inspector` per chosen module

Once the modules are chosen, fan out again - one `step-inspector` per part.
Each downloads the vendor STEP into the project and hands back the mating
interface (bolt circles with their diameter, count, centre and axis; the
shaft; the envelope, and whether it agrees with the datasheet) instead of a
feature dump you have to read. Those few numbers are what your build script
mates to.

Reach for a general web search ONLY when every source has answered and none
carries the part - and then feed what you find THROUGH `mech_part_docs` and
`mech_part_measure` rather than typing numbers into the design, because a rating with no
citation is a number somebody typed, and it will be the one that is wrong.
Put the URL in your answer beside each number either way.

Rated torque governs sizing; peak is a transient, and a joint sized on peak
cannot hold its own pose. Watch the units: 20 kgf.cm is 1.96 N.m, not 196.

If you have the vendor STEP, `import_step()` it and design against the real
bolt circle and shaft rather than a bounding box - but check its bbox against
the datasheet envelope first, because a vendor STEP can be an exploded
assembly or in the wrong units. `mech_inspect_cad` measures it for you.

**3. Draw it.** `mech_build` runs the build123d you write and reports
volume, bbox, mass, centre and solid count. It never edits your design.
Iterate here until the part is right.

**Name AND title every part.** `show(part, "upper_arm", title="Upper arm
shell")` - the snake_case name is the graph identity joints and gates refer
to; the title is what a person reads on the parts list and the BOM. A
machine whose bill reads `upper_arm, j2_module, gripper_base` was named for
the solver and nobody else; the title costs six words at the moment you
know exactly what the part is.

**Then LOOK at it: `mech_look` after every build.** It returns actual
renders (iso, front, right, top) of the mesh you just built. The mate
checks now measure what a picture used to be the only witness for - a link
buried in its own actuator comes back `BURIED_MATES`, a mate holding its
parts apart in space comes back `FLOATING_MATES`, and every bearing's
overlap is on `bearing_overlaps`. Terminate links at the actuator's
mounting face; the joint's volume belongs to the actuator, not your beam.
The picture still catches what no boolean can - proportion, orientation, a
part that is legal and wrong - so a build you have not looked at is still
a build you have not checked.

**A build script draws geometry - that is the whole surface.** You have
build123d and the pure-computation standard library (math, itertools,
functools, dataclasses). You do not have file, process, network or
reflection modules, `eval`/`exec`/`open`, or dunder attributes: the engine
owns every file a build reads or writes, and `import_step()` reads only
what the project owns - the reference `mech_part_cad` returned, like
`cad/ak80_64.step`. A refusal names the construct and the line.

**USE JOINTS, NOT TRANSFORMS.** Assemble with `RigidJoint` / `RevoluteJoint`
and `connect_to()`:

```python
from build123d import *
base = Cylinder(45, 25); base.label = 'pedestal'
RevoluteJoint('shoulder', base, axis=Axis((0,0,12),(0,1,0)),
              angular_range=(-180,180))
ua = Box(220,34,22, align=(Align.MIN,Align.CENTER,Align.CENTER))
ua.label = 'upper_arm'
RigidJoint('prox', ua, Location((0,0,0)))
base.joints['shoulder'].connect_to(ua.joints['prox'], angle=55)
result = Compound(children=[base, ua])
```

Two reasons this matters. Parts placed by mates cannot all end up stacked at
the origin, which is the most common way an assembly comes out wrong and
still renders. And `mech_build` reads the joints back and returns a `graph`
- world positions, hinge axes, parent/child - so the physics model comes out
of your CAD instead of being typed a second time and drifting from it.

Give a part a colour if the design has one - `part.color = Color("orange")`
before you mate it. It reaches the viewer as a real glTF material. Leave it
off and the part stays neutral; the engine will not pick one for you.

## Draw the part the load asks for

The default output of this loop is flat plates bolted at right angles with
the actuators hanging off the outside, and it looks like a bracket kit
because that is what it is. A machine that looks designed is not styled
afterwards - every one of the rules below is structural, and the appearance
is what falls out of applying them. Not one of them is decoration, so none
of them is optional on a part that carries load.

**Terminate the link at the actuator's mounting face, and put the actuator
INSIDE.** A servo sitting proud on a plate is an unsupported cantilever
carrying its own reaction, a snag hazard, and the single biggest reason a
machine reads as a prototype. The limb shell should close around the module
and bolt to its face. Enclosing the module is right; SWALLOWING it is not -
the shell must stop at the mounting face, and `BURIED_MATES` on the build
result is the gate that now catches the difference.

**Taper every limb.** Bending moment on a leg falls roughly linearly from
the hip to the foot, so a constant-section beam is heavy everywhere it is
not loaded. `loft()` from a large section at the driven joint to a small one
at the far end. This alone is most of the visual difference between a stack
of plates and a limb.

**Fillet every structural corner, inside and out.** A sharp re-entrant
corner is a stress concentration - it is where the part will crack - and
`mech_link_stress` computes on section, so it cannot warn you. Radius at
least the wall thickness:

```python
part = fillet(part.edges().filter_by(Axis.Z).group_by(SortBy.LENGTH)[-1], 3)
```

**Shell the body; do not build it from plates.** A closed shell is far
stiffer per gram than the same mass of flat panels bolted at their edges,
and it is what makes a chassis read as one object rather than an assembly.
`offset(solid, -wall, kind=Kind.INTERSECTION)` after the outer form is
right. Build with `features=True` so the section is measured off the real
wall rather than the bounding box - a shelled part judged on its bbox comes
back UNVERIFIED, which is the engine refusing to believe a number it knows
is wrong.

**Colour by function, not per part.** Two or three materials across the
whole machine - structure one colour, limbs another, sourced modules left
dark - reads as deliberate. Twenty different colours, or one grey
everywhere, both read as unfinished.

**What this cannot give you.** Compound-curved surfacing, the kind that
makes a commercial robot look moulded, is the work of an industrial design
team and is painful in build123d. Aim for: enclosed modules, tapered limbs,
consistent radii, a shelled chassis and two colours. That is achievable in
this loop and it is most of the distance.

**Then look.** `mech_look` after the form changes, not just after the
topology does. Every rule above is a claim about a shape, and a shape is the
one thing the numeric gates never see.

**Import the actuator, do not draw a cylinder for it.** Point `import_step`
at the vendor file you downloaded - it costs about two seconds:

```python
motor = import_step(r"<path to the vendor STEP>")
motor.label = "j2_module"        # ALWAYS label an import
RigidJoint("mount", motor, Location((0, 0, 0)))
```

**Then read its features instead of typing them.** `mech_inspect_cad(path)`
returns every hole, boss, bolt circle and mounting face measured off that
solid - a real gearmotor came back with its Ø31 pattern in three bolt sizes.
Mate to `bolt_circle_d_mm` / `count` / `center_mm` / `axis`. A bolt circle
you transcribe from a datasheet is a number nobody checked; one read off the
faces is a measurement. It also tells you fast whether a vendor file is
usable at all - an exploded view shows up as a body count and envelope that
disagree with the datasheet.

The label names the body in the graph, so use it. An import stays one part
either way - the engine splits a result only into children the script itself
made. A multi-solid import is normal and is reported; it is fine to mate and
mass, and not printable. An envelope cylinder is the fallback for parts with
no downloadable STEP, not the default.

**A viewer is not a physics engine.** Driving joints in the Studio is forward
kinematics - solids pass straight through each other and nothing stops them,
in this viewer or any other. What checks solids is `interference` on the
build result, and it checks THE ASSEMBLED POSE ONLY. If you pose the machine
and want to know it still fits, rebuild it in that pose.

**4. Gate it.** Feed that graph to `mech_joint_duty` and `mech_simulate`
**in the same turn** - they take the same graph, they do not depend on each
other, and running them one after the other buys a minute of waiting and
nothing else. `mech_link_stress` comes after, because it consumes duty's
output. If several poses matter, all of their duty calls go in that same
turn too.

`mech_simulate` asks a different question depending on how you say the
machine is mounted, and you have to say:

- `base="free"` — dropped on the floor and nudged. STANDS/FELL.
- `base="fixed"` — root bolted down, every actuated joint commanded to its
  `hold_deg`. HOLDS/SAGS on whether they got there. This is the gate for
  anything on a bench or a column; dropping an arm tells you nothing.

Set `torque_nm` on each joint from the actuator you sourced. It becomes a
hard force limit, so an undersized motor fails here physically instead of
passing a check. Without it the joint gets an unlimited motor, holds
anything, and proves nothing - the result says so.

**Run both gates and compare.** `mech_joint_duty` and `mech_simulate` are
independent - closed-form statics against a stepped simulation. On a correct
model they agree to a few hundredths of a N.m. When they disagree, one of
them is wrong and it is worth knowing which before you buy anything.
`mech_mjcf` returns the XML that was stepped, which is where you look when a
verdict makes no sense.

**A HOLDS can be a joint leaning on something.** Read `constraint_nm` and
`obstructed` on every actuated row. A joint pressed into its travel stop or a
contact holds perfectly and reports a comfortable torque, because the
structure is carrying the rest - a shoulder once read 2% utilised while a
stop took 64% of the moment, and `hold_actuator_nm + constraint_nm` was the
closed-form answer. Move the pose a degree off the stop and the actuator
needs all of it. If the two gates disagree, this is the first thing to check:
`assembled_deg` sitting at the end of `range_deg` puts a joint on its stop as
built.

Gravity is a vector you can set: `gravity_mm_s2` on the sim spec, the same
key the static solver takes. Default `[0,0,-9810]`. Change it for a wall
mount, an incline or off-planet - do not re-derive the loads by hand.

**A failed geometry gate is a WALL, not a warning.** A build whose
interference status is INTERFERENCE, BURIED_MATES or FLOATING_MATES, or
whose sweep says COLLIDES_IN_TRAVEL, will refuse firmware runs, animation
runs and publication until it is fixed - the refusal names every defect.
Do not plan around it; repair the geometry the moment the build reports it,
while the part is still in front of you.

**Do the solids collide?** `mech_build` checks it on the real B-reps and
returns `interference` - the exact overlap volume of every pair of solids the
assembly does not join. Parts a joint connects are exempt; they meet at their
bearing. Anything else sharing volume is a machine that cannot be built and
renders perfectly.

**And does it collide ANYWHERE in its travel?** The build also returns
`sweep`: every revolute posed through its declared range (extremes plus 30°
steps, one joint at a time), the moved subtree boolean-checked against
everything that stayed put. `COLLIDES_IN_TRAVEL` is a gate failure like any
other - each collision names the joint, the angle and the pair, and the fix
is yours: shorten the link, move the joint, or narrow `range_deg` to the
travel that is actually clear. A range you declared IS a claim the machine
can go there. The swept joint's own bearing pair is judged against its home
overlap (`beyond_bearing_mm3`), so a buried shaft does not hide a folding
arm. Combined poses - two joints off home at once - are NOT covered; rebuild
in the pose that matters when one matters.

**What holds the parts on?** `mech_build` also returns `fastening`. A
`RigidJoint` is a mate in the model and NOTHING in the metal - two coincident
faces, no hole, no screw - and a face-contact mate overlaps nothing, so that
machine passes every other gate at 0 mm³ and falls into a heap.
`fastening` counts BOLT LINES: one fastener-sized bore in each part, coaxial
and collinear, measured off the B-reps. Read `parts_held_by_nothing`, and read
`holes_parent`/`holes_child` per mate - they name which side never drilled the
pattern. A sourced actuator arrives with its bolt circles already in the solid,
so cut yours to match what `mech_inspect_cad` measured rather than seating a
plain face against it. It says nothing about thread, grip length or driver
access, and a revolute held by its shaft carries no bolts by design.

**Will the parts survive?** `mech_link_stress(graph, duty, material)` turns
the bending moment the STRUCTURE carries into a stress and a safety factor.
`mech_materials` lists what it knows. First-order beam theory, not FEA - it
says what it did not check.

By default the section comes from the bounding box, which for a shelled link
overstates strength badly - section modulus goes as depth SQUARED - so those
results come back UNVERIFIED rather than as a safety factor you might
believe. **Build with `features=True`** and the runner cuts the real section
at several planes along each link, keeps the thinnest, and the stress is then
computed from measured geometry and trusted. It also validates every solid
(watertight, manifold) and reports pairs that pass within 1 mm without
touching, which is a fit problem an interference check cannot see.

Or declare the graph yourself:

```
{"root": "base", "base": "free"|"fixed",
 "bodies": [{"name","pos_mm" (WORLD),"mass_g","fromto_mm"|"size_mm",
             "radius_mm","geom_center_mm","weld_to"}],
 "joints": [{"name","parent","child","type","axis","pos_mm","actuated",
             "torque_nm","range_deg","assembled_deg","hold_deg"}]}
```

`geom_center_mm` is the offset from the body origin to the solid's centre.
A body's origin is its JOINT, so without it the mass sits on the hinge, every
lever is zero and the duty comes back 0.00 N.m. `mech_build` measures it for
you. `assembled_deg` is where `connect_to` left the joint - build123d
defaults it to the range MINIMUM, so an elbow declared `[28,152]` is built
folded at 28, and every angle you pass is on that same dial.

**5. Wire it.** A machine with no electrical layout is half a design - it is
at least as likely to fail on its loom as on its structure, and no
mechanical gate can see a BEC rated 3 A feeding 3.5 A of load. Design the
loom and SAVE it with `mech_wiring(action="save", layout=...)`:

- components: the pack (`battery`, with `voltage_v`, `capacity_wh`), each
  regulator/BEC (`output_v`, `output_a`), the `mcu` (`min_voltage_v` - the
  brownout threshold), one `motor` per actuated joint
  **whose id IS the joint name** - that is how the firmware simulation maps
  wire drops onto actuators.
- every one of those component ratings should trace to a record you sourced
  at step 2 - the MCU, the regulator and the pack through
  `mech_part_search` / `mech_part_measure` with the citation kept, the same
  loop the actuators went through. A brownout threshold from a cited record
  is a measurement; one recalled while wiring is the number that browns out
  on the bench. If the electronics were not sourced at step 2, source them
  NOW, before the layout is saved - the layout is where their numbers become
  load-bearing.
- nets: power runs with `awg` and `length_mm` MEASURED off your own
  geometry (the joint positions are in the graph - a run to the wrist is
  the sum of the link lengths it travels, not a guess), signal runs with
  their `bus`.
- currents come from `mech_power_budget`, never from memory. Wire ampacity
  and drop are then the checker's problem: read `problems`, fix, re-save.

A layout that only ever passed `mech_check_wiring` is not in the loop -
only the SAVED layout joins firmware runs, where drops come off each
motor's rail and the MCU browns out through its own loom.

**6. Say what the machine is FOR.** Every gate so far asks whether the
machine survives; none asks whether it does the job, because until a job is
declared there is nothing to fail. `mech_task(action="save", ...)` declares
one - the tool body that grasps, the workpiece to pick, where it must end
up, and how close counts. The workpiece must be a body built with
`free=True` in the spec: a body bolted to the flange is CARRIED, not picked,
and the engine refuses to score it. Saved with the project, so every run
reads one declaration.

**7. Program it and run the whole machine.** The last gate runs your
firmware on an emulated MCU against the machine, the loom and the job
together. With a task declared the run answers PLACED / DROPPED_SHORT /
STILL_HOLDING / NEVER_GRASPED, with the cycle time, the place error and the
missed grasp attempts - and carrying the workpiece LOADS the machine, so
the current it costs is real. Those are the numbers to iterate against:
change the firmware, run again, drive them down.
Write the control loop in C against the mailbox contract (mf_mailbox.h:
wait for `MF_SENSOR_MAGIC`, read `joint[]` on the CAD's dial, write
`cmd_deg[]` + `mask`, bump `seq`), then:

1. `mech_firmware_build(source=...)` - compiles it against the vendored
   board support and returns the ELF name; compile errors come back as
   `output` and are yours to fix.
2. `mech_run_firmware(firmware_file=..., spec=..., actuators=..., pack=...)`
   - the same spec `mech_simulate` takes, plus each joint's Kt / winding R /
   reduction from the datasheet you cited. The couplings are real: torque
   draws current, current sags the pack and drops the saved loom, the sagged
   rail caps what every joint can deliver, and an MCU rail under its minimum
   STOPS your control loop mid-run.
   **`torque_nm` on every actuated joint, here exactly as in mech_simulate.**
   An unrated joint runs on the 2 N·m/rad fallback servo and gravity folds
   the machine at its hinges - a replay that looks like parts falling off.
   That is never fasteners (the physics cannot detach a jointed body; screws
   are `fastening`'s gate) and never the machine: the result names the
   culprits in `unrated_joints`.

Judge it like a bench test: `tracking` per joint, `power.brownout`,
`electrical.torque_capped_ticks` (the electronics, not the rating, was the
limit), `electrical.ticks_skipped_mcu_dark` (your controller was off), and
`speed_factor`. A machine that passes statics and sim but browns out on its
first coordinated move fails HERE, and this is the only gate that sees it.

## End with the link

Every build and every `mech_project` call returns `url` - the address of this
machine in the Studio, where the person you are working for can turn it,
simulate it, read its loom and its bill of materials.

**Finish the session with it.** Volumes, safety factors and gate verdicts are
the evidence; they are not something anyone can show a colleague. A design
that ends with numbers in a transcript and no link ends with nobody looking
at the machine.

## Projects

`mech_project(action=...)` with create / describe / open / rename / list /
current / delete. A project belongs to you, so a rename moves nothing and two
projects may share a name. Pass an id when you mean a specific one. Every
build reports `project`, and a build that seems to have produced nothing is
almost always in another one.

**Say what the machine is when you create it.** `create` takes `title` and
`summary` as well as `name`:

```
mech_project(action="create", name="cobot-arm-5kg",
             title="5 kg-payload 6-DoF cobot arm",
             summary="620 mm reach on CubeMars AK-series modules, 24 V, "
                     "sized on rated torque at full extension.")
```

`name` is a label - a folder someone has to pick out of a list. The other two
are what the machine IS, and they become the headline and the description if
it is ever published, so `mech_library(action="publish")` then needs nothing
but the project. Written at the end instead, they are written by whoever is
left after the design is finished. Use `action="describe"` to revise either
one when the machine turns into something else.

## Spend turns like they cost a minute each, because they do

The arithmetic in here runs in microseconds and a CAD build takes about five
seconds. Everything else is the round trip. A design session is slow because
of how many exchanges it takes, not how much it computes.

Which is why the two levers at the top of this skill are the whole game:
**fan the independent work out** (a `part-reader` per candidate, a
`step-inspector` per module, dispatched before you need them), and **put
independent calls in one turn** rather than one after another. Everything
else below is about not spending a turn asking for something you already
have.

**`mech_build` already gates what it built.** It returns `interference` (exact
overlap of every unjointed pair of solids), `duty_self_weight`, and
`link_stress_self_weight` when you pass `material=`. Do not follow a build
with three separate calls to ask what it just told you. Those are self-weight
in the as-built pose - your real load cases and poses are still yours to run.

**Read the whole result before calling again.** `project`, `warning`,
`solids`, `graph_error`, `interference` and the gate rows are all in the build
result. Most follow-up calls are asking for something already on screen.

## Things that have cost real designs

**Booleans reset a part's Location - do them BEFORE placement, never
after.** `part += Rot(...) * solid` silently RE-LOCATES the part, and
shelling (`offset`) discards the rotation a bare `loft()` carried - in both
cases every joint defined afterwards is wrong, the build succeeds, and the
render looks fine. The rule: finish ALL solid modelling (fuse, cut, shell,
fillet) on the part at identity, THEN define joints, THEN connect. If you
must add material to a placed part, subtract instead where possible -
subtraction with a rotated tool is safe; addition is not. Cost four builds
on one arm before it was caught.

**Actuator mass sits ON its joint**, not smeared into the link it drives.
Put it in the wrong place and a six-link arm's shoulder is overstated by
25%.

**On a direct-drive machine the modules outweigh the payload.** Six 1396 g
modules against a 5 kg payload took a shoulder from 47.6 to 81.8 N.m -
across the rating of the module being evaluated. Actuator choice is a fixed
point: pick one, feed its mass back through `joint_masses_g`, re-check.

**Not every joint carries the load.** A yaw about gravity and a roll along
an arm see no gravity moment at all; only the axes perpendicular to it do.
`mech_joint_duty` returns 0.0 for those and puts the moment on the bearing
instead. Sizing all six joints of an arm for the shoulder's load buys three
actuators nobody needs.

**A part in several disjoint solids renders perfectly and cannot be made.**
`mech_build` flags it. Fix it rather than shipping it.

**One pose is not the machine.** `mech_joint_duty` solves the configuration
you hand it. Sweep the poses that matter.

## When a build fails

Read the `hints` on the result - they name the fix for the common failures.
Then **repair and try the NEXT SMALLER STEP.** Do not rewrite the assembly
because one boolean failed; that turns one bad build into five, and each one
costs a whole turn. Fix the line, rebuild the same script.

A build that produced a solid but the wrong one is a different problem, and
the result already tells you: `bbox_mm` per part, `solids` (a part in pieces
renders fine and cannot be made), `validity` with the coordinates of any open
or non-manifold edge, and `interference` with the overlap volume of every
unjointed pair. Check those before rebuilding - most follow-up calls ask for
something already on the screen.

## When a tool refuses

It names the fix. Read it and act on it - do not retry the same call
unchanged, and never paper over a refusal with an assumption. A number you
typed because a solver said no is the one number that will be wrong.
