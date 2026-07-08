# FlowLattice Planning Suite

A dependency-free, single-file planning application for algebraic material flow, network optimization, lot-level allocation, and imported LP/MPS models.

The entire suite—user interface, parsers, model classifier, solvers, sample models, Web Worker source, and export tools—is embedded in one HTML file:

```text
FlowLattice_Planning_Suite_Standalone.html
```

Open it directly in a modern browser. No installation, package manager, server, external script, font, or network connection is required.

> **Project status:** experimental reference implementation. The specialized network, bitmatrix, and lot-quotient paths are exact when their applicability checks pass. The generic dense LP/MILP fallback is intended for small and medium models, not arbitrary industrial-scale mixed-integer programs.

## Why FlowLattice

Conventional planning systems often expand every material, time period, facility, and physical lot into separate scalar variables. FlowLattice keeps the problem’s algebraic composition visible.

This allows the suite to select a solver from the model’s structure rather than sending every instance through one generic optimizer.

The browser app includes:

- packed bitmatrix and bit-shift routing for dense material blocks;
- independent per-commodity minimum-cost flow;
- exact lot-equivalence quotienting with physical identity restoration;
- incidence-network and transportation recognition for imported models;
- coupled LP compilation for recipes and shared resources;
- a bounded branch-and-bound fallback for integer models;
- MPS, CPLEX-LP, FlowLattice JSON, and lot JSON import;
- model, plan, canonical LP, certificate, and lot-allocation export;
- Blob-backed Web Worker execution to keep the interface responsive.

## Quick start

1. Download `FlowLattice_Planning_Suite_Standalone.html`.
2. Open the file in Chrome, Edge, Firefox, or another modern desktop browser.
3. Drop an `.mps`, `.lp`, or `.json` model onto the input area—or load an embedded sample.
4. Leave the engine set to **Adaptive exact** unless testing a particular backend.
5. Review the recommended path and exactness conditions.
6. Select **Solve plan**.
7. Export the normalized model, plan, canonical LP, or lifted lot identities.

No model data leaves the browser.

## Adaptive solver paths

| Backend | Selected when | Main representation |
|---|---|---|
| **Packed bitmatrix** | Materials share a compatible time-expanded topology, each commodity has one inventory source, and there are no recipes, procurement choices, or shared lane/resource constraints | Packed material coordinates and shared shortest-path trees |
| **Independent commodity MCF** | Commodities can be solved independently, including distributed inventory and procurement choices without cross-commodity coupling | One time-expanded graph with independent material flows |
| **Lot-equivalence quotient** | The model contains physical lots or precompiled lot classes | Integer multiplicities over planning-equivalent lot classes, followed by exact identity lifting |
| **Incidence / transportation MCF** | An imported MPS or LP model is recognized as a network or transportation matrix | Sparse minimum-cost flow |
| **Coupled LP** | Recipes, shared capacities, resources, or other continuous coupling prevent decomposition | Canonical scalar LP compiled from the FlowLattice model |
| **Coupled MILP** | The compiled model contains integer or binary variables | Dense LP relaxations with bounded branch-and-bound |

The suite checks applicability before invoking a specialized backend. Forcing an incompatible backend returns `unsupported` rather than silently changing the model.

## Supported input formats

### FlowLattice algebraic JSON

Use schema `flowlattice-algebraic-v2` for facilities, periods, lanes, commodities, recipes, procurement, demands, and shared resources.

```json
{
  "schema": "flowlattice-algebraic-v2",
  "F": 4,
  "H": 5,
  "default_hold_cost": 1,
  "lanes": [
    { "u": 0, "v": 1, "tau": 1, "cost": 2 },
    { "u": 1, "v": 3, "tau": 1, "cost": 1 }
  ],
  "commodities": [
    {
      "id": "part-A",
      "inventory": [{ "v": 0, "t": 0, "qty": 100 }],
      "procurement": [],
      "demands": [{ "v": 3, "t": 4, "qty": 100 }],
      "disposal": []
    }
  ],
  "recipes": [],
  "resources": {},
  "warehouses": {}
}
```

Facility indices range from `0` to `F - 1`; time periods range from `0` to `H - 1`. A lane’s `tau` is its transit time in periods.

### Lot-level JSON

Use schema `flowlattice-lot-v1` for physical-lot allocation with release, expiry, quality, origin, and eligibility restrictions.

```json
{
  "schema": "flowlattice-lot-v1",
  "graph": {
    "F": 3,
    "H": 4,
    "hold_cost": 1,
    "lanes": [{ "u": 0, "v": 2, "tau": 1, "cost": 3 }]
  },
  "lots": [
    {
      "id": "L001",
      "commodity": "part-A",
      "facility": 0,
      "release_period": 0,
      "expiry_period": 3,
      "quality": 2,
      "origin": 0,
      "unit_qty": 1
    }
  ],
  "demands": [
    {
      "commodity": "part-A",
      "facility": 2,
      "target_period": 3,
      "min_quality": 2,
      "allowed_origins": "1",
      "count": 1
    }
  ]
}
```

Lots are grouped only when all planning-relevant attributes match. The result stores class allocations as exact `(class_id, offset, count)` slices; **Lift lot identities** maps those slices back to physical lot IDs.

