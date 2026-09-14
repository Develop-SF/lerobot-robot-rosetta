# CLAUDE.md

Guidance for Claude Code when working in `lerobot-robot-rosetta`.

## What this repo is

The LeRobot `Robot` plugin for Rosetta: it turns a unified Rosetta contract
into LeRobot's `get_observation()` / `send_action()` interface by running a
ROS2 lifecycle node internally. LeRobot discovers it by the
`lerobot_robot_*` distribution-name prefix, so `--robot.type=rosetta` works
once it is installed. It is an `ament_python` package (`setup.py` +
`package.xml`) that depends on the sibling `rosetta` package for
`rosetta.common`.

Three files:

- `lerobot_robot_rosetta/config_rosetta.py` — `RosettaConfig`, the
  `RobotConfig` subclass. Accepts **only unified contracts**
  (`config_path` must be a file that `is_unified_contract` recognises);
  loads the contract and caches the observation/action stream specs. The
  feature dicts themselves are built from those specs by
  `Rosetta.observation_features` / `action_features`, before any node
  exists.
- `lerobot_robot_rosetta/rosetta.py` — `_TopicBridge` (subscriptions,
  publishers, `StreamBuffer` alignment, safety watchdog),
  `_RosettaLifecycleNode` (maps configure/activate/deactivate/cleanup onto
  the bridge), and `Rosetta`, the public `Robot` implementation with two
  modes: standalone (creates its own lifecycle node and spins it on a
  background thread) and injected (attaches to a pre-built `_TopicBridge`
  owned by an external node, via `config._external_bridge` — how
  `rosetta_client_node` uses it; no node or spin thread of its own).
- `README.md` — usage and behaviour table.

## Build and run

From the host, enter the `sns-robot-learning` container (the repo is
mounted at `/root/ws_rl/src/lerobot-robot-rosetta`); then, after
`rosetta` is built:

```bash
docker exec -it sns-robot-learning bash
conda activate sns_robot_learning
cd /root/ws_rl
colcon build --symlink-install --packages-select lerobot_robot_rosetta
source install/setup.bash
```

There is no test suite in this repo. Behaviour is exercised indirectly by
`rosetta_client_node`, which imports `_TopicBridge` directly (a private
name, so treat its signature as a cross-repo API), and by
`sns_robot_learning/tests/contracts`, which validates the contracts this
plugin loads.

## Behaviour to preserve

- Standalone mode: `connect()` ≙ lifecycle `activate`; `disconnect()` ≙
  `deactivate` then `cleanup`, publishing the contract's `safety_behavior`
  action first. Injected mode: `connect()` only attaches to the external
  bridge (already set up and activated); `disconnect()` publishes the
  safety action and resets bridge state but never tears the bridge down —
  the external node owns it.
- Watchdog (both modes): once at least one action has been published, if
  the next one doesn't arrive within `2/fps` seconds the bridge applies
  `safety_behavior` (`hold` / `zeros`). It never fires before the first
  action, and when every spec is `none` no watchdog timer is created at
  all.
- Missing topics return zeros and log once; the client node relies on this
  warm-up behaviour when it swaps contracts between goals.
- Feature names follow the contract's selector order and namespace rules
  (`arm.position.j1`); changing that layout invalidates trained checkpoints.

## Fork status

`origin` is `Develop-SF/lerobot-robot-rosetta`; `upstream` is
`iblnkn/lerobot-robot-rosetta`, still active. As of 2026-09-08 `main` is 10
commits ahead of and 21 behind `upstream/main`. Run `git fetch upstream`
first; the remote is rarely fetched.

SNS-only changes a sync must keep: `RosettaConfig` accepts only unified
contracts (e4cc997) and resolves them with the inference role by default
(07e421d; `role` is a configurable field); publishers are activated explicitly on `activate` (656f2ab); the
safety-action fix (cce1268); shape-replacing resize (51337ba). Upstream
has since paced `get_observation` / `send_action` on the
ROS sim clock and exposed `TopicBridge.clock`, which pairs with `rosetta`
dropping `sim_time_multiplier`; merge the two repos together and retest
sim runs.

## Git

Branches are `<user>/feature/<short-desc>` or `<user>/bugfix/<short-desc>`;
PRs land as merge commits. Commit subjects: imperative, capitalised, no
trailing period, ≤ 50 chars. Changes here usually pair with a `rosetta` PR;
link it in the body.
