# HastyCR — context for Claude Code

A deterministic Clash Royale battle simulator (`sim/`), a PPO agent trained inside it,
and a live bridge that plays the real game on a MuMu emulator (`scripts/`).
We are improving it by following `ROADMAP.md`, one phase at a time.

## Environment

- **Windows.** The live-play launchers are PowerShell (`run.ps1`, `studio.ps1`).
- **Python 3.12 exactly** (`pyproject.toml` pins `>=3.12,<3.13`).
- `scripts/bootstrap.ps1` creates three venvs under `.venvs/`:
  - `.venvs/clashai`: has this project installed editable, plus pytest. **Use it for sim, training and tests.**
  - `.venvs/buildabot`: the venv `run.ps1` uses for live play (BuildABot detector plus `.[live]`).
  - `.venvs/pyclashbot`: adapter tests only.
- The interpreter for everything below is `PY=.venvs/clashai/Scripts/python.exe`.
  On this PC, `clashai` also has `.[live,dev]` and `vendor/ClashRoyaleBuildABot[cpu]` installed,
  because the test suite imports PyQt6 and BuildABot.
- If `.venvs/` is missing, tell the user to run `powershell -ExecutionPolicy Bypass -File scripts\bootstrap.ps1`.
  Don't improvise a different setup.
