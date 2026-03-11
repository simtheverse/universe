# Contract Crate Interface Specification (ICD)
## universe-contract — System-level Contract Crate

---

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Document ID   | TF-SRS-005                                 |
| Version       | 0.1.0 (draft)                              |
| Status        | Draft                                      |
| Layer         | 0 — System (contract crate)                |
| Parent        | UNI-SRS-000 (universe System Requirements) |
| Architecture  | FPA-SRS-000 (Fractal Partition Architecture) |

---

## Table of Contents

1. Purpose and Scope
2. Execution State Machine Interface
3. Requirements Index

---

## 1. Purpose and Scope

This document defines the interface requirements for `universe-contract`, the system-level
contract crate for the universe flight simulation framework. `universe-contract` defines the
types, traits, and behavioral contracts that all layer 0 partitions depend on. It is not
a partition — it is the interface boundary between partitions.

These requirements trace to UNI-SRS-000. The execution state machine interface (Section
2) traces to UNI-025 (Simulation Lifecycle State Machine) and specifies the detailed
states, transitions, request types, event action identifiers, and conflict resolution
policy that `universe-contract` shall export.

---

## 2. Execution State Machine Interface

The execution state machine is the layer 0 instance of the shared state machine
synchronization pattern (FPA-006). It defines how partitions observe and influence the
simulation lifecycle.

---

### TF-005-001 — Execution State Type

**Statement:** `universe-contract` shall export a type representing the simulation execution
state with the following states and transitions:

| State     | Valid Transitions                        |
|-----------|------------------------------------------|
| Idle      | → Running (start)                        |
| Running   | → Paused (pause), → Stopped (stop)       |
| Paused    | → Running (resume), → Stopped (stop)     |
| Stopped   | → Idle (reset)                           |

The type shall include a method or function that validates whether a requested
transition is valid from the current state. The current execution state shall be
available as a named resource accessible to all partitions. No partition shall maintain
a private copy that diverges from the authoritative value held by the orchestrator.

**Traces to:** UNI-025

**Rationale:** All four partitions must agree on whether the simulation is running,
paused, or stopped. A centrally defined type with enforced transition semantics prevents
partitions from reasoning about execution state using ad-hoc booleans or integers that
cannot express transition validity.

**Verification Expectations:**
- Pass: `universe-contract` exports an enum or equivalent type representing Idle, Running, Paused,
  and Stopped, with a validation function for transition legality.
- Pass: Requesting an invalid transition (e.g., Running → Idle) returns an error.
- Pass: All partitions read execution state from the same `universe-contract` resource.
- Fail: The execution state is a bare boolean or integer without enforced transitions.

---

### TF-005-002 — Execution State Request Type

**Statement:** `universe-contract` shall export a typed `ExecutionStateRequest` message carrying
the requested transition and the identity of the requesting partition. Any partition
shall request execution state transitions by emitting this message on its layer's bus.
The orchestrator is the sole authority for evaluating and applying transitions according
to the state machine defined in TF-005-001. At deeper layers, requests follow the
compositor relay chain (FPA-010) to reach the orchestrator.

Requests that represent invalid transitions shall be logged with the requesting
partition's identity and ignored. Valid transitions shall take effect within one tick of
receipt at the orchestrator. A request emitted at layer N may take up to N ticks to reach
the orchestrator due to relay latency. Safety-critical signals that cannot tolerate relay
latency shall use direct signals (FPA-013).

**Traces to:** UNI-025, UNI-016

**Rationale:** Multiple sources may legitimately request execution state changes: the
UI partition (UNI-016), a physics partition detecting a limit exceedance, an event
handler reaching a scripted pause point, or a condition-triggered event. A typed request
message ensures that all sources use the same arbitration path and that the orchestrator
has full information (who requested what) for logging and conflict resolution.

**Verification Expectations:**
- Pass: A layer 0 partition emitting `ExecutionStateRequest::Stop` during Running causes
  the orchestrator to transition to Stopped within one tick.
- Pass: A layer 0 partition emitting `ExecutionStateRequest::Pause` during Running causes
  the orchestrator to transition to Paused within one tick.
