# System Requirements Specification
## universe Flight Simulation Framework

---

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Document ID   | UNI-SRS-000                                |
| Version       | 0.1.0 (draft)                              |
| Status        | Draft                                      |
| Layer         | 0 — System                                 |
| Architecture  | FPA-SRS-000 (Fractal Partition Architecture) |
| Parent        | None (root specification for universe)     |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Applicable Documents](#2-applicable-documents)
3. [Definitions and Abbreviations](#3-definitions-and-abbreviations)
4. [System Architecture](#4-system-architecture)
5. [Multi-vehicle Operation](#5-multi-vehicle-operation)
6. [Physics and Environment](#6-physics-and-environment)
7. [GN&C Plugin Interface](#7-gnc-plugin-interface)
8. [Vehicle Definition](#8-vehicle-definition)
9. [Visualization](#9-visualization)
10. [User Interface](#10-user-interface)
11. [Distribution and Packaging](#11-distribution-and-packaging)
12. [Telemetry](#12-telemetry)
13. [Execution Lifecycle](#13-execution-lifecycle)
14. [Requirements Traceability Matrix](#14-requirements-traceability-matrix)

---

## 1. Purpose and Scope

This document defines the system-level requirements for **universe**, a modular,
multi-fidelity flight simulation framework. The framework is organized according to
the **fractal partition architecture** defined in FPA-SRS-000. This document defines
domain-specific requirements for the universe flight simulation; architectural
primitives — contracts, compositors, events, composition fragments, bus communication,
and testing structure — are defined in FPA-SRS-000 and referenced but not restated here.

The framework is intended to support algorithm development, operator training, hardware-
and software-in-the-loop validation, and conceptual design trades. It is additionally
intended as a teaching platform in which students independently implement GN&C algorithms
and vehicle visualization features against stable, well-defined interfaces.

All child specifications shall include a traceability field referencing the UNI
identifier(s) from this document that each child requirement satisfies.

**Implementation neutrality:** The requirements in this document are intended to specify
behavioral and structural contracts without prescribing implementation technology.
References to specific tools, languages, and frameworks — including Rust, Bevy, TOML,
Cargo, and `egui` — are illustrative examples of one viable realization and shall not be
read as mandating those choices. A conforming implementation may substitute equivalent
technologies provided all stated behavioral and structural requirements are satisfied.

**Out of scope:** Specific aerodynamic databases, certified flight models, and
operational mission data are outside the scope of this specification.

---

## 2. Applicable Documents

| ID            | Title                                           | Location                                     |
|---------------|-------------------------------------------------|----------------------------------------------|
| FPA-SRS-000   | Fractal Partition Architecture Requirements     | [simtheverse/fractal](https://github.com/simtheverse/fractal) `docs/design/SPECIFICATION.md` |
| FPA-CON-000   | Fractal Partition Architecture Conventions      | [simtheverse/fractal](https://github.com/simtheverse/fractal) `docs/design/CONVENTIONS.md`   |
| UNI-SRS-001   | Physics Partition Requirements                  | TBD                                          |
| UNI-SRS-002   | GN&C Partition Requirements                     | TBD                                          |
| UNI-SRS-002A  | GN&C ABI Requirements                           | TBD                                          |
| UNI-SRS-003   | Visualization Partition Requirements            | TBD                                          |
| UNI-SRS-004   | User Interface Partition Requirements           | TBD                                          |
| UNI-SRS-005   | Contract Crate Interface Specification (ICD)    | `universe-contract/docs/design/SPECIFICATION.md` |
| UNI-SRS-006   | Application Requirements                        | TBD                                          |

---

## 3. Definitions and Abbreviations

See FPA-SRS-000 for definitions of architectural terms (compositor, contract crate,
composition fragment, layer, partition, layer and partition uniformity principle,
fractal partition pattern, layer-scoped bus, relay authority, direct signal, delivery
semantic, tick lifecycle, state snapshot, event action).

| Term              | Definition                                                                 |
|-------------------|----------------------------------------------------------------------------|
| ABI               | Application Binary Interface — the binary-level contract for a shared library |
| DoF               | Degrees of Freedom                                                         |
| ECS               | Entity Component System — Bevy's data-oriented architecture                |
| Fidelity tier     | A named composition fragment within the physics domain that selects sub-model implementations by complexity level (e.g., `"low"`, `"mid"`, `"high"`). Not a distinct system primitive — fidelity tiers are named composition fragments expressed using the same mechanism available to every partition |
| GN&C              | Guidance, Navigation, and Control                                          |
| IC                | Initial Conditions                                                         |
| ISA               | International Standard Atmosphere                                          |
| NED               | North-East-Down coordinate frame                                           |
| Plugin            | A Bevy `Plugin` implementor, or a dynamically loaded shared library        |
| Scenario          | A layer 1 composition fragment describing mission-level configuration: vehicle definitions, physics sub-model selections, environment models, events, and initial conditions. A session composes one or more scenarios |
| Session           | The layer 0 composition fragment describing a simulation run: transport mode, timing parameters, active partitions, and which scenarios to compose. A session is the top-level entry point for the compositor |
| VehicleId         | A runtime-unique identifier assigned to a simulated vehicle instance       |
| VehicleTypeId     | A stable string identifier for a vehicle design (e.g., `"cessna172"`)     |
| WGS84             | World Geodetic System 1984 — standard geodetic reference frame             |

---

## 4. System Architecture

---

### UNI-001 — Four-partition Architecture

**Statement:** The system shall implement a four-partition architecture comprising:
(P1) 6-DoF physics and plant modeling, (P2) guidance, navigation, and control (GN&C)
and autonomy, (P3) three-dimensional visualization, and (P4) user interface and execution
control.

**Rationale:** Partitioning by functional domain enforces separation of concerns, allows
each domain to evolve at its own pace, and enables independent substitution of
composition presets, student implementations, and third-party components without
system-wide impact.
These four partitions form the layer 0 decomposition of the fractally partitioned
system (see FPA-001).

**Verification Expectations:**
- Pass: The system executes a complete simulation run with all four partitions active and
  exchanging data across their defined interfaces.
- Pass: Each partition resides in a distinct Rust crate in the workspace with no source
  code from another partition present in its `src/` tree.
- Fail: Any partition's implementation source code is located within another partition's
  crate.
- Fail: The system fails to produce physics output, GN&C commands, rendered frames, or
  UI interaction when any one partition is replaced with its reference stub.

---

## 5. Multi-vehicle Operation

---

### UNI-002 — Multi-vehicle Support

**Statement:** The system shall support simultaneous simulation of N ≥ 1 vehicles, where
each vehicle instance consists of an independently configured plant model and GN&C plugin
pair, and where N is bounded only by available system resources.

**Rationale:** Multi-vehicle operation is required for formation flying exercises,
cooperative autonomy assignments, and adversarial scenarios in which student aircraft
interact. It must be a first-class system property rather than a deferred extension.

**Verification Expectations:**
- Pass: A scenario TOML declaring three vehicles with distinct GN&C plugins and vehicle
  types executes without error, with each vehicle producing independent plant states.
- Pass: Removing one vehicle entry from the scenario TOML reduces the active vehicle
  count by exactly one without affecting the remaining vehicles' behavior.
- Fail: The system requires code modification (outside of configuration) to increase the
  vehicle count beyond one.

---

### UNI-003 — Vehicle Identification

**Statement:** Each simulated vehicle shall be assigned a unique, stable `VehicleId` at
spawn time that is present on all inter-partition messages pertaining to that vehicle for
the duration of its lifetime in the simulation.

**Rationale:** Message routing, Bevy ECS entity association, telemetry logging, and
inter-vehicle awareness in GN&C all require an unambiguous vehicle identifier. Absence
of a stable ID forces partitions to infer vehicle identity from message ordering or
content, which is fragile.

**Verification Expectations:**
- Pass: In a two-vehicle scenario, messages from vehicle A and vehicle B are
  distinguishable solely by `VehicleId` without examining any state field values.
- Pass: The `VehicleId` carried on a `PlantState` message matches the `VehicleId` of the
  Bevy entity created for that vehicle.
- Fail: Any inter-partition message type defined in `universe-contract` that pertains to a
  specific vehicle lacks a `VehicleId` field.
- Fail: Two active vehicles share the same `VehicleId` at any point during a simulation
  run.

---

### UNI-004 — World State Broadcast

**Statement:** The system shall publish an aggregated `WorldState` message each
simulation tick containing the `PlantState` of all currently active vehicles. This
message shall be available to all partitions via the active transport. WorldState shall
be published as an atomic, temporally consistent snapshot — all entries shall correspond
to the same completed tick (see FPA-014). A consumer shall never observe a partially
assembled WorldState regardless of transport mode.

**Rationale:** GN&C algorithms for formation flying, collision avoidance, and
cooperative tasking require awareness of peer vehicle states. A broadcast `WorldState`
provides this without requiring point-to-point subscriptions between individual GN&C
instances. Atomicity and temporal consistency are necessary to satisfy the transport
independence guarantee (FPA-004); without them, consumers under async transport
could observe vehicle states from different ticks.

**Verification Expectations:**
- Pass: In a two-vehicle scenario, the `WorldState` received by each GN&C plugin
  contains entries for both vehicles.
- Pass: When a vehicle is despawned, it is absent from the `WorldState` broadcast in
  all subsequent ticks.
- Pass: All `PlantState` entries in a single `WorldState` correspond to the same
  simulation tick.
- Fail: A GN&C plugin must subscribe to individual per-vehicle topics to obtain the
  states of peer vehicles; no aggregate is available.
- Fail: `WorldState` is broadcast at a rate different from the physics tick rate.
- Fail: A consumer reads a `WorldState` that contains `PlantState` from two different
  simulation ticks.

---

### UNI-005 — Dynamic Vehicle Lifecycle

**Statement:** The system shall support spawning and despawning of vehicle instances
during an active simulation session. Spawning shall load the specified plant model and
GN&C plugin and initialize the vehicle to provided initial conditions. Despawning shall
cleanly unload all associated resources such that, at the time the despawn operation
is reported as complete, for every dynamically loaded module associated with the
vehicle (including the GN&C plugin and any dynamically loaded plant model):
(a) within the despawned vehicle's execution context (i.e., the set of host-managed
threads, tasks, and callbacks that may execute code on behalf of that vehicle
instance), no threads are executing code from that module's binary; (b) within the
host's data structures for that vehicle instance (including any per-vehicle module
handle tables, callback registries, thread entry points, and callback lists), the host
retains no callable references (such as function pointers, callbacks, vtables, or
handles) that would allow further execution of that module's code on behalf of the
despawned vehicle; and (c) if no other active vehicle continues to use the module
(i.e.,
no other active vehicle retains callable references to it), its dynamic library has
been unloaded by the host process. When a dynamically loaded module is shared among
multiple active vehicles, the host shall ensure that the module is unloaded once it is
no longer used by any vehicle (e.g., via reference-counting or an equivalent
mechanism). Spawn and despawn requests shall be processed at tick boundaries (see
FPA-014, Phase 1). All partitions within a tick shall observe the same set of
active vehicles.

**Rationale:** Dynamic vehicle lifecycle enables scenarios in which aircraft launch,
complete their mission, and recover or are destroyed, without requiring a full simulation
restart. It also supports incremental scenario construction during interactive use.
Processing spawn and despawn at tick boundaries (FPA-014) ensures that all partitions
observe a consistent vehicle set for the entirety of each tick, regardless of transport
mode.

**Verification Expectations:**
- Pass: A vehicle spawned mid-simulation at specified initial conditions begins
  publishing `PlantState` messages on the subsequent tick.
- Pass: After despawning a vehicle, no further messages bearing its `VehicleId` appear
  on the bus and its entity is removed from the world.
- Pass: All partitions stepped within the same tick observe the same set of active
  vehicles.
- Fail: Spawning a second vehicle during a running simulation requires a restart.
- Fail: Despawning a vehicle leaves its GN&C plugin loaded in process memory.
- Fail: Under async transport, one partition observes a newly spawned vehicle in tick N
  while another partition does not, because the spawn was processed mid-tick.

---

## 6. Physics and Environment

---

### UNI-006 — Physics Sub-model Composition

**Statement:** The physics partition shall provide at minimum three named composition
presets that select specific layer 1 sub-model implementations: (a) `"low"` — flat-earth
rigid body 6-DoF equations of motion; (b) `"mid"` — WGS84 geodetic reference with
International Standard Atmosphere; (c) `"high"` — mid-tier plus wind field, atmospheric
turbulence, and full propulsion dynamics. The active preset shall be selectable per
vehicle via scenario configuration. Individual sub-model selections within a preset
shall be independently overridable (see FPA-021).

**Rationale:** Physics fidelity is not a special-purpose mechanism — it is the fractal
partition pattern's compositor applied within the physics domain. Each fidelity level is
a named composition of independently replaceable layer 1 partitions (gravity model,
atmosphere model, aerodynamics model, propulsion model). Low-complexity compositions
minimize computational cost for rapid GN&C prototyping. Mid-complexity compositions
introduce geodetic and atmospheric realism for navigation algorithm validation.
High-complexity compositions support aerodynamic and propulsion sensitivity studies. No
single composition satisfies all use cases.

**Verification Expectations:**
- Pass: Each of the three named composition presets executes a straight-and-level flight
  scenario without numerical divergence over a 60-second simulation.
- Pass: Two vehicles in the same scenario may simultaneously operate with different
  composition presets and produce independent results.
- Pass: The active composition preset is changed between runs by modifying the scenario
  TOML only.
- Pass: An individual sub-model within a preset is overridden via inline scenario
  configuration without affecting the other sub-models selected by the preset.
- Fail: Any composition preset shares implementation source files with another preset in
  a way that prevents one from being compiled without the other (see feature flag
  requirement UNI-021).
- Fail: Changing the physics composition requires a mechanism distinct from the
  compositor pattern used at layer 0 or in other partitions.

---

### UNI-007 — Environment Model Composability

**Statement:** The system shall implement environment effects (standard atmosphere, wind
field, atmospheric turbulence, terrain elevation, and gravity) as independently
composable models. Each model shall be individually enabled, disabled, or replaced via
scenario configuration without affecting the behavior of other active environment models.

**Rationale:** Environment models are layer 1 partitions within the physics domain.
Their independent composability is a direct application of the fractal partition
pattern (FPA-001): each sub-model is independently replaceable via the same
contract/compositor structure used at the system level. This allows the instructor or
researcher to isolate the effect of individual environment disturbances, add new
environment models without modifying existing ones, and assign environment model
implementations to student teams independently.

**Verification Expectations:**
- Pass: A scenario with only the ISA atmosphere model enabled produces the same altitude-
  dependent air density as a scenario with ISA atmosphere and Dryden turbulence enabled,
  when turbulence state is zeroed.
- Pass: Removing a single model from the `[environment].models` list in the scenario
  TOML removes its effect from the simulation without altering the output of any
  remaining models.
- Pass: A novel environment model not present in the original codebase is added and
  activated without modifying any existing environment model source file.
- Fail: Disabling turbulence requires modifying the atmosphere model's source code or
  recompiling with a different set of `cfg` flags not governed by the scenario TOML.

---

### UNI-008 — Configurable Simulation Rate

**Statement:** The physics partition update rate shall be configurable at runtime via the
`[sim].dt` field in the scenario TOML. The visualization and user interface partitions
shall execute at their own independent update rates, decoupled from the physics rate.

**Rationale:** High-fidelity physics compositions often demand high update rates (≥ 1 kHz) that are
unnecessary for rendering and UI (typically 60 Hz). Decoupling these rates avoids
unnecessary computational load on the rendering thread and supports faster-than-real-
time batch execution without affecting visualization frame rate.

**Verification Expectations:**
- Pass: Setting `dt = 0.001` (1 kHz) and `dt = 0.01` (100 Hz) in otherwise identical
  scenarios both execute without error, with the higher-rate scenario producing a
  proportionally larger number of physics steps per wall-clock second.
- Pass: The Bevy rendering loop maintains a consistent frame rate independent of the
  configured `dt` value.
- Fail: The visualization frame rate degrades proportionally to physics update rate,
  indicating the render loop is blocked on physics completion.

---

## 7. GN&C Plugin Interface

---

### UNI-009 — Stable GN&C ABI

**Statement:** The GN&C partition interface shall be defined as a stable C-compatible
application binary interface (ABI) in the `sim-gnc-abi` crate. GN&C plugins shall be
compiled as dynamic shared libraries (`.so` / `.dll` / `.dylib`) and loaded at runtime.
The ABI shall use only `#[repr(C)]` types at the boundary.

**Rationale:** A stable C ABI allows student GN&C implementations to be compiled and
submitted as binary artifacts, decouples plugin compilation from simulator compilation,
and permits implementations in any language capable of producing a C-compatible shared
library.

**Verification Expectations:**
- Pass: A GN&C plugin compiled against `sim-gnc-abi` version N loads and executes
  correctly in a simulator compiled against `sim-gnc-abi` version N without recompiling
  the simulator.
- Pass: All types crossing the GN&C ABI boundary are annotated with `#[repr(C)]` and
  contain no Rust-specific types (`Box`, `Vec`, `String`, `&str`, trait objects) as
  direct fields.
- Fail: Loading a valid GN&C `.so` file fails due to symbol name mangling or ABI
  incompatibility between compiler versions on the same platform.

---

### UNI-010 — GN&C Plugin Isolation

**Statement:** A GN&C plugin implementation shall require no compile-time or runtime
dependency on any crate other than `sim-gnc-abi`. The plugin shall receive all
information required for its operation through the ABI function signature.

**Rationale:** Plugin isolation minimizes the knowledge burden on students, prevents
accidental coupling to simulator internals, and ensures that a plugin built in isolation
integrates correctly without environment-specific dependency resolution.

**Verification Expectations:**
- Pass: A GN&C plugin crate with only `sim-gnc-abi` in its `[dependencies]` compiles to
  a loadable shared library and executes correctly in the simulator.
- Pass: The plugin receives own vehicle state and world state entirely through its ABI
  function parameters without calling back into the simulator.
- Fail: The plugin requires a `bevy`, `universe-contract`, `sim-physics`, or `tokio` dependency
  to compile.

---

### UNI-011 — GN&C Peer State Access

**Statement:** The GN&C ABI function signature shall provide each plugin with: its own
`VehicleId`, its own `PlantState`, the current `WorldState` (containing all active
vehicle states), and the elapsed simulation time step `dt`. The plugin shall return a
`GNCCommand`.

**Rationale:** Formation flying, cooperative autonomy, and adversarial scenarios require
each GN&C instance to observe the state of peer vehicles. Providing `WorldState` through
the ABI function signature eliminates the need for plugins to manage their own
subscriptions.

**Verification Expectations:**
- Pass: A GN&C plugin implementing a formation-keeping algorithm correctly reads the
  position of a peer vehicle from `WorldState` and produces commands that reduce
  separation error over time.
- Pass: A plugin that ignores `WorldState` and operates only on `own_state` produces
  correct single-vehicle control without error.
- Fail: The GN&C ABI function signature requires the plugin to perform a separate
  network call or file read to obtain peer vehicle states.

---

## 8. Vehicle Definition

---

### UNI-012 — Vehicle Manifest

**Statement:** Each vehicle type shall be described by a `VehicleManifest` exported from
its shared library. The manifest shall include: a unique `VehicleTypeId`, a physics model
factory function, a visual descriptor containing mesh asset path and control surface bone
mapping, and an optional default GN&C plugin path.

**Rationale:** Bundling physics and visual descriptors into a single manifest ensures
that a student aircraft submission is self-contained: the simulator derives all
vehicle-type-specific behavior and rendering parameters from one artifact, eliminating
configuration fragmentation across multiple files.

**Verification Expectations:**
- Pass: Loading a vehicle manifest `.so` file and invoking the `vehicle_manifest()`
  symbol returns a populated `VehicleManifest` from which the compositor can instantiate
  a physics model and locate mesh assets without additional configuration.
- Pass: Two distinct vehicle types loaded simultaneously each render with their
  respective meshes and surface mappings without cross-contamination.
- Fail: Adding a new vehicle type requires modifying the `MeshPlugin` source code to
  handle the new type explicitly.

---

### UNI-013 — Visual Plant State Interface

**Statement:** The physics plant model shall implement a `VizStateProvider` trait
defined in `universe-contract`, producing a `VehiclePlantVizState` message each tick containing
vehicle-type-specific visual state (e.g., control surface deflection angles, gear
position, rotor RPM). This message shall be published on the bus and mirrored into a
Bevy component for use by the visualization partition.

**Rationale:** The plant model is the authoritative source of actuator and surface states.
Deriving visual animation from GN&C commands or approximations introduces lag and
inaccuracy, particularly when actuator dynamics or rate limits are modeled. A dedicated
visual state output ensures rendering fidelity is bounded only by physics fidelity.

**Verification Expectations:**
- Pass: In a scenario with aileron input commanded, the rendered aircraft mesh deflects
  the aileron bone by an angle consistent with the value in `FixedWingVizState`.
- Pass: The `VehiclePlantVizState` value in the Bevy component matches the value
  produced by `VizStateProvider::viz_state()` on the same tick.
- Fail: The visualization partition reads `GNCCommand` fields to determine control
  surface deflection angles for rendering.

---

## 9. Visualization

---

### UNI-014 — Visualization Modularity

**Statement:** The visualization partition shall implement each visual feature as an
independently addable and removable Bevy plugin: (a) vehicle mesh and surface animation,
(b) environment rendering (sky, terrain, horizon), (c) flight path trail, and
(d) data overlays and HUD. Each plugin shall be enabled or disabled via scenario
configuration without modifying any other plugin's source code.

**Rationale:** Visualization plugins are layer 1 partitions within the visualization
domain — the fractal partition pattern (FPA-001) applied to rendering. Assigning
each visual feature to an independently replaceable plugin allows visualization student
teams to work in isolation, enables features to be enabled selectively to match system
performance targets, and prevents merge conflicts between teams working on different
features simultaneously.

**Verification Expectations:**
- Pass: Removing `"trail"` from `[viz].plugins` in the scenario TOML disables the
  flight path ribbon for all vehicles without affecting mesh rendering, environment
  rendering, or the HUD.
- Pass: A new visualization feature is implemented as a Bevy `Plugin` and activated by
  adding its name to `[viz].plugins` without modifying the source of any existing
  visualization plugin.
- Fail: Disabling one visualization plugin causes a compile error or runtime panic in
  any other visualization plugin.

---

### UNI-015 — Bevy ECS Ownership of Visualization and UI

**Statement:** Partitions P3 (visualization) and P4 (user interface) shall be
implemented using the Bevy Entity Component System. Each simulated vehicle shall
correspond to a Bevy entity. Plant state and GN&C command data shall be mirrored into
Bevy components and resources each simulation tick by an adapter system in `sim-app`.

**Rationale:** Bevy's ECS scales naturally to N vehicles without special-casing,
provides a uniform data model for both rendering and UI systems, and ensures that
visualization and UI plugins share a consistent view of simulation state without
maintaining independent subscriptions to the message bus.

**Verification Expectations:**
- Pass: In a three-vehicle scenario, three distinct Bevy entities exist with distinct
  `VehicleId` components, each carrying independent `PlantState` component values.
- Pass: Bevy visualization and UI systems obtain vehicle state exclusively by querying
  Bevy components and resources; no direct bus subscription occurs within `sim-viz` or
  `sim-ui` source files.
- Fail: Any visualization or UI plugin calls `SimBus::subscribe()` directly.

---

## 10. User Interface

---

### UNI-016 — UI Execution Control

**Statement:** The user interface partition shall provide interactive controls for at
minimum the following simulation lifecycle operations: start, pause, resume, stop, and
reset to initial conditions. These controls shall emit `ExecutionStateRequest` messages
on the simulation bus. The UI is one of potentially many sources of execution state
transition requests; the orchestrator (`sim-app`) is the sole authority for evaluating
and applying transitions (see UNI-025).

**Rationale:** Execution control is the minimum viable interactive capability. Without
it, the simulator cannot be used in classroom or training contexts where an instructor
must interrupt or restart a run in response to student input or anomalous behavior.
Decoupling the UI from direct state mutation ensures that execution state transitions
from all sources (UI, partition events, scripted scenarios) flow through a single
arbitration point.

**Verification Expectations:**
- Pass: Activating pause via the UI emits an `ExecutionStateRequest::Pause` that the
  orchestrator applies within one physics tick, halting physics integration and GN&C
  updates; visualization continues rendering the current frozen state.
- Pass: Activating reset returns all vehicle states to their scenario-defined initial
  conditions and clears telemetry history.
- Pass: An `ExecutionStateRequest::Stop` emitted by a partition event handler (not the
  UI) produces the same execution state change as the UI stop button.
- Fail: Any execution control operation requires more than one physics tick to take
  effect, as measured by the difference in published `PlantState` timestamps before
  and after the operation.
- Fail: The UI partition directly mutates the execution state rather than emitting a
  request on the bus.

---

### UNI-017 — UI Extensibility

**Statement:** The user interface partition shall implement each functional panel
(execution control, parameter tuning, scenario and waypoint editing, telemetry recording)
as an independently addable Bevy plugin. Additional UI panels shall be addable via
scenario configuration without modifying existing panel source files.

**Rationale:** UI panels are layer 1 partitions within the UI domain, mirroring
the visualization modularity requirement — both are instances of the fractal partition
pattern (FPA-001) applied within their respective layer 0 partitions. This allows
UI features to be developed, assigned, and graded independently, and allows laboratories
embedding the framework to add domain-specific control panels without forking the core
UI codebase.

**Verification Expectations:**
- Pass: A new UI panel implemented as a `bevy_egui` plugin is activated by adding its
  identifier to `[ui].panels` in the scenario TOML without modifying any existing UI
  panel source file.
- Fail: Adding a new UI panel requires modification to the `ExecutionPanel` or
  `ParamsPanel` source files.

---

## 11. Distribution and Packaging

---

### UNI-018 — Implementation Language

**Statement:** All partition library crates shall be implemented in the Rust programming
language. The `sim-gnc-abi` boundary types shall use C-compatible (`#[repr(C)]`) layouts
to permit GN&C plugin implementations in any language capable of producing a
C-compatible shared library.

**Rationale:** A uniform Rust stack provides memory safety, a unified build system, and
direct Bevy integration across all partitions. C ABI compatibility at the GN&C boundary
preserves the flexibility for students to submit implementations in C, C++, Python
(via ctypes/cffi), or other languages, broadening accessibility.

**Verification Expectations:**
- Pass: A GN&C plugin implemented in C, compiled to a shared library, and loaded by the
  simulator produces correct `GNCCommand` outputs for a given `PlantState` input.
- Pass: All partition crates compile under `cargo build --workspace` without non-Rust
  build steps (excluding optional C ABI plugin compilation by students).
- Fail: Any partition crate requires a `build.rs` that invokes a C or C++ compiler for
  its own source files.

---

### UNI-019 — Crate Publishability

**Statement:** The following crates shall be publishable to crates.io or a compatible
Cargo registry: `universe-contract`, `sim-gnc-abi`, `sim-physics`, `sim-gnc`, `sim-viz`,
`sim-ui`, and `universe`. The `sim-app` binary target shall be published as part of
the `universe` crate. Internal tooling and scenario runners not intended for external
use shall set `publish = false`.

**Rationale:** Publishing partition crates independently allows other laboratories to
take a dependency on specific partitions (e.g., `universe-contract` for interface compatibility,
`sim-gnc-abi` for student submissions) without depending on the full framework. Publishing
`universe` allows the complete simulator to be embedded in third-party applications.

**Verification Expectations:**
- Pass: `cargo publish --dry-run` succeeds for each listed crate without errors related
  to missing metadata, path dependencies, or unpublished local dependencies.
- Pass: Each listed crate declares `name`, `version`, `description`, `license`, and
  `repository` fields either directly or via workspace inheritance.
- Fail: Any listed crate contains a path dependency that is not also published or
  resolvable via the same registry.

---

### UNI-020 — Independent GN&C ABI Versioning

**Statement:** The `sim-gnc-abi` crate shall be versioned independently of all other
crates in the workspace. Its version shall follow semantic versioning strictly: any
change to an exported type or function signature shall increment the major version. Minor
and patch versions shall be reserved for documentation and non-breaking additions.

**Rationale:** `sim-gnc-abi` is the crate upon which student submissions depend. An
unannounced breaking change invalidates all previously compiled student `.so` files.
Independent versioning makes compatibility commitments explicit and allows students to
pin a stable version for the duration of an assignment.

**Verification Expectations:**
- Pass: A GN&C plugin compiled against `sim-gnc-abi` version 1.0.0 loads without error
  in a simulator linked against `sim-gnc-abi` version 1.x.y for any x and y.
- Pass: A change to any `#[repr(C)]` struct field in `sim-gnc-abi` is accompanied by a
  major version increment.
- Fail: The `sim-gnc-abi` version is updated in lockstep with `universe-contract` or `sim-physics`
  for releases that do not affect the ABI.

---

### UNI-021 — Headless Feature Flag

**Statement:** The `universe` crate shall expose a `headless` Cargo feature that,
when enabled, excludes the visualization and user interface partitions from compilation.
The `default` features shall include visualization and UI. Physics and GN&C shall always
be compiled regardless of feature selection.

**Rationale:** Headless operation is required for server-side batch simulation, Monte
Carlo analysis, and continuous integration test environments where a display is
unavailable. Gating viz and UI behind a feature flag avoids linking Bevy's rendering
stack in contexts where it is unnecessary.

**Verification Expectations:**
- Pass: `cargo build --no-default-features --features headless` compiles successfully in
  a CI environment with no display server present.
- Pass: A headless simulation run produces identical `PlantState` telemetry output to a
  headed run with the same scenario and random seed.
- Fail: Enabling the `headless` feature disables physics updates or GN&C execution.

---

### UNI-022 — Embedding Interface

**Statement:** The `universe` crate shall expose a library interface sufficient for
use as a partition within a larger workflow. Universe shall not assume it owns the
top-level process or entry point. The library interface shall not require callers to
depend on universe's internal partition crates. Universe shall support being invoked
after external pre-processing (e.g., configuration generation, scenario assembly) and
shall produce outputs consumable by external post-processing (e.g., telemetry analysis,
report generation) without requiring those stages to run within universe's process
model.

**Rationale:** The fractal partition pattern applies outward as well as inward: just as
universe's internal partitions are independently replaceable within universe, universe
itself shall be independently replaceable within a larger system. A laboratory's
automation pipeline may generate composition fragments, invoke universe as one stage
in a multi-stage workflow, and consume its outputs — treating universe as a partition
within its own layer 0. The same replaceability guarantee universe provides to its own
sub-partitions shall hold at this outer boundary: swapping universe for an alternative
simulation engine requires no changes to the host program beyond the dependency and
invocation.

**Verification Expectations:**
- Pass: Universe's library interface accepts a session fragment, executes the session,
  and returns control to the caller without retaining ownership of the process lifecycle.
- Pass: Universe's library interface is usable with only `universe` as a dependency;
  internal partition crates (`universe-contract`, `sim-physics`, `sim-gnc`, `sim-viz`, `sim-ui`)
  are not required by callers.
- Pass: Universe's library interface does not require callers to initialize
  framework-specific infrastructure (e.g., an ECS world, a rendering context) that
  universe manages internally.
- Fail: Universe's library interface assumes it is the process entry point or prevents
  the caller from performing work before or after the simulation session.

---

## 12. Telemetry

---

### UNI-023 — Telemetry Recording

**Statement:** The system shall support recording of `PlantState`, `GNCCommand`, and
`EnvState` messages for all active vehicles to a persistent file during simulation
execution. Recording shall be activatable and deactivatable without restarting the
simulation.

**Rationale:** Recorded telemetry is the primary artifact for post-run analysis,
grading student GN&C submissions, and debugging anomalous behavior. The ability to
toggle recording during a run allows targeted capture of specific flight phases without
accumulating unnecessary data.

**Verification Expectations:**
- Pass: A telemetry file produced during a simulation run, when parsed, contains
  time-stamped records of `PlantState` for each vehicle at each recorded tick.
- Pass: Recording activated at t=10s and deactivated at t=20s produces a file
  containing records only within that interval.
- Fail: Activating recording introduces a measurable degradation (> 5%) in physics
  update rate on the target development hardware.

---

### UNI-024 — Telemetry Playback

**Statement:** The system shall support playback of a recorded telemetry file through
the visualization partition. During playback, vehicle state shall be driven by the
recorded `PlantState` values rather than live physics integration. All active
visualization plugins shall render the playback state identically to a live run.

**Rationale:** Post-run visualization of recorded telemetry is essential for student
debrief, algorithm performance review, and report generation. Playback through the same
visualization stack ensures that recorded and live views are visually consistent.

**Verification Expectations:**
- Pass: A telemetry file recorded during a live run, when played back, produces
  rendered vehicle positions and orientations visually indistinguishable from a
  recording of the live visualization.
- Pass: Playback can be paused, scrubbed forward, and scrubbed backward through the
  recorded time range.
- Fail: Playback mode activates physics integration, causing vehicle state to diverge
  from the recorded trajectory.

---

## 13. Execution Lifecycle

---

### UNI-025 — Simulation Lifecycle State Machine

**Statement:** The system-level contract crate (`universe-contract`) shall define a shared state
machine (FPA-006) for simulation lifecycle control with at minimum start, pause, resume,
stop, and reset semantics. The state machine shall be observable by all partitions and
modifiable only through typed bus requests arbitrated by the orchestrator. Execution
state transitions shall be available as event actions (FPA-029) so that pause points,
safety stops, and phase transitions can be expressed declaratively in composition
fragments. The detailed interface — states, transitions, request types, event action
identifiers, and conflict resolution policy — shall be specified in the `universe-contract`
interface specification (UNI-SRS-005).

**Rationale:** Interactive flight simulation requires all four partitions to agree on
whether the simulation is running, paused, or stopped. Multiple sources — UI controls,
scripted events, limit exceedance handlers — may request transitions simultaneously.
Defining the lifecycle as a shared state machine (FPA-006) ensures that transitions are
arbitrated through a single owner, that all partitions observe a consistent state, and
that the mechanism follows the same pattern available at every layer. Deferring the
detailed interface to `universe-contract`'s specification keeps this document functional rather
than prescriptive — a reimplementation of `universe-contract` may define different states or
conflict rules provided it satisfies the lifecycle semantics required here.

**Verification Expectations:**
- Pass: `universe-contract` exports a simulation lifecycle state machine with start, pause,
  resume, stop, and reset operations, implemented using the shared state machine
  synchronization pattern (FPA-006).
- Pass: All four partitions observe the same execution state value; no partition
  maintains a private copy.
- Pass: A partition requesting a lifecycle transition emits a typed request on the bus;
  the orchestrator arbitrates and applies or rejects it within one tick.
- Pass: Lifecycle transitions are available as event actions usable in composition
  fragments with both time-triggered and condition-triggered events.
- Fail: Execution state is modified by direct mutation rather than bus-mediated requests.
- Fail: Lifecycle transition actions require partition source code modifications rather
  than configuration.

---

### UNI-026 — Fail-fast Fault Handling

**Statement:** The system shall not configure fallback implementations for any
partition. When any partition faults during execution, the error shall propagate through
the compositor chain (FPA-011) to the orchestrator, which shall stop the simulation run
with a diagnostic identifying the faulting partition, its layer depth, and the operation
that faulted.

**Rationale:** Universe is a teaching and development platform. Hiding failures behind
fallback implementations would obscure bugs that students and developers need to see.
FPA-011 permits fallback as an architectural option; universe chooses fail-fast as a
domain policy. A structural failure, a panicking GN&C plugin, or a physics sub-model
error should halt the simulation immediately with a clear diagnostic rather than
continuing in a degraded state that may produce subtly incorrect results.

**Verification Expectations:**
- Pass: A partition returning an error from `step()` causes the orchestrator to stop the
  run with a diagnostic identifying the faulting partition.
- Pass: A sub-partition panicking during execution causes the compositor to catch the
  panic, wrap it with context, and propagate it to the orchestrator.
- Pass: No composition fragment in the default configuration specifies a fallback
  implementation for any partition.
- Fail: A partition fault is absorbed and the simulation continues with missing or
  degraded output.

---

## 14. Requirements Traceability Matrix

| ID       | Title                              | Allocated To              | Traces to FPA             |
|----------|------------------------------------|---------------------------|---------------------------|
| UNI-001  | Four-partition Architecture        | UNI-SRS-005, UNI-SRS-006   | FPA-001                   |
| UNI-002  | Multi-vehicle Support              | UNI-SRS-006                | —                         |
| UNI-003  | Vehicle Identification             | UNI-SRS-005                | —                         |
| UNI-004  | World State Broadcast              | UNI-SRS-005                | FPA-004, FPA-014          |
| UNI-005  | Dynamic Vehicle Lifecycle          | UNI-SRS-006                | FPA-014                   |
| UNI-006  | Physics Sub-model Composition      | UNI-SRS-001                | FPA-001, FPA-021          |
| UNI-007  | Environment Model Composability    | UNI-SRS-001                | FPA-001                   |
| UNI-008  | Configurable Simulation Rate       | UNI-SRS-001, UNI-SRS-006   | —                         |
| UNI-009  | Stable GN&C ABI                    | UNI-SRS-002A               | —                         |
| UNI-010  | GN&C Plugin Isolation              | UNI-SRS-002A               | —                         |
| UNI-011  | GN&C Peer State Access             | UNI-SRS-002A               | —                         |
| UNI-012  | Vehicle Manifest                   | UNI-SRS-005, UNI-SRS-006   | —                         |
| UNI-013  | Visual Plant State Interface       | UNI-SRS-001, UNI-SRS-003   | —                         |
| UNI-014  | Visualization Modularity           | UNI-SRS-003                | FPA-001                   |
| UNI-015  | Bevy ECS Ownership of Visualization and UI | UNI-SRS-003, UNI-SRS-004 | —                    |
| UNI-016  | UI Execution Control               | UNI-SRS-004                | FPA-006                   |
| UNI-017  | UI Extensibility                   | UNI-SRS-004                | FPA-001                   |
| UNI-018  | Implementation Language            | UNI-SRS-001 through 006    | —                         |
| UNI-019  | Crate Publishability               | UNI-SRS-006                | —                         |
| UNI-020  | Independent GN&C ABI Versioning    | UNI-SRS-002A               | —                         |
| UNI-021  | Headless Feature Flag              | UNI-SRS-006                | —                         |
| UNI-022  | Embedding Interface                | UNI-SRS-006                | —                         |
| UNI-023  | Telemetry Recording                | UNI-SRS-004                | —                         |
| UNI-024  | Telemetry Playback                 | UNI-SRS-003, UNI-SRS-004   | —                         |
| UNI-025  | Simulation Lifecycle State Machine | UNI-SRS-005, UNI-SRS-006   | FPA-006, FPA-029          |
| UNI-026  | Fail-fast Fault Handling           | UNI-SRS-006                | FPA-011                   |
