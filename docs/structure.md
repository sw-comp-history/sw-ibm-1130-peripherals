# Repository Structure: IBM 1130 Peripheral Emulators

> **Status**: Decision doc (draft). Supersedes any default one-repo-per-device layout.

## Decision

**One repository, one Cargo workspace, separate crates per device.**

Create a single repo `ibm-1130-peripherals-rs` containing a Rust workspace
with one crate (or app) per emulated peripheral:

- `ibm-029-keypunch`
- `ibm-1442-card-io`
- `ibm-1627-plotter` (later)
- multiplexor (later)
- tape drive (later)
- light-pen vector graphics terminal (later)

Do **not** create separate GitHub repos per device.

## Rationale

### Shared, evolving code — the decisive factor

The first two devices consume and produce the **same physical artifact**: the
80-column Hollerith punch card. They must agree on:

- Hollerith encoding (zone + numeric + 8-punch patterns)
- EBCDIC <-> Hollerith byte mapping (80-col interchange)
- The authentic 108-byte IBM 1130 object-deck binary format

That shared model already exists and is well-tested as
`punch-card-core` at `ibm-1130-rs/crates/punch-card-core/`. Splitting these
two into separate repos would immediately force `punch-card-core` to become a
third cross-repo dependency, and a card-format change would require coordinated
commits across repos.

### Precedent already exists in this collection

- `ibm-1130-rs` is already a multi-crate workspace (components crate +
  `crates/punch-card-core`).
- The separate-repo family (`sw-ibm1130-*`, `sw-cdp1802-*`) splits on
  **independent** software layers (ISA / codegen / emulator) that share no
  code. Two devices sharing one card-deck format is the opposite situation.

### Later devices fit without friction

Plotters, tapes, multiplexors, and vector terminals have their own media
models (tape images, vector command streams) and share very little with the
cards. Adding them as thin app crates in the same workspace costs nothing;
each is independent, so one can still be promoted to its own repo later if it
ever gains an independent release/deployment cadence.

### Demo footprint

One repo deploys one coherent demo tree under the org Pages URL:

```
https://sw-comp-history.github.io/ibm-1130-peripherals-rs/<device>/
```

vs. N disconnected per-device sites with no shared navigation.

## Repository Layout

```
ibm-1130-peripherals-rs/
├── Cargo.toml                     # workspace + resolver
├── README.md
├── LICENSE
├── serve.sh                       # local preview (ALWAYS port 9352, per convention)
├── docs/
│   └── structure.md               # this document
├── crates/
│   ├── punch-card-core/           # moved verbatim from ibm-1130-rs
│   │   ├── Cargo.toml
│   │   └── src/ (hollerith.rs, ebcdic.rs, punch_card.rs, lib.rs)
│   └── peripheral-components/     # shared Yew UI components (cards/panels/tab chrome)
│       ├── Cargo.toml
│       └── src/ (keypunch.rs, punch-card-svg, ...)
└── apps/
    ├── ibm-029-keypunch/          # keypunch web demo -> pages/keypunch/
    │   ├── Cargo.toml             # bin crate (cdylib)
    │   ├── index.html
    │   ├── Trunk.toml             # public_url = "/ibm-1130-peripherals-rs/keypunch/"
    │   ├── src/
    │   └── static/                # device CSS
    ├── ibm-1442-card-io/          # reader/punch demo -> pages/card-io/
    │   ├── Cargo.toml
    │   ├── index.html
    │   ├── Trunk.toml
    │   └── src/
    └── ...                        # later devices follow the same shape
```

### Top-level Cargo.toml

```toml
[workspace]
members = [
  "crates/punch-card-core",
  "crates/peripheral-components",
  "apps/ibm-029-keypunch",
  "apps/ibm-1442-card-io",
]
resolver = "2"
```

## Naming Conventions

- Repo: `ibm-1130-peripherals-rs` (matches `ibm-1130-rs` family style).
- App crates use era-accurate model numbers where one exists:
  - `ibm-029-keypunch`
  - `ibm-1442-card-io`
  - `ibm-1627-plotter`
  - light-pen terminal: use the real model number if identified
    (e.g. IBM 2250/2260 family), otherwise `light-pen-vector-terminal`.
- Shared crates: `punch-card-core` (keep the existing name), `peripheral-components`.

## Migration Plan (Phase 1: 029 + 1442)

1. `git init` repo; add workspace `Cargo.toml`.
2. Vendor `crates/punch-card-core` verbatim from `ibm-1130-rs`
   (preserve tests; the 108-byte object-deck format and EBCDIC mapping are the
   cross-device contract).
3. Extract `peripheral-components` from `ibm-1130-rs` shared Yew components
   (keypunch UI, punch-card SVG renderer, tab chrome) as a shared crate.
4. Build `apps/ibm-029-keypunch` first (port the existing demo 1:1).
5. Build `apps/ibm-1442-card-io` on the same `punch-card-core`:
   deck read-back, reader feed, card punching from a source stream.
6. Deploy under one GitHub Pages tree.

## Rules of Thumb

- Keep shared, evolving domain code (`punch-card-core`) in this workspace.
- Promote a crate to its own repo only when it gains an independent release
  cadence, not pre-emptively.
- Every app crate keeps the same `page/Trunk.toml`/9352 (`serve.sh`) shape as
  `ibm-1130-rs` for consistency.