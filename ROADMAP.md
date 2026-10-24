# HastyCR improvement roadmap

Work the phases in order. Each phase has a goal, the files it touches, the steps, and a
**done when** list. Every item on that list must hold before the next phase starts.
Rules and commands are in `CLAUDE.md`. Log decisions in `docs/DECISIONS_ROADMAP.md`.

Progress (Claude Code updates this list as it goes):

- [ ] Phase 0: baseline and setup fixes
- [ ] Phase 0b: repair game-data drift (61 failing tests on current client data)
- [ ] Phase 1: convolutional policy head
- [ ] Phase 2: fix the charging-unit deadlock
- [ ] Phase 3: richer observation (unit roles, enemy elixir, enemy cycle)
- [ ] Phase 4: deck-agnostic hand encoding
- [ ] Phase 5: live HP-bar reading (stretch goal)

---

## Phase 0: baseline and setup fixes

**Goal:** know where we're starting from, and remove the setup friction.

0. Confirm `tmp/gamedata/csv_logic/` exists. If it doesn't, stop and ask the user to run
   `scripts/extract_game_data.py` with MuMu open (see `CLAUDE.md`).
1. Run the full test suite and record the pass, fail and xfail counts in `docs/DECISIONS_ROADMAP.md`.
   Everything later compares against these numbers. If something fails before any change, write it
   down and don't fix it yet.
2. Run `$PY -m sim.check` and save the output to `docs/baseline_sim_check.txt`. Phase 2 diffs against it.
3. `run.ps1`: make `$adb` and `$serial` parameters (`-Adb`, `-Serial`). Keep the current values as defaults.
   Also let the environment variables `HASTYCR_ADB` / `HASTYCR_SERIAL` override them.
   `studio.ps1` / `scrcpy-preview.ps1` may hard-code the same values; apply the same change there.
4. `README.md`: fix the clone path (`runs/clone/clone.pt` → `tmp/rl/clone.pt`).
   Note that live play without trained YOLO weights needs `-Vision buildabot`.
5. Run the smoke clone and smoke PPO from `CLAUDE.md` to prove the training loop runs on this machine.

**Done when**
- Baseline counts are logged.
- `run.ps1 -Adb ... -Serial ...` works. Check with `-Stop`, or by reading the script; don't start a match.
- Both smoke runs finish and write `tmp/rl/smoke_clone.pt` and `tmp/rl/smoke_best.pt` (or `smoke_last.pt`).

**Ask the user** whether to train a full-size baseline clone with the *current* network
(`--name clone_dense`) so Phase 1 has a fair comparison. It takes a while, so they start it.

---

## Phase 0b: repair game-data drift (before any training)

**Why:** the first full run on this machine, with game data extracted 2026-10-24, gave
**61 failed, 1637 passed, 14 skipped, 2 xfailed**. The repo's tests were written against the
author's older extraction. The current client has changed values, added content, and moved some
definitions to places the loader doesn't read. That last group is the dangerous one: the loader
quietly reads **0** and the sim plays wrong without an error. Training on a broken sim teaches
the policy broken physics, so fix this first.

The full failure log is `pytest_failures.txt` in the repo root (UTF-16, made with
`pytest --lf -q -rf --tb=line`). Re-run only those tests with `$PY -m pytest tests --lf`.

Work the categories in this order. Each category below is a first guess from the failure
messages; confirm it by reading the extracted files in `tmp/gamedata/` before changing anything.

**A. Loader reads zero or empty: fix the loader, never the test. Highest priority.**
- **Ice Golem's death blast deals 0** (`test_sim_fidelity::test_a_dying_unit_explodes...`,
  `ice_golemite`). **This card is in the Hog 2.6 deck.** Fix it first.
- **No evolution overlays found at all** (`test_evolution_coverage`). Every evo card currently
  loads as its base card. Most of the evo/hero failures in `test_sim_fidelity` probably follow
  from this.
- **Death-spawn bombs resolve to nothing:** Balloon, Balloon hero, Bomb Tower, Giant Skeleton,
  Skeleton Balloon (`test_death_spawns`).
