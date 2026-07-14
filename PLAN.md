# PLAN.md — Floorplan Signal Project

_Last updated: 2026-07-14_

---

## 1. Overview & Interpretation

### Chosen Interpretation

The "Floorplan Signal" project is a **point-estimate Wi-Fi signal-strength tool**. A user uploads a 2D floor-plan image, nominates a wireless booster location by clicking on the image, and then clicks a second location on the same image to receive an estimated Received Signal Strength Indicator (RSSI) in dBm between those two points. The application also classifies the result as Excellent, Good, Fair, Weak, or Unusable.

The application does **not** generate a full signal heatmap or simulate propagation across the entire floor plan. Signal is estimated for a single selected point-pair at a time using the log-distance path-loss model. This keeps the MVP scope achievable while delivering genuine, end-to-end value.

### Value Delivered

- Gives a quick, approximate answer to "will I get usable Wi-Fi here?" based on distance alone.
- Provides a visual, interactive interface that requires no command-line knowledge.
- Clearly separates the physics calculation from the UI so the model can be tested and improved independently.

### Competing Interpretation (Not Chosen)

A full heatmap interpretation — where signal strength is computed and color-coded for every pixel or grid cell of the floor plan — was considered. This was rejected for the MVP because it requires a rendering pipeline (matplotlib or similar overlay), is significantly more complex, and is not required by the project goals listed in the architect agent definition. It remains a natural extension for a future version.

---

## 2. MVP Scope

### In Scope

- Upload a floor-plan image (PNG, JPG, or JPEG).
- Enter a Wi-Fi network name (SSID) — used as a label only; it has no effect on signal calculations.
- Click the floor-plan image to place a single wireless booster (access point).
- Click the floor-plan image to select a measurement point.
- Compute estimated RSSI in dBm using the log-distance path-loss model.
- Display the RSSI value and a signal-quality classification (Excellent / Good / Fair / Weak / Unusable).
- Allow the user to configure: pixels-per-metre scale, reference distance RSSI (RSSI_0), path-loss exponent (n), and an optional flat wall-obstruction loss (dB).
- Validate all user inputs and display clear error messages for invalid values.
- Automated unit tests for the signal model and input validation.
- A note in the UI stating that the SSID identifies the network but does not affect signal strength.

### Out of Scope (Non-Goals)

- Signal heatmap or coverage-area visualization.
- Multi-access-point placement or best-AP selection.
- Real-world RF obstacle modelling (wall material, frequency-specific attenuation).
- Saving or exporting results.
- Authentication or user accounts.
- Deployment or containerization.
- Support for floor plans with multiple floors.
- Any live or real-device signal measurement.

---

## 3. Architecture

### Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Language | Python 3.11+ | Required by project goals; widely available on Windows |
| UI framework | Streamlit | Required by project goals; zero-boilerplate interactive web UI in pure Python |
| Image handling | Pillow (PIL) | Standard Python image library; Streamlit integrates with it directly |
| Click detection | `streamlit-image-coordinates` | Thin library that returns pixel coordinates from a user click on an image |
| Testing | pytest | Standard, minimal, works on Windows with `python -m pytest` |
| Linting (optional) | None enforced at MVP stage | Keep tooling minimal |

### Major Components

```
UI Layer (app.py / Streamlit)
    |
    |-- calls -->  signal_model.py   (pure-Python, no Streamlit imports)
    |-- calls -->  validators.py     (pure-Python, no Streamlit imports)
```

**`signal_model.py`** — contains all physics.
- `compute_distance_metres(pixel_a, pixel_b, pixels_per_metre)` — Euclidean pixel distance converted to metres.
- `log_distance_path_loss(distance_m, rssi_0, n, wall_loss_db)` — returns RSSI in dBm.
- `classify_signal(rssi_dbm)` — returns one of five quality labels.

