# AI Racing League — TORCS Driver

A Python racing driver improved for the IBM SkillsBuild AI Racing League challenge. The driver uses TORCS sensor readings to control steering, acceleration, braking, and gears.

## Project description / problem statement

The supplied baseline driver could not complete a lap in my tests: it would get stuck at a corner or against a wall. My first priority was to troubleshoot that behaviour and complete a lap successfully. Once the car could finish, I improved its lap time little by little while aiming for zero damage.

The final driver uses rules and sensor feedback rather than a trained machine learning model.

## Results

| Track | Best recorded lap | Top speed | Recorded damage |
| --- | --- | --- | --- |
| Corkscrew | **1:55.10** | **249 km/h** | **0** |

These are recorded development results, not a guarantee for every run. Additional runs are listed in [lap_data.csv](lap_data.csv).

| Development milestone | Lap time | Top speed |
| --- | --- | --- |
| Acceleration ramp increased to 0.20 per step | 1:57.43 | 235 km/h |
| Steering boost of 1.5× during braking | 1:55.65 | 236 km/h |
| Speed-scaled preview steering | **1:55.10** | **249 km/h** |

## Enhancements

1. **Lap completion and recovery:** Reworked steering and speed control, added braking, and added reverse recovery when the car is nearly stationary near the track edge. The initial goal was to stop the car getting stuck and make it complete a lap.
2. **Faster acceleration and better speed control:** Increased the throttle ramp to **0.20 per step**, added a sensor-based corner-speed model and proportional braking, and adjusted automatic gear thresholds to regain speed after turns.
3. **Earlier corner steering:** Added preview steering that responds to differences between left and right track sensors. Its gain increases with speed, and a **1.5× steering boost during braking** helps the car rotate while slowing down.

## Use of IBM Bob

I first used IBM Bob to help troubleshoot why the baseline car was getting stuck. Once it could complete a lap, Bob helped suggest and refine changes to acceleration, braking, steering, and recovery.

I ran the simulator after changes, compared lap times, top speeds, and damage, and used Git to restore working versions when experiments caused crashes or unstable acceleration and braking. This taught me to establish a working baseline first and validate AI suggestions through testing before keeping an improvement.

## Chosen script and dependencies

Run **[snakeoil3_gym_workingcopy.py](snakeoil3_gym_workingcopy.py)**, which contains the improved `drive_example()` function.

| Requirement | Purpose |
| --- | --- |
| Python 3 | Runs the driver |
| A workshop-compatible TORCS/SCR installation | Runs the simulator and exposes the racing server |
| Corkscrew track and an SCR-controlled car | Reproduces the track and client setup used for development |
| Git | Downloads the repository using the commands below |

The selected Python script uses only standard-library modules: `socket`, `sys`, `getopt`, `os`, and `time`. It does not require additional pip packages.

The `vtorcs-RL-color/` directory in this repository currently contains installation documentation only. A complete simulator installation must be supplied separately, such as the environment provided for the workshop.

## Setup and run

These steps assume the workshop-compatible TORCS simulator is already installed and configured.

1. Download the project and check Python:

   ```bash
   git clone https://github.com/c-shawn710/ai-racing-final-ShawnChung.git
   cd ai-racing-final-ShawnChung
   python3 --version
   ```

2. Start the workshop TORCS simulator. If its launcher is available on your PATH:

   ```bash
   torcs
   ```

3. In TORCS, select a **Practice** session on the **Michigan oval** and the workshop's SCR-controlled car. Keep **damage enabled** when evaluating the zero-damage goal. Start the session so the racing server is ready for the Python client.

4. In a second terminal, from the project directory, run:

   ```bash
   python3 snakeoil3_gym_workingcopy.py
   ```

   The client defaults to `localhost` on UDP port `3001`. If your simulator uses another host or port, pass them as command-line options:

   ```bash
   python3 snakeoil3_gym_workingcopy.py --host localhost --port 3001
   ```

   Use the port assigned by the workshop when it differs from the default.

5. Check the completed lap time, top speed, and damage in the simulator. To view client options:

   ```bash
   python3 snakeoil3_gym_workingcopy.py --help
   ```

## Driver behaviour

- **Corner speed:** Combines seven forward track sensors with a weighted average of all 19 sensors to estimate lookahead. The target follows `min(320, k_blend * sqrt(lookahead))`, with `k_blend` ranging from `14.1` to `50.0`. The **320 km/h** value is a configured ceiling, not an achieved top speed.
- **Preview steering:** Uses `0.0012 + 0.000020 * max(0, speed - 50)` as the preview gain. This allows the driver to respond to an upcoming bend before its heading error grows.
- **Steering while braking:** Multiplies the combined steering signal by `1.5` when the car is at or above its target speed.
- **Throttle and braking:** Ramps throttle by `0.20` per step below target speed and applies braking in proportion to excess speed above target.
- **Automatic gears:** Shifts up above 40, 70, 100, 135, and 170 km/h.
- **Recovery:** Selects reverse and 50% throttle when `abs(speedX) < 3` and `abs(trackPos) > 0.8`.

## Known limitations / current project state

- **Corner entry still needs refinement.** The earlier hard-brake-then-turn behaviour improved during development, but smoother braking and turning remain an area for further work.
- **The settings were tuned for the Michigan oval.** Other tracks and simulator configurations need separate testing.
- **Recovery has a narrow trigger.** It may miss some post-crash situations, including a car stuck away from the track edge.
- **Aggressive settings were unstable.** Higher braking steering boosts and softer braking caused crashes or acceleration/braking oscillation in experiments. The final version keeps the 1.5× boost and the `speed * 0.25` braking divisor.
- **Simulator setup is external.** This repository does not include the complete TORCS installation or the workshop race configuration.
- **The legacy reconnect path needs attention.** It references `autostart.sh`, which is not included, and attempts to relaunch TORCS with `-nodamage`. Start the simulator before the driver and ensure damage stays enabled during evaluation.

The current submission version can complete laps and has a recorded best result of **1:55.10 with zero damage**. The next improvement would be smoother corner entry while preserving reliable lap completion.

## Video demonstration

**Video link:** [Watch Demo Here](https://drive.google.com/file/d/1IdUUXGDiMnkDWOjEUCVXuNPKLLLMkno1/view?usp=sharing)

The demonstration should show the driver completing a lap and the resulting lap time, top speed, and damage.

## Repository files

| File | Purpose |
| --- | --- |
| `snakeoil3_gym_baseline.py` | Supplied baseline retained for comparison |
| `snakeoil3_gym_workingcopy.py` | Improved driver used for this submission |
| `lap_data.csv` | Recorded development runs and results |
| `README.md` | Project description, setup, results, improvements, and limitations |

## Credits

Developed from the IBM SkillsBuild workshop materials and the original gym-torcs / Snake Oil Python client work.

- [TORCS](http://torcs.sourceforge.net/)
- [Snake Oil Python client — Chris X Edwards](http://xed.ch/project/snakeoil/index.html)
- [vtorcs](https://github.com/giuseppecuccu/vtorcs/tree/nosegfault)
