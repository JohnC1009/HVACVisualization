# Claude Code — Session 3 of 4: Dash UI
## Hydronic Loop Visualizer — Phase 2

### What to expect from the previous session

Sessions 1 and 2 produced:
- `hydronic/valve.py`, `coil.py`, `controller.py` — all physics classes
- `hydronic/loop.py` — `ClosedLoop.step()` with Mode A/B, UA/LMTD coil, 3-solve
  Mode B sequence, no global mutable state
- `test_phase2.py` — all unit tests including `test_mode_b_integration`, all green
- `SimulationStepResult` schema defined, Group 5 PHE keys present as `None`

Verify before starting:
```
pytest test_solver.py test_phase2.py -v
```
If anything is red, stop and fix it before touching the UI.

---

### Your job this session

Build the Phase 2 Dash UI on top of the existing solver. Three deliverables:
mode selector, new sliders and status panel, two-row time-series chart. Then tune
the PI gains using the load-drop scenario.

No physics changes. If you find yourself editing `valve.py`, `coil.py`, or
`controller.py`, stop — that belongs in a different session.

---

### Spec references

| What | Spec sections |
|------|--------------|
| Mode selector | 5.1 |
| New sliders (IDs, ranges, tooltips) | 5.2 |
| Controller status panel | 5.3 |
| Time-series chart (two-row layout) | 5.4 |
| Non-convergence UI behaviour | 5.4 (warning box) |
| Tuning starting points | 4.4 |
| Load-drop scenario (manual acceptance test) | 8 |
| UI acceptance criteria | 9, criteria 4–5, 8–9 |
| Figure references | Fig 1 (layout), Fig 2 (status panel), Fig 3 (chart), Fig 4 (sliders) |

The figures in the spec document are high-fidelity mockups — use them as the visual
target for layout, colour, and labelling.

---

### Constraints that matter this session

**Slider IDs are fixed.** Use exactly these IDs — Session 4 may add more callbacks
that reference them:

| Slider | Dash ID | Range |
|--------|---------|-------|
| ΔP Setpoint | `dp-setpoint` | 5–60 ft |
| ΔT Setpoint | `dt-setpoint` | 5–40 °F |
| Air Entering Temp | `t-air-enter` | 40–90 °F |
| Air Flow (cfm) | `airflow-cfm` | 5,000–25,000 |

**ΔT tooltip is required.** Add a sub-label or tooltip to the ΔT Setpoint slider:
*"Water-side ΔT (T_supply − T_return). Higher = more heat extracted."*
This is the single most likely point of user confusion (Section 5.2 warning).

**Status panel labels must use A/F terminology.** The fields are `Sizing Authority (A)`
and `Operating Fraction (F)` — not "authority N" or any other label. Green indicator
for A in 0.25–0.5; red otherwise (Section 5.3).

**Non-convergence behaviour is specified.** If `Coil_converged = False` in the result:
show a red banner above the schematic, display last-good temperatures in grey/italic,
keep the simulation running. Do not freeze or crash (Section 5.4 warning).

**cfm slider is a scenario parameter, not real-time control.** Add a note near the
slider: *"Scenario parameter — fixed for this simulation run."* (Section 5.2 note)

**State stays in `dcc.Store`.** Do not introduce any new module-level mutable state.
The rolling 60-point chart history lives in `dcc.Store` alongside the simulation state.

---

### Steps

**Step 8 — Mode selector, sliders, status panel**
Read Sections 5.1–5.3. Add the radio button group for Mode A/B. Wire the mode to
`ClosedLoop.step(mode=...)`. Show/hide correct sliders per mode. Build the status
panel with all fields from Figure 2, including the source-key column.

Manual gate: toggle Mode A/B — correct sliders appear; status panel updates. ✔

**Step 9 — Time-series chart**
Read Section 5.4. Add a second `dcc.Graph` with two subplots (see Figure 3 in spec).

Row 1: `Q_gpm` (blue), `Pump_speed` (green), `Valve_position` (orange dashed)
Row 2: `Coil_dT_F` + flat ΔT setpoint line (amber), `Loop_dp_ft` + flat ΔP setpoint
line (blue)

Rolling 60-point window stored in `dcc.Store`. Each callback appends one point and
trims to 60.

Manual gate: chart updates in real time; window rolls correctly after ~30 seconds. ✔

**Step 10 — Tune PI gains**
Read Section 4.4. Run the load-drop scenario from Section 8 in the UI: move the
Air Entering Temp slider from 55°F to 70°F and watch the time-series chart.

Target: both controllers settle within 20 timesteps (10 seconds) with no sustained
oscillation.

Document the final Kp/Ki values in the PIController construction calls as comments:
```python
# Pump PI — holds ΔP setpoint
# Kp=0.05, Ki=0.004, dt=0.5 — tuned for Phase 1 default system
# (H_shut≈60 ft, Q_design=100 gpm). Scale Kp inversely if H_shut changes.
```

Manual gate: load-drop settles within 20 steps, no oscillation. ✔

---

### Quick reference — numbers for this session

| Parameter | Value | Spec |
|-----------|-------|------|
| Chart history | 60 points = 30 s | §5.4 |
| Dash interval | 500 ms | §4.4 |
| Pump PI: Kp, Ki | 0.05, 0.004 | §4.4 |
| Valve PI: Kp, Ki | 0.05, 0.002 | §4.4 |
| Good sizing authority A | 0.25–0.50 | §2.5 |
| Load-drop settle target | ≤ 20 steps | §8, §9 criterion 4 |

---

### Hand-off checklist

Before ending this session, confirm:
- [ ] `pytest test_solver.py test_phase2.py` — all green (no regressions from UI work)
- [ ] Mode A/B toggle works, correct sliders visible per mode
- [ ] ΔT slider has tooltip/sub-label
- [ ] Status panel shows A/F labels with green/red threshold indicator
- [ ] Time-series chart shows two rows, 60-point rolling window
- [ ] Non-convergence red banner implemented
- [ ] Load-drop scenario settles within 20 steps
- [ ] No new module-level mutable state introduced