- **Zap reads 0 damage and no stun.** Monk is untouched by Zap and Freeze. `Zap_EV1` is missing
  (`test_sim_fidelity`, `test_monk_reflection`, `test_transcribed_graphs`).
- **Goblin Drill surfacing burst** and the E-Wiz / Ice Wiz deploy burst (`test_goblin_drill`).
- **Evolved Valkyrie pull, the Firecracker deflect cost, and hero abilities**
  (`ice_wizard_hero` never offered, `electro_wizard_hero` refused).
- **Lead:** `test_transcribed_graphs` reports new staged action-group files: `dark_magic.toml`,
  `electro_giant_ev1.toml`, `ice_wizard_hero.toml`, `triwizardspawn.toml`, `zap_ev1.toml`.
  Supercell may have moved some spell and ability data into action graphs, which would explain
  Zap reading 0. Diff the old file layout (as described in the tests) against `tmp/gamedata/` and
  extend the loaders.

**B. Real balance changes: update the expected value, after checking it.**
- Examples: Elixir Collector generation 13000→15000 ms, Berserker hero ability 4000→3500,
  and several numeric tuple diffs in `test_sim_fidelity`, `test_skeleton_king` and
  `test_building_pull`.
- Before changing a test, check the new number against the extracted file **and** a public
  source (patch notes, RoyaleAPI). Record each change in the decisions log. If
  `data/royaleapi/combat_rules.json` overrides that stat, update the override too.

**C. New content.** Examples: `GoblinCurse` immunity, `super_lava_hound`, and the attractor set
changing. Update the declared sets in the tests, and check the engine handles the new entries.
Implement them only if they're needed by a deck in the training pool.

**D. Known gaps now fixed by the data.** `test_periodic_spawners` XPASS(strict) for
`super_hog_rider`: the data now has hitpoints. Remove that `xfail` entry.

**E. Strict TOML.** `character_buffs.toml` declares `ImmuneToWeakPushback` twice. `tomllib`
rejects the file and `parse_toml_file` falls back to its own parser. Check that the fallback reads
that file correctly (compare a few buffs). Then make the test accept a client file that is only
rejected for duplicate keys, provided the fallback parses it.

**F. `test_studio_brain_choice` (2 tests).** These belong to the live studio UI, not the sim.
Look at them last.

**Done when**
- Every category-A item is fixed in the loader. Each one keeps a test that fails if the value
  ever reads 0 again.
- B and C changes are logged with their sources.
- The full suite is green except for documented, deliberate `xfail`s.
- `sim.check` output for the Hog 2.6 scenarios is saved as the new baseline.

---

## Phase 1: convolutional policy head

**Why:** `build_network()` flattens a 64×32×18 feature map into a 36,864→512 dense layer.
That layer holds about 19M of the network's ~20M parameters, and it learns every tile separately.
The action space is already a map: actions 1..2304 are `slot*576 + y*18 + x`.
So the policy should be convolutional and output one logit map per hand slot.

**Files:** `sim/train_ppo.py` (network), plus every checkpoint loader listed in `CLAUDE.md`
(`sim/clone.py`, `sim/train_ppo.py`, `sim/vecenv.py`, `scripts/brain/rl_policy.py`,
`scripts/shadow_advisor.py`, `scripts/diagnose_sprint4.py`), plus a new test.

**Design (keep the constraints; the details are yours)**
- **Input:** run the 8 planes through a small conv stack.
  Inject the 47 scalars into every tile, either by broadcasting a scalar MLP's output as extra
  channels or with FiLM.
  The tile logits must know which card sits in each slot.
- **Tile head:** a 1×1 conv down to 4 channels gives logits shaped `(B, 4, 32, 18)`.
  Flatten it slot-major, row-major, to exactly match `ClashEnv.encode(slot, x, y)`.
- **Pass and ability head:** global-pool the features and concatenate the scalar embedding.
  An MLP outputs 1 hold logit plus 16 ability logits.
