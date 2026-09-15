# ROS 2 Fundamentals: Prep Assignments for ESE 6150 Lab 2 (≈10 hours)

> **How to use this document**
> - **Learner:** do A1 → A5 in order. When stuck, open a new AI chat, paste this whole document, then say which assignment/part you're on, what you ran, what you expected, and the exact output. Try for ~15 minutes on your own first.
> - **AI tutor:** read Section 0 before replying.

---

## 0. Instructions for the AI tutor (read first)

### 0.1 Who the learner is and what they want
- A student in Penn **ESE 6150 (RoboRacer / F1TENTH)** with almost no Linux or ROS 2 background.
- Goal: complete **Lab 2 (Automatic Emergency Braking with iTTC) independently, without relying on AI**. These assignments build Lab 2's core skills in about 10 hours, using *different* problems.
- They want to understand *why*. Good help makes them need you less next time.

### 0.2 How to help
1. **Don't solve the task for them.** No complete code for any task, no whole files. Short, *generic* syntax examples are fine when they use a different topic or message than the task.
2. **Use a hint ladder**, one step at a time:
   1. Ask what they ran, expected and got (the **exact** output), and what was sourced in that terminal.
   2. Name the concept involved and point to the relevant part of this document or the official docs.
   3. Narrow down where the problem is ("look at your `install(...)` rule").
   4. Give a small generic example or analogy.
   5. Give a direct fix **only** for environment or tooling problems (sourcing, git remote, command typos), then explain it.
3. **Make them predict** before running something, and explain afterwards why it matched or didn't.
4. **Tutor notes** (collapsed `<details>` blocks) contain verified expected results. Don't reveal them before the learner has attempted the task, unless they're stuck and ask.
5. **Lab 2 boundary:** if asked about Lab 2 itself (iTTC, range rate, thresholds, false-positive filtering, `safety_node` logic), give **conceptual** explanations only. No code, no specific filter design, no threshold choice. Lab 2 is graded.
6. **Keep things safe:** never suggest `sudo rm`, `git reset --hard`, force-pushing branches, or editing anything under `~/sim_ws`. Suggest `git status` before anything that could discard work.
7. **Encourage the error journal** (`~/fund_ws/notes/error_journal.md`).
8. **Protect the time budget.** If the learner is spending much more than the estimate on one part, help them get unstuck rather than go deeper.

### 0.3 Verified facts about the learner's machine (as of 2026-09-14)
- **OS and tools:** Ubuntu 24.04, ROS 2 **Jazzy**, system Python 3.12 with numpy 1.26. Editors: VS Code (`code`), `nano`. `gh` is **not** installed.
- **Terminals:** `~/.bashrc` does **not** source ROS, so every new terminal starts with no ROS environment.
- **Git:** `user.email` is set globally, `user.name` is **not**. The learner has pushed a course repo over HTTPS before.
- **Simulator workspace:** `~/sim_ws` (`src/f1tenth_gym_ros`, dev-jazzy, with venv `~/sim_ws/.venv`). Config `src/f1tenth_gym_ros/config/sim.yaml`: `kb_teleop: True`, map `maps/levine`, 1 agent, start pose `(-12, 0, 0)`.
- **Lab 2 repo:** `~/sim_ws/src/ese-6150-lab-2-automatic-emergency-braking-jacobbau04`. Its `CMakeLists.txt` is **already fixed** (C++ target removed, `RENAME safety_node` added). Don't change it.
- **Unrelated folders:** `~/roboracer_ws` and `~/f1tenth_gym_ros_old_main` aren't used. Don't source them.
- **Topics:**
  - `/scan`: `sensor_msgs/msg/LaserScan`
  - `/ego_racecar/odom`: `nav_msgs/msg/Odometry` (forward speed is `twist.twist.linear.x`)
  - `/drive`: `ackermann_msgs/msg/AckermannDriveStamped`
  - `/cmd_vel`: `geometry_msgs/msg/Twist`, from `teleop_twist_keyboard`