**`validators.py`** — validates and sanitises user inputs.
- `validate_ssid(ssid)` — non-empty string check.
- `validate_scale(pixels_per_metre)` — must be a positive float.
- `validate_model_params(rssi_0, n, wall_loss_db)` — range checks.
- `validate_coordinates(point, image_size)` — coordinates within image bounds.

**`app.py`** — Streamlit entry point.
- Renders sidebar controls (SSID input, scale, model parameters).
- Renders image with click handlers (booster placement, measurement point).
- Calls validators then `signal_model` functions and displays results.
- Displays the SSID disclaimer note.

**`tests/`** — pytest test modules, one per source module.

### Signal Model

The **log-distance path-loss model** computes estimated received signal strength:

```
RSSI(d) = RSSI_0 - 10 * n * log10(d / d_0) - W
```

Where:
- `RSSI_0` — reference RSSI at reference distance `d_0` (default: -30 dBm at 1 m). Assumption: typical indoor Wi-Fi AP transmit power at 1 m.
- `n` — path-loss exponent (default: 3.0). Assumption: n=3 is a reasonable default for a typical furnished indoor environment. Free space is 2; heavy obstruction can reach 5.
- `d_0` — reference distance, fixed at **1 metre** to keep the formula simple.
- `d` — computed distance between booster and measurement point in metres.
- `W` — flat wall-obstruction loss in dB (default: 0). User-configurable. Assumption: a single flat penalty is sufficient for MVP; per-wall attenuation is out of scope.

If `d < d_0`, the model clamps the result to `RSSI_0` (prevents unphysical RSSI above the reference value).

**Signal quality classification thresholds** (industry-standard Wi-Fi RSSI ranges):

| RSSI (dBm) | Classification |
|---|---|
| >= -50 | Excellent |
| -50 to -60 | Good |
| -60 to -70 | Fair |
| -70 to -80 | Weak |
| < -80 | Unusable |

### Data Model

No persistent data store is used. All state lives in `st.session_state` during a Streamlit session.

| Key | Type | Description |
|---|---|---|
| `uploaded_image` | PIL Image | The decoded floor-plan image |
| `ssid` | str | Network name entered by user |
| `booster_point` | (int, int) or None | Pixel coordinates of placed booster |
| `measure_point` | (int, int) or None | Pixel coordinates of measurement location |
| `pixels_per_metre` | float | Scale factor: how many image pixels equal 1 metre |
| `rssi_0` | float | Reference RSSI in dBm at 1 m |
| `n` | float | Path-loss exponent |
| `wall_loss_db` | float | Flat obstruction loss in dB |

---

## 4. File Structure

```
floorplan-signal/
├── app.py                  # Streamlit entry point; all UI code lives here
├── signal_model.py         # Pure-Python signal physics; no Streamlit imports
├── validators.py           # Pure-Python input validation; no Streamlit imports
├── requirements.txt        # Python dependencies (streamlit, pillow, streamlit-image-coordinates)
├── tests/
│   ├── __init__.py         # Makes tests/ a package
│   ├── test_signal_model.py  # Unit tests for all signal_model functions
│   └── test_validators.py    # Unit tests for all validators functions
├── PLAN.md                 # This file; project plan maintained by architect
├── REVIEW.md               # Created/updated by reviewer agent after each stage
├── README.md               # Project title and brief usage notes (updated in Stage 1)
└── .claude/
    └── agents/
        ├── architect.md
        ├── builder.md
        └── reviewer.md
```

No `src/` subdirectory is used. The flat layout keeps imports simple for a small MVP. Assumption: the project will not grow large enough to require a package namespace.

---

## 5. Implementation Stages

### Stage 1 — Project Scaffold

Establish the repository layout, dependency manifest, and test harness. No application logic yet.

**Deliverables:**
- `requirements.txt` listing all dependencies with pinned minor versions.
- `tests/__init__.py` (empty).
- `tests/test_signal_model.py` with one placeholder test (`assert True`).
- `tests/test_validators.py` with one placeholder test (`assert True`).
- `README.md` updated with a one-paragraph description and the command to run the app and tests.

---

