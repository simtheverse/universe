# The Tick Lifecycle and Synchronization Model

This document explains the compositor tick lifecycle (SIM-SYS-062) and bus delivery
semantics (SIM-SYS-063) — the synchronization model that ensures transport-independent
simulation behavior.

## The Problem: When Is Communication Visible?

The specification defines *what* partitions communicate — typed messages on layer-scoped
buses — but the question of *when* a published message becomes visible to other
partitions is equally important for correctness.

Under in-process synchronous transport, the compositor calls `step()` on partitions one
at a time. This imposes a natural ordering: if partition A steps before partition B, then
B can see A's output from the current tick. The behavior is deterministic but
order-dependent.

Under async or network transport, partitions may step concurrently, in different orders,
or at different rates. A message published by partition A might arrive at partition B
before, during, or after B's `step()` call, depending on thread scheduling and network
latency. If the in-process mode allows B to see A's current-tick output but the async
mode does not (or vice versa), the two modes produce different simulation results —
violating the transport independence guarantee (SIM-SYS-005).

The tick lifecycle resolves this by defining explicit phases within each tick, with
precise rules about when messages are visible.

## The Three-Phase Tick Lifecycle

*Specified in SIM-SYS-062.*

Every compositor — at every layer — executes each tick as three sequential phases:

```
                         TICK N
 ┌───────────────────────────────────────────────┐
 │                                               │
 │  Phase 1: Pre-tick                            │
 │  ┌─────────────────────────────────────────┐  │
 │  │ Check direct signals                    │  │
 │  │ Process spawn/despawn requests          │  │
 │  │ Process dump/load requests              │  │
 │  │ Assemble WorldState from tick N-1       │  │
 │  │ Swap read/write buffers                 │  │
 │  │ Publish WorldState + ExecutionState     │  │
 │  └─────────────────────────────────────────┘  │
 │                                               │
 │  Phase 2: Partition stepping                  │
 │  ┌─────────────────────────────────────────┐  │
 │  │ for each partition:                     │  │
 │  │   read from tick N-1 buffer             │  │
 │  │   step(dt)                              │  │
 │  │   write to tick N buffer                │  │
 │  │   check direct signals                  │  │
 │  └─────────────────────────────────────────┘  │
 │                                               │
 │  Phase 3: Post-tick                           │
 │  ┌─────────────────────────────────────────┐  │
 │  │ Evaluate events (against pre-step state)│  │
 │  │ Apply triggered actions (TOML order)    │  │
 │  │ Collect tick N outputs                  │  │
 │  │ Arbitrate bus requests                  │  │
 │  │ Relay to outer bus                      │  │
 │  │ Check direct signals                    │  │
 │  └─────────────────────────────────────────┘  │
 │                                               │
 └───────────────────────────────────────────────┘
```

### Phase 1: Pre-tick Processing

Phase 1 runs after all partitions have completed tick N-1 and before any partition
begins tick N. This is the tick boundary — the synchronization point where the
compositor has exclusive control and no partition is executing.

The compositor uses this window to process requests that must see a consistent view of
the entire system:

**Direct signals** are checked first. A safety-critical signal that arrived during the
previous tick is processed before any new computation begins.

**Spawn and despawn requests** (SIM-SYS-010) are processed here so that all partitions
in the upcoming tick see the same set of active vehicles. If spawn were processed
mid-tick, some partitions would see the new vehicle and others would not.

**Dump and load requests** (SIM-SYS-045) are processed at the tick boundary so that
state snapshots capture temporally consistent data — every partition's contribution
comes from the same completed tick.

**WorldState assembly** (SIM-SYS-009) collects all partition outputs from tick N-1 and
publishes them as an atomic aggregate. The tick barrier ensures that no partition's
output is missing or stale.

**Buffer swap** transitions the double-buffer: the write buffer from tick N-1 (now
complete) becomes the read buffer for tick N. A fresh write buffer is prepared for
tick N outputs.

### Phase 2: Partition Stepping — Intra-tick Message Isolation

Phase 2 is the core simulation work. The normative model is sequential: the compositor
calls `step(dt)` on each partition one at a time, in a deterministic order that is
stable across ticks. Between each partition's step, the compositor checks for direct
signals — giving safety-critical signals a worst-case latency of one partition's step
duration.

If a partition is itself a compositor, its `step()` call executes a complete nested
three-phase tick lifecycle for its own sub-partitions. The fractal structure means tick
lifecycles nest recursively — the outer compositor does not resume until the inner
compositor's entire tick (all three phases) has completed.