- **Value head:** from the same pooled features.
- **Output:** `(logits[B, 2337], value[B])`. Same signature as now, so `masked_distribution` and PPO
  stay unchanged.
- **Receptive field:** the grid is 32 rows tall, and a tile decision depends on what's happening
  across the river. Use dilated convs or a downsample/upsample (U-Net-lite) path, so a tile can
  "see" the whole board.
- **Target:** under 1M parameters.

**Checkpoint versioning**
- Add `arch` to every saved blob (`"conv_v1"`). A blob without `arch` means `"dense"`.
- Keep the old network as `build_network(..., arch="dense")` so old checkpoints still load.
- Add one helper, `load_policy(blob_or_path, device)`, that reads `arch`, builds the right network,
  and loads the weights. Route every loader through it.
- New runs default to `conv_v1`. Add a `--arch` flag to `sim/clone.py` and `sim/train_ppo.py`.
- A shape mismatch must raise an error that names both architectures. Don't load partially.

**New test: `tests/test_policy_head_layout.py`**
- Build the conv net and use hooks (or a forced bias) to put one high logit at map `[slot, y, x]`.
  Check it lands at flat index `ClashEnv.encode(slot, x, y)`.
- Check the parameter count is under 1M.
- Check a dense blob still loads through `load_policy`.
- Check a conv blob round-trips through save and load.

**Done when**
- The full suite is green against the baseline, and the new test passes.
- Smoke clone and smoke PPO run with `--arch conv_v1`.
- `rl_policy.RLBrain` loads a conv checkpoint (see the existing tests in `tests/test_rl_policy.py`).
- **The user has run the full clone:**
  `$PY -m sim.clone --episodes 4000 --epochs 8 --arch conv_v1 --name clone_conv`.
  Its validation accuracy is logged next to `clone_dense` if that run exists.
- **The user has run PPO from `clone_conv`, and it has been evaluated:**
  `evaluate_pilot.py --episodes 60 --seed 8000` against brain and meta.
  The numbers are logged.
  The README's reference is 93.3% vs rule engine and 86.7% vs meta at 6.1M steps, measured on the dense net.

---

## Phase 2: fix the charging-unit deadlock

**Why:** a Battle Ram, Prince, Dark Prince or Ram Rider gets held **for ever** by a single Knight.
`tests/test_body_block.py::test_a_charging_unit_is_also_not_stopped_for_ever` is a strict `xfail`
that documents this. Fix it before Phase 4, because other decks contain charging cards. Otherwise
the agent learns to exploit the bug.

**Files:** `sim/engine.py`, `sim/entities.py`, maybe `sim/pathfind.py`.
`tests/test_body_block.py`, `tests/test_speed_units.py`, `tests/test_stuck_units.py`.

**Steps**
1. Read `docs/RL_SPRINT4_DECISIONS.md` § "Body-block deadlock fixed". Hog Rider and Giant already
   steer around a blocker they can't target; find out why that steering doesn't apply while a unit is
   charging. The charge state uses `charge_speed_multiplier`.
2. Fix it in the same shape as the existing fix. Don't special-case Battle Ram.
3. Remove the `xfail` marker. Parametrize the test over `battle_ram`, `prince`, `dark_prince` and
   `ram_rider`, using every card that exists in `sim/gamedata.py`.
4. Add a test that a charge still breaks or resets when a unit really fights a blocker. That matches
   the real game. Find out what the client data says before assuming.

**Done when**
- The un-xfailed test passes for every charging card.
- The full suite is green, including `test_sim_determinism.py` and `test_sim_symmetry.py`.
- `sim.check` output for the Hog / Musketeer scenarios matches `docs/baseline_sim_check.txt`.
  The 2.6 deck has no charging card, so nothing there should move.
- The decision log says which checkpoints, if any, the fix voids.

---

## Phase 3: richer observation

**Why:** the network can only tell units apart by HP. It has no idea how much elixir the opponent
has, or which of their cards are in cycle. Real players lean on all three.

