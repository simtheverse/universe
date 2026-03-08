# Testing in the Fractal Partition Pattern

## What problem does it solve?

A partitioned system needs a testing strategy that matches its decomposition structure.
The naive approach is a binary split: unit tests for small functions, integration tests
for the assembled system. But this misses the middle — the partition boundaries, the
contract conformance, the compositor assembly — and it doesn't scale with depth. When
layer 1 partitions decompose into layer 2 sub-partitions, a flat unit-vs-integration
distinction doesn't tell you which layer a test is verifying or which contract it is
exercising.

The fractal partition pattern resolves this by making the testing structure mirror the
partition structure. Just as the same structural primitives (contracts, compositors,
events, configuration) appear at every layer, the same testing primitives appear at
every layer. A test at layer N verifies a contract defined at layer N. The test can
run in isolation because the contract defines a boundary. The testing strategy does not
need to be redesigned as the system deepens — it propagates with the pattern.

## Testing follows the layers

### Contract tests

The fundamental unit of testing in a fractally partitioned system is the **contract
test**: a test that verifies an implementation satisfies its layer's contract. At
layer 0, a contract test instantiates a partition implementation, invokes it through the
traits defined in `sim-core`, and asserts that the outputs conform to the contract's
behavioral requirements. At layer 1, a contract test does the same for a sub-partition
implementation against the traits defined in the partition's contract module.

Contract tests are what most frameworks call "unit tests," but the term is misleading
here. A contract test for the physics partition might exercise a complete timestep
involving gravity, atmosphere, and aerodynamics sub-models composed together. It is a
"unit" only in the sense that it tests one unit of replaceability — one partition — in
isolation from its peers at the same layer.

The key property is **isolation at the layer boundary**. A contract test for the physics
partition does not require a running GN&C partition, visualization partition, or UI
partition. It supplies inputs that conform to the contract's input types and asserts
outputs that conform to the contract's output types. The contract boundary is the
isolation boundary.

At layer 1, the same structure applies. A contract test for the atmosphere sub-model
within physics does not require the aerodynamics or gravity sub-models. It tests the
atmosphere implementation against the atmosphere contract trait in isolation.

### Compositor tests

A **compositor test** verifies that the compositor at a given layer correctly assembles
its partitions and that the assembled whole behaves correctly. At layer 0, this means
the orchestrator (`sim-app`) composes the four partitions and they exchange messages
correctly through the bus. At layer 1, this means a partition's internal compositor
assembles its sub-models and they interact correctly.

Compositor tests are what most frameworks call "integration tests." The distinction
matters because a compositor test has a defined scope: it tests one layer's assembly,
not the entire system. A layer 0 compositor test verifies that the four partitions
communicate correctly through `sim-core` contracts and the bus. It does not verify the
internal correctness of each partition's sub-models — that is the job of layer 1
contract tests.

This scope discipline prevents the common failure mode of integration tests: testing
everything at once, so that failures are difficult to localize and tests are slow and
brittle. A compositor test at layer N assumes that layer N+1 contract tests pass. It
tests composition, not implementation.

### System tests

A **system test** exercises the full simulation stack from session configuration to
final output, using the same entry point an operator would use. System tests verify
end-to-end properties that emerge from the interaction of all layers: a vehicle
launched with specific initial conditions reaches a specific final state; a scenario
with scripted events produces the expected sequence of execution state transitions;
telemetry output from a complete run matches a reference.

System tests are expensive and slow relative to contract tests. Their role is not to
catch bugs in individual partitions — contract tests do that — but to verify that the
system-level requirements in this specification are met when all partitions are composed
together. Each system test traces to one or more SIM-SYS requirements.

### The relationship between levels

The three test types form a pyramid that mirrors the layer structure:

```
            ┌─────────────┐
            │ System tests│   Full stack, session-to-output
            │  (layer 0)  │   Verify system requirements
            ├─────────────┤
            │ Compositor  │   One layer's assembly
            │   tests     │   Verify composition correctness
            ├─────────────┤
            │  Contract   │   One partition in isolation
            │   tests     │   Verify contract conformance
            └─────────────┘
```

At each layer N:
- **Contract tests** verify that each partition at layer N+1 satisfies its contract.
- **Compositor tests** verify that the compositor at layer N correctly assembles layer
  N+1 partitions.
- **System tests** exist only at layer 0, verifying end-to-end properties.

Because the fractal pattern allows arbitrary depth, the contract/compositor pair
repeats at every layer. Layer 1 contract tests verify sub-model implementations.
Layer 1 compositor tests verify that the partition's internal compositor assembles
sub-models correctly. Layer 2 contract tests verify sub-sub-model implementations,
and so on.

