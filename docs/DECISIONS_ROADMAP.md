# ROADMAP decisions log

Append-only. Format follows `docs/RL_SPRINT4_DECISIONS.md`.

## 2026-10-24 — Phase 0 setup: emulator connection is configurable
- **Chosen:** `run.ps1` and `scrcpy-preview.ps1` take `-Adb` / `-Serial`. An explicit flag wins,
  then `HASTYCR_ADB` / `HASTYCR_SERIAL`, then the old MuMu values (`127.0.0.1:16480`).
  `scripts/emulator.py` already read `HASTYCR_SERIAL`; its adb path now reads `HASTYCR_ADB` too, and
  the preview passes its serial to the device check. `studio.ps1` hard-codes neither value.
  README: clone path is `tmp/rl/clone.pt`; `-Vision buildabot` is needed without trained YOLO weights.
- **Why:** ROADMAP Phase 0 steps 3–4.
- **Expected effect:** none with no flags or env vars set; the defaults are unchanged.
- **Outcome:** not yet checked on Windows (`.\run.ps1 -Stop -Adb ... -Serial ...`).

## 2026-10-24 — Phase 0 baseline: pending
- Steps 0–2 and 5 (baseline suite, `sim.check`, smoke runs) need `tmp/gamedata/csv_logic/` and the
  `.venvs/` from `scripts/bootstrap.ps1`. Neither exists in the cloud session these edits were made in,
  so the baseline has to be taken on the Windows PC. Last known run (from ROADMAP Phase 0b):
  61 failed, 1637 passed, 14 skipped, 2 xfailed on data extracted 2026-10-24.