### MPS

The parser supports the common free-field sections used by linear and mixed-integer models:

- `NAME`
- `OBJSENSE`
- `OBJNAME`
- `ROWS`
- `COLUMNS`
- `RHS`
- `RANGES`
- `BOUNDS`
- `ENDATA`
- integer markers using `INTORG` / `INTEND`

Imported models are normalized to the suite’s canonical LP representation. Incidence and transportation structures are detected before the generic fallback is considered.

### CPLEX-LP

The LP parser supports the usual sections:

- `Minimize` / `Maximize`
- `Subject To`
- `Bounds`
- `Binary` / `Binaries`
- `General` / `Generals` / `Integer`
- `End`

Linear objectives, named constraints, free variables, ranged bounds, continuous variables, integer variables, and binaries are supported.

## Outputs

Depending on the model and backend, the suite can export:

- **Model JSON** — normalized imported or authored model;
- **Plan JSON** — objective, flows, allocations, routes, solver metadata, and certificates;
- **Canonical LP** — scalar LP generated from a FlowLattice model;
- **Lifted lot identities** — physical lots assigned to each demand event;
- **Infeasibility information** — residual cuts or backend-specific diagnostics where available.

## Lot-level quotienting

The lot backend replaces repeated, symmetric physical-lot decisions with integer class multiplicities.

Two lots may share a class only when they are indistinguishable to every modeled cost and constraint—for example:

- commodity;
- source facility;
- release and expiry periods;
- quality grade;
- origin;
- unit quantity;
- any other attribute used by eligibility or cost logic.

This is an exact symmetry quotient, not aggregation by approximation. Canonical lot identity is restored after optimization through class-contiguous basis ranges.

Split classes whenever the model contains lot-specific restrictions, serial-number decisions, pairwise incompatibilities, or individual costs.

## Packed bitmatrix path

The packed backend exploits repeated graph structure and dense material blocks. Quantities are stored in bit-sliced planes, while shared topology and source/cost profiles reuse shortest-path trees.

It is most useful when many commodities share:

- the same time-expanded network;
- the same source event or a small number of source classes;
- compatible transport and holding-cost profiles;
- no shared capacities or recipe coupling.

Sparse models may be faster through ordinary scalar coordinates. The adaptive classifier uses a density hint to avoid packing where it is unlikely to help.

## Browser benchmark

The **Benchmarks** tab contains:

- embedded historical benchmark CSVs;
- a current-browser benchmark runner;
- separate summaries for the algebraic, bitmatrix, and lot-quotient paths.

Treat benchmark numbers as workload- and hardware-specific. For meaningful comparisons, preserve identical model semantics, confirm objective equality, report whether a value is directly measured or projected, and include model-construction and identity-lifting costs when relevant.

## Limits and non-goals

This is not a replacement for a commercial-scale generic LP/MIP solver.

Important limits include:

- the generic LP fallback materializes a dense tableau and has a configurable dense-cell limit;
- branch-and-bound has explicit node and time limits and is suitable only for modest integer residuals;
- nonlinear objectives and nonlinear constraints are unsupported;
- numerical tolerances use JavaScript floating-point arithmetic;
- the MPS and LP parsers cover common linear-model syntax, not every vendor extension;
- arbitrary lot-specific combinatorics can destroy class symmetry and require a larger MILP;
- very large exports can be limited by browser memory and download behavior.

For large coupled LP/MILP instances, use the suite to classify, reduce, or export the model, then solve the remaining residual with a dedicated optimizer.

## Privacy and deployment

The default distribution is fully offline:

- no telemetry;
- no remote fonts;
- no CDN dependencies;
- no model upload;
- no server component.

When hosting under a restrictive Content Security Policy, allow Blob workers, for example:

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; worker-src 'self' blob:
```

A stricter deployment can extract the inline script and worker into hashed local files instead.

## Repository layout

The minimal repository can contain only:

```text
.
├── FlowLattice_Planning_Suite_Standalone.html
├── README.md
└── LICENSE
```

The HTML contains named embedded source blocks for the parser, canonical solver, bitmatrix engine, lot engine, hybrid planner, worker body, samples, and interface logic.

## Development

There is no build step.

1. Edit `FlowLattice_Planning_Suite_Standalone.html`.
2. Open it locally in a browser.
3. Run each embedded sample.
4. Test imported MPS and LP files.
5. Verify objective values and lot-identity lifting.
6. Run the current-browser benchmark before publishing performance changes.

When extending the suite:

- add new semantics to the canonical model first;
- make backend applicability checks explicit;
- return `unsupported` instead of approximating silently;
- validate every specialized result against a canonical formulation on small instances;
- preserve provenance and identity information through every quotient and lift.

## Contributing

Issues and pull requests are welcome for:

- additional MPS and LP syntax coverage;
- sparse LP and MILP residual solvers;
- stronger infeasibility certificates;
- additional exact algebraic fast paths;
- lot-policy and traceability constraints;
- reproducible benchmark models;
- numerical validation and browser compatibility improvements.

Please include a minimal model, expected status and objective, browser/version information, and whether the issue affects parsing, classification, solving, or export.