### Stage 2 — Signal Model

Implement `signal_model.py` and its full test suite. No UI involved.

**Deliverables:**
- `signal_model.py` with `compute_distance_metres`, `log_distance_path_loss`, and `classify_signal`.
- `tests/test_signal_model.py` fully populated (replaces placeholder).

---

### Stage 3 — Input Validation

Implement `validators.py` and its full test suite. No UI involved.

**Deliverables:**
- `validators.py` with `validate_ssid`, `validate_scale`, `validate_model_params`, and `validate_coordinates`.
- `tests/test_validators.py` fully populated (replaces placeholder).

---

### Stage 4 — Streamlit UI Shell

Implement `app.py` as a runnable Streamlit application with all UI controls wired up but using stub/placeholder values for signal results (i.e., the signal model is not yet called from the UI). The image upload, SSID field, sidebar parameters, and click handling must all function.

**Deliverables:**
- `app.py` with: image uploader, SSID text input with disclaimer note, sidebar sliders/inputs for scale and model parameters, image display with `streamlit-image-coordinates` click capture for both booster and measurement points, session-state management for all stored values, and a placeholder result area that shows the stored coordinates.

---

### Stage 5 — Signal Calculation Integration

Connect the signal model to the UI. When both points are placed, compute and display the real RSSI and classification.

**Deliverables:**
- `app.py` updated to call `validators` and `signal_model` functions.
- Display RSSI in dBm, signal classification, and the pixel distance / metre distance.
- Display user-facing error messages for invalid inputs (via `st.error`).
- The SSID disclaimer note must be visible.

---

### Stage 6 — Polish and Edge Cases

Harden the application against edge cases identified in review: zero distance, very large distances, missing image, missing points, invalid file type. Update tests to cover these cases.

**Deliverables:**
- `signal_model.py` updated with distance clamping (d < d_0 clamps to RSSI_0).
- `validators.py` updated with image-type validation.
- `app.py` updated to guard all missing-state conditions gracefully.
- Additional test cases in both test files.

---

## 6. Acceptance Criteria per Stage

### Stage 1 — Project Scaffold

- [ ] `requirements.txt` exists and lists `streamlit`, `pillow`, and `streamlit-image-coordinates` with pinned minor versions (e.g., `streamlit>=1.35,<2`).
- [ ] `tests/__init__.py` exists (may be empty).
- [ ] `tests/test_signal_model.py` exists and contains at least one test function.
- [ ] `tests/test_validators.py` exists and contains at least one test function.
- [ ] Running `python -m pytest tests/` from the repo root exits with code 0 and reports 0 failures.
- [ ] `README.md` contains a usage paragraph and the exact commands `streamlit run app.py` and `python -m pytest tests/`.
- [ ] No application source files (`app.py`, `signal_model.py`, `validators.py`) exist yet.

### Stage 2 — Signal Model

- [ ] `signal_model.py` exists and imports without error.
- [ ] `compute_distance_metres((0, 0), (300, 400), 100.0)` returns `5.0` (3-4-5 triangle, 500 pixels / 100 px/m).
- [ ] `log_distance_path_loss(1.0, -30.0, 3.0, 0.0)` returns `-30.0` (at reference distance, no loss beyond reference).
- [ ] `log_distance_path_loss(10.0, -30.0, 3.0, 0.0)` returns approximately `-60.0` (10^1 decade * 30 dB).
- [ ] `log_distance_path_loss` with `wall_loss_db=5.0` returns a value 5 dB lower than the same call with `wall_loss_db=0.0`.
- [ ] `classify_signal(-45.0)` returns `"Excellent"`.
- [ ] `classify_signal(-55.0)` returns `"Good"`.
- [ ] `classify_signal(-65.0)` returns `"Fair"`.
- [ ] `classify_signal(-75.0)` returns `"Weak"`.
- [ ] `classify_signal(-85.0)` returns `"Unusable"`.
- [ ] `python -m pytest tests/test_signal_model.py` exits with code 0 and 0 failures.
- [ ] `signal_model.py` contains no Streamlit imports.

