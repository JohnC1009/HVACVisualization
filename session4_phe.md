# Claude Code — Session 4 of 4: PHEXchanger
## Hydronic Loop Visualizer — Phase 2

### What to expect from the previous sessions

Sessions 1–3 produced a fully working Phase 2 application:
- All physics classes implemented and tested (`CvValve`, `Coil`, `PIController`)
- `ClosedLoop.step()` with Mode A/B, UA/LMTD coil, 3-solve Mode B, no global state
- Full Dash UI: mode selector, sliders, status panel, two-row time-series chart
- `SimulationStepResult` schema with Group 5 PHE keys already present as `None`
- `pytest test_solver.py test_phase2.py` — all green

Verify before starting:
```
pytest test_solver.py test_phase2.py -v
```
If anything is red, stop and fix it before writing PHE code.

---

### Your job this session

Implement `PHEXchanger` as a standalone class, write and pass all 6 test groups, then
wire it into `ClosedLoop` with dual-circuit PI control. This session has the highest
complexity per step — take it as three sub-tasks.

---

### Spec references

| What | Spec sections |
|------|--------------|
| Why NTU-ε vs LMTD | 3.9.1 |
| Class attributes | 3.9.2 |
| UA_design calibration + worked example | 3.9.3 |
| UA scaling (both sides) | 3.9.4 |
| `compute_transfer()` source | 3.9.5 (copy this exactly) |
| Sign convention table | 3.9.6 |
| ClosedLoop call sequence | 3.9.7 |
| Extended 9-step Mode B sequence | 3.9.8 |
| SimulationStepResult Group 5 keys | 4.7 Group 5 |
| PHE unit tests (6 groups) | 6.5 |
| Implementation steps 11–12 | 7 |
| Acceptance criteria 13–14 | 9 |
| Glossary (NTU, ε, capacity rate) | 10 |

---

### Constraints that matter this session

**`compute_transfer()` from Section 3.9.5 — copy it exactly.** The formula is
algebraically exact (no iteration). Do not simplify or restructure it. The balanced-
flow special case (`abs(c_r - 1.0) < 1e-6`) must be handled separately from the
general case.

**`calibrate_phe_ua()` must guard against degenerate inputs.** Add this assertion:
```python
assert 0.01 < eps < 0.99, (
    f"PHEXchanger design data implies degenerate ε={eps:.3f} — "
    "check input temperatures (equal inlets or perfect transfer?)"
)
```

**`T_sec_return` comes from the previous timestep.** This is the most likely
integration bug. In `ClosedLoop.step()`, the secondary return temperature passed to
`phe.compute_transfer()` must be `t_sec_out_f` from the previous step's result,
stored in `dcc.Store`. Using the current step's output as its own input creates an
algebraic loop (Section 3.9.7 warning).

**Cold-start value for `T_sec_return`:**
```python
T_sec_return_init = T_primary_supply - DT_setpoint_secondary
```
Store this in `dcc.Store` alongside other initial state.

**Do not add `authority` to `PHEXchanger`.** Same rule as `CvValve` — valve sizing
authority is a loop-level quantity computed by the caller (Section 3.9.2 warning).

**Control-loop coupling.** The primary valve controls `T_sec_out`, which is also what
the secondary valve's coil ΔT depends on. Start with primary valve `Ki ≤` secondary
valve `Ki` to avoid cross-loop oscillation (Section 3.9.8 coupling warning).

**Group 5 keys: present even when PHE is absent.** The schema already has these as
`None` (from Session 2). Do not change the no-PHE path — just fill them in on the
PHE path.

---

### Steps

**Step 11a — `PHEXchanger` class**
Read Sections 3.9.1–3.9.5. Create `hydronic/phe.py`. Implement `PHEXchanger` and
`calibrate_phe_ua()`.

Default fixture for all tests (from Section 3.9.3 worked example):
`ua_design=16300, q_pri_design=100, q_sec_design=120`

Write all 6 Section 6.5 test groups in `test_phase2.py` **before** running them.
Group 6 (calibration roundtrip) is the most important — if it fails, all PHE physics
are wrong and nothing downstream can be trusted.

```
pytest test_phase2.py -k "phe" -v
```
All 6 groups green. ✔

**Step 11b — (full suite gate)**
```
pytest test_solver.py test_phase2.py -v
```
Still all green — `PHEXchanger` import has not broken anything. ✔