- **Bridge behaviour:** `/drive` and `/cmd_vel` overwrite the *same* requested speed and steering. The **most recent message wins** and **stays in effect** until another arrives. A `/drive` message also sets steering (0 if left blank).
- **Scan:** 819 beams, `angle_min` −135° (−2.35619 rad), `angle_max` +135°. `angle_increment = (max − min)/(n − 1)` = **0.0057609 rad (0.33007°)**. Beam **409 = 0°** (straight ahead); nearest +90° (left) is **682**; nearest −90° is **136**. `range_min` 0.05 m, `range_max` 25 m, noise σ 0.01 m.
- **Levine hallway at the start pose:** walls at about **y = +0.70 m (left)** and **y = −1.00 m (right)**, ±5 cm.
- **Collision log:** hitting a wall makes the bridge log a warning containing `ego_racecar hit something`.

### 0.4 Terminal recipes (referred to as T‑SIM, T‑TELEOP, T‑FUND)
```bash
# T-SIM: simulator (Foxglove opens in the browser; import the layout
# ~/sim_ws/install/f1tenth_gym_ros/share/f1tenth_gym_ros/launch/gym_bridge_foxglove.json)
cd ~/sim_ws
source .venv/bin/activate
source /opt/ros/jazzy/setup.bash
source install/local_setup.bash
ros2 launch f1tenth_gym_ros gym_bridge_launch.py

# T-TELEOP: keyboard driving (click this terminal before pressing keys)
source /opt/ros/jazzy/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard

# T-FUND: your workspace (don't activate the sim venv or source sim_ws here)
source /opt/ros/jazzy/setup.bash
source ~/fund_ws/install/setup.bash     # exists only after your first colcon build
```
> Nodes in different workspaces still talk to each other: topics travel over the network layer (DDS), not through the workspace.

---

## 1. Workspace: one workspace, `~/fund_ws`

Use **one workspace, `~/fund_ws`**, which is also one git repo.
- **It matches real ROS use:** one workspace holds many packages (`sim_ws` already holds two).
- **A5 builds on A3:** it adds a node to the package from A3.
- **One overlay to source,** which avoids the most common beginner mistake.
- **A1 and A4 don't need a ROS build;** they live in plain folders that `colcon` ignores, because only folders with a `package.xml` are built.

```
~/fund_ws/                    ← workspace root AND git repository
├── .gitignore                ← build/ install/ log/ __pycache__/
├── fund_assignments.md       ← this document (moved here in A1)
├── notes/                    ← answers + error_journal.md
├── offline/                  ← A4 NumPy scripts (no ROS)
└── src/
    └── fund_pkg/             ← your ROS 2 package (A3 nodes + A5 node)
```

---

## 2. Time budget (≈10 h total)

| Assignment | Most important subjects | Time |
|---|---|---|
| **A1** Terminal, environment and Git | Paths, `source` and environment variables, commit/push/tag | **1.5 h** |
| **A2** ROS 2 detective (no coding) | Nodes, topics, message fields, CLI tools, persistent commands, last message wins | **1.5 h** |
| **A3** Build a package from scratch | Workspace, `colcon`, `package.xml`, CMake `RENAME`, Python nodes, callbacks and state, parameters, common build errors | **3 h** |
| **A4** NumPy and geometry (no ROS) | Beam index ↔ angle, radians, hit points, masks, `inf`/`nan`, empty arrays | **1.5 h** |
| **A5** Capstone: Wall Reporter | Everything combined against the live sim, verification, false triggers | **2.5 h** |
| **Total** | | **≈ 10 h** |

Suggested pace: one assignment per session, over about a week.

**Where Lab 2 uses each subject:**
- **Terminals and sourcing:** running the sim, teleop and your node at the same time (A1, A2).
- **Tag submission:** how Lab 2 is graded (A1).
- **Last message wins:** why a stop command can be overridden (A2, A5).
- **Correct executable name and dependencies:** worth 30 points (A3).
- **State shared between callbacks:** speed arrives from odom, decisions happen on scans (A3, A5).
- **Beam angles and bad readings:** the core of the per-beam math (A4, A5).
- **Checking for false triggers:** the video deliverable (A5).

---

## A1: Terminal, Environment and Git (1.5 h)

### Intent
Most beginner "ROS doesn't work" problems are really Linux problems: wrong folder, environment not sourced, or work never pushed. The key idea is **Part B: why `source` exists**, because it explains why every ROS terminal needs setup.

### Resources
Ubuntu tutorial "The Linux command line for beginners"; `<command> --help`; Pro Git book chapter 2 (git-scm.com/book).