### Stage 3 — Input Validation

- [ ] `validators.py` exists and imports without error.
- [ ] `validate_ssid("")` raises `ValueError` or returns a falsy/error result.
- [ ] `validate_ssid("MyNetwork")` returns without error.
- [ ] `validate_scale(0.0)` raises `ValueError` (zero is invalid).
- [ ] `validate_scale(-5.0)` raises `ValueError` (negative is invalid).
- [ ] `validate_scale(100.0)` returns without error.
- [ ] `validate_model_params(-30.0, 3.0, 0.0)` returns without error.
- [ ] `validate_model_params(-30.0, 0.0, 0.0)` raises `ValueError` (n=0 is invalid).
- [ ] `validate_model_params(-30.0, 3.0, -1.0)` raises `ValueError` (negative wall loss is invalid).
- [ ] `validate_coordinates((100, 200), (800, 600))` returns without error.
- [ ] `validate_coordinates((-1, 200), (800, 600))` raises `ValueError`.
- [ ] `validate_coordinates((100, 700), (800, 600))` raises `ValueError` (y exceeds image height).
- [ ] `python -m pytest tests/test_validators.py` exits with code 0 and 0 failures.
- [ ] `validators.py` contains no Streamlit imports.

### Stage 4 — Streamlit UI Shell

- [ ] `app.py` exists and `streamlit run app.py` starts without a Python import error.
- [ ] A file-upload widget accepting PNG/JPG/JPEG is present.
- [ ] When an image is uploaded it is displayed in the main area.
- [ ] An SSID text-input field is present.
- [ ] A note stating that the SSID identifies the network but does not affect signal strength is visible on the page.
- [ ] Sidebar controls for pixels-per-metre, RSSI_0, path-loss exponent (n), and wall-obstruction loss are present.
- [ ] Clicking the image (using `streamlit-image-coordinates`) stores and displays the booster coordinates in `st.session_state`.
- [ ] A second click interaction allows setting the measurement-point coordinates.
- [ ] The stored coordinates are shown somewhere on the page (even as raw text is acceptable at this stage).
- [ ] No signal calculation is performed yet; the result area shows a placeholder message.

### Stage 5 — Signal Calculation Integration

- [ ] When both the booster point and measurement point are set and the image is uploaded, RSSI is computed and displayed in dBm (e.g., "-62.4 dBm").
- [ ] The signal classification (Excellent / Good / Fair / Weak / Unusable) is displayed alongside the RSSI.
- [ ] The metre distance between the two points is displayed.
- [ ] Changing a sidebar parameter (n, RSSI_0, wall loss, scale) updates the displayed result.
- [ ] If the SSID field is empty, `st.error` is shown and no calculation is performed.
- [ ] The SSID disclaimer note remains visible.
- [ ] `python -m pytest tests/` exits with code 0 and 0 failures (no regression).

### Stage 6 — Polish and Edge Cases

- [ ] If the booster and measurement point are the same pixel, the application does not crash; it displays RSSI_0 (clamped result) or a clear message.
- [ ] If the uploaded file is not a valid image, `st.error` is shown instead of a crash.
- [ ] If the user has not placed both points, the UI shows a clear instruction message rather than an error traceback.
- [ ] `classify_signal` boundary values: -50.0 → "Excellent", -60.0 → "Good", -70.0 → "Fair", -80.0 → "Weak" (exact boundary falls in the better category or is explicitly documented).
- [ ] `compute_distance_metres` with identical points returns `0.0` without raising an exception.
- [ ] `log_distance_path_loss` with `distance_m <= 0` clamps to `RSSI_0` without raising an exception.
- [ ] `python -m pytest tests/` exits with code 0 and 0 failures.

---

## 7. Testing Requirements

### Framework

**pytest** — run with:

```powershell
python -m pytest tests/
```

For verbose output:

```powershell
python -m pytest tests/ -v
```

