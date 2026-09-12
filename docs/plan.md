# Plan: Refactoring + Radio Music Demo

> See also: `structure.md` (repo layout, naming), `research.txt` (design notes).

---

## 1. Goals

1. **Centralize shared peripheral code** — punch-card model, 1442 reader, console
   state/rendering — in `sw-ibm-1130-peripherals`, so multiple demos consume it
   instead of duplicating or owning it locally.
2. **Build the radio-music demo** — a Rust/Yew/WASM exhibit that recreates the
   "Christmas-carols over a transistor radio" trick, fully testable without a browser
   at the model layer.
3. **Preserve existing demos** — every repo that already commits `pages/` must
   still build and deploy after the refactor; no broken live demos at any stage.

---

## 2. What moves, and where it goes

Everything below moves **from `ibm-1130-rs`** unless noted otherwise.
The new workspace structure in `sw-ibm-1130-peripherals`:

```
sw-ibm-1130-peripherals/
├── Cargo.toml                  # workspace
├── crates/
│   ├── punch-card-core/        # ← move verbatim from ibm-1130-rs
│   ├── machine-1130-core/      # ← extract: ConsoleState, Registers, ControlState,
│   │                            #    SpeedMode, LampState, ConsoleAction (pure model)
│   └── ibm-1442/               # ← new: CardReaderState, ReaderPhase, ReaderEvent
├── components/
│   └── console-ui/             # ← extract: ConsolePanel, IndicatorLights, CircularKnob,
│                                #    EmergencyStop, PowerSwitch, LampTestButton,
│                                #    SixteenBitPanel, ToggleSwitch (Yew)
├── apps/
│   ├── keypunch/               # ← extract: Keypunch, PunchCardSvg, Deck
│   ├── card-reader/            # ← new: 1442 animation app
│   └── radio-music/            # ← new: the feature demo
└── docs/
```

**Not moving** (stays in `ibm-1130-rs`):

- `src/cpu/` — the full 1130 instruction emulator
- `src/assembler.rs`, `src/challenge.rs` — the assembler game
- `components/sidebar`, `header`, `modal`, `program_area`, `register_panel`,
  `memory_viewer`, `printer` — assembler-game-specific UI
- `TabContainer`, `Tab` — trivially extractable later if needed

---

## 3. Refactoring sequence (ordered steps)

Each step is independently committable and testable. Nothing below breaks
an existing live demo until Step 3 is complete.

### Step 1 — Create workspace skeleton in `sw-ibm-1130-peripherals`

- `Cargo.toml` with `[workspace]` + resolver 2.
- `crates/punch-card-core/`: copy verbatim from `ibm-1130-rs/crates/punch-card-core/`.
  Keep all tests; run `cargo test` locally. No other crate depends on it yet.
- Verify: `cargo test -p punch-card-core` passes on a clean checkout.

**Commit message**: `chore: create workspace, add punch-card-core`

### Step 2 — Build the radio-music app skeleton (no shared deps yet)

Using `punch-card-core` only (it is already local):

- Create `apps/radio-music/` with a minimal Yew app skeleton (Trunk.toml,
  `public_url = "/sw-ibm-1130-peripherals/radio-music/"`, `index.html`, `src/app.rs`).
- Implement `music/deck.rs`: the text-based deck format from `research.txt`
  (`NOTE G4 1` lines → `Vec<Note>`), using `punch-card-core` for card encoding.
- Implement `music/timing.rs`: pure-Rust `TimingLoop` struct (nop_count,
  loop_cycles_us, resulting_hz). Testable with `cargo test`, no WebAudio.
- Implement `radio/synth.rs`: a minimal WebAudio harness (square-wave oscillator
  → `BiquadFilterNode` → `GainNode` → destination). No simulation-mode logic
  yet — just "verify we can make a tone."
- Wire into Yew: PLAY DECK button, card display (reuse `PunchCardSvg` or build
  a simpler one), frequency readout.
- Verify: `trunk build --release` produces `pages/radio-music/`; open in
  browser, hear a tone when PLAY is clicked.

**Commit message**: `feat: radio-music skeleton — deck, timing model, single tone`

### Step 3 — Move `keypunch` app out of `ibm-1130-rs` → `apps/keypunch/`

This is the first step that touches `ibm-1130-rs`.

- Create `apps/keypunch/` in the new repo; move `keypunch.rs`,
  `PunchCardSvg`, `Deck`, and the `keypunch.css` into it.
- Add a git dependency in `ibm-1130-rs/Cargo.toml`:
  ```toml
  punch-card-core = { git = "https://github.com/sw-comp-history/sw-ibm-1130-peripherals" }
  ```
  (or a `[patch]` override pointing at the local checkout during dev).
- In `ibm-1130-rs`, update the keypunch tab to import the shared
  `Keypunch`/`Deck`/`PunchCardSvg` types from the new crate.
- **Rebuild and verify `ibm-1130-rs` pages**:
  1. `./build-all.sh` (runs `trunk build --release` → `pages/` updated).
  2. `./serve.sh` on port 9352; manually exercise keypunch tab (type, navigate,
     save/load deck).
  3. `cargo clippy --all-targets` + `cargo test` in `ibm-1130-rs`.
  4. `git add pages && git commit` the rebuilt pages.
- Verify: `https://sw-comp-history.github.io/ibm-1130-rs/` keypunch tab still works
  after push.

**Commit message**: `refactor: pull keypunch from shared peripherals crate`

### Step 4 — Extract `machine-1130-core` (console model)

Extract from `ibm-1130-rs/components/src/components/console_panel.rs`
the **pure model types** (no Yew imports):