## What emerges from the pattern

### Test isolation matches partition isolation

A contract test's isolation boundary is the same as the partition's replaceability
boundary. If a partition can be replaced without modifying its peers, a contract test
can run without instantiating its peers. The structural guarantee that makes the system
modular is the same guarantee that makes the tests fast and independent.

This is not a coincidence — it is a direct consequence of the contract-centric design.
The contract defines both what a replacement must provide (for modularity) and what a
test must assert (for correctness). A single artifact serves both purposes.

### Alternative implementations are testable by construction

When a contributor provides an alternative implementation of a partition — a student's
GN&C plugin, a lab's custom atmosphere model — the contract tests for that partition's
contract apply unchanged. The test suite for the atmosphere contract verifies any
atmosphere implementation, not just the default one. If the alternative passes the
contract tests, it is a valid replacement. If it doesn't, the failures identify exactly
which contract obligations are unmet.

This eliminates the common problem of alternative implementations that "mostly work"
but fail in edge cases that aren't covered by ad-hoc tests. The contract tests are
comprehensive with respect to the contract, and the contract is the complete definition
of what an implementation must do.

### Transport independence is testable

The requirement that all three transport modes produce identical results (SIM-SYS-005)
becomes a compositor test parameterized over transport mode. The same scenario runs
three times — in-process, async, networked — and the final vehicle state is compared.
This test exists at layer 0 because transport is a layer 0 concern. Layer 1 tests are
unaware of transport because layer 1 partitions are unaware of transport.

### Fidelity tiers are testable without special machinery

A fidelity tier is a named composition fragment that selects a set of sub-model
implementations. Testing a fidelity tier is a compositor test at layer 1: compose the
sub-models selected by the tier, run a reference scenario, and compare outputs against
the expected reference for that fidelity level. No fidelity-specific test infrastructure
is needed — it is the same compositor test with a different composition fragment input.

### Snapshot round-trip is testable

State snapshot dump and load (SIM-SYS-044) is testable as a compositor test: run a
scenario to time T, dump state, load the dumped state into a fresh simulation, continue
to time T+N, and compare the result with a continuous run from 0 to T+N. Because state
snapshots are composition fragments, the dump output is human-readable and diffable,
making test failures diagnosable.

## Design choices and tradeoffs

### Why contract tests, not mock-based unit tests

The conventional approach to testing modular systems is to mock dependencies and test
each module in isolation against its mocks. The fractal partition pattern avoids mocks
in favor of contract tests for a specific reason: the contract is already a precisely
defined interface. A mock is an ad-hoc reimplementation of that interface for testing
purposes. If the mock diverges from the real contract behavior, the tests pass but the
integration fails.

Contract tests avoid this by testing against the contract itself, not against a mock of
it. The test supplies real inputs and asserts real outputs as defined by the contract
types. Where a test needs to supply data that would normally come from another partition,
it constructs instances of the contract's input types directly — this is not a mock, it
is using the contract's type system as intended.

There is a cost: contract tests for complex partitions may require nontrivial input
construction. A physics contract test must supply a plausible `GNCCommand` as input.
This is more work than mocking a `GNCCommandProvider` trait. The tradeoff is that the
test exercises the actual contract boundary and cannot silently diverge from the real
interface.

### Why compositor tests have a defined scope

It would be simpler to have two levels — unit tests and end-to-end tests — without the
intermediate compositor test. The problem is that end-to-end tests are slow, brittle,
and hard to debug. When an end-to-end test fails, the failure could be in any partition,
any sub-model, or in the composition itself. Localizing the failure requires
investigation that the test structure doesn't help with.

Compositor tests localize composition failures to a specific layer. If a layer 0
compositor test fails but all layer 1 contract tests pass, the problem is in the
inter-partition communication or the orchestrator's assembly logic, not in any
partition's internal implementation. This diagnostic property is worth the additional
test category.

### Why system tests trace to requirements

System tests could be organized by scenario, by vehicle type, or by feature. Organizing
them by requirement — each system test traces to one or more SIM-SYS identifiers — has
a specific benefit: coverage analysis is trivial. If every SIM-SYS requirement has at
least one system test, and all system tests pass, the system satisfies its specification.
If a requirement has no system test, the gap is visible in the traceability matrix.

This is the same traceability discipline that the fractal partition pattern applies to
specifications (SIM-SYS-033) and to test files (SIM-SYS-034), extended to system-level
verification.
