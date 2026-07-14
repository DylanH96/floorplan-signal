---
name: project-plan-baseline
description: Core decisions made when PLAN.md was first created — interpretation, stack, stage count, and key model parameters.
metadata:
  type: project
---

PLAN.md was created on 2026-07-14 for a greenfield repo. The project is a point-estimate Wi-Fi signal-strength tool (not a heatmap).

**Why:** The architect agent definition lists discrete click-to-place and click-to-measure interactions, which maps cleanly to a point-estimate model. A heatmap was explicitly considered and rejected as out of scope for MVP.

**How to apply:** Do not expand scope toward heatmap generation, multi-AP placement, or real obstacle modelling unless the user explicitly requests it. Record any such request as a new assumption.

Key decisions:
- Python + Streamlit (mandated)
- `streamlit-image-coordinates` for click detection
- Pillow for image handling
- pytest for tests (command: `python -m pytest tests/`)
- Signal model lives in `signal_model.py` (no Streamlit imports)
- Validators live in `validators.py` (no Streamlit imports)
- UI entry point is `app.py`
- 6 stages: Scaffold → Signal Model → Validators → UI Shell → Integration → Polish
- RSSI_0 default: -30 dBm at 1 m; n default: 3.0; d_0 fixed at 1 m
- Classification thresholds: Excellent >= -50, Good >= -60, Fair >= -70, Weak >= -80, Unusable < -80
- SSID is label only — no effect on calculations