- Pass: A layer 1 sub-partition emitting `ExecutionStateRequest::Stop` on the layer 1 bus
  causes the compositor to relay it to the layer 0 bus for orchestrator arbitration.
- Pass: An `ExecutionStateRequest::Resume` emitted during Running is logged as invalid
  and ignored.
- Pass: Each applied transition is logged with the requesting partition's identity.
- Fail: A partition directly mutates the execution state without emitting a request.
- Fail: A sub-partition emits an `ExecutionStateRequest` directly on the layer 0 bus,
  bypassing compositor relay.

---

### TF-005-003 — Execution State Event Actions

**Statement:** `universe-contract` shall declare `"sim_pause"`, `"sim_stop"`, and `"sim_resume"`
as event action identifiers in its contract-crate action vocabulary (FPA-029). When an
event with one of these action identifiers fires, the event dispatcher shall emit the
corresponding `ExecutionStateRequest` on the bus at the layer where the event is
defined. These actions shall be usable with both time-triggered and condition-triggered
events.

**Traces to:** UNI-025

**Rationale:** Encoding pause points, safety stops, and phase transitions as event
actions keeps the logic declarative and configurable in composition fragments. Because
every partition depends on `universe-contract`, these actions are available at every layer via the
contract-crate dependency graph. Connecting the event system to the execution state
request mechanism (TF-005-002) ensures all event-driven state changes flow through the
same arbitration path as UI-initiated changes.

**Verification Expectations:**
- Pass: A partition-level event with `action = "sim_pause"` and a time trigger causes the
  simulation to pause at the specified time.
- Pass: A partition-level event with `action = "sim_stop"` and a condition trigger causes
  the simulation to stop when the condition is met.
- Pass: A system-level event with `action = "sim_resume"` and a time trigger resumes a
  paused simulation.
- Pass: The `ExecutionStateRequest` emitted by an event action is indistinguishable from
  one emitted by the UI or partition code directly.
- Fail: Execution state event actions require source code modifications rather than
  configuration.

---

### TF-005-004 — Execution State Conflict Resolution

**Statement:** When the orchestrator receives multiple `ExecutionStateRequest` messages
within the same tick that request conflicting transitions, it shall apply the following
deterministic priority order: (1) Stop takes priority over all other requests. (2) Pause
takes priority over Resume. (3) Among requests of equal priority, the request shall be
designated the primary request for logging purposes using a deterministic,
transport-independent rule (the specific tie-breaking mechanism is
implementation-defined). All received requests and the resolution outcome shall be logged
with the requesting partition identities.

**Traces to:** UNI-025

**Rationale:** In a flight simulation, a structural failure or limit exceedance must
always win over a scripted pause point. Concurrent events may independently request
conflicting transitions — for example, a physics partition requesting stop while a
scripted event requests pause. Prioritizing stop over pause ensures safety-critical
transitions are not overridden. Deterministic, transport-independent resolution ensures
reproducible audit logs across deployment configurations.

**Verification Expectations:**
- Pass: `ExecutionStateRequest::Stop` and `ExecutionStateRequest::Pause` in the same tick
  results in Stopped.
- Pass: `ExecutionStateRequest::Resume` and `ExecutionStateRequest::Pause` in the same
  tick results in Paused.
- Pass: Two `ExecutionStateRequest::Pause` in the same tick deterministically designates
  the same partition as primary regardless of transport mode.
- Pass: The orchestrator log includes all requests and identifies which was applied.
- Pass: The audit log is identical across synchronous, async, and network transport modes.
- Fail: Conflicting requests produce non-deterministic behavior depending on transport.
- Fail: A lower-priority request silently overrides a higher-priority request.

---

## 3. Requirements Index

| ID          | Title                                            | Traces to     |
|-------------|--------------------------------------------------|---------------|
| TF-005-001  | Execution State Type                             | UNI-025       |
| TF-005-002  | Execution State Request Type                     | UNI-025, UNI-016 |
| TF-005-003  | Execution State Event Actions                    | UNI-025       |
| TF-005-004  | Execution State Conflict Resolution              | UNI-025       |