**Files:** `sim/env.py` (`PLANES`, `observe`, `NUM_SCALARS`), `scripts/brain/rl_policy.py`
(`observation_from_live`), `tests/test_rl_policy.py`, `tests/test_sim_env_contract.py`.
Change both observation files in the same commit (see `CLAUDE.md`).

**Steps**
1. **Role planes**, split into ally and enemy:
   - `targets_buildings`
   - `splash`
   - `tank` (HP above a threshold at the current level)
   - `swarm` (spawned in a group of 3 or more)
   - `ranged` (range above a threshold)

   Derive every role from `sim/gamedata.py` stats, not a hand list, so it works for every card.
   The live side derives the same roles from the detected unit name through the same function.
   Put that function in one shared module.
2. **Enemy elixir estimate** as a scalar. Track it from the elixir clock and the cost of each
   enemy deploy you observe. Clamp it to 0..10. The sim side reads the real value; mark it
   "estimated" in `rl_policy.py`. Add a test that drives an estimate through a recorded deploy
   sequence.
3. **Enemy recent cycle** as scalars: a one-hot or feature vector for the opponent's last 4 played
   cards, plus "cards seen" so far. Use the same tracker in sim and live.
4. Bump the observation version. Store `obs_version` in checkpoints, and make `load_policy` refuse
   a mismatch with a clear message.

**Done when**
- The agreement test in `tests/test_rl_policy.py` compares every new plane and scalar cell for cell
  on a played-out sim match.
- The full suite is green.
- The user has retrained clone and PPO on the new observation, and the evaluation is logged against
  the Phase 1 numbers. The change is kept only if it doesn't regress against both opponents. If it
  does regress, log that and revert.

---

## Phase 4: deck-agnostic hand encoding

**Why:** the hand is a one-hot over the 8 Hog 2.6 cards, so the agent can't play anything else.

**Files:** `sim/env.py` (`DECK_26`, hand scalars, `legal_mask`, the spell set), `sim/deck_builder.py`,
`sim/meta_decks.py`, `sim/vecenv.py`, `sim/train_ppo.py`, `scripts/brain/rl_policy.py`,
`scripts/cr_bot.py` (`DECK`).

**Steps**
1. Build a per-card feature vector from `sim/gamedata.py`. Include elixir cost, HP, DPS, range,
   hit speed, air/ground targeting, flying, building-only targeting, spell radius and spawn count,
   plus the Phase 3 roles. Normalise each feature. Encode each hand slot, and the next card, as
   this vector instead of a one-hot.
2. Remove the hard-coded `SPELLS = {"fireball", "the_log"}`; spell placement rules come from card data.
3. Make the training deck configurable: `--deck hog26`, `--deck <name from data/meta_decks.json>`
   or `--deck random`. Add an option to train on a mix.
4. `cr_bot.py`: read the deck from a CLI flag or config instead of the `DECK` constant. Check that
   the BuildABot detector supports every card in the chosen deck, and refuse to start if it doesn't.

**Done when**
- One network trained on a deck mix plays at least 2 decks in the sim.
- Its Hog 2.6 results are no worse than Phase 3's on the same evaluation protocol.
- The full suite is green, and the live/sim agreement tests cover a non-Hog deck.

---

## Phase 5 (stretch goal): live HP reading

**Why:** live vision gives every detected unit its card's full HP, so a half-dead Musketeer
looks healthy to the policy. The `ally_hp` / `enemy_hp` planes already exist; live mode just fills
them with guesses.

**Approach:** crop the HP bar above each YOLO box and estimate the filled fraction from the bar's
colour run. Calibrate on frames in `data/` or new recordings that the user captures. Check it
offline against frames labelled by hand before wiring it into `observation_from_live`.

**Done when** the offline HP error is measured and logged, and it's wired in behind a flag
(`--live-hp`) that defaults to off until the user has watched it in friendly matches.

---

## Quick reference: decisions log entry

```markdown
## YYYY-MM-DD — <short title>
- **Chosen:** what changed (files, flags, checkpoint names)
- **Why:** evidence (test output, eval table, numbers with seeds and episode counts)
- **Expected effect:** what should move, and what must not
- **Outcome:** filled in after measurement
```
