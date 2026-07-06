# Nav2 Local Planner Benchmark (`nav2_benchmark`)

A ROS 2 package for benchmarking the three local trajectory controllers provided by the
Nav2 navigation stack — **DWB**, **MPPI**, and **Regulated Pure Pursuit (RPP)** — on a
simulated **TurtleBot3 Waffle** in a custom **L-shaped corridor**, under both **static** and
**dynamic** obstacle conditions.

## Overview

The goal of this project is to compare how the three Nav2 local controllers perform when
navigating the same set of start–goal tasks, first with all obstacles stationary and then
with a set of cylindrical obstacles moving along deterministic paths. For each trial the
package records success, time-to-goal, path length, path efficiency, minimum obstacle
clearance, trajectory smoothness (jerk), and the number of close-approach events. These
per-trial results are written to CSV files, one per controller and condition, which are then
used for statistical comparison of the controllers.

The experiment runs entirely in simulation (Gazebo Classic), using a static occupancy map of
the corridor for localization and global planning, with the moving obstacles added on top at
run time for the dynamic condition.

## Environment

- Ubuntu 22.04
- ROS 2 Humble
- Gazebo Classic 11
- Nav2
- TurtleBot3 Waffle (`TURTLEBOT3_MODEL=waffle`)

## Package contents

- `worlds/corridor.world` — the custom L-shaped corridor Gazebo world (walls, desks, cylinders)
- `maps/corridor.yaml` (+ `.pgm`) — the occupancy map used by AMCL and the global costmap
- `config/nav2_dwb.yaml`, `config/nav2_mppi.yaml`, `config/nav2_rpp.yaml` — the per-controller Nav2 parameter files
- `launch/benchmark.launch.py` — brings up Gazebo, the map, and the Nav2 stack with the selected controller
- `scripts/dynamic_obstacles.py` — moves the cylinders and exposes `/obstacles/start` and `/obstacles/stop`
- `scripts/simple_driver.py` — runs the trials and writes the per-trial CSV

## Build

From the workspace root:

```bash
export TURTLEBOT3_MODEL=waffle
colcon build --packages-select nav2_benchmark
source install/setup.bash
```

Set `use_sim_time` to `true` everywhere (the commands below already do this), since the whole
experiment runs against Gazebo simulation time.

## Running the benchmark

Open three terminals and source the workspace (`source install/setup.bash`) in each.
Run the commands in the order below.

### 1. Launch the simulation and Nav2 with the chosen controller

Brings up Gazebo with the corridor world, loads the map, and starts the Nav2 stack using the
selected controller. Change `controller:=` to `dwb`, `mppi`, or `rpp`:

```bash
ros2 launch nav2_benchmark benchmark.launch.py \
  world:=$(ros2 pkg prefix --share nav2_benchmark)/worlds/corridor.world \
  map:=$(ros2 pkg prefix --share nav2_benchmark)/maps/corridor.yaml \
  controller:=mppi
```

Wait until Nav2 is fully active (RViz shows the map and the robot is localized) before
continuing.

### 2. Start the dynamic-obstacle node

Starts the node that controls the moving cylinders. It stays idle until the driver triggers
it, so it is safe to launch it now for both static and dynamic runs:

```bash
ros2 run nav2_benchmark dynamic_obstacles.py --ros-args -p use_sim_time:=true
```

### 3. Run the trials

Executes the start–goal trials and writes the results to a CSV. Set a descriptive
`--run-id` and an `--output` path for each run:

```bash
ros2 run nav2_benchmark simple_driver.py \
  --run-id dwb_corridor_001 \
  --output results/run_rpp2.csv \
  --ros-args -p use_sim_time:=true
```

To benchmark all three controllers, repeat steps 1–3 with a different `controller:=` value in
step 1 and a different `--run-id` / `--output` in step 3, so each controller and condition is
written to its own CSV file.

## Output

Each run produces a CSV where every row is one trial, with columns for the run id, trial
index, start and goal poses, success, timeout, time-to-goal, path length, path efficiency,
minimum clearance, mean jerk, and close-approach count. Collecting one CSV per controller and
condition gives the full data set used for the comparison.