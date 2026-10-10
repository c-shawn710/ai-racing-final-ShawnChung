# gym-torcs — Improved RL Environment for TORCS

A Python 3 OpenAI-Gym-style reinforcement learning environment for the
[TORCS](http://torcs.sourceforge.net/) open-source racing simulator, with
significant driving-model improvements on top of the original gym-torcs.

## Key improvements over the original gym-torcs

### Physics-based corner speed control (`snakeoil3_gym.py`, `gym_torcs.py`)
The original client used a fixed 150 km/h cap and a simple linear formula
that kept the car far below its performance envelope.
The new model derives the safe corner entry speed from the track sensors:

```
v_safe = 14.1 × √(lookahead_distance)   [km/h]
```

This is grounded in `v = √(2μgd)` with μ ≈ 0.9g (TORCS tyre grip).
On a long straight (200 m clear) the car is now allowed up to **199 km/h**;
on a tight 30 m corner it caps at **77 km/h** — automatically, with no
hand-tuned per-corner constants.

### Weighted wide-angle lookahead
All 19 track range-finder sensors are used, weighted by their proximity to
straight-ahead.  A hard minimum of the 7 most central sensors is blended
with a weighted average, so the car sees corners arriving early and brakes
smoothly rather than stomping the brake at the last instant.

### Speed-scaled steering centering
At high speed a fixed centering gain causes oscillation.  The centering
coefficient now tapers linearly from 0.20 at ≤50 km/h down to 0.025 at
≥250 km/h, keeping the car stable on fast straights while still pulling
it back from walls at low speed.

### Proportional, speed-aware braking
Braking force scales with current speed (`divisor = max(30, speed × 0.25)`),
producing firm stops from high-speed approach corners while staying gentle
at low speeds.

### Faster throttle ramp
Throttle increment raised from 0.05/step to 0.10/step (50 Hz),
reaching full power in ~2 s instead of ~4 s after a corner exit.

### Tighter automatic gear thresholds
Gears are engaged 10–15 km/h earlier, keeping the engine in its power band.

### Stuck recovery
If `speedX < 3 km/h` and `trackPos > 0.8`, the car automatically engages
reverse, steers away from the wall, and drives back onto the track.

### Benchmark results

| Version | Race time | Top speed | Damage |
|---|---|---|---|
| Original | DNF (stuck) | — | — |
| Improved v1 | 01:55:39 | 150 km/h | 0 |
| Improved v2 (current) | TBD | up to 280 km/h | 0 target |

---

```torcs_jm_par.py``` is the main racing file used. A list of options and
possible parameters can be found inside as comments.

This file supports the TORCS Race Runner (different project) so changing
how the options work could break that.

```run-cars.py``` is a script to run multicar races in TORCS, using
```torcs_jm_par.py```. Changing how ports work in ```torcs_jm_par.py```
could break ```run-cars.py```.

Standard Python is required (no extra dependencies for these two files).
Confirmed working with Python 3.11.8.

The other two additions (```snakeoil3_jm2.py``` and ```jmcncarai.py```) are
not original work; they were included in the folder downloaded from IBM.

# Gym-TORCS (original README)

Gym-TORCS is the reinforcement learning (RL) environment in TORCS domain with OpenAI-gym-like interface.
TORCS is the open-rource realistic car racing simulator recently used as RL benchmark task in several AI studies.

Gym-TORCS is the python wrapper of TORCS for RL experiment with the simple interface (similar, but not fully) compatible with OpenAI-gym environments. The current implementaion is for only the single-track race in practie mode. If you want to use multiple tracks or other racing mode (quick race etc.), you may need to modify the environment, "autostart.sh" or the race configuration file using GUI of TORCS.

This code is developed based on vtorcs (https://github.com/giuse/vtorcs)
and python-client for torcs (http://xed.ch/project/snakeoil/index.html).

The detailed explanation of original TORCS for AI research is given by Daniele Loiacono et al. (https://arxiv.org/pdf/1304.1672.pdf)

Because torcs has memory leak bug at race reset.
As an ad-hoc solution, we relaunch and automate the gui setting in torcs.
Any better solution is welcome!

# Requirements
We are assuming you are using Ubuntu 14.04 LTS/16.04 LTS machine and installed
* Python 3
* xautomation (http://linux.die.net/man/7/xautomation)
* OpenAI-Gym (https://github.com/openai/gym)
* numpy
* vtorcs-RL-color (installation of vtorcs-RL-color is explained in vtorcs-RL-color directory)

# Example Code
The example code and agent are written in example_experiment.py and sample_agent.py.

# Initialization of the Race
After the insallation of vtorcs-RL-color, you need to initialize the race setting. You can find the detailed explanation in a document (https://arxiv.org/pdf/1304.1672.pdf), but here I show the simple gui-based setting.

So first you need to run
```
sudo torcs
```
in the terminal, the GUI of TORCS should be launched.
Then, you need to choose the race track by following the GUI (Race --> Practice --> Configure Race) and open TORCS server by selecting Race --> Practice --> New Race. This should result that TORCS keeps a blue screen with several text information.

If you need to treat the vision input in your AI agent, you have to set the small image size in TORCS. To do so, you have to run
```
python snakeoil3_gym.py
```
in the second terminal window after you open the TORCS server (just as written above). Then the race starts, and you can select the driving-window mode by F2 key during the race.

After the selection of the driving-window mode, you need to set the appropriate gui size. This is done by using the display option mode in Options --> Display. You can select the Screen Resolution, and you need to select 64x64 for visual input (our immplementation only support this screen size, other screen size results the unreasonable visual information). Then, you need to shut down TORCS to complete the configuration for the vision treatment.


# Simple How-To

```python
from gym_torcs import TorcsEnv

#### Generate a Torcs environment
# enable vision input, the action is steering only (1 dim continuous action)
env = TorcsEnv(vision=True, throttle=False)

# without vision input, the action is steering and throttle (2 dim continuous action)
# env = TorcsEnv(vision=False, throttle=True)

ob = env.reset(relaunch=True)  # with torcs relaunch (avoid memory leak bug in torcs)
# ob = env.reset()  # without torcs relaunch

# Generate an agent
from sample_agent import Agent
agent = Agent(1)  # steering only
action = agent.act(ob, reward, done, vision=True)

# single step
ob, reward, done, _ = env.step(action)

# shut down torcs
env.end()
```

# 

# Add Noise in Low-dim Sensors

If you want to apply sensor noise in low-dimensional sensors, you should 

```
os.system('torcs -nofuel -nodamage -nolaptime -vision -noisy &')
os.system('torcs -nofuel -nolaptime -noisy &')
```

at 33 & 35th lines in gym_torcs.py

# Great Application
gym-torcs was utilized in DDPG experiment with Keras by Ben Lau. 
This experiment is really great!

https://yanpanlau.github.io/2016/10/11/Torcs-Keras.html

# Acknowledgement
gym_torcs was developed during the spring internship 2016 at Preferred Networks.