- **Card data is not in the repo.** `sim/gamedata.py` reads stats from `tmp/gamedata/csv_logic/`,
  which is gitignored. It's extracted from the Clash Royale install on the user's MuMu emulator:
  `$PY scripts/extract_game_data.py --adb "<path to MuMu adb.exe>" --serial 127.0.0.1:<port>`
  (it pulls the APK and overlays the device's live-patch files).
  If that folder is missing, the simulator and most tests fail with `FileNotFoundError`.
  Ask the user to run the extraction with MuMu open. Don't fake or download the data.

## Commands

```bash
$PY -m pytest tests -q                       # full suite (~1,700 tests, ~25 min on this PC). Run before calling anything done
$PY -m pytest tests --lf -q                  # only the tests that failed last run (seconds). Use while iterating
$PY -m pytest tests/test_<area>.py -q        # one file while working on one area
$PY -m pytest tests/test_rl_policy.py -q     # live-bridge / simulator agreement
$PY -m pytest tests/test_sim_determinism.py tests/test_sim_symmetry.py -q
$PY -m sim.check                             # engine prints checkable predictions (Hog vs tower etc.)
$PY -m sim.batch --matches 50 --workers 4 --opponent brain    # headless matches
$PY -m sim.watch --skin debug --rings        # visual; needs a display, the user runs it

# smoke-scale training. Fine to run yourself
$PY -m sim.clone --episodes 20 --epochs 1 --name smoke_clone
$PY -m sim.train_ppo --envs 2 --steps 20000 --eval-every 10000 --eval-episodes 2 \
    --init tmp/rl/smoke_clone.pt --name smoke

# full-scale training. ASK the user first; these take hours
$PY -m sim.clone --episodes 4000 --epochs 8 --name clone
$PY -m sim.train_ppo --envs 8 --steps 2000000 --init tmp/rl/clone.pt \
    --opponent meta --brain-share 0.5 --value-warmup 300000 --name <run>
$PY scripts/evaluate_pilot.py --ckpt tmp/rl/<run>_best.pt --episodes 60
$PY scripts/head_to_head.py --a <old.pt> --b <new.pt> --games 100
```

Training output goes to `tmp/rl/` (`<name>.pt` for clones, `<name>_best.pt` and `<name>_last.pt` for PPO).
The README says `runs/clone/clone.pt`. That's wrong; the code saves to `tmp/rl/`.

## Map of the code

| path | role |
|---|---|
| `sim/engine.py`, `sim/entities.py`, `sim/spells.py`, `sim/towers.py`, `sim/pathfind.py` | the battle engine |
| `sim/gamedata.py` | card stats decoded from the client's CSV tables |
| `sim/match.py`, `sim/arena.py` | match loop and arena geometry (millitiles) |
| `sim/env.py` | **RL contract**: `observe()`, `legal_mask()`, action encode/decode, rewards, `ClashEnv` |
| `sim/train_ppo.py` | PPO trainer. **`build_network()` is the only place the network is defined** |
| `sim/clone.py` | behaviour cloning of the rule engine (the PPO starting point) |
| `sim/vecenv.py` | parallel envs plus the self-play league (loads past checkpoints) |
| `scripts/brain/rl_policy.py` | live bridge: `observation_from_live()` / `mask_from_live()` must mirror `sim/env.py` |
| `scripts/brain/policy.py` | the hand-written rule engine ("brain") |
| `scripts/cr_bot.py` | live runner (vision → brain → adb taps). The deck is hard-coded as `DECK` |
| `scripts/brain/vision.py` | YOLO vision. Weights are not in the repo (`tmp/yolo/runs/*/weights/best.pt`) |
| `tests/` | one file per behaviour, each named for the failure it prevents |
| `docs/RL_SPRINT4_DECISIONS.md` | the author's append-only decision log. Read it before touching training |

## Facts that matter

- **Units.** Positions are integer **millitiles** (1 tile = 1000) and time moves in fixed **50 ms ticks**.
  Keep the engine integer-only. Floats in positions break determinism.
- **Determinism.** The same seed gives the same match. `test_sim_determinism.py` guards it, and every
  measured win rate depends on it.
- **Policy grid.** 18 wide × 32 tall. Row `y >= 16` is our half. Each seat sees the board as the
  bottom player, and `observe()` mirrors the board for the top seat.
- **Observation.** `planes` is `(NUM_PLANES=8, 32, 18)`:
  ally/enemy units, ally/enemy hp/1000, ally/enemy air, ally/enemy buildings.
  `scalars` is `(NUM_SCALARS=47,)`: elixir, time, elixir multiplier, 4 tower fractions,
  4-slot × 8-card hand one-hot, and an 8-card next-card one-hot.
- **Actions.** `ACTIONS = 1 + 4*576 + 16 = 2337`.
  - index 0: hold elixir
  - `1 + slot*576 + y*18 + x`: play a hand slot on a tile
  - the last 16: champion/hero abilities, by UID rank
  - Illegal actions are masked to `-inf` before the softmax (`masked_distribution`).
- **Deck.** The agent only knows Hog 2.6 (`DECK_26` in `sim/env.py`, `DECK` in `scripts/cr_bot.py`).
- **Checkpoints.** A checkpoint is `{"state_dict": ...}` with no architecture tag. Loaders call
  `build_network(NUM_PLANES, NUM_SCALARS, ACTIONS)` and then `load_state_dict`. They live in
  `sim/clone.py`, `sim/train_ppo.py` (init and anchor), `sim/vecenv.py`, `scripts/brain/rl_policy.py`,
  `scripts/shadow_advisor.py` and `scripts/diagnose_sprint4.py`.
- **Engine fixes void old checkpoints.** A policy learned against the old physics. After changing
  mechanics, say so in the decision log and expect to retrain.

## The rule that bites: simulator and live must agree

`sim/env.py:observe()` and `scripts/brain/rl_policy.py:observation_from_live()` must produce
the same tensors for the same board. A mismatch throws nothing. The live bot just plays badly.
Any change to the observation or action space changes **both** files in the same commit.
Extend `tests/test_rl_policy.py` so it checks the new fields cell for cell.

## When Clash Royale updates

If the user says the game has updated, work through this list in order. Report what changed
before retraining anything.

1. Save the current data: copy `tmp/gamedata/` to `tmp/gamedata_<old-date>/`.
   Save the output of `$PY -m sim.check` to `docs/sim_check_<old-date>.txt`.
2. The user updates CR inside MuMu, then re-runs `scripts/extract_game_data.py`.
   Delete `tmp/apk/` first, or the old APK is reused.
3. Diff the old and new data folders (`tools/research/csv_history.py` may help). List every changed
   stat for the Hog 2.6 cards and for the meta-deck cards, and any new or renamed columns or TOML keys.
4. Check `data/royaleapi/combat_rules.json`. Its hand-verified values **override** the extracted
   data, so an entry the patch changed is now stale. Flag it for the user and don't guess the new
   value.
5. Run the full suite and `sim.check`, then diff `sim.check` against the old output. Card-catalog,
   level-audit and ability-coverage tests catch schema changes and unimplemented new mechanics.
6. Retraining. If a 2.6 card or an opponent card changed meaningfully, propose fine-tuning the
   current best checkpoint (`train_ppo --init <best>`), not training from scratch, then re-evaluate.
   Log the decision either way.
7. Live side. New cards, skins or a UI change need new screenshots from the user. Don't change
   tap coordinates or detector classes blind.

## Working rules

1. Work one roadmap phase at a time, on its own git branch (`phase-N-short-name`). Commit in small steps.
2. Before saying a phase is done, run the full suite. Report the pass, fail and xfail counts against the baseline.
3. **Never weaken, skip or delete a test to get green.** Strict `xfail`s are on purpose. Remove an
   `xfail` marker only when the behaviour now passes. Add a test for every behaviour you change.
4. Don't start runs longer than about 10 minutes (full clone, PPO, large `sim.batch`) without asking.
   Give the user the exact command and let them start it.
5. Don't touch the live-play path (`run.ps1`, `cr_bot.py`, emulator scripts) unless the phase calls for it.
   Never drive the emulator or the game yourself.
6. Don't commit `.pt` files, `tmp/`, `vendor/*/` or `.venvs/`. `.gitignore` already covers them.
7. Record each material decision in `docs/DECISIONS_ROADMAP.md`. Use the format of
   `docs/RL_SPRINT4_DECISIONS.md`: what was decided, the evidence, the expected effect, and later the outcome.
8. Win-rate claims need held-out seeds (`--seed 8000`), ≥60 episodes, and the opponent named.
   A number from training-time eval is not a result.
9. If the code contradicts this file, trust the code and fix this file.
