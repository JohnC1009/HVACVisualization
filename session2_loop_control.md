# Claude Code — Session 2 of 4: Loop Control & State
## Hydronic Loop Visualizer — Phase 2

### What to expect from the previous session

Session 1 produced:
- `hydronic/valve.py` — `CvValve` (and deprecated `Valve` stub)
- `hydronic/coil.py` — `Coil` with UA/LMTD model + `calibrate_coil_ua()`
- `hydronic/controller.py` — `PIController` (velocity form, conditional anti-windup)
- `test_phase2.py` — unit tests for all three classes, all green

Verify before starting:
```
pytest test_solver.py test_phase2.py -v
```
If anything is red, stop and fix it before proceeding.

---

### Your job this session

Wire the new physics into `ClosedLoop.step()`, implement Mode B (PI-controlled pump
and valve), add bumpless transfer, and move all mutable state into `dcc.Store`.
Still no UI changes. Everything is testable with `pytest`.

---

### Spec references

| What | Spec sections |
|------|--------------|
| What changes in the solver step | 3.8 |
| Mode B step sequence (3-solve) | 4.6 |
| Startup and bumpless transfer | 4.5 |
| Setpoint-change derivative kick | 4.3 |
| SimulationStepResult schema (Groups 1–4) | 4.7 |
| Mode B integration test | 6.4 |

---

### Constraints that matter this session

**No global mutable state.** There must be no module-level `ClosedLoop` instance,
controller instance, or any other mutable object that persists between Dash callbacks.
All state lives in `dcc.Store`. The Dash callback reconstructs objects from store data,
runs one step, and writes results back. Grep for module-level assignments before
committing.

**Bumpless transfer uses `reset()` only.** On Mode A → B transition, call:
```python
pump_ctrl.reset(current_pump_speed)
valve_ctrl.reset(current_valve_position)
```
Never write directly to `_u_prev`. This is the only safe external state interaction
with `PIController` (Section 4.5).

**Schema contract.** Every call to `ClosedLoop.step()` must return a dict containing
ALL keys from the SimulationStepResult schema (Section 4.7, Groups 1–4). Keys for
PHE equipment (Group 5) are not needed yet — but when you define the TypedDict or
schema dict, add them as `None` placeholders so Session 4 can fill them in cleanly.

**Three-solve sequence.** Mode B runs three hydraulic solves per timestep, not one:
1. Solve with current actuators → Q₁ (feed to coil)
2. Pump PI update → new pump speed → re-solve → Q₂
3. Valve PI update → new valve position → re-solve → Q₃
Use Q₃ values in the output dict. See Section 4.6 for the exact sequence.

**`dt_seconds = 0.5`** throughout. If you see any PIController construction without an
explicit `dt_seconds`, add it.

---

### Steps

**Step 4 — Wire Coil into ClosedLoop**
Read Section 3.8. Update `ClosedLoop.step()` to call `coil.compute_load()` instead of
the old `500×Q` formula. `T_supply` remains a fixed input (PHE comes in Session 4).
Add Group 2 coil keys to the return dict.

Re-calibrate any `test_solver.py` expected values that changed due to the new coil
model. Update them with a comment:
`# updated Step 4: UA/LMTD model replaces 500*Q formula`

```
pytest test_solver.py
```
All green (with updated expected values). ✔

**Step 5 — (carry-over gate)**
Already done in Session 1 — just confirm:
```
pytest test_phase2.py -k "pi" -v
```
All green. ✔

**Step 6 — Mode B step sequence**
Read Sections 4.5 and 4.6. Update `ClosedLoop.step()` to accept `mode="A"` or
`mode="B"`. Implement the three-solve sequence. Add bumpless transfer on first Mode B
call.

Write the Section 6.4 integration test before running it:
```python
def test_mode_b_integration():
    # 80 steps, then assert:
    # abs(Loop_dp_ft - dp_setpoint) < 2.0
    # abs(Coil_dT_F  - dt_setpoint) < 2.0
    # energy balance within 1%
```
```
pytest test_phase2.py::test_mode_b_integration -v
```
Green. ✔

**Step 7 — Eliminate global mutable state**
Audit all modules. Move any persistent mutable state into `dcc.Store`. The Dash
callback pattern (even if the UI isn't updated yet) should be:

```python
@app.callback(Output('store', 'data'), Input('interval', 'n_intervals'),
              State('store', 'data'))
def simulation_step(n, store_data):
    # reconstruct from store_data → run step → serialise back
    return updated_store_data
```

```
grep -rn "^[a-zA-Z].*= \(ClosedLoop\|PIController\|CvValve\|Coil\)" hydronic/ app.py
```
Result should be empty (no module-level instances).

```
pytest test_solver.py test_phase2.py -v
```
All still green. ✔

---

### Quick reference — numbers for this session

| Parameter | Value | Spec |
|-----------|-------|------|
| `dt_seconds` | 0.5 | §4.4 |
| Pump PI: Kp, Ki | 0.05, 0.004 | §4.4 |
| Valve PI: Kp, Ki | 0.05, 0.002 | §4.4 |
| ΔP setpoint default | 25 ft | §4.4 |
| ΔT setpoint default | 20 °F | §4.4 |
| Mode B solves per step | 3 | §4.6 |

---

### Hand-off checklist

Before ending this session, confirm:
- [ ] `pytest test_solver.py test_phase2.py` — all green
- [ ] `test_mode_b_integration` passes
- [ ] No module-level mutable objects (grep confirms)
- [ ] SimulationStepResult schema defined with Group 5 PHE keys as `None` placeholders
- [ ] `ClosedLoop.step()` accepts `mode="A"` and `mode="B"`