### Part A: Paths and files (20 min)
1. Run `pwd`, `ls`, `ls -la`. What are `.` and `..`?
2. Create the workspace in one command: `mkdir -p ~/fund_ws/{notes,offline,src}`, and check it with `ls ~/fund_ws`.
3. Move this document to `~/fund_ws/fund_assignments.md` with `mv`.
4. From `~/fund_ws/src`, `cd` to `~/fund_ws/notes` once with an absolute path and once with a relative path (`..`).
5. Use `grep map_path ~/sim_ws/src/f1tenth_gym_ros/config/sim.yaml` to find a line without opening the file. **Don't edit `sim_ws`.**

### Part B: Environment variables and `source` (30 min) ★
1. In terminal 1: `export FUND_MSG="hello"` then `echo $FUND_MSG`. Open **terminal 2** and `echo $FUND_MSG`. Explain the result.
2. Create `~/fund_ws/notes/setvar.sh` containing `export FUND_MSG="from file"`. In a **fresh** terminal:
   - `bash ~/fund_ws/notes/setvar.sh`, then `echo $FUND_MSG`
   - `source ~/fund_ws/notes/setvar.sh`, then `echo $FUND_MSG`

   Why do they differ? (Key words: *child process* vs. *current shell*.)
3. In a fresh terminal, run `which ros2`. Then `source /opt/ros/jazzy/setup.bash` and run `which ros2` and `printenv | grep ROS_DISTRO`. What changed, and why?

### Part C: Git and tags (40 min)
1. Run `git config --global user.name "Your Name"`.
2. In `~/fund_ws`: `git init`, then create `.gitignore` containing `build/`, `install/`, `log/` and `__pycache__/`.
3. Run `git status` → `git add .` → `git status` → `git commit -m "Initial workspace"`. What does "staged" mean?
4. On github.com, create a **private, empty** repo `fund_ws`. Add it as `origin` and push. (If your branch is `master`, run `git branch -M main` first.)
5. **Tags, exactly how Lab 2 is submitted:**
   1. Run `git tag checkpoint` and `git push origin checkpoint`, then find the tag on GitHub.
   2. Make a new commit and push it, then run `git tag -f checkpoint` and `git push --force origin checkpoint`.
   3. Confirm on GitHub that the tag moved.

   Why doesn't a plain `git push` send tags?

### Deliverables and checkpoint questions
Create `notes/a1_answers.md` and answer, then commit and push:
1. Why did `bash setvar.sh` not keep the variable, while `source setvar.sh` did?
2. A new terminal says `ros2: command not found`. What's wrong, and what's the fix?
3. What's the difference between pushing commits and pushing a tag?

Also start `notes/error_journal.md` using the template in the Appendix.

**Done when** you can open a fresh terminal, source ROS, commit, and move and push a tag without looking anything up.

<details>
<summary><b>Tutor notes: A1</b></summary>

- **Part B1:** empty output in terminal 2. Variables exist only in the shell (and its children) where they were exported.
- **Part B2:**
  - `bash` runs the file in a child process whose variables vanish when it exits, so the output is empty.
  - `source` runs it in the *current* shell, so the output is `from file`.
  - `source /opt/ros/jazzy/setup.bash` relies on exactly this.
- **Part B3:** before sourcing, `which ros2` prints nothing. After sourcing, it prints `/opt/ros/jazzy/bin/ros2` and `ROS_DISTRO=jazzy` appears.
- **Part C4:** `gh` isn't installed, so create the repo on the website. If push asks for a password, GitHub needs a token or credential manager. They've pushed over HTTPS before, so an existing helper probably works.
- **Part C5:** `git push` sends branch commits only; tags need `git push origin <tag>`. Moving a tag that's already on the remote needs `--force` on that tag push only. That's the Lab 2 submission flow.
</details>

---

## A2: ROS 2 Detective, with No Coding (1.5 h)

### Intent
You can't debug a node until you can *see* the system. Using only CLI tools and Foxglove, discover the facts Lab 2 depends on:
- what's in a `LaserScan`
- where speed lives in `Odometry`
- that `/drive` commands persist
- that two publishers fight over the car

### Resources
ROS 2 Jazzy docs, *Beginner: CLI tools* → "Understanding nodes", "Understanding topics"; `ros2 topic echo --help`.

### Setup
Open **T‑SIM** (load the Foxglove layout), **T‑TELEOP**, and a third terminal with only `/opt/ros/jazzy/setup.bash` sourced. Record answers in `notes/a2_answers.md`.

