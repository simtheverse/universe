# The Fractal Partition Pattern

## What problem does it solve?

Flight simulation frameworks face a tension between modularity and coherence. A system
that lets you swap out the atmosphere model should also let you swap out the entire
physics partition, and ideally let someone swap out *universe itself* as a component in
their larger pipeline. But modularity at multiple scales typically requires multiple
mechanisms — plugin systems for small components, service interfaces for large ones,
embedding APIs for the framework as a whole — and each mechanism has its own conventions
for configuration, events, composition, and documentation.

The fractal partition pattern resolves this by making the answer the same at every scale.

## The core idea

The system is decomposed into **layers** and **partitions**. At each layer, partitions
are independently replaceable units that conform to contracts defined by the layer above.
The defining property — the **layer and partition uniformity principle** — is that the
same structural primitives appear at every layer:

- **Contracts** define what a partition must provide and what it may consume.
- **Compositors** select and assemble partition implementations at startup.
- **Events** are defined, armed, and triggered using the same schema at every layer.
- **Composition fragments** configure partition selections using the same TOML format,
  override semantics, and inheritance rules at every scope.
- **Specifications** follow the same requirement format with bidirectional traceability
  to the layer above.
- **Documentation** follows the same Diátaxis structure at every partition.

The pattern is called "fractal" because zooming into any partition reveals the same
structure as the system as a whole, and zooming out reveals that universe itself is a
partition in someone else's system.

## Layers in practice

In universe, the layers are:

- **Layer 0 (system):** The four top-level partitions — physics, GN&C, visualization,
  and UI — composed by the orchestrator (`sim-app`). Contracts live in `sim-core`.
  Configuration at this layer is the **session** — transport mode, timestep, which
  scenarios to compose.

- **Layer 1 (partition):** Sub-components within each partition — atmosphere model,
  gravity model, aerodynamics within physics; estimator, guidance law, control law within
  GN&C; mesh rendering, trails, HUD within visualization. Configuration at this layer is
  the **scenario** — vehicle definitions, sub-model selections, environment models,
  events, initial conditions.

- **Layer 2+ (sub-partition):** Where a layer 1 sub-component is itself complex enough
  to warrant decomposition, the pattern continues. The atmosphere model might compose
  a base atmosphere, a turbulence model, and a wind model as independently replaceable
  units with their own contracts.

The pattern does not prescribe a fixed number of layers. Decomposition continues as deep
as the domain requires, and stops where it doesn't.

## What emerges from the pattern

Several properties that would otherwise need to be designed and maintained separately
fall out naturally from the uniformity principle.

### Fidelity is not a special concept

A "fidelity tier" is a named composition fragment that selects a specific set of layer 1
physics sub-models. `"low"` selects flat-earth gravity and simplified aerodynamics.
`"high"` selects WGS84 gravity, ISA atmosphere, Dryden turbulence, and full propulsion
dynamics. There is no fidelity-specific machinery — it is the same named composition
fragment mechanism available to visualization presets, GN&C configurations, or any other
domain. Fidelity emerges from composition rather than being engineered into the system.

### Configuration collapses to one concept

Sessions, scenarios, vehicle definitions, physics presets, and visualization presets are
all **composition fragments** — TOML blocks that select partition implementations at some
scope. They all support the same `extends` inheritance, the same inline override
semantics, and the same named-reference resolution. A session is a composition fragment
at layer 0 that references scenario fragments at layer 1. A scenario is a composition
fragment that references vehicle and physics fragments at narrower scopes.

An outer-layer fragment can override any field defined by an inner-layer fragment it
composes. This means a session file can fully define all scenario content inline —
useful for simple cases or self-contained examples — or it can reference separate
scenario files and override only the parameters relevant to a specific run. The same
mechanism works at every scope: a scenario can override individual sub-model selections
within a physics preset, a vehicle definition can override specific initial conditions
from a template.

### Events work everywhere

The event system — time-triggered, condition-triggered, declaratively defined in TOML —
is the same at every layer. A system-level event that pauses the simulation at wall-clock
T+30s uses the same schema as a GN&C event that fires a mode transition at scenario time
T+60s, which uses the same schema as a physics event that stops the simulation when
structural load exceeds a threshold.

Because events can request execution state transitions (`sim_pause`, `sim_stop`,
`sim_resume`), any partition at any layer can influence the simulation lifecycle through
the same mechanism. A plant model detecting a limit exceedance, a GN&C algorithm reaching
a scripted checkpoint, and an operator clicking a pause button all emit the same
`ExecutionStateRequest` on the bus. The orchestrator arbitrates — stop beats pause beats
resume — but the mechanism is uniform.

### Documentation and specifications propagate

Each partition maintains a `docs/` directory with the same Diátaxis quadrants (tutorials,
how-to, reference, explanation, design) and a `SPECIFICATION.md` with the same format.
A contributor navigating from the system-level physics partition into its atmosphere
sub-partition finds the same documentation layout, the same specification structure, and
the same traceability conventions. Specifications at each layer nucleate specifications
at the next layer — this document nucleates partition specs, which nucleate sub-partition
specs where warranted — propagating traceability and structural discipline to arbitrary
depth.

### Universe is a partition in your system

The pattern applies outward. Universe exposes a library interface that accepts a session
fragment, executes the simulation, and returns control without assuming process ownership.
A laboratory's automation pipeline can generate composition fragments, invoke universe as
one stage in a pre-process/simulate/post-process workflow, and consume its outputs.
Universe is a partition within that pipeline's layer 0, swappable for an alternative
simulation engine through the same replaceability guarantee universe provides to its own
sub-partitions.

## Design choices and tradeoffs

### Why uniformity over specialization

Specialized mechanisms are often more ergonomic for their specific case. A purpose-built
fidelity system could offer a cleaner API for selecting physics complexity levels. A
purpose-built event system for execution control could avoid the generality of condition
predicates.

The fractal partition pattern trades this local ergonomic advantage for global
consistency. A contributor who learns how composition fragments work in physics
configuration immediately understands how they work in visualization configuration,
scenario authoring, and session setup. A contributor who learns how events work at the
system level immediately knows how to define events within a partition. The number of
concepts in the system stays constant as the system grows deeper.

### Why layers over a flat plugin architecture

A flat plugin system (everything is a plugin at one level) avoids the complexity of
layers but sacrifices the ability to reason about scope and override direction.
Configuration override semantics depend on knowing which scope is "outer" and which is
"inner" — a session overrides a scenario, not the reverse. Event time semantics depend
on knowing which layer you are at — wall-clock at layer 0, scenario time at layer 1.
The layer concept makes these relationships explicit.

### Why not enforce a fixed depth

The pattern does not mandate that every partition decompose further. A partition with
a single implementation and no meaningful sub-components maintains the `docs/` structure
and specification format but has nothing to decompose. Forcing decomposition where the
domain doesn't warrant it produces empty abstractions. The pattern provides the mechanism
for decomposition and lets the domain determine the depth.

### Why specifications perpetuate

Requiring each layer's spec to nucleate the next layer's specs is more overhead than a
single system-level spec with all requirements. The tradeoff is traceability at scale:
when a system has dozens of independently replaceable sub-components across multiple
layers, a single flat spec becomes unmanageable. Layered specs with bidirectional
traceability make it possible to verify allocation (every system intent reaches an
implementor) and coverage (every implementation traces to an intent) at each layer
boundary independently. And besides, a single system-level specs can possibly be generated by hydrating it with a recursion of inner layer specs.