```rust
// In sw-ibm-1130-peripherals/crates/machine-1130-core/src/lib.rs
pub struct ConsoleState { ... }
pub struct Registers { ... }
pub struct ControlState { ... }
pub enum ConsoleAction { ... }
pub enum SpeedMode { ... }
```

- Publish in `crates/machine-1130-core/` with unit tests.
- In `ibm-1130-rs`, replace the inline definitions with a dependency on
  `machine-1130-core`. Update `app.rs` and `console_panel.rs` accordingly.
- Rebuild `ibm-1130-rs` pages; verify console tab still renders/works.

**Commit message**: `refactor: extract ConsoleState to shared machine-1130-core`

### Step 5 — Extract `console-ui` (Yew console panel)

Move `ConsolePanel` + its sub-components into `components/console-ui/`:

```
components/console-ui/src/
  lib.rs
  console_panel.rs
  circular_knob.rs
  indicator_lights.rs
  emergency_stop.rs
  power_switch.rs
  lamp_test_button.rs
  sixteen_bit_panel.rs
  toggle_switch.rs
  static/styles/
```

- Depends on `machine-1130-core` (model) + `yew` (UI).
- `ibm-1130-rs` switches to depending on the shared `console-ui` crate
  for the Console tab.
- Rebuild and verify `ibm-1130-rs` pages.

**Commit message**: `refactor: extract ConsolePanel into shared console-ui`

### Step 6 — Build `ibm-1442` crate + `card-reader` app

- Create `crates/ibm-1442/src/lib.rs`: `CardReaderState`, `ReaderPhase`,
  `ReaderEvent`, `feed_column()` (pure Rust, async-friendly, fully tested).
- Create `apps/card-reader/`: a Yew demo with 1442 side-view animation
  (input hopper, read station, stacker) driven by `ReaderEvent`s.
- Optional: `apps/keypunch/` gets a "Send to 1442" button that pipes
  `Deck` bytes through the reader model, demonstrating the round-trip.

**Commit message**: `feat: ibm-1442 crate + card-reader app`

### Step 7 — Integrate radio-music with shared peripherals

Replace the standalone timing model with the full chain:

```
Music Deck → CardReader1442 → TimingLoop/CPU activity model → RFI → WebAudio
```

- Wire the 1442 app components into `apps/radio-music/`.
- Implement "Explain" mode (separates deck → 1442 → 1131 → RFI diagram).
- Implement AM radio UI (tuning dial, signal/static/volume sliders,
  60 Hz hum / RF distortion checkboxes).
- Refine audio: pulse-train + jitter + hum + noise, via sample-level
  `AudioWorklet` / generated buffer in Rust.

**Commit message**: `feat: radio-music demo — full chain integration`

### Step 8 — Polish, built-in songs, help

- Bundle `jingle-bells.deck` and `silent-night.deck` as built-in songs.
- Add `File → Import/Export deck` (IBM 1130 format via `punch-card-core`).
- Final museum-style UI pass (IBM blue-gray palette, explanatory text).
- Verify all live demo URLs.

---

## 4. Testing after code moves

The core risk: `ibm-1130-rs` depends on crates that live in a different repo.
The guard:

| Action | When | Command |
|---|---|---|
| `cargo test` in `sw-ibm-1130-peripherals` | Before every push to main | `cargo test --workspace` |
| `cargo clippy --all-targets` in `sw-ibm-1130-peripherals` | Before every push | `cargo clippy --all-targets` |
| Rebuild `ibm-1130-rs` pages | After changing the git dep ref | `./build-all.sh` |
| Serve + smoke-test locally | After every pages rebuild | `./serve.sh` → `localhost:9352` |
| `cargo clippy --all-targets` in `ibm-1130-rs` | After pulling dep change | `cargo clippy --all-targets` |
| Rebuild radio-music pages | After every feature commit | `trunk build --release` |
| Visual verification | After every radio-music build | Open in browser; confirm tone + card animation |

**Git dep pinning**: use a git dependency (not branch HEAD) once the shared
crates stabilize. For active development, `branch = "main"` is acceptable
but pin to a specific commit hash before cutting any "release" demo.

**Local override for rapid iteration**: during development, use a
`[patch."https://..."]` override in `ibm-1130-rs/Cargo.toml` to point at
the local checkout:

```toml
[patch."https://github.com/sw-comp-history/sw-ibm-1130-peripherals"]
punch-card-core = { path = "../sw-ibm-1130-peripherals/crates/punch-card-core" }
machine-1130-core = { path = "../sw-ibm-1130-peripherals/crates/machine-1130-core" }
```

Remove the patch section before committing to main.

---

## 5. Radio music feature — design summary

### UI

Museum-exhibit style (not dashboard/grid). Three synchronized views:

1. **Card/deck view** — current card with column highlight, card count
2. **CPU/timing view** — NOP loop visualization, instruction count, loop period
3. **Radio view** — AM transistor radio drawing with tuning dial and audio controls

Two modes: *"As I remember it"* (experiential) and *"How it probably worked"*
(technical diagram).

### Audio model

Web Audio API from Rust via `web-sys`:

```
pulse train (square wave, Hz from timing loop)
  + timing jitter
  + 60 Hz hum
  + radio static (band-limited noise)
  + reader mechanical noise
      ↓ BiquadFilter (band-pass)
      ↓ GainNode (volume)
      ↓ destination
```

Initially: `OscillatorNode` (square) + noise layer — validate the pipeline.
Later: sample-level buffer generation via `AudioWorklet`.

### Testing

- `music/note.rs`, `music/deck.rs`, `music/timing.rs`: pure Rust, `#[test]`.
- `radio/synth.rs`: integration test via `wasm-bindgen-test` (create
  `AudioContext`, verify node connections, no audible output needed).
- Manual browser test: confirm tone plays, card animates, tuning dial affects pitch.
