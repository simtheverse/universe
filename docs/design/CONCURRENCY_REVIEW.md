# Concurrency and Thread Safety Architecture Review

---

| Field         | Value                                           |
|---------------|-------------------------------------------------|
| Document ID   | TF-REV-001                                      |
| Version       | 0.1.0 (draft)                                   |
| Status        | Open — pending specification amendments         |
| Scope         | System-wide (all layers)                        |
| References    | TF-SRS-000 (SPECIFICATION.md)                   |

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Scope and Method](#2-scope-and-method)
3. [Summary of Findings](#3-summary-of-findings)
4. [Findings](#4-findings)
   - [CR-001 — Intra-tick Read/Write Ordering](#cr-001--intra-tick-readwrite-ordering)
   - [CR-002 — WorldState Consistency Under Async Transport](#cr-002--worldstate-consistency-under-async-transport)
   - [CR-003 — Bus Backpressure and Overflow Semantics](#cr-003--bus-backpressure-and-overflow-semantics)
   - [CR-004 — State Dump Temporal Consistency](#cr-004--state-dump-temporal-consistency)
   - [CR-005 — Direct Signal Processing Timing](#cr-005--direct-signal-processing-timing)
   - [CR-006 — Event Evaluation Ordering and Side-Effect Visibility](#cr-006--event-evaluation-ordering-and-side-effect-visibility)
   - [CR-007 — Vehicle Spawn/Despawn Timing Within Tick](#cr-007--vehicle-spawndespawn-timing-within-tick)
   - [CR-008 — Cross-Layer Conflict Resolution Ordering](#cr-008--cross-layer-conflict-resolution-ordering)
   - [CR-009 — Partition Behavior After Emitting Stop](#cr-009--partition-behavior-after-emitting-stop)
   - [CR-010 — Fault Handling Scope for Non-Step Trait Calls](#cr-010--fault-handling-scope-for-non-step-trait-calls)
   - [CR-011 — WorldState Publish Atomicity](#cr-011--worldstate-publish-atomicity)
   - [CR-012 — Dynamic Library Unload Safety](#cr-012--dynamic-library-unload-safety)
5. [Cross-Cutting Recommendation: Synchronization Model](#5-cross-cutting-recommendation-synchronization-model)
6. [Resolution Tracking](#6-resolution-tracking)

---

## 1. Purpose

This document records the results of a deep review of the universe architecture for race
conditions, thread safety gaps, and concurrency-related specification ambiguities. Each
finding identifies a specific gap in the current specification (TF-SRS-000) that, if left
unresolved, could produce non-deterministic behavior, data corruption, or violation of the
transport independence guarantee (SIM-SYS-005) during implementation.

The document is intended as a working framework: each finding should be resolved by
amending the specification, adding a new requirement, or documenting a deliberate
acceptance of the risk. Resolved findings are tracked in
[Section 6](#6-resolution-tracking).

---

## 2. Scope and Method

**In scope:** All specification requirements (SIM-SYS-001 through SIM-SYS-061), all
architecture explainer documents, and all communication architecture documents.

**Method:** Each requirement and explainer was evaluated against the three transport modes
(in-process synchronous, asynchronous cross-thread, network publish-subscribe) to
determine whether the stated guarantees hold under all modes. Particular attention was
given to:

- Ordering assumptions that hold in synchronous mode but not in async or network mode
- Shared mutable state accessed from multiple execution contexts
- Timing windows where intermediate state is visible to observers
- Missing synchronization points between independently clocked components
- Guarantees stated in prose but not enforceable without additional specification

**Out of scope:** Implementation-specific concerns (e.g., specific Rust lifetime issues,
Bevy ECS scheduling details). This review addresses architectural-level concurrency
semantics.

---

## 3. Summary of Findings

| ID | Title | Priority | Spec References | Status |
|----|-------|----------|-----------------|--------|
| CR-001 | Intra-tick read/write ordering | Critical | SIM-SYS-005, 006, 056 | Open |
| CR-002 | WorldState consistency under async | Critical | SIM-SYS-005, 009, 013 | Open |
| CR-003 | Bus backpressure and overflow | Critical | SIM-SYS-005 | Open |
| CR-004 | State dump temporal consistency | High | SIM-SYS-044, 045 | Open |
| CR-005 | Direct signal processing timing | High | SIM-SYS-060 | Open |
| CR-006 | Event evaluation ordering | High | SIM-SYS-035–039, 061 | Open |
| CR-007 | Spawn/despawn timing within tick | High | SIM-SYS-010, 009 | Open |
| CR-008 | Cross-layer conflict resolution ordering | Medium | SIM-SYS-043, 057 | Open |
| CR-009 | Partition behavior after emitting stop | Medium | SIM-SYS-041, 057, 058 | Open |
| CR-010 | Fault handling scope for non-step calls | Medium | SIM-SYS-058, 059 | Open |
| CR-011 | WorldState publish atomicity | Medium | SIM-SYS-009 | Open |
| CR-012 | Dynamic library unload safety | Low | SIM-SYS-010, 014 | Open |

---

## 4. Findings

---

### CR-001 — Intra-tick Read/Write Ordering

**Priority:** Critical

**Specification references:** SIM-SYS-005, SIM-SYS-006, SIM-SYS-056

**Observation:**

The compositor calls `step()` on partitions sequentially (SIM-SYS-056). During its step,
each partition publishes outputs on the bus and reads inputs from the bus. The
specification does not define whether partition B (stepped second) sees partition A's
output from the *current* tick or the *previous* tick.

This is the classic read-before-write vs. write-before-read ambiguity in lockstep
simulation:

- If B sees A's current-tick output, then step ordering creates an implicit directional
  dependency: A's output influences B within the same tick, but not vice versa.
- If B sees A's previous-tick output, both partitions compute from consistent
  previous-tick data, but there is a one-tick transport delay on all inter-partition data.

Under synchronous transport, whichever behavior the bus implementation chooses becomes
implicit contract. Under async transport, the behavior may differ — partitions may read
stale or fresh data depending on thread scheduling. The transport independence guarantee
(SIM-SYS-005: "identical final vehicle state") is unenforceable without specifying this
semantic.

**Impact:**

Different transport implementations will make different choices about message visibility
within a tick, producing different simulation results. This directly violates the
transport independence guarantee.

**Recommendation:**

Add a requirement specifying one of:

**(a) Lockstep double-buffered (recommended):** All partitions read from tick N-1 outputs
and write to tick N outputs. The compositor swaps read/write buffers between ticks. This
produces transport-independent behavior because no partition sees another's current-tick
output, regardless of step order or thread scheduling.

```
Tick N:
  compositor publishes WorldState(N-1), ExecutionState(N-1) on bus
  compositor swaps read buffer to tick N-1 outputs
  for each partition in step order:
    partition reads from tick N-1 buffer
    partition writes to tick N buffer
  compositor collects tick N outputs
```

**(b) Sequential passthrough:** Partitions stepped later in the same tick see earlier
partitions' current-tick outputs. Step order is fixed and documented. This is simpler but
makes behavior depend on step order, which must be identical across transport modes.

Option (a) is strongly preferred because it eliminates ordering sensitivity entirely and
makes the transport independence guarantee trivially enforceable for intra-tick data flow.

**Proposed specification amendment:**

> **SIM-SYS-0XX — Intra-tick Message Visibility**
>
> **Statement:** Within a single simulation tick, partitions shall read inter-partition
> messages from the previous tick's published values. A message published by partition A
> during tick N shall not be visible to any other partition until tick N+1. The compositor
> shall ensure this isolation regardless of the active transport mode and regardless of
> the order in which partitions are stepped.
>
> **Rationale:** Intra-tick message isolation eliminates ordering sensitivity — the
> simulation result is identical regardless of which partition steps first. This is
> necessary to satisfy the transport independence guarantee (SIM-SYS-005), because
> different transport modes may step partitions in different orders or concurrently.
> Without isolation, a partition reading "current-tick" data from a peer would see
> different values depending on whether the peer has already stepped, producing
> transport-dependent results.

---

### CR-002 — WorldState Consistency Under Async Transport

**Priority:** Critical

**Specification references:** SIM-SYS-005, SIM-SYS-009, SIM-SYS-013

**Observation:**

SIM-SYS-009 requires that WorldState contain the PlantState of all active vehicles and
be published each tick. The inter-partition communication explainer states: "All partitions
that read WorldState in a given tick see the same snapshot of vehicle states. There is no
risk of one partition seeing a partially updated set of vehicles while another sees a
different partial update."

This guarantee is trivially satisfied under in-process synchronous transport but
**unspecified for async and network modes**. SIM-SYS-013 allows physics to run at 1 kHz
while visualization runs at 60 Hz. Under async transport, partitions run on separate
threads at different update rates. When the compositor assembles WorldState:

- Vehicle A's physics may have completed tick N while vehicle B's physics is still
  computing tick N-1.
- The resulting WorldState contains temporally inconsistent PlantState entries.
- A GN&C plugin receiving this WorldState computes formation-keeping commands against
  stale peer data.

No tick barrier or consistency protocol is specified.

**Impact:**

Formation flying, collision avoidance, and any multi-vehicle coordination algorithm will
produce different results under async transport than under synchronous transport, violating
the transport independence guarantee.

**Recommendation:**

Specify a tick-barrier protocol:

**(a) Compositor-driven barrier (recommended):** The compositor must collect all partition
outputs for tick N before publishing WorldState for tick N. Under async transport, this
means the compositor waits for the slowest partition to complete its tick before
assembling and publishing WorldState. Partitions running at higher rates produce
intermediate outputs that are buffered but not included in WorldState until all partitions
have reached the same tick boundary.

**(b) Rate-group synchronization:** Define synchronization points at the least-common-
multiple of partition rates. WorldState is published only at these synchronization points.
Between synchronization points, partitions use the most recent WorldState.

**(c) Timestamped staleness tolerance:** Each PlantState in WorldState carries a timestamp.
Consumers are responsible for interpolation or extrapolation. This weakens the consistency
guarantee but may be appropriate for visualization (which is display-only) while
maintaining strict synchronization for physics-GN&C interaction.

Option (a) is simplest and preserves the existing guarantee. It may introduce idle time
for fast partitions but ensures correctness. Option (c) is appropriate as an optimization
for visualization only, not for physics-GN&C data flow.

**Proposed specification amendment:**

> **SIM-SYS-0XX — Tick Barrier for WorldState Assembly**
>
> **Statement:** The compositor shall not publish WorldState for tick N until all
> partitions at that layer have completed their tick N step and published their outputs.
> Under async transport, the compositor shall wait for all partition outputs before
> assembling and publishing WorldState. Partitions running at higher update rates than the
> WorldState publication rate shall buffer intermediate outputs; only the output
> corresponding to the WorldState tick boundary shall be included.

---

### CR-003 — Bus Backpressure and Overflow Semantics

**Priority:** Critical

**Specification references:** SIM-SYS-005

**Observation:**

The specification describes three transport modes but does not address what happens when
a producer publishes faster than a consumer reads. Under async transport with partitions
at different rates (SIM-SYS-013: physics at 1 kHz, visualization at 60 Hz), the physics
partition produces approximately 16 messages for every one the visualization partition
reads.

The specification does not define:
- Whether bus channels are bounded or unbounded
- What happens when a bounded buffer is full (block producer? drop oldest? drop newest?)
- Whether consumers receive every published message or only the latest value
- Whether different message types may have different delivery semantics

**Impact:**

- Unbounded buffers cause memory exhaustion in long-running simulations
- Dropping messages causes missed ExecutionStateRequest messages (a correctness failure)
- Blocking producers causes physics to stall waiting for visualization (a performance
  failure that changes timing behavior)
- Different transport implementations making different choices produces transport-dependent
  behavior

**Recommendation:**

Specify per-message-type delivery semantics:

| Message Type | Delivery Semantic | Rationale |
|---|---|---|
| PlantState, WorldState, EnvState | Latest-value (last-writer-wins) | Consumers need current state, not history |
| GNCCommand | Latest-value | Commands are instantaneous; stale commands are incorrect |
| ExecutionStateRequest | Queued (all delivered) | Every request must reach the arbitrator |
| VehiclePlantVizState | Latest-value | Visualization needs current frame, not backlog |
| Direct signals | Queued (all delivered) | Safety-critical, must not be dropped |

**Proposed specification amendment:**

> **SIM-SYS-0XX — Bus Delivery Semantics**
>
> **Statement:** The bus shall support two delivery semantics per message type, specified
> in the contract crate alongside the type declaration:
>
> (a) **Latest-value:** The bus retains only the most recently published value for each
> (message type, VehicleId) pair. A consumer reading the bus always receives the latest
> published value. No backlog accumulates. Suitable for continuous state messages.
>
> (b) **Queued:** The bus retains all published messages in order. A consumer receives
> every message published since its last read. The queue shall be bounded; if the bound
> is reached, the compositor shall log a warning and the producer shall block until space
> is available. Suitable for request messages where every instance must be processed.
>
> The delivery semantic for each message type shall be declared in the contract crate
> and enforced identically across all transport modes.

---

### CR-004 — State Dump Temporal Consistency

**Priority:** High

**Specification references:** SIM-SYS-044, SIM-SYS-045

**Observation:**

SIM-SYS-045 states: "Dump shall be invocable during any execution state (Running, Paused,
Stopped)" and "A dump operation invoked while the simulation is Running produces a valid
snapshot fragment without interrupting physics integration."

If dump occurs while partitions are mid-`step()`, the `contribute_state()` calls
interleave with active computation. Under async transport, partition A may have completed
tick N while partition B is still computing tick N-1. The resulting snapshot contains
state from different simulation times — it is temporally inconsistent.

Under synchronous transport, the compositor controls ordering and could insert the dump
between ticks. But the specification does not require this — it only says dump must not
"interrupt physics integration," which could be read as allowing concurrent execution.

**Impact:**

A snapshot captured during Running and then loaded initializes the simulation to an
internally inconsistent state (partition A at time T, partition B at time T-dt),
potentially causing divergence, conservation violations, or crashes on reload.

**Recommendation:**

Specify one of:

**(a) Tick-boundary dump (recommended):** The compositor processes dump requests between
ticks, after all partitions have completed their step for tick N and before any partition
begins tick N+1. All `contribute_state()` calls see post-tick-N state. The "without
interrupting physics integration" guarantee means the dump does not delay tick N's
computation — it occurs in the inter-tick gap.

**(b) Shadow state:** Each partition maintains a committed state snapshot from the last
completed tick. `contribute_state()` reads from this shadow state, not from the partition's
live working state. Dump can occur at any time because it reads from a consistent,
immutable snapshot. This is more complex but allows truly concurrent dump.

Option (a) is simpler and sufficient for most use cases. Option (b) is worth considering
if dump latency during high-rate simulation is a concern.

**Proposed specification amendment:**

> **SIM-SYS-045 amendment:** Add: "When the simulation is in the Running state, the
> compositor shall process dump requests at a tick boundary — after all partitions have
> completed their current tick and before any partition begins the next tick. The
> `contribute_state()` calls shall observe the state produced by the most recently
> completed tick."

---

### CR-005 — Direct Signal Processing Timing

**Priority:** High

**Specification references:** SIM-SYS-060

**Observation:**

SIM-SYS-060 states that direct signals "bypass all intermediate bus instances" and reach
the orchestrator directly. The implementation mechanism "might be a separate channel, a
shared atomic, or a direct callback."

The specification does not define **when** the orchestrator processes direct signals
relative to its step loop:

- If the mechanism is a shared atomic polled between partition steps, there is a window
  (the duration of a single partition's `step()`) where a safety-critical signal is not
  yet observed. The worst-case latency is one full step-loop cycle.
- If the mechanism is a callback or interrupt that fires during the orchestrator's
  execution, it may execute while the orchestrator holds mutable references to partition
  state or bus structures, causing aliasing violations in Rust or reentrancy hazards in
  other languages.
- If the mechanism is an async channel, the signal may be queued behind bus processing
  and subject to the same latency as a relayed request — defeating its purpose.

**Impact:**

Either delayed response to emergency signals (defeating the purpose of direct signals) or
memory safety violations from reentrancy, depending on implementation choice.

**Recommendation:**

Specify that the orchestrator polls for direct signals at defined points in the step loop:

```
Step loop:
  check direct signals  ← HERE (before any partition steps)
  for each partition in step order:
    call partition.step(dt)
    check direct signals  ← HERE (between partition steps)
  collect outputs
  check direct signals  ← HERE (after all partitions step)
  process bus messages
```

This gives direct signals a worst-case latency of one partition's step duration (not one
full tick), while avoiding reentrancy hazards. The polling approach is Rust-friendly
because the orchestrator checks the signal while it has exclusive control, not while
holding mutable references into partition state.

**Proposed specification amendment:**

> **SIM-SYS-060 amendment:** Add: "The orchestrator shall check for pending direct signals
> at least once between each pair of partition `step()` calls within a tick, and once
> before the first partition steps. When a direct signal is detected, the orchestrator
> shall process it before stepping the next partition. The worst-case latency for a direct
> signal is the duration of one partition's `step()` call, not one full tick."

---

### CR-006 — Event Evaluation Ordering and Side-Effect Visibility

**Priority:** High

**Specification references:** SIM-SYS-035, SIM-SYS-036, SIM-SYS-037, SIM-SYS-038,
SIM-SYS-039, SIM-SYS-061

**Observation:**

The events explainer states: "When multiple events fire in the same tick, ordering follows
the event system's evaluation semantics. Actions whose handlers modify internal state take
effect in the order the event dispatcher evaluates them."

The evaluation order is not specified. If event A's action modifies a signal that event
B's condition monitors, event B may or may not fire depending on whether it is evaluated
before or after A's side effects are applied.

Example:
```toml
[[physics.events]]
trigger = { time = 60.0 }
action = "inject_sensor_fault"
parameters = { sensor = "altimeter", bias = 50.0 }

[[physics.events]]
trigger = { condition = "altimeter_bias > 0.0" }
action = "sim_stop"
```

If the fault injection event is evaluated first and its side effect is visible, the
condition event fires in the same tick. If evaluated second, or if side effects are
deferred, the condition event fires on the next tick.

**Impact:**

Non-deterministic event behavior depending on TOML array ordering, hash map iteration
order, or other implementation details. Different implementations may produce different
event sequences from the same scenario.

**Recommendation:**

Specify one of:

**(a) Snapshot evaluation (recommended):** All event conditions are evaluated against
pre-step state. All triggered actions are collected, then applied in declaration order.
No action's side effects are visible to other events' conditions within the same tick.
Events whose conditions depend on action side effects fire on the next tick.

**(b) Sequential evaluation with defined order:** Events are evaluated in TOML declaration
order. Each action's side effects are visible to subsequently evaluated conditions within
the same tick. This is deterministic but order-dependent — reordering event entries in
TOML changes behavior.

Option (a) is preferred because it eliminates ordering sensitivity and is consistent with
the double-buffered approach recommended in CR-001.

**Proposed specification amendment:**

> **SIM-SYS-0XX — Event Evaluation Semantics**
>
> **Statement:** Within a single tick, all event conditions shall be evaluated against
> the partition state as it existed at the beginning of the tick, before any event actions
> have been applied. The set of events whose conditions are satisfied shall be collected,
> and their actions shall be applied in TOML declaration order after all conditions have
> been evaluated. An action's side effects shall not be visible to other event conditions
> until the following tick.

---

### CR-007 — Vehicle Spawn/Despawn Timing Within Tick

**Priority:** High

**Specification references:** SIM-SYS-010, SIM-SYS-009

**Observation:**

SIM-SYS-010 states: "A vehicle spawned mid-simulation at specified initial conditions
begins publishing PlantState messages on the subsequent tick" and "After despawning a
vehicle, no further messages bearing its VehicleId appear on the bus."

The specification does not define when spawn/despawn requests are processed relative to
the step loop. If physics has already been stepped for the current tick when a despawn
request arrives, physics has already published a PlantState for that vehicle. Partitions
stepped after the despawn point would not see the vehicle, while partitions stepped before
would — producing an inconsistent view within a single tick.

Under async transport, the timing is even less predictable: a spawn request might arrive
between partition steps, during a partition step, or after all partitions have stepped.

**Impact:**

Partitions receive inconsistent views of the vehicle set within the same tick. WorldState
may include a despawned vehicle's state or exclude a newly spawned vehicle that another
partition can see.

**Recommendation:**

Specify that spawn/despawn requests take effect only at tick boundaries:

```
Tick boundary processing (between tick N and tick N+1):
  1. Process pending spawn requests → initialize new vehicles
  2. Process pending despawn requests → remove vehicles and clean up
  3. Assemble WorldState from current vehicle set
  4. Begin tick N+1 stepping
```

All partitions see the same vehicle set for the entirety of a tick. Spawn/despawn
requests received during tick N are queued and applied before tick N+1 begins.

**Proposed specification amendment:**

> **SIM-SYS-010 amendment:** Add: "Spawn and despawn requests shall be processed at tick
> boundaries, after all partitions have completed their current tick step and before any
> partition begins the next tick. All partitions within a tick shall observe the same set
> of active vehicles. A spawn request received during tick N shall result in the vehicle
> being active and included in WorldState starting at tick N+1. A despawn request received
> during tick N shall result in the vehicle being removed before tick N+1 begins."

---

### CR-008 — Cross-Layer Conflict Resolution Ordering

**Priority:** Medium

**Specification references:** SIM-SYS-043, SIM-SYS-057

**Observation:**

SIM-SYS-043 states: "Among requests of equal priority, the first received shall be
applied." The phrase "first received" implies arrival order, which is transport-dependent.

A layer 0 partition emitting `Pause` and a layer 1 sub-partition emitting `Pause` (relayed
by its compositor) will arrive at the orchestrator at different times depending on
transport mode and compositor step ordering. Under async transport, relay latency is
non-deterministic.

While the *outcome* is often the same (both are Pause), the audit log records which
request was "primary" and which was "superseded." This log will differ across transport
modes, making test reproducibility difficult.

More critically, if the tie-break ever affects observable behavior (e.g., a future
requirement where the "winning" requester's identity influences the response), the
transport-dependent ordering becomes a correctness issue.

**Impact:**

Audit log non-determinism across transport modes. Potential future correctness issue if
tie-break logic gains behavioral significance.

**Recommendation:**

Replace arrival-order tie-breaking with a deterministic criterion:

**(a) Partition-ID tie-break:** Among equal-priority requests in the same tick, the
request from the partition with the lowest ID (or highest layer depth, or another stable
ordering) is designated primary. All are applied (since they agree), but the log records
a deterministic primary.

**(b) Accept non-deterministic logging:** Restrict the transport independence guarantee
to state outcomes only, explicitly excluding audit log ordering for equal-priority
requests.

Option (a) is preferred for reproducibility.

**Proposed specification amendment:**

> **SIM-SYS-043 amendment:** Replace "the first received shall be applied" with: "the
> request from the partition with the numerically lowest partition identifier shall be
> designated the primary request for logging purposes. All valid requests of the same
> type shall be applied (the outcome is the same regardless of which is designated
> primary). This ensures audit log determinism across transport modes."

---

### CR-009 — Partition Behavior After Emitting Stop

**Priority:** Medium

**Specification references:** SIM-SYS-041, SIM-SYS-057, SIM-SYS-058

**Observation:**

When a partition (especially a deep sub-partition) emits `ExecutionStateRequest::Stop`,
the request may take up to N ticks to reach the orchestrator through the relay chain
(SIM-SYS-041). During those N ticks, the compositor continues calling `step()` on the
partition that has already decided it needs to stop.

The specification does not define what the partition should do:
- Continue computing normally? This may produce invalid outputs if the stop was triggered
  by a safety limit exceedance.
- Return an error from `step()`? This enters the fault handling path (SIM-SYS-058), which
  may or may not produce the same outcome as the stop request.
- Produce degraded/safe outputs? There is no mechanism for this.

**Impact:**

N ticks of physics integration after a safety limit exceedance, with the partition
potentially producing outputs from an invalid state that other partitions consume.

**Recommendation:**

Specify expected partition behavior:

**(a) Continue with flag (recommended):** The partition continues computing but sets an
internal flag indicating a stop has been requested. The compositor, having received the
stop request on its bus, may choose to fast-track the relay (process it immediately rather
than waiting for its next step cycle) and/or skip stepping the requesting partition on
subsequent ticks until the stop is applied.

**(b) Error return:** The partition returns an error from `step()` on subsequent calls.
The compositor handles this via SIM-SYS-058 fault logic. This is simpler but conflates
"requested stop" with "faulted," which have different semantics.

**Proposed specification amendment:**

> **SIM-SYS-0XX — Compositor Fast-Track Relay for Stop Requests**
>
> **Statement:** When a compositor receives an `ExecutionStateRequest::Stop` on its inner
> bus, it shall relay the request to the outer bus immediately — during the current tick's
> processing, not deferred to the next tick. After relaying a stop request, the compositor
> may skip stepping the requesting partition on subsequent ticks until the execution state
> transition is applied by the orchestrator. This reduces the worst-case relay latency
> for stop requests from N ticks to 1 tick regardless of layer depth.

---

### CR-010 — Fault Handling Scope for Non-Step Trait Calls

**Priority:** Medium

**Specification references:** SIM-SYS-058, SIM-SYS-059

**Observation:**

SIM-SYS-058 specifies that the compositor catches faults from sub-partitions "during
execution — returning an error from a trait method call, panicking, or timing out." The
rationale and examples focus exclusively on `step()`.

SIM-SYS-059 defines recursive state contribution via `contribute_state()`, which is
also a trait call into sub-partitions. A sub-partition that panics or hangs during
`contribute_state()` is not addressed by SIM-SYS-058's examples. Similarly, `init()`,
`shutdown()`, and `load_state()` can fault.

Under async transport, a sub-partition blocked during `contribute_state()` could hold
resources that the compositor or other sub-partitions need, causing deadlock.

**Impact:**

A hung sub-partition during state dump freezes the entire state contribution chain. No
timeout or fault recovery is specified for non-step trait calls.

**Recommendation:**

Amend SIM-SYS-058 to explicitly cover all trait calls, not just `step()`:

**Proposed specification amendment:**

> **SIM-SYS-058 amendment:** Replace "when a sub-partition faults during execution" with
> "when a sub-partition faults during any trait method call — including `step()`,
> `init()`, `shutdown()`, `contribute_state()`, and `load_state()`." Add: "For
> `contribute_state()` specifically, the compositor shall apply a configurable timeout. If
> a sub-partition does not return within the timeout, the compositor shall log the timeout,
> exclude that sub-partition's contribution from the snapshot (with a marker indicating the
> omission), and continue collecting contributions from remaining sub-partitions."

---

### CR-011 — WorldState Publish Atomicity

**Priority:** Medium

**Specification references:** SIM-SYS-009

**Observation:**

WorldState contains PlantState for all active vehicles. Under network transport, a large
WorldState message with many vehicles might be fragmented at the transport layer. Under
async transport with ECS-style per-type channels, a subscriber might read WorldState while
it is being updated — seeing a partially constructed aggregate.

If the bus implementation uses a shared mutable resource for WorldState (e.g., a Bevy
Resource), concurrent reads and writes require synchronization.

**Impact:**

Torn reads on WorldState, especially under network transport with many vehicles. A
partition reading a partially updated WorldState sees some vehicles at tick N and others
at tick N-1.

**Recommendation:**

Specify that WorldState publication is atomic from the consumer's perspective.

**Proposed specification amendment:**

> **SIM-SYS-009 amendment:** Add: "WorldState shall be published as an atomic unit. A
> consumer reading WorldState shall always receive a complete, internally consistent
> snapshot — never a partially assembled aggregate. The bus implementation shall ensure
> this atomicity regardless of transport mode."

This is naturally satisfied by the double-buffering approach in CR-001 (consumers read
from a completed previous-tick buffer) and the tick barrier in CR-002 (WorldState is
assembled only after all partitions complete).

---

### CR-012 — Dynamic Library Unload Safety

**Priority:** Low

**Specification references:** SIM-SYS-010, SIM-SYS-014

**Observation:**

GN&C plugins are loaded as dynamic shared libraries (SIM-SYS-014). Vehicle despawn
"shall cleanly unload all associated resources" (SIM-SYS-010), including the library.

If the unloaded library has registered callbacks, thread-local storage, or vtable entries
that the runtime still references, unloading causes undefined behavior. Under async
transport, a GN&C plugin's execution thread may still be active when the library is
unloaded. Even under synchronous transport, if any partition retains a reference to a
type whose vtable lives in the unloaded library (e.g., a trait object or function
pointer), subsequent use is undefined behavior.

**Impact:**

Use-after-free from `dlclose` while code is executing or from dangling function pointers.
This is a memory safety issue that bypasses Rust's safety guarantees (dynamic loading is
inherently `unsafe`).

**Recommendation:**

Specify a safe unload protocol:

**Proposed specification amendment:**

> **SIM-SYS-010 amendment:** Add: "Before unloading a GN&C plugin's shared library, the
> system shall ensure: (a) no thread is executing code from the library, (b) no function
> pointers, trait objects, or callbacks referencing the library's code remain reachable,
> and (c) the plugin's `shutdown()` method has returned successfully. Under async
> transport, the thread executing the plugin's `step()` shall have joined before unload
> proceeds. The system shall verify that the library's reference count (or equivalent
> mechanism) has reached zero before calling `dlclose` or equivalent."

---

## 5. Cross-Cutting Recommendation: Synchronization Model

The findings above share a common root cause: the specification defines *what* is
communicated but not *when* communication is visible relative to the step loop. This is
sufficient for synchronous in-process execution (where the compositor's sequential
`step()` calls impose a natural ordering) but insufficient for async and network modes.

The recommended resolution is a new specification section — **Synchronization Model** —
that defines the tick lifecycle as a state machine with explicit phases:

```
┌─────────────────────────────────────────────────────────┐
│                    TICK N LIFECYCLE                      │
│                                                         │
│  Phase 1: Pre-tick processing                           │
│    - Check direct signals                               │
│    - Process pending spawn/despawn requests              │
│    - Process pending dump/load requests                  │
│    - Assemble WorldState from tick N-1 outputs           │
│    - Publish WorldState, ExecutionState on bus           │
│    - Swap read/write buffers (double-buffer)             │
│                                                         │
│  Phase 2: Partition stepping                            │
│    for each partition in defined order:                  │
│      - partition reads from tick N-1 buffer              │
│      - partition.step(dt)                                │
│      - partition writes to tick N buffer                 │
│      - check direct signals                             │
│                                                         │
│  Phase 3: Post-tick processing                          │
│    - Evaluate event conditions (against pre-step state)  │
│    - Apply triggered event actions                       │
│    - Collect tick N outputs                              │
│    - Process bus requests (ExecutionStateRequest, etc.)  │
│    - Relay qualified requests to outer bus               │
│    - Check direct signals                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

This lifecycle model resolves CR-001, CR-002, CR-004, CR-005, CR-006, and CR-007
simultaneously. It provides implementers with a precise execution contract that is
enforceable across all transport modes.

The model should be specified as a new requirement (or family of requirements) and
cross-referenced from the affected existing requirements.

---

## 6. Resolution Tracking

Each finding should be resolved by one of:
- **Specification amendment:** A new or modified SIM-SYS requirement addressing the gap
- **Accepted risk:** A documented decision that the gap is acceptable, with rationale
- **Deferred:** The gap is acknowledged but resolution is deferred to a later version

| ID | Resolution | Amendment ID | Date | Notes |
|----|------------|-------------|------|-------|
| CR-001 | | | | |
| CR-002 | | | | |
| CR-003 | | | | |
| CR-004 | | | | |
| CR-005 | | | | |
| CR-006 | | | | |
| CR-007 | | | | |
| CR-008 | | | | |
| CR-009 | | | | |
| CR-010 | | | | |
| CR-011 | | | | |
| CR-012 | | | | |
