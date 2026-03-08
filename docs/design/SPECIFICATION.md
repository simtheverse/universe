# System Requirements Specification
## universe Flight Simulation Framework

---

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Document ID   | TF-SRS-000                                 |
| Version       | 0.1.0 (draft)                              |
| Status        | Draft                                      |
| Layer         | 0 — System                                 |
| Parent        | None (root specification)                  |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Applicable Documents](#2-applicable-documents)
3. [Definitions and Abbreviations](#3-definitions-and-abbreviations)
4. [System Architecture](#4-system-architecture)
5. [Inter-partition Communication](#5-inter-partition-communication)
6. [Multi-vehicle Operation](#6-multi-vehicle-operation)
7. [Physics and Environment](#7-physics-and-environment)
8. [GN&C Plugin Interface](#8-gnc-plugin-interface)
9. [Vehicle Definition](#9-vehicle-definition)
10. [Visualization](#10-visualization)
11. [User Interface](#11-user-interface)
12. [Configuration and Composition](#12-configuration-and-composition)
13. [Distribution and Packaging](#13-distribution-and-packaging)
14. [Telemetry](#14-telemetry)
15. [Events](#15-events)
16. [Verification and Traceability](#16-verification-and-traceability)
17. [Requirements Traceability Matrix](#17-requirements-traceability-matrix)

---

## 1. Purpose and Scope

This document defines the system-level requirements for **universe**, a modular,
multi-fidelity flight simulation framework. It establishes the behavioral and structural
contracts that all partitions must satisfy, and serves as the parent document to which
all partition-level specifications trace. The framework is organized according to the
**fractal partition pattern**: the system is decomposed into layers (layer 0 at the
system level, layer 1 at the partition level) and partitions at each layer, with each
partition applying the same structural primitives — contracts, events, configuration,
and composition — regardless of its position in the hierarchy. This layer and partition
uniformity principle ensures that constructs learned at one level apply identically at
every other level.

The framework is intended to support algorithm development, operator training, hardware-
and software-in-the-loop validation, and conceptual design trades. It is additionally
intended as a teaching platform in which students independently implement GN&C algorithms
and vehicle visualization features against stable, well-defined interfaces.

All child specifications shall include a traceability field referencing the SIM-SYS
identifier(s) from this document that each child requirement satisfies.

**Implementation neutrality:** The requirements in this document are intended to specify
behavioral and structural contracts without prescribing implementation technology.
References to specific tools, languages, and frameworks — including Rust, Bevy, TOML,
Cargo, and `egui` — are illustrative examples of one viable realization and shall not be
read as mandating those choices. An conforming implementation may substitute equivalent
technologies provided all stated behavioral and structural requirements are satisfied.

**Out of scope:** Specific aerodynamic databases, certified flight models, and
operational mission data are outside the scope of this specification.

---

## 2. Applicable Documents

| ID            | Title                                           | Location                                     |
|---------------|-------------------------------------------------|----------------------------------------------|
| TF-SRS-001    | Physics Partition Requirements                  | `sim-physics/docs/design/SPECIFICATION.md`   |
| TF-SRS-002    | GN&C Partition Requirements                     | `sim-gnc/docs/design/SPECIFICATION.md`       |
| TF-SRS-002A   | GN&C ABI Requirements                           | `sim-gnc-abi/docs/design/SPECIFICATION.md`   |
| TF-SRS-003    | Visualization Partition Requirements            | `sim-viz/docs/design/SPECIFICATION.md`       |
| TF-SRS-004    | User Interface Partition Requirements           | `sim-ui/docs/design/SPECIFICATION.md`        |
| TF-SRS-005    | Core Interface Requirements                     | `sim-core/docs/design/SPECIFICATION.md`      |
| TF-SRS-006    | Application Requirements                        | `sim-app/docs/design/SPECIFICATION.md`       |

---

## 3. Definitions and Abbreviations

| Term              | Definition                                                                 |
|-------------------|----------------------------------------------------------------------------|
| ABI               | Application Binary Interface — the binary-level contract for a shared library |
| Compositor        | A component that selects and assembles implementations at startup          |
| Contract crate    | A Rust crate that defines traits and data types but contains no implementation |
| DoF               | Degrees of Freedom                                                         |
| ECS               | Entity Component System — Bevy's data-oriented architecture                |
| Composition preset | A named, reusable scenario fragment that selects a specific set of layer 1 partition implementations within a domain. Applicable to any fractally partitioned domain (physics, visualization, GN&C), not only physics. See SIM-SYS-025 |
| Fidelity tier     | A composition preset within the physics domain that selects sub-model implementations by complexity level (e.g., `"low"`, `"mid"`, `"high"`). Not a distinct system primitive — fidelity tiers are an emergent application of the fractal partition pattern's compositor mechanism |
| GN&C              | Guidance, Navigation, and Control                                          |
| IC                | Initial Conditions                                                         |
| INCOSE            | International Council on Systems Engineering                               |
| ISA               | International Standard Atmosphere                                          |
| Fractal partition pattern | The architectural principle that the system is decomposed into layers and partitions, where each partition at every layer applies the same contract/implementation/compositor structure and the same event, configuration, and communication primitives as the system level. Named for the self-similarity of structure at every scale. See SIM-SYS-004 |
| Layer             | A level in the system's hierarchical decomposition. Layer 0 is the system level; layer 1 is the partition level. The fractal partition pattern applies at every layer: each uses the same structural primitives (contracts, events, composition) as the layer above it |
| Layer and partition uniformity principle | The defining property of the fractal partition pattern: structural primitives (contracts, events, configuration, composition) are identical in kind across all layers and partitions. A construct available at layer 0 is available in the same form at layer 1 and beyond |
| NED               | North-East-Down coordinate frame                                           |
| Partition         | A functional subdivision of the system at a given layer. At layer 0, the four top-level partitions (physics, GN&C, visualization, UI). At layer 1, sub-components within a partition (e.g., atmosphere model, navigation estimator). Each partition is independently replaceable provided it conforms to its layer's interface contracts |
| Plugin            | A Bevy `Plugin` implementor, or a dynamically loaded shared library        |
| Scenario          | A TOML file fully describing a simulation run                              |
| VehicleId         | A runtime-unique identifier assigned to a simulated vehicle instance       |
| VehicleTypeId     | A stable string identifier for a vehicle design (e.g., `"cessna172"`)     |
| WGS84             | World Geodetic System 1984 — standard geodetic reference frame             |

---

## 4. System Architecture

---

### SIM-SYS-001 — Four-partition Architecture

**Statement:** The system shall implement a four-partition architecture comprising:
(P1) 6-DoF physics and plant modeling, (P2) guidance, navigation, and control (GN&C)
and autonomy, (P3) three-dimensional visualization, and (P4) user interface and execution
control.

**Rationale:** Partitioning by functional domain enforces separation of concerns, allows
each domain to evolve at its own pace, and enables independent substitution of
composition presets, student implementations, and third-party components without
system-wide impact.
These four partitions form the layer 0 decomposition of the fractally partitioned
system (see SIM-SYS-004).

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

### SIM-SYS-002 — Partition Independence

**Statement:** Each partition shall be independently replaceable at the crate level
without requiring modification to the source code of any other partition, provided the
replacement implementation conforms to the inter-partition interface contracts defined in
`sim-core`.

**Rationale:** Independent replaceability at every layer is the defining structural
guarantee of a fractally partitioned system. At layer 0, it is the primary mechanism by
which the framework accommodates student GN&C submissions, student visualization
contributions, and laboratory-specific physics models, without requiring those
contributors to understand or modify the broader system.

**Verification Expectations:**
- Pass: An alternative implementation of any single partition is substituted in `sim-app`
  and the system compiles and executes without modifying source files in any other
  partition crate.
- Pass: The substitution requires changes only to `sim-app` configuration or dependency
  declarations.
- Fail: Substituting one partition requires editing source code in any other partition.
- Fail: The compiler reports unresolved imports from a replaced partition's internal
  modules in any other partition.

---

### SIM-SYS-003 — Inter-partition Interface Ownership

**Statement:** All inter-partition data structures and behavioral contracts shall be
defined exclusively within the `sim-core` crate. No partition shall import types or
traits directly from another partition's crate.

**Rationale:** Centralizing interface ownership in a single contract crate prevents
circular dependencies, enforces the direction of coupling, and provides a single source
of truth for interface evolution and versioning.

**Verification Expectations:**
- Pass: A dependency graph of the workspace shows no direct edges between partition
  crates other than through `sim-core` or `sim-gnc-abi`.
- Pass: `cargo tree` for each partition crate lists no other partition crate as a
  direct or transitive dependency (except `sim-core` and `sim-gnc-abi`).
- Fail: Any partition crate's `Cargo.toml` lists another partition crate under
  `[dependencies]`.

---

### SIM-SYS-004 — Fractal Partition Pattern

**Statement:** The system shall apply the **fractal partition pattern** at every level
of its decomposition. At layer 0 (system level), the four top-level partitions use
contract/implementation/compositor structure with shared interface types in `sim-core`.
At layer 1 (partition level), the internal structure of each partition shall apply the
same contract/implementation/compositor pattern, such that sub-components within a
partition (e.g., aerodynamic model, atmosphere model, navigation estimator) are each
independently replaceable via trait objects without modifying their siblings. The
**layer and partition uniformity principle** shall hold: the same structural
primitives — contracts, events (see SIM-SYS-035), configuration, and composition — shall
be identical in kind at every layer.

**Rationale:** The fractal partition pattern ensures that the modularity, event handling,
and composition mechanisms proven at the system level are available in the same form
within each partition. Like a fractal, zooming into any partition reveals the same
contract/compositor/event structure as the system as a whole. This maximizes the surface
area of independent replaceability, allows composition to be adjusted at the sub-model
level (including fidelity selection via composition presets), and allows instructors to
assign isolated sub-components to student teams — all
using the same constructs they encounter at the system level.

**Verification Expectations:**
- Pass: Within the physics partition, substituting the atmosphere sub-model with an
  alternative implementation requires no changes to the aerodynamics, propulsion, or
  gravity sub-model source files.
- Pass: Each partition contains a `contracts/` module or equivalent defining internal
  sub-component traits that are imported by implementations but not by callers outside
  the partition.
- Pass: Events defined at layer 1 (within a partition) use the same definition schema
  and trigger types as events defined at layer 0 (system level).
- Fail: A sub-model within a partition is instantiated by name (e.g., via `match` on a
  string) in more than one location, indicating the compositor pattern is not applied.
- Fail: A partition implements domain-specific mechanisms (events, composition,
  configuration) using constructs that differ from those defined at the system level.

---

## 5. Inter-partition Communication

---

### SIM-SYS-005 — Transport Abstraction

**Statement:** The system shall support at minimum three inter-partition communication
modes: (a) in-process synchronous channels, (b) asynchronous message-passing across
threads, and (c) network-based publish-subscribe over a configurable endpoint. The active
mode shall be selectable at runtime via scenario configuration without recompilation.

**Rationale:** In-process channels minimize latency for single-machine development.
Asynchronous channels support partitions running on separate threads at different update
rates. Network-based transport enables distributed execution across machines and
integration with external tools. No single mode satisfies all deployment contexts.

**Verification Expectations:**
- Pass: The same scenario executes to completion under all three transport modes with
  identical final vehicle state (within floating-point determinism limits) when run on
  a single machine.
- Pass: Switching transport mode requires only a change to the `[sim]` section of the
  scenario TOML; no source files are modified.
- Fail: A partition contains a compile-time import or branch that is specific to one
  transport mode (i.e., transport choice is not fully abstracted behind the `SimBus`
  trait).
- Fail: The system fails to initialize or exchange messages under any of the three
  transport modes.

---

### SIM-SYS-006 — Typed Message Contracts

**Statement:** All data exchanged between partitions shall be transmitted as instances
of named, versioned message types declared in `sim-core`. Untyped byte buffers, raw
pointers, and JSON-serialized dynamic payloads shall not be used as the primary message
format at partition boundaries.

**Rationale:** Typed messages make interface contracts explicit and compiler-enforced,
prevent silent data misinterpretation across partitions, and provide a stable basis for
version compatibility tracking.

**Verification Expectations:**
- Pass: Every field consumed by a receiving partition from an inter-partition message is
  statically typed and verifiable by the Rust compiler at the sender's crate boundary.
- Pass: `sim-core` declares `PlantState`, `GNCCommand`, `EnvState`, and `WorldState` as
  named structs with documented field semantics.
- Fail: Any inter-partition exchange relies on a `Vec<u8>` or `serde_json::Value` as the
  message payload without an enclosing typed wrapper defined in `sim-core`.

---

## 6. Multi-vehicle Operation

---

### SIM-SYS-007 — Multi-vehicle Support

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

### SIM-SYS-008 — Vehicle Identification

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
- Fail: Any inter-partition message type defined in `sim-core` that pertains to a
  specific vehicle lacks a `VehicleId` field.
- Fail: Two active vehicles share the same `VehicleId` at any point during a simulation
  run.

---

### SIM-SYS-009 — World State Broadcast

**Statement:** The system shall publish an aggregated `WorldState` message each
simulation tick containing the `PlantState` of all currently active vehicles. This
message shall be available to all partitions via the active transport.

**Rationale:** GN&C algorithms for formation flying, collision avoidance, and
cooperative tasking require awareness of peer vehicle states. A broadcast `WorldState`
provides this without requiring point-to-point subscriptions between individual GN&C
instances.

**Verification Expectations:**
- Pass: In a two-vehicle scenario, the `WorldState` received by each GN&C plugin
  contains entries for both vehicles.
- Pass: When a vehicle is despawned, it is absent from the `WorldState` broadcast in
  all subsequent ticks.
- Fail: A GN&C plugin must subscribe to individual per-vehicle topics to obtain the
  states of peer vehicles; no aggregate is available.
- Fail: `WorldState` is broadcast at a rate different from the physics tick rate.

---

### SIM-SYS-010 — Dynamic Vehicle Lifecycle

**Statement:** The system shall support spawning and despawning of vehicle instances
during an active simulation session. Spawning shall load the specified plant model and
GN&C plugin and initialize the vehicle to provided initial conditions. Despawning shall
cleanly unload all associated resources.

**Rationale:** Dynamic vehicle lifecycle enables scenarios in which aircraft launch,
complete their mission, and recover or are destroyed, without requiring a full simulation
restart. It also supports incremental scenario construction during interactive use.

**Verification Expectations:**
- Pass: A vehicle spawned mid-simulation at specified initial conditions begins
  publishing `PlantState` messages on the subsequent tick.
- Pass: After despawning a vehicle, no further messages bearing its `VehicleId` appear
  on the bus and its Bevy entity is removed from the world.
- Fail: Spawning a second vehicle during a running simulation requires a restart.
- Fail: Despawning a vehicle leaves its GN&C dynamic library loaded in process memory
  (verified by library handle count or equivalent).

---

## 7. Physics and Environment

---

### SIM-SYS-011 — Physics Sub-model Composition

**Statement:** The physics partition shall provide at minimum three named composition
presets that select specific layer 1 sub-model implementations: (a) `"low"` — flat-earth
rigid body 6-DoF equations of motion; (b) `"mid"` — WGS84 geodetic reference with
International Standard Atmosphere; (c) `"high"` — mid-tier plus wind field, atmospheric
turbulence, and full propulsion dynamics. The active preset shall be selectable per
vehicle via scenario configuration. Individual sub-model selections within a preset
shall be independently overridable (see SIM-SYS-025).

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
  requirement SIM-SYS-020).
- Fail: Changing the physics composition requires a mechanism distinct from the
  compositor pattern used at layer 0 or in other partitions.

---

### SIM-SYS-012 — Environment Model Composability

**Statement:** The system shall implement environment effects (standard atmosphere, wind
field, atmospheric turbulence, terrain elevation, and gravity) as independently
composable models. Each model shall be individually enabled, disabled, or replaced via
scenario configuration without affecting the behavior of other active environment models.

**Rationale:** Environment models are layer 1 partitions within the physics domain.
Their independent composability is a direct application of the fractal partition
pattern (SIM-SYS-004): each sub-model is independently replaceable via the same
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

### SIM-SYS-013 — Configurable Simulation Rate

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

## 8. GN&C Plugin Interface

---

### SIM-SYS-014 — Stable GN&C ABI

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

### SIM-SYS-015 — GN&C Plugin Isolation

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
- Fail: The plugin requires a `bevy`, `sim-core`, `sim-physics`, or `tokio` dependency
  to compile.

---

### SIM-SYS-016 — GN&C Peer State Access

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

## 9. Vehicle Definition

---

### SIM-SYS-017 — Vehicle Manifest

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

### SIM-SYS-018 — Visual Plant State Interface

**Statement:** The physics plant model shall implement a `VizStateProvider` trait
defined in `sim-core`, producing a `VehiclePlantVizState` message each tick containing
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

## 10. Visualization

---

### SIM-SYS-019 — Visualization Modularity

**Statement:** The visualization partition shall implement each visual feature as an
independently addable and removable Bevy plugin: (a) vehicle mesh and surface animation,
(b) environment rendering (sky, terrain, horizon), (c) flight path trail, and
(d) data overlays and HUD. Each plugin shall be enabled or disabled via scenario
configuration without modifying any other plugin's source code.

**Rationale:** Visualization plugins are layer 1 partitions within the visualization
domain — the fractal partition pattern (SIM-SYS-004) applied to rendering. Assigning
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

### SIM-SYS-020 — Bevy ECS Ownership of Visualization and UI

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

## 11. User Interface

---

### SIM-SYS-021 — UI Execution Control

**Statement:** The user interface partition shall provide interactive controls for at
minimum the following simulation lifecycle operations: start, pause, resume, stop, and
reset to initial conditions. These controls shall emit `ExecutionStateRequest` messages
on the simulation bus. The UI is one of potentially many sources of execution state
transition requests; the orchestrator (`sim-app`) is the sole authority for evaluating
and applying transitions (see SIM-SYS-041).

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

### SIM-SYS-022 — UI Extensibility

**Statement:** The user interface partition shall implement each functional panel
(execution control, parameter tuning, scenario and waypoint editing, telemetry recording)
as an independently addable Bevy plugin. Additional UI panels shall be addable via
scenario configuration without modifying existing panel source files.

**Rationale:** UI panels are layer 1 partitions within the UI domain, mirroring
the visualization modularity requirement — both are instances of the fractal partition
pattern (SIM-SYS-004) applied within their respective layer 0 partitions. This allows
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

## 12. Configuration and Composition

---

### SIM-SYS-023 — TOML Scenario File

**Statement:** The system shall accept a TOML file as its primary runtime configuration
interface. A scenario file shall fully specify: simulation parameters (timestep, bus
mode), environment model list and per-model parameters, per-vehicle configuration
(manifest path, composition preset or inline sub-model selection, GN&C plugin path,
initial conditions), active
visualization plugins, and active UI panels.

**Rationale:** A human-readable, file-based configuration surface separates operational
intent from implementation, allows scenarios to be version-controlled alongside code,
and provides a portable artifact that students and laboratories can exchange without
access to simulator source code.

**Verification Expectations:**
- Pass: The system initializes and executes a complete simulation run from a TOML file
  without additional command-line arguments beyond the file path.
- Pass: Two operators on different machines produce identical initial physics states
  from the same scenario TOML (excluding non-deterministic transport effects).
- Fail: Any simulation parameter that is documented as configurable requires a source
  code change or environment variable to override.

---

### SIM-SYS-024 — Scenario Inheritance

**Statement:** A scenario TOML file shall support an `extends` field whose value is a
path to a base scenario file. Fields present in the inheriting file shall override the
corresponding fields in the base. Fields absent from the inheriting file shall be
inherited unchanged from the base.

**Rationale:** Composition preset variants and vehicle configuration variants of the
same scenario are typically small diffs against a common base. Inheritance allows these
variants to be expressed minimally, keeping them automatically synchronized with base
scenario changes and reducing the maintenance burden of scenario libraries.

**Verification Expectations:**
- Pass: A scenario file containing only `extends = "base.toml"` and a single overridden
  field produces a simulation that differs from the base scenario only in the overridden
  field's effect.
- Pass: Modifying a shared field in the base scenario is reflected in all inheriting
  scenarios without modifying the inheriting files.
- Fail: A circular `extends` chain (A extends B extends A) is silently accepted; the
  system shall detect and report it as a configuration error.

---

### SIM-SYS-025 — Named Composition Presets

**Statement:** The scenario file shall support named composition presets resolvable to
preset files in a configurable presets directory. Presets shall be applicable to any
fractally partitioned domain — physics (e.g., `"low"`, `"mid"`, `"high"`),
visualization, GN&C, or any other partition that composes layer 1 sub-components.
Inline configuration tables shall be supported to override or replace individual
sub-model selections within a named preset without defining an entirely new preset.

**Rationale:** Composition presets are a natural consequence of the fractal partition
pattern: since every partition composes independently replaceable sub-components via
the same compositor mechanism, a reusable named selection of those sub-components is
useful at any layer and in any domain. Named presets give instructors a stable,
communicable vocabulary (e.g., "run your controller at mid fidelity") while inline
overrides allow individual sub-model substitution without defining an entirely new
preset. Generalizing presets beyond physics ensures that the same composition mechanism
serves visualization quality levels, GN&C estimator configurations, or any future
domain without introducing a domain-specific preset system.

**Verification Expectations:**
- Pass: A vehicle configured with `preset = "mid"` produces the same behavior as a
  vehicle configured with an inline table containing the fields defined in
  `config/presets/physics/mid.toml`.
- Pass: An inline override of a single field (e.g., `wind = "dryden"`) in an otherwise
  `"mid"` preset vehicle activates only that sub-model difference, leaving all other
  sub-models selected by the preset unchanged.
- Pass: A visualization preset (e.g., `[viz] preset = "minimal"`) selects a specific
  set of visualization plugins, and inline overrides add or remove individual plugins
  without replacing the entire preset.
- Fail: The system accepts an unrecognized preset name without reporting an error.
- Fail: The preset mechanism is available only for physics; other partitions require
  a different mechanism to achieve named sub-component selections.

---

## 13. Distribution and Packaging

---

### SIM-SYS-026 — Implementation Language

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

### SIM-SYS-027 — Crate Publishability

**Statement:** The following crates shall be publishable to crates.io or a compatible
Cargo registry: `sim-core`, `sim-gnc-abi`, `sim-physics`, `sim-gnc`, `sim-viz`,
`sim-ui`, and `universe`. The `sim-app` binary target shall be published as part of
the `universe` crate. Internal tooling and scenario runners not intended for external
use shall set `publish = false`.

**Rationale:** Publishing partition crates independently allows other laboratories to
take a dependency on specific partitions (e.g., `sim-core` for interface compatibility,
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

### SIM-SYS-028 — Independent GN&C ABI Versioning

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
- Fail: The `sim-gnc-abi` version is updated in lockstep with `sim-core` or `sim-physics`
  for releases that do not affect the ABI.

---

### SIM-SYS-029 — Headless Feature Flag

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

### SIM-SYS-030 — Embedding Interface

**Statement:** The `universe` library crate shall expose a `FlightSimPlugin`
implementing the Bevy `Plugin` trait. Third-party Bevy applications shall be able to
incorporate the full simulator by adding `FlightSimPlugin` to their `App` without
depending on individual partition crates.

**Rationale:** Laboratories with existing Bevy-based applications (visualization
pipelines, hardware interfaces, mission control displays) should be able to embed the
simulator as a component of a larger system rather than treating it as a standalone
executable.

**Verification Expectations:**
- Pass: A minimal third-party Bevy application that declares only `universe` as a
  dependency and calls `app.add_plugins(FlightSimPlugin::from_toml(...))` compiles and
  executes a simulation run.
- Pass: The third-party application can add its own Bevy plugins alongside
  `FlightSimPlugin` without conflict.
- Fail: Embedding `FlightSimPlugin` requires the third-party application to also
  declare `sim-core`, `sim-physics`, `sim-gnc`, `sim-viz`, or `sim-ui` as direct
  dependencies.

---

## 14. Telemetry

---

### SIM-SYS-031 — Telemetry Recording

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

### SIM-SYS-032 — Telemetry Playback

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

## 15. Events

---

### SIM-SYS-035 — Event System Architecture

**Statement:** The system shall provide a unified event system in which discrete events
can be defined, armed, triggered, and handled at both layer 0 (system level) and layer 1
(partition level). Consistent with the fractal partition pattern (SIM-SYS-004), each
partition (physics, GN&C, visualization, environment) shall define, arm, and handle
events using the same primitives and interfaces available at the system level.

**Rationale:** Mission timelines, failure injection, scenario scripting, and conditional
logic all require a mechanism for scheduling and reacting to discrete occurrences during
a simulation run. The fractal partition pattern requires that the event system be
uniform across layers: partition-specific events (e.g., a sensor failure in the plant
model, a mode transition in GN&C, a camera cut in visualization) are expressed and
managed with the same constructs as system-level events (e.g., simulation start, scenario
phase transitions), avoiding redundant or incompatible mechanisms across partitions.

**Verification Expectations:**
- Pass: A system-level event and a partition-level event defined in the same scenario
  both trigger at their specified conditions during a simulation run.
- Pass: A partition-level event defined within the GN&C partition uses the same event
  definition schema and trigger types as a system-level event defined outside any
  partition scope.
- Fail: A partition implements its own ad-hoc event mechanism that does not conform to
  the system event interface defined in `sim-core`.
- Fail: Events can only be defined at the system level; partition-scoped events are not
  supported.

---

### SIM-SYS-036 — Time-triggered Events

**Statement:** The event system shall support events triggered at a specified time.
Consistent with the fractal partition pattern (SIM-SYS-004), time semantics vary by
layer: at layer 0 (system level), time-triggered events shall reference wall-clock time
elapsed since simulation start; at layer 1 (partition level), time-triggered events shall
reference mission scenario time as defined by the simulation clock.

**Rationale:** Wall-clock triggers at layer 0 support infrastructure concerns such as
telemetry snapshots, periodic health checks, and real-time synchronization boundaries
that are independent of simulation time scaling. Scenario-time triggers at layer 1
(partition level) support mission timeline events (e.g., "ignite second stage at T+120s")
that must track the simulated clock, including during time-scaled or paused execution.

**Verification Expectations:**
- Pass: A layer 0 event configured to trigger at wall-clock T+5s fires at approximately
  5 seconds of real elapsed time, regardless of whether the simulation is running at 0.5x
  or 2x real-time speed.
- Pass: A layer 1 event configured to trigger at scenario time T+60s fires when the
  simulation clock reaches 60 seconds, regardless of wall-clock elapsed time.
- Pass: A layer 1 time-triggered event does not advance while the simulation is paused.
- Fail: A scenario-time event fires based on wall-clock time, causing it to trigger at
  the wrong mission phase when time scaling is active.

---

### SIM-SYS-037 — Condition-triggered Events

**Statement:** The event system shall support events triggered by logical conditions
evaluated against observable signals. A condition shall be expressible as a boolean
predicate over one or more named signals (e.g., `altitude_msl < 100.0`,
`mach > 1.0 && dynamic_pressure > 500.0`). The set of observable signals shall include
any value published on the simulation bus or exposed as a named field within a
partition's state.

**Rationale:** Many mission events are defined not by clock time but by physical
conditions: staging at a target altitude, deploying landing gear below a threshold
airspeed, or injecting a fault when a sensor reading enters a specific range. Condition
triggers allow scenario authors to express event logic declaratively without embedding
procedural checks in partition source code.

**Verification Expectations:**
- Pass: An event conditioned on `altitude_msl < 500.0` triggers on the first tick in
  which the monitored vehicle's altitude falls below 500 meters, and does not trigger
  while altitude remains above 500 meters.
- Pass: A compound condition using logical AND over two signals triggers only when both
  sub-conditions are simultaneously satisfied.
- Pass: Condition predicates reference signals by name as published on the bus or defined
  in partition state, without requiring the event author to specify memory addresses or
  internal data paths.
- Pass: A condition-triggered event with action `"sim_stop"` conditioned on
  `altitude_msl < 0.0` causes the orchestrator to transition the simulation to the
  Stopped state on the first tick the condition is met (see SIM-SYS-042).
- Fail: Condition-triggered events can only monitor system-level signals; partition-
  internal state is not observable by the event system.

---

### SIM-SYS-038 — Partition-scoped Event Arming

**Statement:** Each partition shall be capable of defining and arming events scoped to
its own domain (layer 1). Partition-scoped events shall be evaluated against that
partition's internal signals and shall invoke handlers within the partition's execution
context. Consistent with the fractal partition pattern (SIM-SYS-004), the mechanism
for defining and arming events shall be uniform across all partitions and identical in
structure to system-level (layer 0) event definitions.

**Rationale:** The fractal partition pattern requires that each partition have the same
event capabilities as the system level. Partitions contain domain-specific state that may
not be published on the inter-partition bus but is meaningful for triggering domain-
specific behavior. A GN&C partition may arm an event on an internal estimator convergence
metric; a plant model may arm an event on a structural load threshold; a visualization
partition may arm a camera transition on proximity to a waypoint. Requiring all such
events to be defined at the system level would violate partition encapsulation and create
coupling between the event configuration and partition internals.

**Verification Expectations:**
- Pass: A GN&C partition arms an event on an internal signal not published to the bus,
  and the event triggers correctly when the condition is met during GN&C execution.
- Pass: A physics partition and a visualization partition each arm independent events
  using the same event definition schema, and both trigger correctly without interference.
- Pass: A partition-scoped event is defined in the partition's section of the scenario
  TOML and does not require modification to any other partition's configuration.
- Pass: A physics partition event armed on an internal structural load signal fires
  a `"sim_stop"` action that the orchestrator applies, halting the simulation without
  UI involvement (see SIM-SYS-042).
- Fail: Arming an event within a partition requires modifying `sim-core` or the system-
  level event dispatcher source code.

---

### SIM-SYS-039 — Event Definition in Scenario Configuration

**Statement:** Events shall be declaratively defined in the scenario TOML file.
Layer 0 (system-level) events shall be defined in a top-level `[[events]]` array.
Layer 1 (partition-level) events shall be defined within the corresponding partition's
configuration section (e.g., `[[physics.events]]`, `[[gnc.events]]`, `[[viz.events]]`).
Each event entry shall specify a trigger (time or condition), an action identifier, and
optional parameters. Consistent with the fractal partition pattern (SIM-SYS-004), the
event entry schema shall be identical at both layers.

**Rationale:** Declarative event definition in the scenario file keeps mission timelines
version-controlled, portable, and inspectable alongside other scenario configuration.
The fractal partition pattern requires that the same event schema be usable at every
layer, so that scenario authors learn one event syntax and apply it uniformly. It avoids
hard-coding event logic in partition source code and allows instructors to modify mission
event sequences without recompilation.

**Verification Expectations:**
- Pass: A scenario TOML containing a `[[physics.events]]` entry with a time trigger and
  an action identifier causes the specified action to execute in the physics partition at
  the specified scenario time.
- Pass: A scenario TOML containing both `[[events]]` (system-level) and
  `[[gnc.events]]` (partition-level) entries executes both event categories correctly in
  the same run.
- Pass: Modifying the trigger time or condition of an event requires only a change to the
  scenario TOML; no source code modification is needed.
- Fail: Events can only be defined programmatically in Rust source; no scenario-file
  representation exists.

---

### SIM-SYS-040 — Simulation Execution State Machine

**Statement:** The system shall define a formal execution state machine in `sim-core`
with the following states and transitions:

| State     | Valid Transitions                        |
|-----------|------------------------------------------|
| Idle      | → Running (start)                        |
| Running   | → Paused (pause), → Stopped (stop)       |
| Paused    | → Running (resume), → Stopped (stop)     |
| Stopped   | → Idle (reset)                           |

The current execution state shall be available as a named resource or type in `sim-core`
accessible to all partitions. No partition shall maintain a private copy of the execution
state that diverges from the authoritative value held by the orchestrator.

**Rationale:** In a fractally partitioned system, any partition at any layer may need to
observe or influence the simulation lifecycle. Execution state is currently implicit in
SIM-SYS-021's description of UI controls. Without a formal state machine, partitions
cannot consistently reason about valid transitions, detect invalid requests, or
synchronize their behavior with the global execution lifecycle. An explicit, centrally
defined state machine prevents undefined behavior when multiple sources (UI, events,
partition logic) attempt to influence execution state.

**Verification Expectations:**
- Pass: The `sim-core` crate exports an enum or equivalent type representing the four
  execution states (Idle, Running, Paused, Stopped) and a function or method that
  validates whether a requested transition is valid from the current state.
- Pass: Requesting an invalid transition (e.g., Running → Idle) returns an error or
  rejection rather than silently succeeding.
- Pass: All partitions read execution state from the same `sim-core` resource; no
  partition defines its own execution state enum.
- Fail: The execution state is represented as a bare boolean or integer flag without
  enforced transition semantics.

---

### SIM-SYS-041 — Execution State Transition Requests

**Statement:** Any partition shall be capable of requesting a simulation execution state
transition by emitting an `ExecutionStateRequest` message on the simulation bus. The
`ExecutionStateRequest` type shall be defined in `sim-core` and shall carry the requested
transition and the identity of the requesting partition. The orchestrator (`sim-app`)
shall be the sole authority for evaluating and applying state transitions according to
the state machine defined in SIM-SYS-040. Requests that represent invalid transitions
shall be logged with the requesting partition's identity and ignored. Valid transitions
shall take effect within one physics tick of receipt.

**Rationale:** The fractal partition pattern (SIM-SYS-004) implies that partitions at
any layer may generate events that affect execution state. Multiple sources may
legitimately need to change execution state: the UI for interactive control, a plant
model detecting a safety limit exceedance, a GN&C event handler reaching a scripted
pause point, or a condition-triggered event firing a stop. Routing all requests through
a single arbitration point (the orchestrator) prevents race conditions, ensures
transition validity, and provides a single audit point for execution state changes.

**Verification Expectations:**
- Pass: A physics partition emitting `ExecutionStateRequest::Stop` during a Running
  simulation causes the orchestrator to transition to the Stopped state within one
  physics tick.
- Pass: A GN&C partition emitting `ExecutionStateRequest::Pause` during a Running
  simulation causes the orchestrator to transition to the Paused state within one
  physics tick.
- Pass: An `ExecutionStateRequest::Resume` emitted while the simulation is in the
  Running state is logged as an invalid transition and ignored; the simulation continues
  running without interruption.
- Pass: Each applied transition is logged with the identity of the partition that
  requested it.
- Fail: A partition directly mutates the execution state resource without emitting an
  `ExecutionStateRequest` on the bus.
- Fail: Two partitions emitting conflicting requests in the same tick causes undefined
  behavior (see SIM-SYS-043 for conflict resolution).

---

### SIM-SYS-042 — Execution State Change as Event Action

**Statement:** The event system shall support `"sim_pause"`, `"sim_stop"`, and
`"sim_resume"` as built-in action identifiers available at both layer 0 (system level)
and layer 1 (partition level). When an event with one of these action identifiers fires, the event
dispatcher shall emit the corresponding `ExecutionStateRequest` on the simulation bus.
These actions shall be usable with both time-triggered and condition-triggered events.

**Rationale:** Execution state changes are among the most common event-driven actions in
mission simulation. Encoding a GN&C pause point, a plant safety stop, or a scenario
phase transition as event actions keeps the logic declarative and configurable in the
scenario TOML, avoiding hard-coded procedural checks in partition source code. Connecting
the event system to the execution state request mechanism (SIM-SYS-041) ensures all
event-driven state changes flow through the same arbitration path as UI-initiated
changes.

**Verification Expectations:**
- Pass: A scenario TOML entry `[[gnc.events]]` with `trigger = { time = 30.0 }` and
  `action = "sim_pause"` causes the simulation to pause when the GN&C partition's
  simulation clock reaches 30 seconds.
- Pass: A scenario TOML entry `[[physics.events]]` with
  `trigger = { condition = "structural_load > 9.0" }` and `action = "sim_stop"` causes
  the simulation to stop when the monitored signal exceeds the threshold.
- Pass: A system-level `[[events]]` entry with `action = "sim_resume"` and a time
  trigger successfully resumes a paused simulation at the specified wall-clock time.
- Pass: The `ExecutionStateRequest` emitted by an event action is indistinguishable from
  one emitted by the UI or by partition code directly; the orchestrator processes it
  identically.
- Fail: Execution state change actions require partition source code modifications
  rather than scenario TOML configuration.

---

### SIM-SYS-043 — Execution State Transition Conflict Resolution

**Statement:** When the orchestrator receives multiple `ExecutionStateRequest` messages
within the same physics tick that request conflicting transitions, it shall apply the
following deterministic priority order: (1) Stop takes priority over all other requests.
(2) Pause takes priority over Resume. (3) Among requests of equal priority, the first
received shall be applied. All received requests and the resolution outcome shall be
logged with the requesting partition identities.

**Rationale:** In a fractally partitioned system, any partition at any layer may
generate events that request execution state changes. Concurrent events may independently
request conflicting transitions in the same tick — for example, a GN&C event requesting
pause while a physics safety limit simultaneously requests stop. Without a deterministic
resolution policy, the resulting execution state would depend on message ordering, which
varies with transport mode and scheduling. Prioritizing stop over pause reflects the
principle that safety-critical transitions should not be overridden by less severe
requests.

**Verification Expectations:**
- Pass: When a physics partition emits `ExecutionStateRequest::Stop` and a GN&C
  partition emits `ExecutionStateRequest::Pause` in the same tick, the orchestrator
  transitions to Stopped.
- Pass: When a UI partition emits `ExecutionStateRequest::Resume` and a GN&C event
  emits `ExecutionStateRequest::Pause` in the same tick, the orchestrator transitions
  to Paused (if currently Paused, the pause wins and the state remains Paused).
- Pass: The orchestrator log for the tick includes both requests and identifies which
  was applied and which was superseded, with partition identities.
- Fail: Conflicting requests in the same tick produce non-deterministic behavior
  depending on transport mode or thread scheduling.
- Fail: A lower-priority request silently overrides a higher-priority request without
  logging.

---

## 16. Verification and Traceability

---

### SIM-SYS-033 — Partition-level Specifications

**Statement:** Each partition crate shall maintain a `docs/design/SPECIFICATION.md`
containing requirements that individually trace to one or more SIM-SYS identifiers in
this document. No partition-level requirement shall exist without a parent SIM-SYS
requirement.

**Rationale:** The fractal partition pattern requires traceability at every layer of
decomposition. Bidirectional traceability between layer 0 (system) and layer 1
(partition) requirements ensures that all system intents are verifiably allocated to an
implementing partition, and that no partition-level requirement is orphaned from
system-level intent.

**Verification Expectations:**
- Pass: Every requirement in each partition SPECIFICATION.md includes a `Traces to:`
  field referencing at least one `SIM-SYS-NNN` identifier present in this document.
- Pass: Every `SIM-SYS` requirement in this document is referenced by at least one
  partition-level requirement.
- Fail: A partition SPECIFICATION.md contains a requirement with no `Traces to:` field.
- Fail: A `SIM-SYS` requirement exists with no corresponding partition-level child
  requirement in any applicable document listed in Section 2.

---

### SIM-SYS-034 — Test Coverage of Requirements

**Statement:** Each requirement in every partition SPECIFICATION.md shall be verified by
at least one test in that crate's `tests/` directory. Test files shall be named using
the requirement identifier they primarily verify (e.g., `tests/sim_phy_001.rs`).

**Rationale:** Named test files create a direct audit trail from requirement to
verification evidence, making it straightforward to confirm coverage during design
reviews and to identify which tests must be updated when a requirement changes.

**Verification Expectations:**
- Pass: For every requirement `SIM-XYZ-NNN` in a partition specification, a file
  `tests/sim_xyz_nnn.rs` exists in that partition's crate and contains at least one
  `#[test]` function.
- Pass: `cargo test --workspace` executes all requirement-linked tests and reports
  results.
- Fail: A requirement exists in a partition specification with no corresponding file
  in `tests/`.
- Fail: A test file in `tests/` exists with no linkage to a named requirement (i.e.,
  its filename does not correspond to any requirement identifier).

---

## 17. Requirements Traceability Matrix

| ID          | Title                              | Allocated To              |
|-------------|------------------------------------|---------------------------|
| SIM-SYS-001 | Four-partition Architecture        | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-002 | Partition Independence             | TF-SRS-006                |
| SIM-SYS-003 | Inter-partition Interface Ownership| TF-SRS-005                |
| SIM-SYS-004 | Fractal Partition Pattern           | TF-SRS-001 through 004    |
| SIM-SYS-005 | Transport Abstraction              | TF-SRS-005                |
| SIM-SYS-006 | Typed Message Contracts            | TF-SRS-005                |
| SIM-SYS-007 | Multi-vehicle Support              | TF-SRS-006                |
| SIM-SYS-008 | Vehicle Identification             | TF-SRS-005                |
| SIM-SYS-009 | World State Broadcast              | TF-SRS-005                |
| SIM-SYS-010 | Dynamic Vehicle Lifecycle          | TF-SRS-006                |
| SIM-SYS-011 | Physics Sub-model Composition      | TF-SRS-001                |
| SIM-SYS-012 | Environment Model Composability    | TF-SRS-001                |
| SIM-SYS-013 | Configurable Simulation Rate       | TF-SRS-001, TF-SRS-006    |
| SIM-SYS-014 | Stable GN&C ABI                    | TF-SRS-002A               |
| SIM-SYS-015 | GN&C Plugin Isolation              | TF-SRS-002A               |
| SIM-SYS-016 | GN&C Peer State Access             | TF-SRS-002A               |
| SIM-SYS-017 | Vehicle Manifest                   | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-018 | Visual Plant State Interface       | TF-SRS-001, TF-SRS-003    |
| SIM-SYS-019 | Visualization Modularity           | TF-SRS-003                |
| SIM-SYS-020 | Bevy ECS Ownership                 | TF-SRS-003, TF-SRS-004    |
| SIM-SYS-021 | UI Execution Control               | TF-SRS-004                |
| SIM-SYS-022 | UI Extensibility                   | TF-SRS-004                |
| SIM-SYS-023 | TOML Scenario File                 | TF-SRS-006                |
| SIM-SYS-024 | Scenario Inheritance               | TF-SRS-006                |
| SIM-SYS-025 | Named Composition Presets          | TF-SRS-001 through 006    |
| SIM-SYS-026 | Implementation Language            | TF-SRS-001 through 006    |
| SIM-SYS-027 | Crate Publishability               | TF-SRS-006                |
| SIM-SYS-028 | Independent GN&C ABI Versioning    | TF-SRS-002A               |
| SIM-SYS-029 | Headless Feature Flag              | TF-SRS-006                |
| SIM-SYS-030 | Embedding Interface                | TF-SRS-006                |
| SIM-SYS-031 | Telemetry Recording                | TF-SRS-004                |
| SIM-SYS-032 | Telemetry Playback                 | TF-SRS-003, TF-SRS-004    |
| SIM-SYS-033 | Partition-level Specifications     | TF-SRS-001 through 006    |
| SIM-SYS-034 | Test Coverage of Requirements      | TF-SRS-001 through 006    |
| SIM-SYS-035 | Event System Architecture          | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-036 | Time-triggered Events              | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-037 | Condition-triggered Events         | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-038 | Partition-scoped Event Arming      | TF-SRS-001 through 004    |
| SIM-SYS-039 | Event Definition in Scenario Config| TF-SRS-006                |
| SIM-SYS-040 | Simulation Execution State Machine | TF-SRS-005                |
| SIM-SYS-041 | Execution State Transition Requests| TF-SRS-005, TF-SRS-006    |
| SIM-SYS-042 | Execution State Change as Event Action | TF-SRS-005, TF-SRS-006 |
| SIM-SYS-043 | Execution State Transition Conflict Resolution | TF-SRS-006 |