### What Must Be Unit-Tested

| Module | Functions | Key cases |
|---|---|---|
| `signal_model.py` | `compute_distance_metres` | Known 3-4-5 triangle, zero distance, scale variations |
| `signal_model.py` | `log_distance_path_loss` | At reference distance, 1 decade, wall loss additive, clamping at d<=0 |
| `signal_model.py` | `classify_signal` | All five classifications, boundary values |
| `validators.py` | `validate_ssid` | Empty string, whitespace-only, valid string |
| `validators.py` | `validate_scale` | Zero, negative, positive float |
| `validators.py` | `validate_model_params` | n=0, negative n, negative wall loss, valid combination |
| `validators.py` | `validate_coordinates` | In-bounds, x<0, y<0, x>=width, y>=height |

### Coverage Expectations

- All functions in `signal_model.py` and `validators.py` must have at least one test covering the happy path and at least one test covering each documented error condition.
- There is no enforced percentage target for the MVP, but the reviewer will flag any untested public function as a defect.
- No tests are required for `app.py` (Streamlit UI testing is out of scope for MVP).

### Integration / E2E

No automated integration or browser tests are required for the MVP. The reviewer agent will perform a manual smoke test of `streamlit run app.py` and document the result in REVIEW.md.

---

## 8. Risks & Assumptions

### Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| `streamlit-image-coordinates` does not return reliable pixel coordinates for all image sizes | Medium | High | Test manually during Stage 4 review; fall back to `st.image` + manual coordinate text inputs if needed |
| Streamlit session-state resets on widget interaction cause coordinate loss | Medium | Medium | Use `st.session_state` keys explicitly for all stored values; document in code |
| Windows path separators cause issues with Pillow image loading | Low | Low | Use `pathlib.Path` or file-like objects from `st.file_uploader` instead of string paths |
| Very large floor-plan images slow down the Streamlit render loop | Low | Low | Document a recommended max image size (e.g., 2000x2000 px) in README; do not add resizing logic at MVP |
| pytest is not in the user's PATH | Low | Low | All test commands use `python -m pytest` (module invocation) which bypasses PATH issues |

### Assumptions

| # | Assumption | Rationale |
|---|---|---|
| A1 | Reference distance `d_0` is fixed at 1 metre. | Simplifies the formula; -30 dBm at 1 m is a reasonable indoor Wi-Fi reference. |
| A2 | Default RSSI_0 is -30 dBm. | Represents a typical 2.4 GHz or 5 GHz AP at 1 m in a residential environment. |
| A3 | Default path-loss exponent n is 3.0. | A commonly cited value for typical furnished indoor environments. |
| A4 | Wall-obstruction loss is a single flat penalty (0 dB default). | MVP does not model individual walls drawn on the floor plan. |
| A5 | The SSID is a display/label field only and has no effect on calculations. | Confirmed by the project goals: "SSID identifies the network but does not provide signal strength." |
| A6 | Pixel-to-metre scale is user-supplied, not inferred from the image. | The image contains no embedded scale metadata. |
| A7 | Only one booster location is supported at MVP. | Multiple APs add significant complexity (best-AP selection, interference) which is out of scope. |
| A8 | The flat-layout (no `src/` package) is appropriate for MVP. | The project has three Python source files; namespace packaging is unnecessary overhead. |
| A9 | Boundary RSSI values (exactly -50, -60, -70, -80 dBm) fall into the better classification (e.g., -60 dBm is "Good", not "Fair"). | A defensible convention; must be documented in `signal_model.py` and verified in tests. |
| A10 | Python 3.11 or later is installed on the development machine. | Streamlit 1.35+ and modern Pillow require Python 3.8+ minimum; 3.11 is the current stable release on Windows. |

### Interpretation Assumption

The chosen interpretation (point-estimate tool) is simpler and more directly satisfies the stated project goals than a heatmap interpretation. The heatmap variant is recorded under Section 1 as a competing interpretation and remains a natural v2 extension.
