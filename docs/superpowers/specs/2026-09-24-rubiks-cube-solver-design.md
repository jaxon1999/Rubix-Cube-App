# Rubik's Cube Solver — Design Spec

**Date:** 2026-09-24
**Status:** Approved in brainstorming; awaiting written-spec review

## 1. Purpose and success criteria

A browser app that takes two photos of a scrambled 3×3 Rubik's cube, reconstructs the full cube state, and produces a near-optimal solution the user can step through.

This is a **portfolio / learning project**: the computer-vision pipeline and the solver are the point, and both are written by us (not delegated to cube-specific libraries).

**Success criteria (v1):**
- On the author's own cube in normal indoor lighting, the vision pipeline needs 0–3 manual sticker corrections per scan.
- Solve completes in < 2 s and returns ≤ 22 moves (typically ≤ 20).
- Solver has a unit + property test suite; vision has a fixture-photo regression test.
- Deployed as a static site on GitHub Pages; README explains the algorithms.

**Out of scope for v1:** live camera scanning, cube sizes other than 3×3, human methods (CFOP/beginner), accounts/backend, native mobile apps, the 3D animated player (milestone M5, separate spec later).

## 2. Architecture

Everything runs client-side; no server.

```
Browser (static site)
├─ UI (React + TypeScript + Vite)
│    Capture → Align → Review (2D net) → Solution stepper
├─ vision/   TypeScript in a Web Worker, OpenCV.js for primitives
│    locate cube → split into 3 faces → warp → sample stickers → classify
├─ cube/     TypeScript: facelet model, validation, move application for the net
└─ solver    Rust → WASM (wasm-bindgen) in its own Web Worker
     solve(facelets, budget_ms) -> moves | error
```

**Interface contract:** components exchange the standard 54-character facelet string in Kociemba order `U1..U9 R1..R9 F1..F9 D1..D9 L1..L9 B1..B9`, each char one of `U R F D L B` (the face whose center has that color). Solver and vision are testable independently through this string.

**Division of labor:** OpenCV.js supplies low-level operations only (blur, Canny, contours, convex hull, `approxPolyDP`, `warpPerspective`, color conversion). All cube-specific logic (cube localization, face splitting, sampling, classification) is our code.

**Threads:** vision and solver each run in a Web Worker so the UI never blocks.

## 3. Vision pipeline

### 3.1 Capture protocol

Standard color scheme assumed (white opposite yellow, green opposite blue, red opposite orange).

- **Photo 1:** white on top, green facing the user, camera aimed at the U-F-R corner → shows U, F, R.
- **Photo 2:** cube turned so the opposite D-B-L corner faces the camera, following the guide image → shows D, B, L.

The protocol is fixed and illustrated in the UI, so each face's identity **and in-image rotation** are known; the pipeline maps each sampled grid to facelet indices with a fixed per-face lookup table.

### 3.2 Locate the cube (7 key points)

A corner view of a cube is a hexagon with a central Y-junction. Seven points define the geometry: the 6 hull vertices and the center vertex.

- **Auto:** downscale to ~800 px long edge → Gaussian blur → Canny → dilate → largest external contour → convex hull → `approxPolyDP` with increasing epsilon until 6 vertices. The center vertex is found by searching candidate points near the hull centroid and choosing the one minimizing total color variance of the three resulting warped faces.
- **Fallback:** if no 6-vertex hull is found, place a default hexagon centered in the image. In all cases the 7 points are shown as draggable handles and the user can adjust them.

### 3.3 Split and warp

Each face is the quadrilateral (center vertex, hull vertex, hull vertex, hull vertex) sharing the center. `warpPerspective` maps each to a 300×300 square.

### 3.4 Sample

Divide each square into a 3×3 grid. For each cell take the per-channel **median** of the central 40% patch (avoids black borders; median rejects glare). Convert to CIE Lab. Output: 27 Lab samples per photo, 54 total.

### 3.5 Classify