### Part 1: What's running, and what's in the messages? (30 min)
1. Run `ros2 node list` and `ros2 topic list -t`.
2. Run `ros2 topic info` on `/scan`, `/ego_racecar/odom`, `/drive` and `/cmd_vel`. Record each type and the publisher and subscriber counts.
3. Sketch a graph with nodes as boxes and topics as arrows from publisher to subscriber, covering teleop, the bridge, `/cmd_vel`, `/drive`, `/scan` and odom.
4. Run `ros2 interface show sensor_msgs/msg/LaserScan`. Explain `angle_min`, `angle_increment`, `range_min`, `range_max` and `ranges` in your own words. What units are the angles in?
5. Run `ros2 interface show nav_msgs/msg/Odometry` and write the full dotted path to forward speed.

### Part 2: Live data (25 min)
1. Run `ros2 topic echo /scan --once --no-arr` and record the angle and range fields.
2. **By hand:** how many beams are there, which index points straight ahead, and which is closest to +90° (left)? Watch out for `n` vs. `n − 1`.
3. Run `ros2 topic echo /ego_racecar/odom --field twist.twist.linear.x`. In teleop, tap `i`, then `k`, then `,`, and record what you see.
4. In Foxglove, use the 3D panel's ruler tool to measure the distance from the car to the left and right walls at the start pose.

### Part 3: Commands persist, and the last message wins (35 min) ★
1. Stop teleop. **Predict**, then run:
   `ros2 topic pub --once /drive ackermann_msgs/msg/AckermannDriveStamped "{drive: {speed: 1.0}}"`
   Does the car move for an instant, or keep going? What does T‑SIM log when it hits the wall?
2. Publish `speed: 0.0` once, then reset the pose with Foxglove's pose tool. Why do the slides say to stop *before* resetting?
3. Restart teleop. Publish zeros **repeatedly** with `ros2 topic pub -r 10 /drive … "{drive: {speed: 0.0}}"`. While that runs, **tap** `i`, then **hold** `i`, and describe each.
4. Stop the repeating publisher and tap `i` once. What happens?
5. **Think, don't code:** is publishing a stop command *once* enough to keep a car stopped? Why?

### Checkpoint questions
1. How do you find a topic's type and that type's fields using only the CLI?
2. Given `angle_min`, `angle_increment` and index `i`, what's the beam's angle? Which index is straight ahead here?
3. Two nodes publish different speeds to the car. Which one wins, and for how long?

**Done when** you can find any topic's type, fields, publishers and live values in under two minutes, and explain Part 3.

<details>
<summary><b>Tutor notes: A2</b></summary>

- **Part 1:**
  - The bridge node name comes from the launch file (probably `/bridge`); trust `ros2 node list`.
  - `/scan` has one publisher, the bridge. `/cmd_vel` is subscribed by the bridge and published by teleop. `/drive` is subscribed by the bridge.
  - Angles are in **radians**. Speed is at `twist.twist.linear.x`.
- **Part 2:**
  - Scan fields: `angle_min` ≈ −2.35619, `angle_increment` ≈ 0.0057609, `range_min` 0.05, `range_max` 25.0.
  - Beam count: `n = (max − min)/inc + 1 = 819` (819 beams have 818 gaps). Straight ahead is **409** (exactly 0°); nearest +90° is **682**.
  - Teleop: `i` → about +0.5 m/s (ramps up), `k` → 0, `,` → about −0.5.
  - Walls at the start pose: about **0.70 m left**, **1.00 m right**.
- **Part 3:**
  1. The car keeps driving indefinitely, and the log says `ego_racecar hit something`.
  2. Resetting the pose doesn't clear the requested speed.
  3. Tapping: zeros override each keypress within ≤0.1 s, so the car barely creeps. Holding: auto-repeat alternates with the zeros, so the car jerks forward.
  4. It moves and keeps moving.
  5. Once isn't enough: any later message overrides it, so a stop must be re-sent while the danger lasts. Don't go further into Lab 2 design.
</details>

---

## A3: Build a ROS 2 Package from Scratch (3 h)

