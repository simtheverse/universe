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
6. [Inter-layer Communication](#6-inter-layer-communication)
7. [Multi-vehicle Operation](#7-multi-vehicle-operation)
8. [Physics and Environment](#8-physics-and-environment)
9. [GN&C Plugin Interface](#9-gnc-plugin-interface)
10. [Vehicle Definition](#10-vehicle-definition)
11. [Visualization](#11-visualization)
12. [User Interface](#12-user-interface)
13. [Configuration and Composition](#13-configuration-and-composition)
14. [Distribution and Packaging](#14-distribution-and-packaging)
15. [Telemetry](#15-telemetry)
16. [Events](#16-events)
17. [Verification and Traceability](#17-verification-and-traceability)
18. [Requirements Traceability Matrix](#18-requirements-traceability-matrix)

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
| Compositor        | A component that selects and assembles partition implementations at startup and, at runtime, owns the layer's bus instance, drives partition execution via trait calls, publishes shared context on the bus, arbitrates requests, and relays inter-layer messages to the outer bus with authority to filter, transform, or suppress |
| Contract crate    | A Rust crate that defines traits and data types but contains no implementation |
| Diátaxis          | A documentation framework organizing content into four quadrants by user need: tutorials (learning-oriented), how-to guides (task-oriented), reference (information-oriented), and explanation (understanding-oriented). See SIM-SYS-033 |
| DoF               | Degrees of Freedom                                                         |
| ECS               | Entity Component System — Bevy's data-oriented architecture                |
| Composition fragment | A configuration block — inline or named — that selects partition implementations at a given scope within the fractal structure. A session is a composition fragment at layer 0. A scenario is a composition fragment at layer 1. A vehicle section, a physics preset, and a visualization preset are composition fragments at progressively narrower scopes. All composition fragments share the same override and inheritance semantics (see SIM-SYS-024, SIM-SYS-025) |
| Fidelity tier     | A named composition fragment within the physics domain that selects sub-model implementations by complexity level (e.g., `"low"`, `"mid"`, `"high"`). Not a distinct system primitive — fidelity tiers are named composition fragments expressed using the same mechanism available to every partition |
| GN&C              | Guidance, Navigation, and Control                                          |
| IC                | Initial Conditions                                                         |
| INCOSE            | International Council on Systems Engineering                               |
| ISA               | International Standard Atmosphere                                          |
| Event action      | An action identifier specified in an event's TOML definition. All event actions are declared in contract crates and scoped to the declaring crate's hierarchy. Actions defined in `sim-core` (e.g., `"sim_stop"`, `"sim_pause"`, `"sim_resume"`) are available at every layer because all partitions depend on `sim-core`. Actions defined in a partition's contract crate (e.g., `"stage_separate"`, `"inject_sensor_fault"`) are available to events within that partition's hierarchy. The event mechanism is uniform; the action vocabulary is contract-crate-scoped. See SIM-SYS-061 |
| Fractal partition pattern | The architectural principle that the system is decomposed into layers and partitions, where each partition at every layer applies the same contract/implementation/compositor structure and the same event, configuration, and communication primitives as the system level. Named for the self-similarity of structure at every scale. See SIM-SYS-004 |
| Layer             | A level in the system's hierarchical decomposition. Layer 0 is the system level; layer 1 is the partition level. The fractal partition pattern applies at every layer: each uses the same structural primitives (contracts, events, composition) as the layer above it |
| Layer and partition uniformity principle | The defining property of the fractal partition pattern: structural primitives (contracts, events, configuration, composition, specification, and documentation structure) are identical in kind across all layers and partitions. A construct available at layer 0 is available in the same form at layer 1 and beyond |
| NED               | North-East-Down coordinate frame                                           |
| Partition         | A functional subdivision of the system at a given layer. At layer 0, the four top-level partitions (physics, GN&C, visualization, UI). At layer 1, sub-components within a partition (e.g., atmosphere model, navigation estimator). Each partition is independently replaceable provided it conforms to its layer's interface contracts |
| Plugin            | A Bevy `Plugin` implementor, or a dynamically loaded shared library        |
| Scenario          | A layer 1 composition fragment describing mission-level configuration: vehicle definitions, physics sub-model selections, environment models, events, and initial conditions. A session composes one or more scenarios |
| Session           | The layer 0 composition fragment describing a simulation run: transport mode, timing parameters, active partitions, and which scenarios to compose. A session is the top-level entry point for the compositor |
| State snapshot    | A composition fragment produced by capturing the complete simulation state at a point in time. A state snapshot is not a distinct system primitive — it is a composition fragment whose fields happen to have been machine-generated rather than hand-authored. Snapshots are loadable, inheritable, and overridable using the same mechanisms as any other composition fragment (see SIM-SYS-044) |
| Delivery semantic | A per-message-type specification of how the bus delivers messages to consumers. Latest-value retains only the most recent value (suitable for continuous state). Queued retains all messages in order (suitable for requests that must not be dropped). Declared in the contract crate alongside the type. See SIM-SYS-063 |
| Direct signal     | A safety-critical signal type declared in a contract crate that bypasses the compositor relay chain within that contract crate's hierarchy and reaches the declaring crate's orchestrator directly. Scoped to the declaring crate's jurisdiction — does not propagate beyond the boundary when the system is embedded as a partition in an outer system. Reserved for scenarios where compositor suppression would be unsafe (e.g., emergency stop, hardware fault). See SIM-SYS-060 |
| Layer-scoped bus  | A bus instance owned by the compositor at a given layer, connecting that layer's partitions. Each compositor owns a separate bus instance; sub-partitions publish only to their layer's bus, not to buses at other layers. Inter-layer communication occurs through compositor relay. See SIM-SYS-055 |
| Relay authority   | The compositor's right to decide whether a message received on its inner bus is forwarded to the outer bus. The compositor may relay as-is, transform, suppress, or aggregate messages before re-emitting them. See SIM-SYS-057 |
| Tick lifecycle    | The three-phase execution model for each simulation tick: Phase 1 (pre-tick processing: direct signals, spawn/despawn, dump/load, WorldState assembly, buffer swap), Phase 2 (partition stepping with intra-tick message isolation), Phase 3 (post-tick processing: event evaluation, output collection, bus request processing, relay). See SIM-SYS-062 |
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
of its decomposition, in both runtime architecture and specification structure. At
layer 0 (system level), the four top-level partitions use contract/implementation/
compositor structure with shared interface types in `sim-core`. At layer 1 (partition
level), the internal structure of each partition shall apply the same contract/
implementation/compositor pattern, such that sub-components within a partition (e.g.,
aerodynamic model, atmosphere model, navigation estimator) are each independently
replaceable via trait objects without modifying their siblings. The **layer and partition
uniformity principle** shall hold: the same structural primitives — contracts, events
(see SIM-SYS-035), configuration (see SIM-SYS-023), composition, specification,
documentation structure (see SIM-SYS-033), and testing structure (see SIM-SYS-047,
SIM-SYS-048) — shall be identical in kind at every layer.

**Rationale:** The fractal partition pattern ensures that the modularity, event handling,
composition mechanisms, and specification structure proven at the system level are
available in the same form within each partition. Like a fractal, zooming into any
partition reveals the same contract/compositor/event/specification structure as the
system as a whole — and zooming out, universe itself is an independently replaceable
partition within a larger system (see SIM-SYS-030). A laboratory invoking universe as
one stage in a pre-process/simulate/post-process pipeline treats universe as a partition
within its own layer 0; the same contracts and interfaces that make universe's internal
partitions swappable make universe itself swappable in that outer context. This
maximizes the surface area of independent replaceability at every scale, allows
composition to be adjusted at the sub-model level (including fidelity selection via
composition presets), and allows instructors to assign isolated sub-components to student
teams — all using the same constructs they encounter at the system level. Applying the
pattern to specification structure ensures that each layer's spec nucleates the next
layer's specs, propagating traceability and structural discipline to arbitrary depth.

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
- Pass: Each partition's specification identifies its own sub-partitions and uses the
  same specification structure (statement, rationale, verification expectations,
  traceability) as this document.
- Fail: A partition implements domain-specific mechanisms (events, composition,
  configuration) using constructs that differ from those defined at the system level.
- Fail: A partition's specification defines requirements without identifying
  independently replaceable sub-partitions or without using the same structural
  primitives as this document.

---

## 5. Inter-partition Communication

---

### SIM-SYS-005 — Transport Abstraction

**Statement:** The system shall support at minimum three inter-partition communication
modes: (a) in-process synchronous channels, (b) asynchronous message-passing across
threads, and (c) network-based publish-subscribe over a configurable endpoint. The active
mode shall be selectable at runtime via configuration without recompilation. Consistent
with the fractal partition pattern (SIM-SYS-004), each compositor at every layer owns a
bus instance for its partitions (see SIM-SYS-055). Bus instances at different layers are
independent and may use different transport modes — the layer 0 bus might use network
transport while a layer 1 bus uses in-process transport. The transport independence
guarantee (identical results across modes) applies per bus instance.

**Rationale:** In-process channels minimize latency for single-machine development.
Asynchronous channels support partitions running on separate threads at different update
rates. Network-based transport enables distributed execution across machines and
integration with external tools. No single mode satisfies all deployment contexts.
Layer-scoped bus instances allow transport mode to be selected independently at each
layer, matching the deployment needs of each compositor's partitions without imposing a
system-wide choice.

**Verification Expectations:**
- Pass: The same scenario executes to completion under all three transport modes with
  identical final vehicle state (within floating-point determinism limits) when run on
  a single machine.
- Pass: Switching transport mode requires only a change to the configuration for the
  relevant layer; no source files are modified.
- Pass: A layer 0 bus configured for network transport and a layer 1 bus configured for
  in-process transport operate correctly in the same simulation run.
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

### SIM-SYS-046 — Shared State Machine Synchronization

**Statement:** When a state machine must be observed by multiple partitions at the same
layer, its type and transition rules shall be defined in the contract crate for that
layer. Exactly one owner — the orchestrator at layer 0, or a designated partition at
deeper layers — shall hold the authoritative value. All other partitions at that layer
shall observe the current state as a read-only value published through the contract
crate. State transitions shall be requested via the simulation bus, never by direct
mutation of the authoritative value. The owner shall evaluate requests against the state
machine's transition rules and reject invalid transitions. Consistent with the fractal
partition pattern (SIM-SYS-004), this mechanism shall be identical at every layer.

**Rationale:** Partitions at the same layer frequently need to coordinate around shared
state machines — execution lifecycle, mission phase, mode selections — without coupling
to each other's internals. Placing the type and transition rules in the contract crate
makes the state machine part of the layer's interface contract rather than an
implementation detail of any single partition. Single-owner authority with bus-mediated
requests prevents conflicting mutations and provides a single audit point for
transitions. The fractal partition pattern requires this mechanism to be available at
every layer: the execution state machine (SIM-SYS-040) is one instance at layer 0, but
sub-partitions at layer 1 or deeper may define their own shared state machines using
the same pattern.

**Verification Expectations:**
- Pass: All partitions at a given layer read a shared state machine's current value from
  the same contract-crate-defined type; no partition defines its own copy of the state
  enum.
- Pass: A partition requesting a transition emits a typed request on the bus; the owner
  evaluates and applies or rejects it according to the defined transition rules.
- Pass: An invalid transition request is rejected by the owner and logged; the state
  machine value remains unchanged.
- Pass: At layer 1, a sub-partition defines a shared state machine in its layer's
  contract module using the same owner/request/observe pattern used for the execution
  state machine at layer 0.
- Fail: A partition directly mutates a shared state machine's value without emitting a
  request on the bus.
- Fail: The mechanism for defining and synchronizing a shared state machine at layer 1
  differs structurally from the mechanism used at layer 0.

---

### SIM-SYS-063 — Bus Delivery Semantics

**Statement:** Each message type declared in a contract crate shall specify its
delivery semantic as part of the type's interface contract. Two delivery semantics
are defined:

- **Latest-value:** The bus retains only the most recent published value for the
  message type. A consumer that reads slower than the producer publishes will see
  only the most recent value, not intermediate values. Suitable for continuous state
  (e.g., `PlantState`, `WorldState`).
- **Queued:** The bus retains all published instances in order. A consumer receives
  every instance regardless of rate mismatch. Suitable for requests and commands
  where dropping an instance is a correctness failure (e.g., `ExecutionStateRequest`).

The specified semantic shall be enforced identically across all transport modes.
Queued messages shall not be silently dropped under any transport mode.

**Rationale:** Under async transport with partitions at different update rates
(SIM-SYS-013), a producer may publish faster than a consumer reads. Without a
specified delivery semantic per message type, different transport implementations make
different choices — leading to transport-dependent behavior. Declaring the semantic in
the contract crate alongside the type makes bus delivery behavior part of the interface
contract rather than a transport implementation detail, supporting the transport
independence guarantee (SIM-SYS-005).

**Verification Expectations:**
- Pass: The delivery semantic for each message type is declared in the contract crate
  and behaves identically under all three transport modes.
- Pass: A latest-value message type published multiple times within a tick is read by a
  slower consumer as a single value — the most recent — with no error or data loss
  concern.
- Pass: A queued message type published multiple times within a tick is received by the
  consumer as a complete, ordered sequence with no dropped instances.
- Fail: A transport implementation silently drops queued messages to maintain throughput.
- Fail: The delivery behavior for a message type varies across transport modes.

---

## 6. Inter-layer Communication

---

### SIM-SYS-055 — Layer-scoped Bus

**Statement:** Each compositor shall own a bus instance for its layer's partitions.
The layer 0 orchestrator owns the layer 0 bus; a compositor at layer 1 owns a layer 1
bus for its sub-partitions; the pattern continues at deeper layers. Bus instances at
different layers are independent: they may use different transport modes, and a
sub-partition at layer N publishes only to the layer N bus, never to a bus at any other
layer. Inter-layer communication between buses occurs exclusively through compositor
relay (SIM-SYS-057) or direct signals (SIM-SYS-060).

**Rationale:** Layer-scoped buses preserve the encapsulation guarantee of the fractal
partition pattern (SIM-SYS-004). The outer layer sees its partitions as opaque units —
it does not observe messages from sub-partitions and cannot depend on a partition's
internal decomposition. A physics partition that decomposes into atmosphere,
aerodynamics, and gravity at layer 1 looks identical on the layer 0 bus to a monolithic
physics partition. This ensures that replacing a partition's internal structure does not
change the messages visible on the outer bus, maintaining the replaceability guarantee.
Independent transport modes per layer allow deployment flexibility — the layer 0 bus can
use network transport for distributed execution while a layer 1 bus uses in-process
transport for tightly coupled sub-partitions.

**Verification Expectations:**
- Pass: A compositor at layer 1 creates and owns a bus instance that connects its
  sub-partitions; this bus is distinct from the layer 0 bus.
- Pass: A sub-partition at layer 1 publishes a typed message that is received by its
  peers on the layer 1 bus but is not visible on the layer 0 bus.
- Pass: The layer 0 bus and a layer 1 bus operate with different transport modes in the
  same simulation run without interference.
- Fail: A sub-partition at layer 1 or deeper holds a reference to or publishes directly
  on a bus at a different layer.
- Fail: Replacing a partition's internal decomposition (adding or removing
  sub-partitions) changes the set of messages visible on the outer bus.

---

### SIM-SYS-056 — Compositor Runtime Role

**Statement:** The compositor at each layer shall be active at runtime, not only at
assembly time. Its runtime responsibilities shall include: (a) driving partition
execution by calling lifecycle trait methods (`init`, `step`, `shutdown`) on each
partition in a controlled order; (b) owning the layer's bus instance and publishing
shared context — WorldState, execution state, environment context — as typed messages
on that bus; (c) receiving and arbitrating requests from partitions on its bus, acting
as the single owner for shared state machines at that layer (SIM-SYS-046); (d) relaying
inter-layer requests to the outer bus according to its relay authority (SIM-SYS-057);
and (e) catching and handling faults from its partitions (SIM-SYS-058).

**Rationale:** The existing specification describes the compositor's assembly-time role
(selecting and instantiating partition implementations) but leaves its runtime
communication responsibilities implicit. Making these explicit is necessary because the
compositor sits at the boundary between layers: it is simultaneously the bus owner for
its inner layer (role a–c) and a partition on the outer layer (role d). Without a
defined runtime role, the compositor's responsibilities for relay, fault handling, and
downward data broadcast are ambiguous. The principled split is: trait calls for
imperative lifecycle control (the compositor controls execution order), bus broadcast for
shared context (partitions subscribe to typed messages regardless of source), and bus
requests for upward communication (partitions emit, compositor arbitrates).

**Verification Expectations:**
- Pass: The compositor calls `step()` on each partition in a defined order; partitions
  do not self-schedule.
- Pass: Shared context (WorldState, execution state) is available to partitions as typed
  messages on the layer's bus, published by the compositor.
- Pass: A partition consumes shared context using the same bus subscription mechanism it
  uses for peer data — there is no distinct mechanism for compositor-originated vs.
  peer-originated data.
- Pass: The compositor receives `ExecutionStateRequest` messages on its bus and
  arbitrates them as the single owner at its layer.
- Fail: A compositor is inert after assembly — it does not participate in runtime
  communication or bus management.
- Fail: Shared context is passed to partitions exclusively through trait method
  arguments, preventing uniform bus-based consumption.

---

### SIM-SYS-057 — Compositor Relay Authority

**Statement:** When a compositor receives a request on its inner bus that is relevant to
the outer layer — such as an `ExecutionStateRequest` that must reach the orchestrator —
the compositor shall act as a relay gateway. The compositor shall have full authority
over what crosses its layer boundary: it may relay the request as-is, transform it
(adding context, changing the request type), suppress it (handling internally without
forwarding), or aggregate multiple requests from a single tick into one consolidated
message. Inter-layer requests propagate through the relay chain — each compositor in the
chain independently decides whether to relay — until the request reaches the layer 0
orchestrator for final arbitration.

**Rationale:** Compositor relay authority preserves the encapsulation guarantee: the
outer layer sees only what the compositor's contract promises, regardless of internal
events. Without relay authority, any inner request would automatically appear on the
outer bus, creating an implicit dependency on the partition's internal structure. A
compositor that relays everything transparently is making a deliberate choice, not
following a default. The relay chain also provides a natural audit point at each layer
boundary, and allows compositors to handle certain situations locally (e.g., falling back
to an alternative sub-partition implementation) without propagating to the outer layer.

**Verification Expectations:**
- Pass: A sub-partition emitting `ExecutionStateRequest::Stop` on a layer 1 bus causes
  the compositor to receive the request and emit a corresponding request on the layer 0
  bus; the orchestrator arbitrates the relayed request.
- Pass: A compositor suppresses an internal request (e.g., by switching to a fallback
  implementation) without any message appearing on the outer bus.
- Pass: A compositor transforms a request before relaying — for example, adding context
  about which sub-partition originated the request.
- Pass: The relay chain operates correctly through multiple layers: a layer 2 request
  relayed by the layer 1 compositor, then relayed by the layer 0 partition, reaches the
  orchestrator.
- Fail: A request from a sub-partition appears on the outer bus without passing through
  the compositor's relay logic.
- Fail: The compositor is a transparent pipe with no ability to filter, transform, or
  suppress inter-layer messages.

---

### SIM-SYS-058 — Compositor Fault Handling

**Statement:** When a sub-partition faults during any trait method call — including
`step()`, `init()`, `shutdown()`, `contribute_state()`, and `load_state()` — by
returning an error, panicking, or timing out, the compositor at that layer shall catch
the fault, log it with the faulting sub-partition's identity and layer depth, and
propagate the error to the outer layer by returning an error from the compositor's own
trait method call. The error shall include the compositor's context (which sub-partition
faulted, during which operation) but the failure itself shall not be suppressed — it
cascades through the compositor chain until the orchestrator receives it and stops the
simulation with a clear diagnostic. There shall be no fault-specific bus channel or
message type; the compositor's error return from its own trait call is the propagation
mechanism.

**Rationale:** A sub-partition fault means the simulation is in an invalid state.
Silently continuing without a failed physics sub-model produces incorrect results;
silently omitting a partition's state from a snapshot produces a snapshot that loses
data on reload; running after a failed `init()` means the system was never correctly
assembled. The compositor's role in fault handling is to catch raw panics (preventing
undefined behavior from escaping), add diagnostic context (which partition, which layer,
which operation), and propagate a clean error — not to absorb the failure. The outer
layer sees the compositor's error return, not the raw panic, preserving encapsulation of
the internal structure while ensuring the failure is visible. A separate fault channel
or fault message type would create a parallel communication path with its own transport,
relay, and arbitration semantics — complexity that is unnecessary because the
compositor's call-and-return relationship with the outer layer already provides an
error propagation path. Graceful degradation (e.g., falling back to an alternative
partition implementation on fault) is not included — there is no current use case that
requires it. If one arises, a dedicated requirement can be added without weakening the
cascade guarantee defined here.

**Verification Expectations:**
- Pass: A sub-partition returning an error from `step()` is caught by the compositor;
  the compositor returns an error from its own `step()` call, which cascades to the
  orchestrator and stops the simulation.
- Pass: A sub-partition panicking during `init()` is caught by the compositor; the
  compositor returns an error from its own `init()` call, preventing the simulation
  from starting.
- Pass: A sub-partition returning an error from `contribute_state()` is caught by the
  compositor; the compositor returns an error from its own `contribute_state()` call,
  and the dump operation fails with a diagnostic identifying the faulting sub-partition.
- Pass: The error returned by the compositor includes context identifying the faulting
  sub-partition's identity, layer depth, and the operation that faulted.
- Pass: The compositor logs the fault with full details regardless of whether the outer
  layer also logs it.
- Fail: A sub-partition fault is silently absorbed by the compositor and the simulation
  continues running with missing or degraded partition output.
- Fail: A sub-partition fault propagates as a raw panic or unhandled exception without
  compositor context wrapping.
- Fail: A dedicated fault bus or fault message channel exists alongside the regular bus.

---

### SIM-SYS-059 — Recursive State Contribution

**Statement:** A partition that is itself a compositor shall implement the state
contribution contract (SIM-SYS-044) by recursively invoking `contribute_state()` on its
sub-partitions and assembling their contributions into a nested TOML fragment. The
compositor's contribution shall include both its own internal state and the nested
contributions of its children, preserving the hierarchical structure. Loading a snapshot
shall reverse the process: the compositor shall decompose its fragment section along the
same boundaries used for assembly and delegate each sub-section to the corresponding
sub-partition's `load_state()` implementation. The outer layer shall receive one
contribution per partition — it shall not observe the internal decomposition.

**Rationale:** State snapshots must capture state at every layer without breaking
encapsulation. If the orchestrator needed to know about sub-partitions to collect their
state, replacing a partition's internal structure would require modifying the
orchestrator. Recursive delegation through compositors ensures that each layer handles
its own collection and assembly. Nested TOML fragments preserve compatibility with the
`extends` and override mechanisms (SIM-SYS-024, SIM-SYS-025) — a snapshot fragment can
be used as a configuration input, and state values at any depth can be overridden using
the same composition semantics as all other configuration.

**Verification Expectations:**
- Pass: A compositor at layer 1 calls `contribute_state()` on each of its sub-partitions
  and assembles the results into a nested TOML fragment under its partition's key.
- Pass: The orchestrator receives one state contribution per layer 0 partition; it does
  not observe whether the contribution was assembled from sub-partitions or produced
  monolithically.
- Pass: A state snapshot captured from a compositor partition, when loaded via
  `load_state()`, correctly restores the state of each sub-partition through recursive
  decomposition.
- Pass: The nested fragment structure is compatible with `extends` inheritance — a
  fragment that extends a snapshot can override a specific sub-partition's state value.
- Fail: The orchestrator must enumerate or know about sub-partitions to capture a
  complete state snapshot.
- Fail: State contributions from sub-partitions are flat key-value pairs rather than
  nested TOML sections, breaking composition fragment compatibility.

---

### SIM-SYS-060 — Direct Signals

**Statement:** A contract crate at any layer may declare a set of **direct signal
types** — safety-critical signals that bypass the compositor relay chain within that
contract crate's jurisdiction and reach the orchestrator that owns that contract crate
directly, regardless of the emitting partition's depth within the hierarchy. Direct
signals are scoped to the contract crate that declares them: signals declared in
`sim-core` reach universe's orchestrator; they do not propagate beyond universe's
boundary when universe is embedded as a partition in an outer system. An outer system
that embeds universe may declare its own direct signals in its own contract crate,
independent of universe's internal signals. Any partition within the declaring contract
crate's hierarchy may emit a declared direct signal. Direct signals shall carry minimal
payload: a signal type identifier, a reason string, and the identity of the emitting
partition. Every direct signal emission shall be logged with the emitting partition's
identity and layer depth. The set of direct signal types at any layer shall be small,
stable, and reserved for scenarios where compositor relay suppression would be unsafe
(e.g., emergency stop, hardware fault).

**Rationale:** The compositor relay chain (SIM-SYS-057) is the correct default for
inter-layer communication — it preserves encapsulation and gives each compositor control
over what crosses its boundary. However, for safety-critical signals, the cost of a
compositor inadvertently suppressing or delaying a relay exceeds the cost of bypassing
encapsulation. A hardware fault or emergency condition detected deep in the hierarchy
must reach the responsible orchestrator without depending on every compositor in the
chain making the correct relay decision. Direct signals are the escape hatch: declared
sparingly, constrained to minimal payload, and audited. They complement the relay chain
rather than replacing it — normal inter-layer communication uses compositor relay,
safety-critical communication uses direct signals. Scoping direct signals to the
declaring contract crate's jurisdiction preserves encapsulation at the embedding
boundary: when universe is a partition in an outer system, its internal direct signals
are an implementation detail invisible to the outer system. The outer system defines its
own safety mechanisms in its own contract crate. Universe communicates the outcome of
internal emergencies through its contract interface on the outer bus, just as any other
partition would.

**Verification Expectations:**
- Pass: A sub-partition at layer 2 emitting `DirectSignal::EmergencyStop` causes
  universe's orchestrator to receive the signal without any intermediate compositor
  relay step.
- Pass: When universe is embedded as a partition in an outer system, a direct signal
  declared in `sim-core` reaches universe's orchestrator but does not appear on the
  outer system's bus. The outer system learns of the event only through universe's
  contract interface (e.g., a status change or request on the outer bus).
- Pass: An outer system declares its own direct signal types in its own contract crate,
  independent of `sim-core`'s direct signal types.
- Pass: Every direct signal emission is logged with the emitting partition's identity
  and layer depth.
- Pass: A direct signal carries only a type identifier, reason string, and emitter
  identity — not arbitrary data payloads.
- Fail: A compositor at an intermediate layer intercepts or suppresses a direct signal
  within the declaring contract crate's hierarchy.
- Fail: A direct signal declared in `sim-core` propagates beyond universe's boundary
  when universe is embedded as a partition in an outer system.
- Fail: Direct signals are used for non-safety-critical communication that could be
  handled through the normal compositor relay chain.
- Fail: The set of direct signal types is large or changes frequently, indicating misuse
  as a general communication mechanism.

---

### SIM-SYS-062 — Compositor Tick Lifecycle

**Statement:** The compositor at each layer shall execute each simulation tick as a
three-phase lifecycle. All transport modes shall enforce this lifecycle identically. A
partition that is itself a compositor shall execute its own complete three-phase tick
lifecycle within the outer compositor's Phase 2 `step()` call for that partition —
the fractal structure nests tick lifecycles recursively.

**Phase 1 — Pre-tick processing (between tick N-1 and tick N):**

1. Check for pending direct signals (SIM-SYS-060) and process them.
2. Process pending spawn and despawn requests (SIM-SYS-010). Spawned vehicles become
   active; despawned vehicles are removed and their resources released.
3. Process pending dump and load requests (SIM-SYS-045). Dump invokes
   `contribute_state()` on all partitions using post-tick-N-1 state. Load replaces
   partition state via `load_state()`.
4. Assemble WorldState from tick N-1 partition outputs and publish it on the bus
   (SIM-SYS-009). Swap the read/write buffers: the read buffer now contains tick N-1
   outputs; the write buffer is cleared to receive tick N outputs.
5. Publish ExecutionState and shared context on the bus.

**Phase 2 — Partition stepping:**

The normative execution model is sequential: for each partition in the compositor's step
order (a deterministic ordering defined by the compositor and stable across ticks; the
mechanism by which the compositor determines this order is implementation-defined):

1. The partition reads inter-partition messages from the read buffer (tick N-1 outputs).
   A message published by partition A during tick N-1 is visible; a message published by
   partition A during the current tick N is not visible to any other partition until tick
   N+1.
2. The compositor calls `partition.step(dt)`. The partition writes its outputs to the
   write buffer (tick N outputs).
3. After each partition's `step()` returns, the compositor checks for pending direct
   signals and processes them before stepping the next partition.

A compositor may step partitions concurrently as an optimization (e.g., under async or
network transport where sequential stepping would impose unnecessary serialization
latency). Concurrent stepping is permitted provided the compositor upholds the following
invariants:

- **Intra-tick message isolation:** Each partition reads from the read buffer and writes
  to the write buffer. No partition observes another partition's current-tick output.
  Write paths shall be isolated — concurrent `step()` calls shall not contend on shared
  write buffer state.
- **Direct signals:** The compositor shall check for pending direct signals at least once
  before Phase 3. Under concurrent stepping the worst-case direct signal latency is the
  duration of the longest partition's `step()` call, rather than one partition's step
  duration under sequential stepping.
- **Request collection:** Bus requests (e.g., `ExecutionStateRequest`) emitted by
  concurrently stepping partitions shall be collected in a thread-safe manner for
  arbitration in Phase 3.
- **Tick barrier:** All partition `step()` calls shall complete before Phase 3 begins.
- **Determinism:** The simulation result shall be identical to that produced by sequential
  stepping in any order — the double-buffered approach guarantees this provided the
  invariants above are upheld.

**Phase 3 — Post-tick processing:**

1. Evaluate all event conditions against the partition state as it existed at the
   beginning of the tick (pre-step state), before any event actions have been applied
   (SIM-SYS-035 through SIM-SYS-039). Collect the set of events whose conditions are
   satisfied. Apply triggered event actions in TOML declaration order. An action's side
   effects are not visible to other event conditions until the following tick.
2. Collect tick N outputs from all partitions.
3. Process bus requests (ExecutionStateRequest, etc.) received during this tick.
   Arbitrate conflicting requests per SIM-SYS-043.
4. Relay qualified requests to the outer bus per SIM-SYS-057.
5. Check for pending direct signals.

The intra-tick message isolation guarantee — that no partition sees another partition's
current-tick output during Phase 2 — shall hold regardless of step order, thread
scheduling, and transport mode. This eliminates ordering sensitivity: the simulation
result is identical regardless of which partition steps first.

**Rationale:** The specification defines what is communicated between partitions but
prior to this requirement did not fully define when communication becomes visible
relative to the step loop. Under in-process synchronous transport, the compositor's
sequential `step()` calls impose a natural ordering that makes visibility implicit. Under
async and network transport, sequential stepping would impose unnecessary serialization
latency — particularly over a network, where each sequential `step()` call incurs a
round-trip. Without an explicit tick lifecycle, different transport implementations make
different choices about message visibility, producing different simulation results —
violating the transport independence guarantee (SIM-SYS-005).

The double-buffered approach (partitions read from tick N-1, write to tick N) is the
simplest model that eliminates ordering sensitivity. No partition sees another's
current-tick output, so the result is the same whether partitions step sequentially,
concurrently, or in any order. This makes the transport independence guarantee trivially
enforceable for intra-tick data flow and also makes concurrent stepping safe as an
optimization — the double-buffer provides isolation without requiring locks on the read
path. The tick barrier (all partitions complete before WorldState assembly) ensures
temporal consistency for WorldState (SIM-SYS-009) and state dumps (SIM-SYS-045).

The sequential model is normative because it is simplest to reason about and implement
correctly. Concurrent stepping is an allowed optimization because the double-buffered
design already provides the isolation needed — an implementation that upholds the stated
invariants produces identical results regardless of whether partitions step sequentially
or concurrently. This allows compositors to choose the strategy appropriate to their
transport mode: sequential for in-process transport (simple, no threading overhead),
concurrent for network transport (avoids serialized round-trips).

Under sequential stepping, direct signal polling between partition steps (SIM-SYS-060)
gives safety-critical signals a worst-case latency of one partition's step duration.
Under concurrent stepping, the worst case is the longest partition's step duration.
Both are bounded and both avoid reentrancy hazards — the compositor checks signals while
no partition state is being mutated.

Snapshot evaluation of event conditions (Phase 3, step 1) eliminates event ordering
sensitivity: reordering event entries in TOML does not change which events fire, only
the order in which their actions are applied. This is consistent with the double-buffered
approach — just as partitions see a snapshot of inter-partition data, events see a
snapshot of partition state.

**Verification Expectations:**
- Pass: Two partitions exchanging data via the bus produce identical simulation results
  regardless of step order, when the step order is varied between test runs.
- Pass: A message published by partition A during tick N is not visible to partition B
  during tick N; partition B reads A's tick N output during tick N+1.
- Pass: Under async transport, the compositor waits for all partition `step()` calls to
  complete before assembling WorldState.
- Pass: Under sequential stepping, direct signals are checked at least once between
  each pair of partition `step()` calls; the worst-case signal latency is one partition's
  step duration. Under concurrent stepping, direct signals are checked at least once
  before Phase 3; the worst-case latency is the longest partition's step duration.
- Pass: Event conditions are evaluated against pre-step state; an event action that
  modifies a signal does not cause a second event conditioned on that signal to fire in
  the same tick.
- Pass: Spawn and despawn requests are processed in Phase 1; all partitions in the
  subsequent tick see the updated vehicle set.
- Pass: Dump requests during Running are processed in Phase 1 at a tick boundary; all
  contributed state corresponds to the same completed tick.
- Pass: The same scenario produces identical results under synchronous, async, and
  network transport modes (within floating-point determinism limits).
- Fail: A partition reads another partition's output from the current tick during
  Phase 2.
- Fail: Event conditions are evaluated after event actions have been applied, causing
  cascading event firing within a single tick.
- Fail: WorldState is assembled before all partitions have completed their current tick.

---

## 7. Multi-vehicle Operation

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
message shall be available to all partitions via the active transport. WorldState shall
be published as an atomic, temporally consistent snapshot — all entries shall correspond
to the same completed tick (see SIM-SYS-062). A consumer shall never observe a partially
assembled WorldState regardless of transport mode.

**Rationale:** GN&C algorithms for formation flying, collision avoidance, and
cooperative tasking require awareness of peer vehicle states. A broadcast `WorldState`
provides this without requiring point-to-point subscriptions between individual GN&C
instances. Atomicity and temporal consistency are necessary to satisfy the transport
independence guarantee (SIM-SYS-005); without them, consumers under async transport
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

### SIM-SYS-010 — Dynamic Vehicle Lifecycle

**Statement:** The system shall support spawning and despawning of vehicle instances
during an active simulation session. Spawning shall load the specified plant model and
GN&C plugin and initialize the vehicle to provided initial conditions. Despawning shall
cleanly unload all associated resources, ensuring no code from the unloaded plugin
remains executing or reachable at the time of unload. Spawn and despawn requests shall
be processed at tick boundaries (see SIM-SYS-062, Phase 1). All partitions within a
tick shall observe the same set of active vehicles.

**Rationale:** Dynamic vehicle lifecycle enables scenarios in which aircraft launch,
complete their mission, and recover or are destroyed, without requiring a full simulation
restart. It also supports incremental scenario construction during interactive use.
Processing spawn and despawn at tick boundaries (SIM-SYS-062) ensures that all partitions
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

## 8. Physics and Environment

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

## 9. GN&C Plugin Interface

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

## 10. Vehicle Definition

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

## 11. Visualization

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

## 12. User Interface

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

## 13. Configuration and Composition

---

### SIM-SYS-023 — TOML Composition Fragments

**Statement:** The system shall use TOML files as its primary runtime configuration
interface. Each file is a composition fragment at a specific scope within the fractal
structure. A layer 0 fragment (session) specifies simulation parameters (timestep,
transport mode) and references one or more layer 1 fragments (scenarios). A scenario
fragment specifies mission content: vehicle definitions, physics sub-model selections,
environment models, events, initial conditions, active visualization plugins, and active
UI panels. Composition fragments at narrower scopes (vehicle definitions, physics
presets, visualization presets) follow the same structure. An outer-layer fragment may
inline or override any field that would otherwise be defined by an inner-layer fragment
it composes; this allows a session fragment to fully define scenario content inline
without referencing separate scenario files, or to override individual fields of a
referenced scenario. The system shall accept a session fragment as its entry point.
When no session fragment is specified, `sim-app` shall load a default session. When a
path to a session fragment is provided as a command-line argument, `sim-app` shall use
it instead of the default.

**Rationale:** A human-readable, file-based configuration surface separates operational
intent from implementation. Structuring configuration as composition fragments at every
layer — rather than a single monolithic file — mirrors the fractal partition pattern:
the same TOML structure, override semantics, and inheritance rules apply at every scope.
Sessions, scenarios, vehicle definitions, and sub-model presets are all portable
artifacts that can be independently version-controlled, exchanged between students and
laboratories, and reused across different compositions without access to simulator
source code.

**Verification Expectations:**
- Pass: `sim-app` launched with no arguments loads the default session and executes a
  complete simulation run.
- Pass: `sim-app path/to/custom-session.toml` loads the specified session fragment
  instead of the default and executes a complete simulation run.
- Pass: A session fragment referencing a scenario fragment by path correctly composes
  the scenario's vehicle definitions, environment models, and events into the session.
- Pass: A session fragment that inlines all scenario content (vehicle definitions,
  environment models, events, initial conditions) without referencing any external
  scenario fragment executes a complete simulation run.
- Pass: A session fragment that references a scenario fragment and overrides a single
  field (e.g., a vehicle's initial altitude) produces a simulation that matches the
  referenced scenario in all respects except the overridden field.
- Pass: Two operators on different machines produce identical initial physics states
  from the same session and scenario fragments (excluding non-deterministic transport
  effects).
- Fail: Any simulation parameter that is documented as configurable requires a source
  code change or environment variable to override.
- Fail: Configuration at any scope requires a format or structure distinct from the
  TOML composition fragment format used at other scopes.

---

### SIM-SYS-024 — Composition Fragment Inheritance

**Statement:** Any TOML composition fragment shall support an `extends` field whose
value is a path to a base fragment of the same scope. Fields present in the inheriting
fragment shall override the corresponding fields in the base. Fields absent from the
inheriting fragment shall be inherited unchanged from the base. This mechanism shall be
available at every scope: sessions, scenarios, vehicle definitions, and sub-model
presets. Additionally, an outer-layer fragment shall be able to override fields defined
by any inner-layer fragment it composes: a session may override scenario fields, a
scenario may override vehicle or physics preset fields, and so on to arbitrary depth.

**Rationale:** Variants of a composition fragment at any scope are typically small diffs
against a common base — a scenario that differs only in wind model, a session that
differs only in transport mode, a vehicle definition that differs only in initial
conditions. Inheritance allows these variants to be expressed minimally, keeping them
automatically synchronized with base fragment changes and reducing the maintenance
burden of configuration libraries. Cross-layer overrides allow outer fragments to
customize inner fragments without forking them: an instructor can distribute a standard
scenario and have each session override only the parameters relevant to that session,
or a single session file can fully define all content without separate fragment files.
Consistent with the fractal partition pattern, the same inheritance and override
mechanism applies at every layer and scope.

**Verification Expectations:**
- Pass: A scenario fragment containing only `extends = "base.toml"` and a single
  overridden field produces a simulation that differs from the base scenario only in
  the overridden field's effect.
- Pass: A session fragment extending a base session inherits the base session's scenario
  references and simulation parameters, overriding only specified fields.
- Pass: A session fragment that references a scenario by name and overrides a physics
  sub-model within that scenario produces a simulation where only the overridden
  sub-model differs from the named scenario's defaults.
- Pass: Modifying a shared field in a base fragment is reflected in all inheriting
  fragments without modifying the inheriting files.
- Fail: A circular `extends` chain (A extends B extends A) is silently accepted; the
  system shall detect and report it as a configuration error.
- Fail: The `extends` mechanism is available only for scenario fragments; session or
  vehicle definition fragments require a different override mechanism.

---

### SIM-SYS-025 — Named Composition Fragments

**Statement:** Any composition fragment reference within a TOML file shall be
expressible as either an inline table or a named reference resolvable to a fragment file
in a configurable directory. Named fragments shall be applicable at any scope within the
fractal structure — physics sub-model selections (e.g., `"low"`, `"mid"`, `"high"`),
visualization plugin sets, GN&C configurations, vehicle definitions, or scenarios.
Inline overrides shall be supported to modify individual fields within a named fragment
without defining an entirely new fragment (via the inheritance mechanism of SIM-SYS-024).

**Rationale:** Named composition fragments are a natural consequence of the fractal
partition pattern: since every scope composes independently replaceable sub-components
via the same compositor mechanism, a reusable named selection of those sub-components is
useful at every layer and in every domain. Named fragments give instructors a stable,
communicable vocabulary (e.g., "run your controller against the mid scenario") while
inline overrides allow individual field substitution without creating a new fragment.
Because the naming and override mechanism is the same at every scope, there is no
domain-specific preset system — physics presets, visualization presets, vehicle
templates, and scenario variants are all composition fragments.

**Verification Expectations:**
- Pass: A vehicle referencing `physics = "mid"` produces the same behavior as a vehicle
  with an inline table containing the fields defined in the corresponding named fragment
  file.
- Pass: An inline override of a single field (e.g., `wind = "dryden"`) in an otherwise
  `"mid"` physics fragment activates only that sub-model difference, leaving all other
  sub-models selected by the fragment unchanged.
- Pass: A visualization fragment (e.g., `[viz] preset = "minimal"`) selects a specific
  set of visualization plugins, and inline overrides add or remove individual plugins
  without replacing the entire fragment.
- Pass: A session referencing `scenario = "case-2"` resolves to the corresponding
  named scenario fragment file and composes its contents into the session.
- Fail: The system accepts an unrecognized fragment name without reporting an error.
- Fail: The named fragment mechanism is available only at specific scopes; other scopes
  require a different mechanism to achieve named sub-component selections.

---

### SIM-SYS-044 — State Snapshot as Composition Fragment

**Statement:** The system shall support capturing the complete simulation state at any
point during execution and emitting it as a TOML composition fragment. The contract
crate for each layer shall define a state contribution contract that each partition at
that layer implements. The orchestrator shall assemble the complete snapshot by invoking
each partition's state contribution implementation and composing the results into a
single fragment. A partition that is itself a compositor shall implement the state
contribution contract by recursively invoking `contribute_state()` on its sub-partitions
and assembling their contributions into a nested TOML fragment (see SIM-SYS-059). Loading
a snapshot shall use the same contract in reverse: the orchestrator decomposes the
fragment and passes each partition's section to that partition's load implementation; a
compositor partition decomposes its section further and delegates to its sub-partitions.
The resulting snapshot fragment shall be a valid composition fragment loadable by the same
mechanism used for sessions and scenarios (SIM-SYS-023). The snapshot shall include the
current simulation time, execution state, and the complete internal state of each
partition for all active vehicles. The snapshot fragment shall be human-readable,
inspectable, and editable using standard text tools.

**Rationale:** State persistence is a natural extension of the composition fragment
mechanism rather than a separate system. Initial conditions are already composition
fragment content — a state snapshot is a complete set of initial conditions captured at
a specific simulation time. By expressing snapshots as composition fragments, they
inherit all existing fragment capabilities: `extends` inheritance (SIM-SYS-024), named
references (SIM-SYS-025), inline overrides, and cross-layer override semantics. An
operator can extend a snapshot and override a single vehicle's position, or reference a
named snapshot in a session fragment. Defining the state contribution contract in the
contract crate (consistent with SIM-SYS-003) ensures that all partitions serialize and
deserialize state through a uniform interface rather than each inventing its own
mechanism. The orchestrator coordinates dump and load without needing knowledge of any
partition's internal state structure — it invokes the contract and composes or
decomposes the resulting fragment sections. Recursive delegation through compositors
ensures that the snapshot captures state at every layer without the orchestrator needing
to know the internal decomposition of any partition.

**Verification Expectations:**
- Pass: A state snapshot captured during a Running simulation produces a valid TOML file
  that parses without error as a composition fragment.
- Pass: The snapshot fragment, when loaded as the basis for a new session, produces a
  simulation whose initial state matches the captured state (within floating-point
  determinism limits).
- Pass: A snapshot fragment supports the `extends` field: a fragment containing only
  `extends = "snapshot.toml"` and a single overridden vehicle position produces a
  simulation that differs from the snapshot only in the overridden vehicle's initial
  position.
- Pass: Each partition's state section in the snapshot uses the same TOML structure as
  the corresponding partition's configuration section in a scenario fragment.
- Pass: A snapshot fragment is usable as a named fragment: `state = "checkpoint-3"`
  resolves to a snapshot file and loads correctly.
- Fail: State snapshots are emitted in a binary or opaque format that cannot be parsed
  as a TOML composition fragment.
- Fail: Loading a snapshot requires a mechanism distinct from the composition fragment
  loader used for sessions and scenarios.
- Pass: The state contribution contract is defined in the contract crate for the layer;
  no partition defines its own serialization interface for state snapshots.
- Pass: An alternative partition implementation that conforms to the state contribution
  contract produces a snapshot whose section is loadable by the orchestrator without
  modification to the orchestrator or any other partition.
- Fail: The snapshot omits partition state that would be necessary to reproduce the
  captured simulation state on reload.
- Fail: A partition implements state dump or load through a mechanism other than the
  contract defined in the layer's contract crate.

---

### SIM-SYS-045 — State Dump and Load Operations

**Statement:** The system shall provide operations to dump and load simulation state.
A dump operation shall capture the current state and write it to a specified file path
as a composition fragment (SIM-SYS-044). A load operation shall accept a state snapshot
composition fragment and restore the simulation to the captured state, replacing the
current state of all partitions and vehicles. Dump shall be invocable during any
execution state (Running, Paused, Stopped). Load shall be invocable from the Paused or
Stopped execution states; loading while Running shall not be supported. When the
simulation is Running, dump shall be processed at a tick boundary (see SIM-SYS-062,
Phase 1) so that all partition contributions correspond to the same completed tick.
Both operations shall be requestable via the simulation bus, the UI partition, and the
event system (as event actions), consistent with the uniform request mechanisms used for
execution state transitions (SIM-SYS-041, SIM-SYS-042).

**Rationale:** Dump and load are the operational interface to state snapshots. Dump
during a Running simulation enables capturing transient conditions without pausing;
dump while Paused or Stopped enables deliberate checkpointing. Restricting load to
non-Running states prevents mid-tick state corruption. Processing dump at tick boundaries
ensures temporal consistency across transport modes. Making dump and load available
through the same request channels as execution state transitions — bus messages, UI
controls, and event actions — keeps the operational surface uniform. An event action
`"state_dump"` with a path parameter allows scenario authors to script automatic
checkpoints at specific times or conditions without UI interaction.

**Verification Expectations:**
- Pass: A dump operation invoked while the simulation is Running produces a valid
  snapshot fragment without interrupting physics integration.
- Pass: A dump captured during Running contains state from a single completed tick —
  all partition contributions correspond to the same simulation time.
- Pass: A dump operation invoked while Paused produces a snapshot fragment, and
  loading that fragment in a new session produces identical initial state.
- Pass: A load operation invoked while Paused replaces all vehicle and partition state
  with the snapshot's state; on resume, the simulation continues from the loaded state.
- Pass: An event defined as `action = "state_dump"` with `path = "checkpoint.toml"`
  triggers at the specified condition and produces a snapshot file at the given path.
- Pass: A load request emitted while the simulation is Running is rejected (logged and
  ignored), and the simulation continues unaffected.
- Fail: Dump or load requires a dedicated API distinct from the bus message and event
  action mechanisms used for execution state control.
- Fail: A dump captured during Running contains partition states from different
  simulation ticks.

---

## 14. Distribution and Packaging

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
  internal partition crates (`sim-core`, `sim-physics`, `sim-gnc`, `sim-viz`, `sim-ui`)
  are not required by callers.
- Pass: Universe's library interface does not require callers to initialize
  framework-specific infrastructure (e.g., an ECS world, a rendering context) that
  universe manages internally.
- Fail: Universe's library interface assumes it is the process entry point or prevents
  the caller from performing work before or after the simulation session.

---

## 15. Telemetry

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

## 16. Events

---

### SIM-SYS-035 — Event System Architecture

**Statement:** The system shall provide a unified event system in which discrete events
can be defined, armed, triggered, and handled at every layer of the fractal partition
hierarchy. The event mechanism — trigger types, arming lifecycle, TOML schema, and
evaluation semantics — shall be identical at every layer and in every partition, consistent
with the fractal partition pattern (SIM-SYS-004). The semantic meaning of events — what
actions they invoke and what domain concerns they express — shall vary by layer and
partition. Each layer's contract crate defines the action vocabulary available to events
scoped at that layer (see SIM-SYS-061).

**Rationale:** The fractal partition pattern requires that structural primitives be
uniform in kind across all layers. The event primitive is no exception: its mechanism
(trigger + action + parameters, declaratively defined in TOML) is the same everywhere.
But just as the bus carries different typed messages at different layers, and contracts
specify different behavioral obligations at different layers, events express different
domain concerns at different layers. Layer 0 events address infrastructure and execution
lifecycle (telemetry snapshots, health checks, execution state transitions). Layer 1
events address mission and domain concerns (scenario phase transitions, failure
injection, mode changes). Layer 2+ events address subsystem-internal concerns
(model transitions, convergence thresholds). The event mechanism is the uniform
primitive; the action vocabulary is the layer-scoped semantic content.

**Verification Expectations:**
- Pass: A system-level event and a partition-level event defined in the same scenario
  both trigger at their specified conditions during a simulation run.
- Pass: A partition-level event defined within the GN&C partition uses the same event
  definition schema and trigger types as a system-level event defined outside any
  partition scope.
- Pass: A partition-level event uses a domain-specific action identifier defined in that
  partition's contract crate, and the action is handled within the partition without
  requiring system-level awareness of the action type.
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

**Rationale:** The execution state machine is the primary instance of the shared state
machine synchronization pattern (SIM-SYS-046) at layer 0. In a fractally partitioned
system, any partition at any layer may need to observe or influence the simulation
lifecycle. Execution state is currently implicit in SIM-SYS-021's description of UI
controls. Without a formal state machine, partitions cannot consistently reason about
valid transitions, detect invalid requests, or synchronize their behavior with the
global execution lifecycle. An explicit, centrally defined state machine prevents
undefined behavior when multiple sources (UI, events, partition logic) attempt to
influence execution state.

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
transition by emitting an `ExecutionStateRequest` message on its layer's bus. The
`ExecutionStateRequest` type shall be defined in `sim-core` and shall carry the requested
transition and the identity of the requesting partition. At layer 0, the orchestrator
(`sim-app`) receives requests directly on the layer 0 bus and is the sole authority for
evaluating and applying state transitions according to the state machine defined in
SIM-SYS-040. At deeper layers, requests emitted on an inner bus are received by the
compositor at that layer, which relays them to the outer bus according to its relay
authority (see SIM-SYS-057). The request propagates through the compositor relay chain
until it reaches the layer 0 orchestrator for arbitration. Requests that represent
invalid transitions shall be logged with the requesting partition's identity and ignored.
Valid transitions shall take effect within one physics tick of receipt at the
orchestrator. Because requests from deeper layers traverse the compositor relay chain,
a request emitted at layer N may take up to N physics ticks to reach the orchestrator.
Safety-critical signals that cannot tolerate relay latency shall use the direct signal
mechanism (SIM-SYS-060) instead.

**Rationale:** The fractal partition pattern (SIM-SYS-004) implies that partitions at
any layer may generate events that affect execution state. Multiple sources may
legitimately need to change execution state: the UI for interactive control, a plant
model detecting a safety limit exceedance, a GN&C event handler reaching a scripted
pause point, or a condition-triggered event firing a stop. Because buses are
layer-scoped (SIM-SYS-055), requests from inner layers reach the orchestrator through
compositor relay rather than direct emission on a global bus. Each compositor in the
relay chain may add context or transform the request (SIM-SYS-057), ensuring that the
outer layer sees only what the compositor's contract promises. The relay chain introduces
latency proportional to layer depth — each compositor processes its inner bus during its
own `step()` and relays on the next outer-bus read cycle. This latency is acceptable for
normal execution state transitions; safety-critical scenarios that require immediate
response use direct signals (SIM-SYS-060), which bypass the relay chain entirely. The
orchestrator remains the single arbitration point for execution state changes.

**Verification Expectations:**
- Pass: A layer 0 physics partition emitting `ExecutionStateRequest::Stop` during a
  Running simulation causes the orchestrator to transition to the Stopped state within
  one physics tick.
- Pass: A layer 0 GN&C partition emitting `ExecutionStateRequest::Pause` during a
  Running simulation causes the orchestrator to transition to the Paused state within
  one physics tick.
- Pass: A layer 1 sub-partition emitting `ExecutionStateRequest::Stop` on the layer 1
  bus causes the compositor to relay the request to the layer 0 bus, where the
  orchestrator arbitrates and transitions to the Stopped state.
- Pass: An `ExecutionStateRequest::Resume` emitted while the simulation is in the
  Running state is logged as an invalid transition and ignored; the simulation continues
  running without interruption.
- Pass: Each applied transition is logged with the identity of the partition that
  requested it.
- Fail: A partition directly mutates the execution state resource without emitting an
  `ExecutionStateRequest` on the bus.
- Fail: A sub-partition at layer 1 or deeper emits an `ExecutionStateRequest` directly
  on the layer 0 bus, bypassing its compositor's relay authority.
- Fail: Two partitions emitting conflicting requests in the same tick causes undefined
  behavior (see SIM-SYS-043 for conflict resolution).

---

### SIM-SYS-042 — Execution State Change as Event Action

**Statement:** `sim-core` shall declare `"sim_pause"`, `"sim_stop"`, and `"sim_resume"`
as event action identifiers in its contract-crate action vocabulary. When an event with
one of these action identifiers fires, the event dispatcher shall emit the corresponding
`ExecutionStateRequest` on the bus at the layer where the event is defined. At layer 0,
this reaches the orchestrator directly. At layer 1 or deeper, the request follows the
compositor relay chain (SIM-SYS-057, SIM-SYS-041) to reach the orchestrator for
arbitration. These actions shall be usable with both time-triggered and condition-
triggered events.

**Rationale:** Execution state transitions are event actions declared in `sim-core`
using the same contract-crate action vocabulary mechanism as any other event action
(SIM-SYS-061). Because every partition in the system depends on `sim-core`, these
actions are available at every layer — the contract-crate dependency graph makes them
visible everywhere. Their handlers emit typed bus messages (`ExecutionStateRequest`)
that enter the arbitration pipeline. Encoding a GN&C pause point, a plant
safety stop, or a scenario phase transition as event actions keeps the logic declarative
and configurable in the scenario TOML, avoiding hard-coded procedural checks in partition
source code. Connecting the event system to the execution state request mechanism
(SIM-SYS-041) ensures all event-driven state changes flow through the same arbitration
path as UI-initiated changes. Because buses are layer-scoped (SIM-SYS-055), event-driven
requests at inner layers reach the orchestrator through the same compositor relay path as
all other inter-layer requests.

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
(2) Pause takes priority over Resume. (3) Among requests of equal priority, the request
shall be designated the primary request for logging purposes using a deterministic,
transport-independent rule (the specific tie-breaking mechanism is
implementation-defined). All valid requests of the same type shall be applied (the
outcome is the same regardless of which is designated primary). All received requests
and the resolution outcome shall be logged with the requesting partition identities.

**Rationale:** In a fractally partitioned system, any partition at any layer may
generate events that request execution state changes. Concurrent events may independently
request conflicting transitions in the same tick — for example, a GN&C event requesting
pause while a physics safety limit simultaneously requests stop. Without a deterministic
resolution policy, the resulting execution state would depend on message ordering, which
varies with transport mode and scheduling. Prioritizing stop over pause reflects the
principle that safety-critical transitions should not be overridden by less severe
requests. Tie-breaking by a deterministic, transport-independent rule rather than
arrival order ensures that audit logs are reproducible across transport modes and
deployment configurations.

**Verification Expectations:**
- Pass: When a physics partition emits `ExecutionStateRequest::Stop` and a GN&C
  partition emits `ExecutionStateRequest::Pause` in the same tick, the orchestrator
  transitions to Stopped.
- Pass: When a UI partition emits `ExecutionStateRequest::Resume` and a GN&C event
  emits `ExecutionStateRequest::Pause` in the same tick, the orchestrator transitions
  to Paused (if currently Paused, the pause wins and the state remains Paused).
- Pass: When two partitions emit `ExecutionStateRequest::Pause` in the same tick, the
  same partition is deterministically logged as the primary request regardless of
  transport mode; the outcome (Paused) is the same regardless of which is designated.
- Pass: The orchestrator log for the tick includes both requests and identifies which
  was applied and which was superseded, with partition identities.
- Pass: The audit log for a given scenario is identical across synchronous, async, and
  network transport modes.
- Fail: Conflicting requests in the same tick produce non-deterministic behavior
  depending on transport mode or thread scheduling.
- Fail: A lower-priority request silently overrides a higher-priority request without
  logging.
- Fail: Equal-priority requests from different partitions produce different audit log
  entries depending on transport mode.

---

### SIM-SYS-061 — Contract-crate-scoped Event Action Vocabulary

**Statement:** All event action identifiers shall be declared in contract crates. The
set of action identifiers available to events at a given scope is the union of actions
declared in the contract crates that the partition at that scope transitively depends on.
Each contract crate declares its action vocabulary as part of its contract interface,
alongside trait definitions and typed message declarations. The event TOML schema —
trigger, action identifier, and parameters — shall be identical regardless of which
contract crate declares the action. Action identifiers shall be validated at
configuration load time: an action identifier used in an event entry must be declared in
a contract crate visible at that event's scope, or the configuration shall be rejected.

There is one mechanism for declaring event actions, one mechanism for dispatching them,
and one TOML schema for defining them. What varies across layers is the action
vocabulary — which actions are available — determined by the contract-crate dependency
graph.

**Rationale:** The fractal partition pattern produces a system where events at different
layers express fundamentally different domain concerns. Layer 0 events address
infrastructure: execution lifecycle, telemetry checkpoints, periodic health checks.
Layer 1 events address mission and domain concerns: scenario phase transitions
(`"stage_separate"`, `"deploy_landing_gear"`), failure injection
(`"inject_sensor_fault"`), mode changes (`"switch_aero_model"`). Layer 2+ events address
subsystem-internal concerns: model transitions, convergence thresholds, internal mode
switches.

Actions declared in `sim-core` — such as `"sim_stop"`, `"sim_pause"`, `"sim_resume"`
(SIM-SYS-042) — are available at every layer because every partition transitively
depends on `sim-core`. Actions declared in a physics contract crate are available to
events within the physics partition because the physics implementation depends on its
contract crate. `sim-core`'s actions are available everywhere for the same reason
`sim-core`'s typed messages are available everywhere — the dependency graph makes them
visible.

This mirrors how all other contract-crate-scoped primitives work. Typed messages are
declared in contract crates and available wherever the dependency graph reaches. Traits
are declared in contract crates and implemented by partitions that depend on those
crates. Event actions follow the same pattern: declared in contract crates, available
wherever the dependency graph reaches, handled by the partition whose dispatcher owns
that scope.

Without contract-crate scoping, the event system must either restrict actions to a
hardcoded execution-state set — forcing partitions to encode domain logic procedurally
rather than declaratively — or maintain a single global action registry that couples all
partitions to each other's domain vocabulary. Contract-crate scoping avoids both: each
contract crate owns its action namespace, the dependency graph determines visibility,
and no global coordination is needed.

**Verification Expectations:**
- Pass: An action (`"sim_stop"`) declared in `sim-core` is usable in a layer 1 partition
  event because the partition transitively depends on `sim-core`.
- Pass: An action (`"inject_sensor_fault"`) declared in the physics contract crate is
  used in a `[[physics.events]]` entry and handled by the physics partition's event
  dispatcher.
- Pass: An action declared in the physics contract crate is rejected at configuration
  load time if used in a `[[gnc.events]]` entry (the GN&C partition does not depend on
  the physics contract crate).
- Pass: Event entries using actions from `sim-core` and actions from a partition's
  contract crate have identical TOML syntax — the same `trigger`, `action`, and
  `parameters` fields.
- Pass: An action's effects on the outer layer are visible only through whatever the
  partition publishes on the outer bus as part of its normal contract obligations — not
  through a separate event propagation mechanism.
- Fail: Event actions are defined in a global registry visible to all partitions,
  creating cross-partition coupling on domain vocabulary.
- Fail: A partition handles domain-specific event logic procedurally in its `step()`
  implementation because the event system does not support contract-crate-scoped actions.
- Fail: Actions declared in `sim-core` use a different declaration mechanism, dispatch
  path, or TOML schema than actions declared in a partition's contract crate.

---

## 17. Verification and Traceability

---

### SIM-SYS-033 — Partition-level Specifications and Documentation Structure

**Statement:** Each partition shall maintain a `docs/` directory whose structure
follows the Diátaxis documentation framework and is uniform across all partitions at
all layers:

| Directory             | Diátaxis Quadrant | Content                                    |
|-----------------------|-------------------|--------------------------------------------|
| `docs/tutorials/`    | Tutorial          | Learning-oriented guided walkthroughs       |
| `docs/how-to/`       | How-to guide      | Task-oriented procedural instructions       |
| `docs/reference/`    | Reference         | Information-oriented technical descriptions |
| `docs/explanation/`  | Explanation       | Understanding-oriented conceptual discussion|
| `docs/design/`       | —                 | `SPECIFICATION.md` and design artifacts     |

The `docs/design/SPECIFICATION.md` shall contain requirements that individually trace
to one or more identifiers in the parent layer's specification. No requirement shall
exist without a parent requirement in the layer above. Where a partition identifies its
own independently replaceable sub-partitions (layer 2), those sub-partitions shall
maintain the same `docs/` structure and their own `SPECIFICATION.md` tracing to the
partition-level specification — perpetuating the fractal partition pattern to arbitrary
depth.

**Rationale:** The fractal partition pattern requires that specification structure and
documentation structure propagate uniformly to every layer of decomposition. A Diátaxis-
aligned `docs/` folder ensures that each partition — regardless of its layer — presents
its documentation in the same four quadrants. A contributor navigating from the system-
level physics partition into its atmosphere sub-partition finds the same documentation
layout, the same specification format, and the same traceability conventions.
Bidirectional traceability between each layer's specification and the layer above ensures
that all intents are verifiably allocated downward and no requirement is orphaned from
its parent.

**Verification Expectations:**
- Pass: Every partition's `docs/` directory contains the five subdirectories listed
  above (directories may be empty if no content exists yet, but the structure is
  present).
- Pass: Every requirement in each partition's `SPECIFICATION.md` includes a `Traces to:`
  field referencing at least one identifier in the parent layer's specification.
- Pass: Every requirement in this document is referenced by at least one partition-level
  requirement.
- Pass: Where a partition defines independently replaceable sub-partitions, each
  sub-partition maintains its own `docs/` directory with the same Diátaxis structure
  and a `SPECIFICATION.md` tracing to the partition-level specification.
- Fail: A partition's `SPECIFICATION.md` contains a requirement with no `Traces to:`
  field.
- Fail: A requirement in any layer's specification exists with no corresponding child
  requirement in any applicable specification at the layer below.
- Fail: A partition's `docs/` structure differs from the Diátaxis layout used by its
  parent or sibling partitions.

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

### SIM-SYS-047 — Contract Tests at Every Layer

**Statement:** Each independently replaceable partition at every layer shall have
**contract tests**: tests that instantiate the partition implementation in isolation,
invoke it through the contract traits defined at its layer, and verify that outputs
conform to the contract's behavioral requirements. Contract tests shall not require
instantiation of peer partitions at the same layer. Where a partition defines
independently replaceable sub-partitions (layer 1 and beyond), each sub-partition shall
have its own contract tests against the sub-partition contract traits, following the
same structure.

**Rationale:** The fractal partition pattern guarantees independent replaceability at
every layer. That guarantee is only verifiable if each partition can be tested in
isolation against its contract — without its peers. Contract tests make the
replaceability guarantee concrete: if an alternative implementation passes the contract
tests, it is a valid replacement. The same testing structure propagates to every layer
because the same contract structure propagates to every layer. Contract tests use the
contract's own input and output types to supply data, not mocks of peer partitions,
ensuring that tests exercise the actual contract boundary and cannot silently diverge
from the real interface.

**Verification Expectations:**
- Pass: For each partition at layer 0, a test exists that instantiates the partition
  implementation, invokes it through `sim-core` traits, and asserts behavioral
  properties — without any other partition crate compiled or instantiated.
- Pass: For each independently replaceable sub-partition at layer 1, a test exists that
  instantiates the sub-partition implementation, invokes it through the partition's
  internal contract traits, and asserts behavioral properties — without sibling
  sub-partitions instantiated.
- Pass: When an alternative implementation of a partition is provided, the same contract
  test suite runs against it without modification and reports pass/fail against the
  contract.
- Fail: A contract test for a partition requires instantiation of a peer partition at
  the same layer (indicating the test is not isolated at the contract boundary).
- Fail: A partition at any layer has no tests that exercise its contract traits in
  isolation.

---

### SIM-SYS-048 — Compositor Tests at Every Layer

**Statement:** Each layer that composes partitions shall have **compositor tests**:
tests that verify the compositor correctly assembles its partitions and that the
assembled partitions interact correctly through their shared contracts. At layer 0, this
means the orchestrator composes the four top-level partitions and they exchange messages
correctly through the bus. At layer 1, this means a partition's internal compositor
assembles its sub-models and they interact correctly through the partition's contract
module. Compositor tests shall assume that lower-layer contract tests pass and shall
focus on composition correctness, not on re-verifying individual partition behavior.

**Rationale:** Contract tests verify individual partitions in isolation. Compositor
tests verify that the compositor's assembly logic — selection, wiring, initialization
ordering — is correct and that the assembled partitions communicate through their
contracts as expected. Without compositor tests, composition bugs (incorrect wiring,
missing initialization, message routing errors) are only caught by expensive system-level
tests where failure localization is difficult. Scoping compositor tests to a single
layer's assembly prevents the common failure mode of integration tests that test
everything at once.

**Verification Expectations:**
- Pass: A layer 0 compositor test composes all four partitions using the orchestrator,
  runs at least one simulation tick, and verifies that inter-partition messages are
  exchanged correctly (e.g., physics produces `PlantState`, GN&C consumes it and
  produces `GNCCommand`).
- Pass: For each partition that decomposes into sub-partitions, a layer 1 compositor
  test composes the sub-partitions using the partition's internal compositor and verifies
  that they interact correctly through the partition's contract module.
- Pass: When a layer 0 compositor test fails but all layer 1 contract tests pass, the
  failure is localizable to inter-partition communication or orchestrator assembly logic.
- Fail: A layer in the system that composes independently replaceable sub-partitions has
  no compositor test.
- Fail: A compositor test re-tests internal behavior of individual partitions rather
  than focusing on their composition and interaction.

---

### SIM-SYS-049 — System Tests Trace to Requirements

**Statement:** System-level tests shall exercise the full simulation stack from session
configuration to final output. Each system test shall trace to one or more SIM-SYS
requirement identifiers. System tests shall use the same entry points available to an
operator or embedder — session-level composition fragments, the orchestrator's public
API, or the command-line interface — and shall not bypass composition or initialization
to reach internal partition interfaces directly.

**Rationale:** System tests verify end-to-end properties that emerge from the
interaction of all layers: a vehicle reaching an expected final state, an event sequence
producing the expected execution state transitions, telemetry output matching a
reference. Requiring traceability to SIM-SYS requirements ensures that coverage analysis
is trivial — every requirement with a system test is verified end-to-end, and
requirements without system tests represent visible gaps. System tests complement, but
do not replace, contract and compositor tests at lower layers.

**Verification Expectations:**
- Pass: Each system test file or test function includes a comment or attribute
  identifying the SIM-SYS requirement(s) it verifies.
- Pass: Every SIM-SYS requirement that specifies observable system behavior has at least
  one system test.
- Pass: System tests use session-level composition fragments as input and assert against
  simulation outputs (final vehicle state, telemetry, event logs) — not against internal
  partition state.
- Fail: A system test bypasses the orchestrator or compositor to directly instantiate
  and invoke partition internals.
- Fail: A SIM-SYS requirement with observable system behavior has no corresponding
  system test.

---

### SIM-SYS-050 — Transport-parameterized Compositor Tests

**Statement:** Layer 0 compositor tests shall be parameterized over transport mode. The
same compositor test scenario shall execute under in-process synchronous, asynchronous
cross-thread, and network-based transport modes, and shall produce identical final
vehicle state (within floating-point determinism limits as specified in SIM-SYS-005).

**Rationale:** Transport independence (SIM-SYS-005) is a correctness constraint that
requires verification across all three transport modes. Making this a parameterized
compositor test — rather than a separate test category — follows from the fractal
partition pattern: transport is a layer 0 composition concern, so transport verification
belongs in layer 0 compositor tests. Layer 1 tests are unaware of transport because
layer 1 partitions are unaware of transport.

**Verification Expectations:**
- Pass: A compositor test runs the same scenario under all three transport modes and
  the final vehicle state matches across modes within floating-point determinism limits.
- Pass: The parameterization requires no changes to partition code or partition-level
  test code — only the transport selection in the session-level composition fragment
  differs.
- Fail: A transport mode produces a different final vehicle state from the other modes
  beyond floating-point determinism limits.
- Fail: Transport-mode testing requires partition-specific test code or partition-aware
  test infrastructure.

---

### SIM-SYS-051 — Test Reference Data Ownership at Contract Boundaries

**Statement:** Each contract shall own the reference data used to verify implementations
against it. Reference data shall consist of two elements: (a) **canonical inputs** —
representative instances of the contract's input types, and (b) **expected output
properties** — invariants, tolerances, and constraints that any conforming implementation
must satisfy for those inputs. Contract tests shall assert against the contract's stated
output properties, not against exact output values captured from a specific
implementation. Where a contract defines numerical tolerances, those tolerances shall be
stated in the contract itself and referenced by the contract tests.

**Rationale:** In a fractal partition system, contracts exist at every layer and tests
exist at every layer. If test reference data is maintained independently of the contracts
it verifies, a contract change at layer N can silently invalidate reference data at
layers N through 0 — each layer's compositor and system tests may assert against stale
expected values without any test failing. By making the contract the single owner of its
reference data, the reference data changes when and only when the contract changes, and
both changes are made by the same author in the same artifact. This eliminates cascade
invalidation for contract tests and bounds the propagation of reference data changes to
the contract boundary where they originate.

**Verification Expectations:**
- Pass: Each contract defines canonical inputs as part of its test support module,
  constructed from the contract's own input types.
- Pass: Each contract defines expected output properties (invariants, tolerances,
  constraints) alongside the canonical inputs, not in a separate golden file.
- Pass: Contract tests assert against the contract-defined output properties, not
  against exact values captured from any particular implementation.
- Pass: When a contract's behavioral requirements change, the canonical inputs and
  expected output properties are updated as part of the same change.
- Fail: A contract test compares outputs against a golden file that is maintained
  separately from the contract definition.
- Fail: A contract's expected output properties reference tolerances or constraints
  not stated in the contract itself.

---

### SIM-SYS-052 — Compositor Tests Assert Compositional Properties

**Statement:** Compositor tests shall assert **compositional properties** — invariants
that must hold when partitions are correctly assembled — rather than exact output values.
Compositional properties include: messages sent by one partition are received by the
intended consumer; conserved quantities (energy, mass, momentum) are preserved across
partition boundaries within stated tolerances; execution ordering respects the declared
dependency graph; and state that one partition publishes is visible to its declared
consumers in the same tick or the next tick as specified by the contract. Where a
compositor test requires regression baselines with exact output values, those baselines
shall be generated mechanically from the current contract-conforming implementations,
not maintained by hand.

**Rationale:** Compositor tests verify composition correctness — wiring, ordering,
message delivery — not the numerical behavior of individual partitions. Asserting exact
output values in compositor tests couples them to every partition's implementation
details, so that a legitimate improvement in any partition invalidates compositor-level
golden files across the layer boundary. Compositional properties are stable across
implementation changes because they derive from the composition structure, not from any
specific partition's output values. Where exact regression baselines are unavoidable,
mechanical generation from the current implementations makes regeneration a deterministic
operation triggered by any partition change, rather than a manual cross-team coordination
effort.

**Verification Expectations:**
- Pass: Each compositor test asserts at least one compositional property (message
  delivery, conservation, ordering, or visibility).
- Pass: Compositor tests do not fail when a partition's internal implementation is
  replaced with an alternative that passes its contract tests.
- Pass: Where regression baselines with exact values exist, a documented generation
  command can regenerate them from the current implementations without manual editing.
- Fail: A compositor test asserts exact output values that were captured from a specific
  implementation and maintained as a hand-edited golden file.
- Fail: Replacing a partition with a contract-conforming alternative causes compositor
  test failures unrelated to composition correctness.

---

### SIM-SYS-053 — System Test Reference Generation

**Statement:** Where system tests require exact end-to-end reference outputs for
requirements traceability, those references shall be generated by a documented,
repeatable process that runs the full stack with known-good implementations and captures
the output. The generation process shall follow the layer structure: when a partition
implementation changes, reference regeneration shall proceed bottom-up — the changed
partition's contract test reference data is verified first, then compositor-level
references are regenerated, then system-level references are regenerated. The
regeneration command, the implementations used, and the version of each contract shall
be recorded alongside the generated reference.

**Rationale:** System tests sometimes require exact reference outputs to verify
requirements traceability (e.g., "a vehicle launched with these initial conditions
reaches this final state"). Hand-maintaining these references across a fractal structure
is fragile — a sub-model improvement at layer 2 changes outputs that propagate through
layer 1 compositor assembly to layer 0 system test references. Mechanical generation
with recorded provenance makes regeneration deterministic and auditable. Bottom-up
ordering ensures that no reference is regenerated against a partition that has not itself
been verified, preserving the diagnostic layering of the test pyramid.

**Verification Expectations:**
- Pass: Each system test that asserts exact output values has a corresponding reference
  file generated by a documented command.
- Pass: Each generated reference file records the generation command, the implementation
  versions used, and the contract versions in effect.
- Pass: After a partition implementation change, running the documented regeneration
  command produces updated references that reflect the change, and the system test passes
  against the new references.
- Fail: A system test reference file has no recorded provenance (no generation command
  or version information).
- Fail: A partition implementation change requires manual editing of system test
  reference files rather than regeneration.

---

### SIM-SYS-054 — Contract Versioning Scopes Reference Data Propagation

**Statement:** When a contract's behavioral requirements change, the change shall be
expressed as a new contract version. Each contract version shall carry its own canonical
inputs and expected output properties (per SIM-SYS-051). Alternative implementations
targeting the previous contract version shall remain testable against that version's
reference data until they migrate to the new version. The contract version boundary
shall be the propagation boundary for reference data changes — implementations targeting
an unchanged contract version shall not require reference data updates.

**Rationale:** The fractal partition pattern supports alternative implementations at
every layer. Without contract versioning, a contract change forces all alternative
implementations to update simultaneously, and their reference data becomes invalid in a
single event. Versioning bounds the propagation: a new contract version creates new
reference data, but implementations targeting the old version continue to use the old
version's reference data until they choose to migrate. This allows alternatives to
migrate on their own schedule while maintaining test coverage throughout the transition.

**Verification Expectations:**
- Pass: Each contract that has undergone a behavioral change maintains version-scoped
  canonical inputs and expected output properties for each supported version.
- Pass: An alternative implementation targeting contract version N passes contract tests
  using version N's reference data, even after version N+1 exists.
- Pass: A contract version change does not cause test failures in implementations
  targeting the previous version.
- Fail: A contract behavioral change invalidates reference data for implementations
  that have not migrated to the new contract version.
- Fail: A contract carries canonical inputs and expected output properties that are not
  scoped to a specific contract version.

---

## 18. Requirements Traceability Matrix

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
| SIM-SYS-023 | TOML Composition Fragments         | TF-SRS-006                |
| SIM-SYS-024 | Composition Fragment Inheritance   | TF-SRS-006                |
| SIM-SYS-025 | Named Composition Fragments        | TF-SRS-001 through 006    |
| SIM-SYS-026 | Implementation Language            | TF-SRS-001 through 006    |
| SIM-SYS-027 | Crate Publishability               | TF-SRS-006                |
| SIM-SYS-028 | Independent GN&C ABI Versioning    | TF-SRS-002A               |
| SIM-SYS-029 | Headless Feature Flag              | TF-SRS-006                |
| SIM-SYS-030 | Embedding Interface                | TF-SRS-006                |
| SIM-SYS-031 | Telemetry Recording                | TF-SRS-004                |
| SIM-SYS-032 | Telemetry Playback                 | TF-SRS-003, TF-SRS-004    |
| SIM-SYS-033 | Partition-level Specifications and Documentation Structure | TF-SRS-001 through 006 |
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
| SIM-SYS-044 | State Snapshot as Composition Fragment | TF-SRS-006          |
| SIM-SYS-045 | State Dump and Load Operations     | TF-SRS-004, TF-SRS-006  |
| SIM-SYS-046 | Shared State Machine Synchronization | TF-SRS-005            |
| SIM-SYS-047 | Contract Tests at Every Layer   | TF-SRS-001 through 006    |
| SIM-SYS-048 | Compositor Tests at Every Layer | TF-SRS-001 through 006    |
| SIM-SYS-049 | System Tests Trace to Requirements | TF-SRS-001 through 006 |
| SIM-SYS-050 | Transport-parameterized Compositor Tests | TF-SRS-005        |
| SIM-SYS-051 | Test Reference Data Ownership at Contract Boundaries | TF-SRS-001 through 006 |
| SIM-SYS-052 | Compositor Tests Assert Compositional Properties | TF-SRS-001 through 006 |
| SIM-SYS-053 | System Test Reference Generation    | TF-SRS-006              |
| SIM-SYS-054 | Contract Versioning Scopes Reference Data Propagation | TF-SRS-001 through 006 |
| SIM-SYS-055 | Layer-scoped Bus                   | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-056 | Compositor Runtime Role             | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-057 | Compositor Relay Authority          | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-058 | Compositor Fault Handling           | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-059 | Recursive State Contribution        | TF-SRS-006                |
| SIM-SYS-060 | Direct Signals                      | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-061 | Contract-crate-scoped Event Action Vocabulary | TF-SRS-001 through 006    |
| SIM-SYS-062 | Compositor Tick Lifecycle           | TF-SRS-005, TF-SRS-006    |
| SIM-SYS-063 | Bus Delivery Semantics              | TF-SRS-005                |