**Step 12a — Call sequence**
Read Section 3.9.7. Insert `phe.compute_transfer()` between the hydraulic solve and
the coil call in `ClosedLoop.step()`. Wire `t_sec_out_f` → `coil.compute_load()` as
`t_water_in_f`. Wire `T_sec_return` from `dcc.Store` (previous step's `T_sec_out_F`).

Add `ClosedLoop` constructor parameter `phe: Optional[PHEXchanger] = None`. When
`phe is None`, the loop runs exactly as before (Mode B, single circuit). The no-PHE
path must be unchanged.

```
pytest test_phase2.py::test_mode_b_integration -v
```
Still green — existing Mode B unaffected. ✔

**Step 12b — Four PI controllers**
Read Section 3.9.8. Extend to the 9-step dual-circuit sequence when a PHE is present:
- `pump_pri_ctrl` — holds primary ΔP
- `pump_sec_ctrl` — holds secondary ΔP (this replaces the single `pump_ctrl` from
  the no-PHE path when PHE is active)
- `valve_pri_ctrl` — controls `T_sec_out` to setpoint `T_sec_out_setpoint`
- `valve_sec_ctrl` — controls secondary coil ΔT (unchanged from Session 2)

Set starting gains: primary valve `Ki ≤` secondary valve `Ki`.

**Step 12c — Group 5 keys**
Fill in all 13 Group 5 keys in `SimulationStepResult` when PHE is active. They are
already `None` on the no-PHE path — leave that unchanged.

Write the two PHE acceptance tests before running:

```python
def test_phe_energy_balance():
    # At any operating point, both sides must balance to within 1 Btuh:
    # abs(PHE_load_btuh - 500*Q_pri*(T_pri_in - T_pri_out)) < 1.0
    # abs(PHE_load_btuh - 500*Q_sec*(T_sec_out - T_sec_in)) < 1.0

def test_phe_primary_valve_control():
    # Run 80 steps with PHE active, then assert:
    # abs(T_sec_out_F - T_sec_out_setpoint) < 2.0
    # 0.0 < Pump_pri_speed < 1.0   (not railed)
    # 0.0 < Pump_speed < 1.0       (not railed)
    # 0.05 < PHE_effectiveness < 0.95
```

**Final gate:**
```
pytest test_solver.py test_phase2.py -v
```
All green. No regressions. ✔

---

### Quick reference — numbers for this session

| Parameter | Value | Spec |
|-----------|-------|------|
| UA exponent (each PHE side) | 0.4 | §3.9.4 |
| `Q_PRI/SEC_FLOOR_GPM` | 1.0 | §3.9.4 |
| ε numerical ceiling | `1.0 - 1e-9` | §3.9.5 |
| Balanced flow threshold | `abs(c_r - 1.0) < 1e-6` | §3.9.5 |
| Calibration ε guard | `0.01 < ε < 0.99` | §3.9.3 |
| PHE ε acceptance bounds | 0.05 – 0.95 | §9 criterion 14 |
| T_sec_out control tolerance | ±2 °F after 80 steps | §9 criterion 14 |
| PHE energy balance tolerance | < 1 Btuh | §9 criterion 13 |
| Worked example: UA_design | 16,300 Btuh/°F | §3.9.3 |
| Worked example: T_sec_out | ≈ 116.7 °F | §3.9.3 |

---

### Final acceptance checklist (all 14 criteria — Section 9)

- [ ] `pytest test_solver.py test_phase2.py` — all green
- [ ] Criteria 1–3: all Phase 1 + Phase 2 unit tests pass
- [ ] Criterion 4: load-drop settles ≤ 20 steps (manual)
- [ ] Criterion 5: status panel shows A/F labels + threshold (manual)
- [ ] Criterion 6: Mode A still works (manual + pytest)
- [ ] Criterion 7: no global mutable state (grep)
- [ ] Criterion 8: bumpless transfer — no visible spike (manual)
- [ ] Criterion 9: T_air_out displayed and updates (manual)
- [ ] Criterion 10: UA calibration comments match spec numbers (code review)
- [ ] Criterion 11: cooling path pytest test passes
- [ ] Criterion 12: bumpless transfer pytest test passes (±0.02, ±2 ft)
- [ ] Criterion 13: PHE energy balance pytest test passes
- [ ] Criterion 14: PHE primary valve control pytest test passes