### Intent
In Lab 2 the build system can silently run the wrong code (the skeleton's `safety_node` executable was an empty C++ stub). Build a package from an empty folder so you understand `package.xml` and `CMakeLists.txt`, then break it deliberately so the common errors become familiar.

The listener practices Lab 2's node *pattern* on a harmless problem:
- two subscriptions
- state shared between callbacks
- a parameter
- a conditional, repeated publish

### Resources
ROS 2 Jazzy docs, *Beginner: Client libraries*: "Using colcon to build packages", "Creating a package", "Writing a simple publisher and subscriber (Python)", "Using parameters in a class (Python)". CMake `install()` docs.
**Note:** the tutorials use `ament_python` with `setup.py`. The course uses **`ament_cmake`** with Python scripts installed by CMake, so do it the course way.

### Part A: Create and build (30 min)
1. In `~/fund_ws/src`, create package `fund_pkg` (build type `ament_cmake`, dependencies `rclpy` and `std_msgs`). Read `ros2 pkg create --help` for the argument order.
2. Open `package.xml` and `CMakeLists.txt`, and note in one line each what the main blocks do.
3. Create `fund_pkg/scripts/`. From **`~/fund_ws`** (the root), run `colcon build`. Which folders appeared, and which one do you edit?

### Part B: Two nodes (1 h 15 min)
Each script needs a shebang and the structure `main()` → `rclpy.init` → node → `spin` → cleanup.

- **`counter_talker`** (`scripts/counter_talker.py`): a timer every 0.5 s publishes an increasing `std_msgs/msg/Int32` on `/fund/count`.
- **`sum_listener`** (`scripts/sum_listener.py`):
  - **Subscription 1:** `/fund/scale` (`std_msgs/msg/Float32`). Store the latest value in `self.scale` (start at 1.0).
  - **Subscription 2:** `/fund/count` (`Int32`). Add `count × self.scale` to `self.total` and log it.
  - **Parameter:** `threshold` (default 50.0).
  - **Alert:** on **every** count message while `total > threshold`, publish `std_msgs/msg/Bool` `True` on `/fund/alert`.
- **`CMakeLists.txt`:** install the scripts so the executables are named `counter_talker` and `sum_listener`, with **no `.py`**.

Build with `colcon build --packages-select fund_pkg`, source the overlay, and confirm with `ros2 pkg executables fund_pkg`. Run each node in its own T‑FUND terminal.

### Part C: Interact (30 min)
1. Confirm the wiring with `ros2 topic echo /fund/count` and `ros2 node info /sum_listener`.
2. **Predict** how the total changes, then run:
   `ros2 topic pub --once /fund/scale std_msgs/msg/Float32 "{data: 2.0}"`
3. Run `ros2 run fund_pkg sum_listener --ros-args -p threshold:=200.0`. While it runs, try `ros2 param set /sum_listener threshold 500.0`. Does the change take effect? It depends on whether your code reads the parameter in `__init__` or in the callback, so explain which yours does.

### Part D: Break it on purpose (45 min) ★
For each experiment:
- **predict** the result,
- run it and paste the **exact** output,
- explain the cause,
- undo the change and confirm it works again,
- add any new error to the journal.

| # | Break it | Observe with |
|---|---|---|
| 1 | Remove `RENAME` from one install rule and rebuild | `ros2 pkg executables fund_pkg`, `ros2 run fund_pkg counter_talker` |
| 2 | Put `RENAME` back and rebuild **without** cleaning. Then delete `build/fund_pkg` and `install/fund_pkg`, and rebuild | `ros2 pkg executables fund_pkg` after each |
| 3 | Edit a log message but **don't rebuild** | `ros2 run …` |
| 4 | Misspell a method (e.g. `get_loger`) and rebuild | Did the build succeed? Then `ros2 run …` |
| 5 | Open a **new** terminal and source only `/opt/ros/jazzy/setup.bash` | `ros2 run fund_pkg counter_talker` |
| 6 | Import `from sensor_msgs.msg import LaserScan` without declaring it, and rebuild | Does it build and run? Why is it still a bug? |
| 7 | Misspell the topic in the listener only | Any error? Diagnose with `ros2 topic list` and `ros2 topic info` |

### Checkpoint questions (in `notes/a3_answers.md`)
1. Why must you rebuild after editing a Python script in this package?
2. Which line in `CMakeLists.txt` decides the executable name?
3. Why can a missing `package.xml` dependency work on your laptop but fail on a grader's machine?
4. How does the count callback use a value that arrived in the scale callback? Why not a local variable?
5. A subscriber never receives anything. Which CLI commands would you use to find out why?

**Done when** you can recreate a working two-node `ament_cmake` Python package from an empty folder (docs only for syntax) and explain its `CMakeLists.txt`.

<details>
<summary><b>Tutor notes: A3</b> (break-it results verified on this machine with a throwaway package)</summary>

- **Part A1 trap:** `ros2 pkg create --build-type ament_cmake --dependencies rclpy std_msgs fund_pkg` **fails** with `the following arguments are required: package_name`, because `--dependencies` swallows every word after it. Put the name first: `ros2 pkg create fund_pkg --build-type ament_cmake --dependencies rclpy std_msgs`.
- **Part A3:** only `src/` is edited; `build/`, `install/` and `log/` are generated.
- **Part B CMake:**
  - `RENAME` works only with a **single** file per `install()` call, so they need **one `install(PROGRAMS …)` per script**. Listing two files fails at configure time with `install PROGRAMS given RENAME option with more than one file.` (verified).
  - `find_package(ament_cmake_python)` is not needed just to install scripts.
- **Part C2:** after scale 2.0, each count adds twice as much.
- **Part C3:** `param set` changes the stored value, but a copy read once in `__init__` stays old. Reading it in the callback makes it live.
- **Break-it results (verified):**
  1. `ros2 pkg executables` shows `fund_pkg counter_talker.py`; `ros2 run fund_pkg counter_talker` → **`No executable found`**.
  2. Without cleaning, **both** `counter_talker` and `counter_talker.py` are listed, because colcon doesn't remove old installed files. After deleting `build/fund_pkg` and `install/fund_pkg`, only the correct name remains.
  3. The **old** text still prints, because the installed file is a copy made at build time.
  4. **Build succeeds** (Python isn't compiled). At runtime: `AttributeError: 'Node' object has no attribute 'get_loger'. Did you mean: 'get_logger'?`, then `[ros2run]: Process exited with failure 1`.
  5. **`Package 'fund_pkg' not found`**. Only the underlay is sourced, not `~/fund_ws/install/setup.bash`.
  6. It builds and runs locally, because `sensor_msgs` is installed system-wide. It's still a bug: clean grader containers and `rosdep` rely on `package.xml`. Lab 2 requires declared dependencies.
  7. No error; the listener is just silent. `ros2 topic list` shows both names, and `ros2 topic info <typo>` shows 0 publishers.
- **Other verified facts, if they come up:**
  - A missing shebang gives `OSError: [Errno 8] Exec format error`.
  - A source script without `+x` still runs, because `install(PROGRAMS)` makes the installed copy executable.
  - `colcon build` run inside `src/` creates `build/`, `install/` and `log/` in the wrong place.
</details>

---

## A4: NumPy and Geometry, Offline (1.5 h)

### Intent
Lab 2's hardest bugs are math and data bugs: the wrong angle for an index, degrees vs. radians, or a single `nan` or `inf` breaking a result. Practice them **without ROS**, checking every answer by hand, so in A5 you know whether a bug is in the math or in ROS.

### Resources
NumPy "absolute beginners" guide; NumPy docs for `isfinite`, `nanmin`, `argmin`, `deg2rad`; ROS REP 103 (units and coordinate conventions: +x forward, +y left, angles counter-clockwise, radians).

### Setup
Work in `~/fund_ws/offline/` using plain `python3` scripts with **no ROS imports**. Do each task **by hand first**, then in code, and compare.

### Part 1: Beam index ↔ angle (30 min)
Use the sim's values: `n = 819`, `angle_min = −135°`, `angle_max = +135°`, `angle_increment = (max − min)/(n − 1)`.
1. Compute `angle_increment` in radians and degrees.
2. Build an array of all 819 angles in radians with `np.arange`, with no `for` loop.
3. What angle, in degrees, is index 0? 409? 818?
4. Write `index_for_angle(deg)` returning the nearest index, and test it on +90° and −90°.
5. Build a boolean mask for beams within ±10° of straight ahead. How many beams is that?

### Part 2: Geometry (25 min)
1. A beam at +30° measures `r = 2.0 m`. Compute the hit point `(x, y)`.
2. The car drives straight with a wall on its left at y = 0.70 m. For the beam at +45°: what's `r`, and how far ahead (`x`) and to the side (`y`) is the hit point?
3. The car moves forward at `v = 2.0 m/s`. What's the velocity component along directions at 0°, 90° and 180°? What does a negative value mean?

### Part 3: Messy data (35 min) ★
Use `ranges = np.array([1.2, np.inf, 0.03, np.nan, 30.0, 0.8])`, `range_min = 0.05`, `range_max = 25.0`.
1. **Predict, then run:** `np.min(ranges)`, `np.nanmin(ranges)`, `np.nan < 1`, `np.nan > 1`. What surprised you?
2. Build a `valid` mask: finite **and** within `[range_min, range_max]`. What's the smallest valid range, and at which **original** index?
3. What do Python `1/0` and NumPy `np.float64(1)/np.float64(0)` each do?
4. What happens with `np.min(ranges[ranges < 0])`? Find a safe way to handle "nothing matched" (hint: `initial=`).

### Checkpoint questions (in `notes/a4_answers.md`)
1. Why does `angle_increment` use `n − 1`?
2. For range `r` at angle θ, what are `x` and `y`? Which one means "how far to the side"?
3. Why is a `nan` in an array dangerous for code that reacts when a value is below a threshold?

**Done when** you can compute beam angles and the closest *valid* reading, by hand and in vectorised NumPy, without crashing on `inf`, `nan` or empty selections.

<details>
<summary><b>Tutor notes: A4</b> (computed with numpy on this machine)</summary>

- **Part 1:**
  - `angle_increment` = 0.0057609 rad = 0.33007°.
  - Index 0 → −135°, 409 → **0°**, 818 → +135°.
  - Nearest index for +90° → **682** (90.11°); for −90° → **136**. Formula: `round((θ − angle_min)/inc)`.
  - ±10° → **61 beams** (indices 379–439).
  - A common bug: `np.cos(60)` treats 60 as radians and gives ≈ −0.952 instead of 0.5.
- **Part 2:**
  1. (1.732, 1.000).
  2. `r = 0.70/sin 45° = 0.990`, `x = r cos θ = 0.700`, `y = 0.700`. **Don't** connect this to Lab 2 filtering; let the learner draw conclusions later.
  3. `v cos θ`: 0° → 2.0, 90° → 0.0, 180° → −2.0. Negative means moving away from that direction. **Don't** go into range rate or TTC sign conventions.
- **Part 3:**
  1. `np.min` → **nan** (it propagates). `np.nanmin` → **0.03**, which is still invalid because it's below `range_min`. `nan < 1` and `nan > 1` are both False, so threshold checks silently fail.
  2. Valid indices are 0 and 5; the minimum valid range is **0.8** at original index **5**. (`argmin` on the filtered array indexes the *filtered* array.)
  3. Python `1/0` → `ZeroDivisionError`. NumPy gives `inf` with `RuntimeWarning: divide by zero encountered in scalar divide`.
  4. `ValueError: zero-size array to reduction operation minimum which has no identity`. Fix: `np.min(x, initial=np.inf)`, or check `x.size > 0`.
</details>

---

## A5: Capstone, a Wall Reporter and Speed Governor (2.5 h)

### Intent
Combine everything in a node that runs against the live simulator:
- subscribe to scan and odom
- keep state between callbacks
- turn beams into angles
- handle bad data
- use a parameter
- publish to `/drive` in competition with teleop
- **verify** that it triggers when it should and not when it shouldn't

> **This is NOT Lab 2 and must not be reused for it.** It uses a simple **distance** rule, not Lab 2's **time-to-collision** method. The reflection questions explore why distance alone is limited, and Lab 2 is where you design the better answer.

### Specification: node `wall_reporter` in `fund_pkg`
File `scripts/wall_reporter.py`, executable `wall_reporter`.
- **Subscribes:** `/scan` and `/ego_racecar/odom`. Store the latest forward speed.
- **Parameters:** `slow_distance` (default 2.0 m) and `slow_speed` (default 0.5 m/s).
- **On every scan:**
  1. Compute beam angles from the message fields. **Don't hard-code 819 or 409.**
  2. Ignore readings that are `inf`, `nan`, or outside `[range_min, range_max]`.
  3. **Front distance:** the smallest valid range within ±10° of straight ahead (infinite if none).
  4. **Closest obstacle:** the smallest valid range across all beams, and its angle in degrees.
  5. Log the front distance, closest distance and angle, and speed, at most twice per second (`throttle_duration_sec=0.5`).
  6. **Governor:** if `front distance < slow_distance` **and** `speed > slow_speed`, publish `AckermannDriveStamped` with `speed = slow_speed` on `/drive`.
- **Package:** add the install rule, and declare `sensor_msgs`, `nav_msgs` and `ackermann_msgs` in `package.xml` and `CMakeLists.txt`.

### Part A: Build and wire-check (50 min)
1. Implement, build and source the node, and confirm the executable name.
2. Run T‑SIM and T‑TELEOP, then `wall_reporter` in T‑FUND (only ROS and `fund_ws` sourced).
3. Prove with `ros2 node info /wall_reporter` that it subscribes to `/scan` and `/ego_racecar/odom` and publishes `/drive`.

### Part B: Verify the numbers (40 min)
Check against an **independent** measurement; don't use your code to check itself.
1. At the start pose, **predict** the closest obstacle's distance and angle using A2 Part 2.4 and A4 Part 1. Compare with your log. Is the sign of the angle right?
2. Use Foxglove's pose tool to face the car at a wall, measure the distance with the ruler, and compare it with your front distance.

### Part C: Test the governor (40 min)
**Before testing**, write each case's *expected* result in `notes/a5_answers.md`, then record what actually happened.

| Case | Setup | Expected |
|---|---|---|
| 1 | Toward a wall, slower than `slow_speed` | *(you predict)* |
| 2 | Toward a wall, faster (raise teleop speed with `w`) | *(you predict)* |
| 3 | Fast, straight down the Levine hallway | *(should it trigger? does it?)* |
| 4 | Governor active: **tap** vs. **hold** a teleop key | *(use A2 Part 3)* |

### Part D: Reflection (20 min) ★
1. **False positive vs. false negative:** define both for this governor, and give an example of each.
2. At 0.6 m/s, is `slow_distance = 2.0 m` cautious or too late? At 5 m/s? What does this say about using **distance alone** to judge danger?
3. Case 3 used a ±10° cone. What would happen with ±60°, and why?

### Checkpoint questions
1. Walk through what happens, in order, from a `LaserScan` arriving to your node publishing (or not).
2. Why didn't `wall_reporter`'s terminal need `sim_ws` sourced?
3. How did you make sure `ros2 run fund_pkg wall_reporter` ran your latest code?

**Done when** all four cases have a prediction, a result and an explanation, and your numbers match independent measurements. Commit, push, and tag `a5-done`.

<details>
<summary><b>Tutor notes: A5</b></summary>

- **Part A:** needs a separate `install(PROGRAMS scripts/wall_reporter.py DESTINATION lib/${PROJECT_NAME} RENAME wall_reporter)`, plus `<depend>` entries for the three message packages. System numpy works in T‑FUND with no venv. `throttle_duration_sec` is supported by this rclpy.
- **Part B1:** at the start pose, expect about **0.70 m at about +90°** (left wall). The front distance is large and depends on the map; readings beyond `range_max` must be handled.
- **Part C:**
  - **Case 1:** no publish.
  - **Case 2:** the speed is capped once within `slow_distance`.
  - **Case 3:** with ±10° the side walls stay outside the cone while driving straight, so there's normally no trigger.
  - **Case 4:** tapping lets the cap win quickly; holding makes the speed jitter as auto-repeat and the governor alternate.
  - A likely extra observation: the governor's `/drive` has `steering_angle = 0`, so the car straightens whenever it publishes.
- **Part D:** guide with questions only. The learner should notice that danger depends on speed as well as distance, and that a wide cone pulls in side walls. **Don't** introduce the iTTC formula, range-rate sign conventions, lateral filtering designs, or threshold values.
</details>

---

## Ready for Lab 2 when you can, without help:
1. Create, build and run a Python `ament_cmake` package with correct executable names and declared dependencies.
2. Prove with CLI tools that a node receives `/scan` and `/ego_racecar/odom` and publishes `/drive`.
3. Calculate a beam's angle and the nearest valid range by hand, ignoring `inf`/`nan`.
4. Explain why a single stop command can be overridden.
5. Commit, push, and submit with a moved git tag.

---

## Appendix: Error journal template
```markdown
## <short name of the error>
- **Assignment / part:** 
- **Terminal & what was sourced:** 
- **Command I ran:** 
- **Exact error / behaviour:** 
- **What I predicted:** 
- **Actual cause:** 
- **Fix:** 
```