1. The 6 center samples are the reference colors (centers never move, so they are the ground truth for each face's color under this lighting).
2. Compute CIEDE2000 distance from every sticker to every center.
3. Solve a **balanced assignment**: each of the 54 stickers gets exactly one color, each color exactly 9 stickers, minimizing total distance (Hungarian algorithm on a 54×54 cost matrix: stickers × 6 colors × 9 slots). Centers are fixed to their own face.
4. Confidence per sticker = distance to 2nd-nearest center − distance to assigned center. Stickers below a threshold are flagged in the UI.

### 3.6 Validation (in `cube/`, used by the review step)

A state is solvable only if all hold; each failure produces a specific, human-readable reason and the offending pieces:
- 6 distinct centers; exactly 9 stickers of each color.
- Every corner and edge is a real piece (valid color combination, no duplicates).
- Sum of corner twists ≡ 0 (mod 3).
- Sum of edge flips ≡ 0 (mod 2).
- Corner permutation parity = edge permutation parity.

## 4. Solver (Rust, Kociemba two-phase)

### 4.1 Representation

Cubie level: corner permutation + orientation (8 corners, twist 0–2) and edge permutation + orientation (12 edges, flip 0–1). `facelet.rs` converts facelet string ↔ cubie cube. The 18 moves (U, U2, U′, R, … B′) are defined as cubie compositions.

### 4.2 Coordinates

| Phase | Coordinate | Size |
|---|---|---|
| 1 | Corner twist | 2,187 |
| 1 | Edge flip | 2,048 |
| 1 | UD-slice edge positions | 495 |
| 2 | Corner permutation | 40,320 |
| 2 | U/D-layer edge permutation | 40,320 |
| 2 | UD-slice edge permutation | 24 |

Phase 1 reaches the subgroup G1 = ⟨U, D, R2, L2, F2, B2⟩. Phase 2 solves within G1 using only those 10 moves.

### 4.3 Tables

- **Move tables:** coordinate × move → coordinate.
- **Pruning tables** (4 bits/entry, BFS from solved): phase 1 (twist × slice), (flip × slice); phase 2 (corner perm × slice perm), (edge perm × slice perm). Heuristic = max over applicable tables.
- Generated by a `build-tables` binary at build time → `tables.bin` (target a few MB, served compressed). The WASM module fetches and loads it at startup.

### 4.4 Search

- IDA* on phase 1 by increasing depth; for each phase-1 solution, run phase-2 IDA* bounded by `best_total − phase1_len − 1`.
- Move pruning: no two consecutive turns of the same face; for opposite faces only one order (e.g. U before D).
- **Anytime:** keep exploring longer phase-1 solutions; return the best total when the budget expires (default 1500 ms) or immediately when total ≤ target (default 20). Deadline checked periodically, compatible with a WASM worker (`performance.now()` via bindings).

### 4.5 API

```rust
pub fn solve(facelets: &str, budget_ms: u32) -> Result<Vec<Move>, SolveError>;
pub enum SolveError { BadInput(String), IllegalState(String), NoSolution }
```

WASM binding exposes `init(tables: &[u8])` and `solve(facelets, budget_ms) -> String` (space-separated moves, or a JSON error).

## 5. UI

One page, four steps, React.

1. **Capture:** two upload slots with guide images (drag-drop or file picker; the mobile picker offers the camera).
2. **Align:** each photo with 7 draggable handles pre-placed by auto-detection; live preview of the 3 warped faces; "Looks good" to continue.
3. **Review:** 2D net of 54 stickers; low-confidence stickers pulse; click cycles color or opens a 6-color palette. Validation runs live; Solve disabled while illegal, with the specific reason and affected pieces highlighted.
4. **Solution:** move chips (`R U R' F2 …`) with count; prev/next buttons and arrow keys step through; the net updates per move; the current move is highlighted with a plain-English hint.

## 6. Error handling

| Situation | Behavior |
|---|---|
| Image fails to load / unsupported type | Inline error on that slot |
| Auto-detection finds no hexagon | Default hexagon handles + "Couldn't find the cube — drag the points to its corners" |
| Two centers classify to similar colors | Warning on net; user fixes before continuing |
| Illegal state | Specific reason, pieces highlighted, Solve disabled |
| WASM / tables fail to load | Full-width error with retry; vision steps still work |
| Solver returns no solution in budget | Should not occur; return first solution found and log |

## 7. Project layout

```
/solver          Rust crate: cubie, facelet, coords, tables, search, wasm bindings, bin/build-tables
/web             Vite + React + TypeScript
  /src/vision    pipeline (worker) + fixture-photo tests
  /src/cube      facelet model, validation, move application
  /src/ui        components
/docs            specs, plans, algorithm write-ups
```

CI (GitHub Actions): `cargo test`, build tables, `wasm-pack build`, Vitest, build web app, deploy to GitHub Pages.

## 8. Testing

**Solver (`cargo test`):**
- Each face move applied 4× is the identity.
- Facelet → cubie → facelet round-trips.
- Coordinate get/set round-trips for every coordinate.
- Pruning values are admissible (never exceed true distance) on sampled states, checked by brute force at low depth.
- Property test: 1,000 random scrambles → solve → apply → solved, length ≤ 22.
- Superflip solves in ≤ 20 moves.

**Web (Vitest):**
- Validation: each illegal-state class is detected with the right reason.
- Move application on the facelet model matches the Rust solver's definition (shared fixture of scramble → expected facelet string).
- Classifier: synthetic Lab inputs with noise → correct balanced assignment.
- Vision regression: `web/src/vision/fixtures/` holds real photo pairs with hand-labeled facelet strings; test reports per-sticker accuracy and fails if it drops below the recorded baseline.

## 9. Milestones

1. **M1 — Solver core:** Rust crate, tables, search, full test suite, CLI `solve <facelets>`.
2. **M2 — WASM + minimal app:** manual facelet editor → solver → solution stepper. Usable without vision.
3. **M3 — Vision:** capture, align, review; classification; fixture regression tests.
4. **M4 — Polish + deploy:** guide images, error handling, README with algorithm write-ups, GitHub Pages.
5. **M5 — 3D player (later, separate spec):** Three.js animated cube with play/pause/step/speed, starting from the scanned state.

This spec covers M1–M4.
