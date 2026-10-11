# gym-torcs — High-Performance Rule-Based Driver for TORCS

A Python 3 OpenAI-Gym-style environment for the
[TORCS](http://torcs.sourceforge.net/) open-source racing simulator,
with a fully rewritten rule-based driving model built on top of the
original gym-torcs.

---

## Best result

| Track | Lap time | Top speed | Damage |
|---|---|---|---|
| Michigan oval (20 laps) | **01:55:10** | **249 km/h** | **0** |

---

## How to run

1. Launch TORCS and start a Practice race on the Michigan oval.
2. In a second terminal:

```bash
python snakeoil3_gym.py
```

The driver connects automatically on the default port. Do **not** hardcode
the port — the `Client` class reads it from the command line.

---

## Driver architecture (`snakeoil3_gym.py` → `drive_example()`)

All driving logic lives in a single function called every simulation step
(~50 Hz). No machine learning, no pre-recorded data — pure sensor-reactive
control.

### 1. Physics-based corner speed model

```
target_speed = min(320, k_blend × √lookahead)
```

- **`soft_min`** — the maximum of the 7 most forward-facing track sensors
  (indices 6–12). Using the maximum rather than minimum filters out spurious
  short readings caused by the inner wall receding at an angle, so the car
  does not brake prematurely on long straights.
- **`lookahead`** — blends `soft_min` (60%) with a weighted average of all
  19 sensors (40%) for smoothness.
- **`k_blend`** — variable coefficient: `14.1` at `soft_min ≤ 50 m`
  (tight hairpin, physics grip limit), linearly rising to `50.0` at
  `soft_min ≥ 150 m` (open straight). The hairpin value is derived from
  `v = √(2μgd)` with μ ≈ 0.9g.

### 2. Anticipatory (preview) steering

The 19 sensors are arranged in a fan. Asymmetry between the left and right
near-forward sensors (indices 3–8 vs 10–15) reveals an upcoming corner
before the car's heading angle changes:

```python
imbalance   = right_near_avg − left_near_avg   # positive = left corner ahead
preview_gain = 0.0012 + 0.000020 × max(0, speed − 50)
preview_steer = −imbalance × preview_gain
```

The gain **scales with speed**: at 200 km/h the car starts rotating into the
corner ~3.5× earlier than at low speed, so it arrives at the apex already
pointed correctly instead of braking to near-zero and then turning.

### 3. Braking-phase steer boost

During braking (`speed > target_speed`), the computed steer signal is
amplified by **1.5×**:

```python
R['steer'] = clip(R['steer'] * 1.5, -1, 1)
```

This coordinates braking and steering simultaneously — the car rotates into
the corner *while* slowing down rather than after.

### 4. Speed-scaled steering centering

```python
center_gain = max(0.025, 0.20 − 0.0007 × max(0, speed − 50))
```

At ≤50 km/h: gain = 0.20 (strong wall-avoidance).
At 200 km/h: gain = 0.07 (gentle, no wobble on straights).

### 5. Throttle and braking

- **Accel ramp**: `+0.20/step` — reaches full throttle in ~0.1 s after
  corner exit.
- **Braking divisor**: `max(30, speed × 0.25)` — firm at high speed,
  gentle at low speed.
- **TCS**: backs off throttle by 0.2 when rear wheel spin exceeds front
  by >5 rad/s.

### 6. Automatic transmission

| Speed (km/h) | Gear |
|---|---|
| < 40 | 1 |
| 40–70 | 2 |
| 70–100 | 3 |
| 100–135 | 4 |
| 135–170 | 5 |
| > 170 | 6 |

### 7. Stuck recovery

If `|speedX| < 3 km/h` and `|trackPos| > 0.8`, engage reverse, steer away
from the wall, apply 50% throttle.

---

## Optimisation history

| Commit | Change | Lap time | Top speed |
|---|---|---|---|
| `a9664b1` | soft-min + variable-k baseline | 02:10:91 | — |
| `e8e1e52` | low-speed boost safety fix | 02:10:66 | — |
| `0d7d6e3` | soft_min index `[2]→[3]` | 02:09:78 | 206 |
| `8cdc782` | soft_min index `[3]→[4]` | 02:09:27 | 202 |
| `d9bc553` | soft_min index `[4]→[5]` | 02:08:02 | 208 |
| `02cbdd4` | soft_min index `[5]→[6]` (max) | 02:06:66 | 209 |
| `92ed4b4` | k ceiling `17.0→17.5` | 02:06:25 | 212 |
| `888f287` | k ceiling `17.5→19.0` | 02:05:82 | 208 |
| `eab67b7` | k ceiling `19.0→25.0` | 02:03:93 | 216 |
| `3ae2e6a` | k ceiling `25.0→30.0` | 02:02:90 | 219 |
| `b385878` | k ceiling `30.0→40.0` | 01:59:04 | 231 |
| `bff8305` | anticipatory steer + ceiling 320 | 02:01:28 | 220 |
| `f7a8dc7` | k ceiling `40.0→50.0` | 01:58:49 | 233 |
| `eab5bf5` | tighter sensor range for preview steer | 01:58:32 | 235 |
| `fe1e9f7` | accel ramp `0.10→0.20/step` | 01:57:43 | 235 |
| `26d00b9` | braking-phase steer boost `1.5×` | 01:55:65 | 236 |
| `d931060` | braking-phase steer boost `2.0×` | 01:55:45 | 236 |
| `8cb33a1` | **speed-scaled preview gain** | **01:55:10** | **249** |

---

## Files

| File | Purpose |
|---|---|
| `snakeoil3_gym.py` | **Main driver** — all driving logic in `drive_example()` |
| `gym_torcs.py` | OpenAI-Gym wrapper for RL use |
| `practice.xml` | Race config: Michigan oval, 20 laps |
| `jmcncarai.py` | Reference client (IBM-supplied, unmodified) |
| `snakeoil3_jm2.py` | Reference client (IBM-supplied, unmodified) |

---

## Track: Michigan Oval

| Segment | Type | Radius | Distance |
|---|---|---|---|
| 0 | Left sweeper | 600 m | 0–356 m |
| 100 | Left sweeper | 535 m | 356–524 m |
| **200** | **Tight hairpin** | **120 m** | **524–792 m** |
| 400–600 | Pit straight | — | 792–1520 m |
| **700** | **Tight hairpin** | **120 m** | **1520–1787 m** |
| 800 | Left sweeper | 535 m | 1787–1955 m |
| 900 | Left sweeper | 600 m | 1955–2312 m |

All corners are left-hand turns. Track width: 18 m.
The two hairpins (r=120 m) set the physics safety floor: `k=14.1`,
`target_speed ≈ 117 km/h` at the apex.

---

# Original gym-torcs README

Gym-TORCS is the reinforcement learning (RL) environment in TORCS domain
with OpenAI-gym-like interface. TORCS is the open-source realistic car racing
simulator recently used as RL benchmark task in several AI studies.

This code is developed based on
[vtorcs](https://github.com/giuse/vtorcs) and
[python-client for torcs](http://xed.ch/project/snakeoil/index.html).

The detailed explanation of original TORCS for AI research is given by
Daniele Loiacono et al. (https://arxiv.org/pdf/1304.1672.pdf)

## Requirements

- Python 3
- xautomation
- OpenAI-Gym
- numpy
- vtorcs-RL-color

## Simple How-To

```python
from gym_torcs import TorcsEnv

env = TorcsEnv(vision=False, throttle=True)
ob = env.reset(relaunch=True)

from sample_agent import Agent
agent = Agent(1)
action = agent.act(ob, reward, done, vision=True)
ob, reward, done, _ = env.step(action)
env.end()
```