**Concurrent stepping** is an allowed optimization. A compositor may step partitions in
parallel — for example, under network transport where sequential stepping would impose
a round-trip per partition. This is safe because the double-buffer already isolates
partitions from each other's current-tick outputs: concurrent writers go to isolated
write paths, and all readers see the same immutable tick N-1 buffer. The invariants
that must hold are: write paths don't contend, bus requests are collected thread-safely,
all steps complete before Phase 3 (the tick barrier), and direct signals are checked at
least once before Phase 3 (worst-case latency becomes the longest partition's step
duration rather than one partition's). The simulation result is identical to sequential
stepping — the double-buffer guarantees this.

The key invariant is **intra-tick message isolation**:

> A message published by partition A during tick N is not visible to any other partition
> until tick N+1.

All partitions read from the tick N-1 buffer (the read buffer) and write to the tick N
buffer (the write buffer). No partition sees another's current-tick output, regardless
of step order. This is the double-buffered approach:

```
Tick N-1 outputs (read buffer):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Physics  │  │  GN&C    │  │   Env    │
  │ output   │  │  output  │  │  output  │
  │ (N-1)    │  │  (N-1)   │  │  (N-1)   │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │read         │read         │read
  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
  │ Physics  │  │  GN&C    │  │   Env    │
  │ step(dt) │  │  step(dt)│  │  step(dt)│
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │write        │write        │write
  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
  │ Physics  │  │  GN&C    │  │   Env    │
  │ output   │  │  output  │  │  output  │
  │ (N)      │  │  (N)     │  │  (N)     │
  └──────────┘  └──────────┘  └──────────┘
Tick N outputs (write buffer)
```

This eliminates ordering sensitivity entirely. Whether Physics steps before or after
GN&C, both read the same tick N-1 data and produce the same outputs. The simulation
result is identical regardless of step order — which is exactly what makes the transport
independence guarantee enforceable, because different transport modes may step partitions
in different orders or concurrently.

**Direct signal polling** under sequential stepping occurs between each pair of partition
steps, giving safety-critical signals a worst-case latency of one partition's `step()`
duration. Under concurrent stepping, the compositor checks signals at least once after
all steps complete, giving a worst-case latency of the longest partition's step duration.
In both cases the compositor checks signals while it has exclusive control — not while
partition state is being mutated — avoiding reentrancy hazards.

### Phase 3: Post-tick Processing

Phase 3 runs after all partitions have completed their step for tick N.

**Event evaluation** uses snapshot semantics: all event conditions are evaluated against
the partition state as it existed at the beginning of the tick (pre-step state), before
any event actions have been applied. The set of triggered events is collected, and their
actions are applied in TOML declaration order. No action's side effects are visible to
other event conditions until the following tick. This is consistent with the
double-buffered approach — events see a snapshot of state, just as partitions see a
snapshot of inter-partition data.

**Bus request processing** handles `ExecutionStateRequest` messages and other requests
emitted during the tick, with conflict resolution per SIM-SYS-043.

**Relay** forwards qualified requests to the outer bus per SIM-SYS-057.

## Bus Delivery Semantics

*Specified in SIM-SYS-063.*

The tick lifecycle defines when messages are visible. Delivery semantics define *how
many* messages a consumer sees, addressing what happens when a producer publishes faster
than a consumer reads.

Each message type declares its delivery semantic in the contract crate as part of its
interface contract. The key distinction is between continuous state messages (where the
consumer only needs the current value) and request messages (where every instance must
be processed and dropping one is a correctness failure). Under async transport with
partitions at different rates (SIM-SYS-013), this distinction determines whether a
backlog accumulates. The declared semantic must behave identically across all transport
modes, supporting the transport independence guarantee.

## Relationship to Other Architecture Elements

The tick lifecycle is the compositor's runtime contract — it extends the compositor
runtime role (SIM-SYS-056) with precise timing semantics. It integrates with:

- **Layer-scoped buses** (SIM-SYS-055): The double-buffer operates per bus instance.
  Each compositor at each layer runs its own tick lifecycle on its own bus.
- **Direct signals** (SIM-SYS-060): Polling between partition steps gives signals
  sub-tick latency while avoiding reentrancy hazards.
- **Recursive state contribution** (SIM-SYS-059): The tick boundary in Phase 1 ensures
  that `contribute_state()` calls observe consistent, completed state.
- **Compositor fault handling** (SIM-SYS-058): Faults during any trait call in any
  phase are caught by the compositor, wrapped with diagnostic context, and propagated
  upward — the compositor does not silently absorb failures.
- **Transport abstraction** (SIM-SYS-005): The lifecycle is the mechanism that makes
  the transport independence guarantee enforceable — by isolating partitions from
  each other's current-tick outputs, the result becomes independent of step order and
  timing, which is what varies across transport modes.

## Cross-Reference

| Requirement | Relationship |
|---|---|
| SIM-SYS-005 | Tick lifecycle makes transport independence enforceable |
| SIM-SYS-009 | WorldState assembled in Phase 1 with tick barrier and atomicity |
| SIM-SYS-010 | Spawn/despawn processed in Phase 1 at tick boundary |
| SIM-SYS-035–039 | Event evaluation in Phase 3 with snapshot semantics |
| SIM-SYS-045 | Dump/load processed in Phase 1 at tick boundary |
| SIM-SYS-055 | Double-buffer operates per layer-scoped bus instance |
| SIM-SYS-056 | Tick lifecycle extends compositor runtime role |
| SIM-SYS-057 | Relay occurs in Phase 3 |
| SIM-SYS-058 | Fault handling covers all trait calls in all phases |
| SIM-SYS-060 | Direct signals polled between partition steps in Phase 2 |
| SIM-SYS-062 | This explainer |
| SIM-SYS-063 | Bus delivery semantics declared per message type |
