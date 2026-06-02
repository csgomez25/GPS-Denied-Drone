# Week-1 Simulation Setup Checklist

> Goal for the week: a simulated X500 takes off and flies a waypoint **autonomously via offboard control**, with a depth camera streaming into ROS 2, and an occupancy map building from that depth. No hardware, no crash risk.
>
> Companion doc: [BUILD.md](./BUILD.md). Target stack: PX4 SITL + Gazebo + ROS 2 **Jazzy** + Isaac ROS (cuVSLAM/nvblox).
>
> ⚠️ **These are starting-point commands.** Package names, repo paths, and versions drift — cross-check each against the official PX4 / ROS 2 / Isaac ROS docs (linked at the bottom) as you go. Don't paste blindly.

---

## Day 0 — Machine check & the one branch that decides your week

Run these and write down the answers:

```bash
lsb_release -a            # must be Ubuntu 24.04 (Noble) for Jazzy tier-1
nvidia-smi                # NVIDIA GPU present? note the answer
df -h ~                   # need ~40+ GB free (PX4 + ROS + Gazebo + containers)
nproc && free -h          # SITL + Gazebo + RViz is heavy; 8+ cores / 16+ GB ideal
```

**Confirmed: dev box has an NVIDIA RTX A3000 (Ampere, CUDA) → you are on the FULL GPU path.** cuVSLAM + nvblox run locally in sim. Notes for the A3000:

- Isaac ROS runs in **Docker** + the **NVIDIA Container Toolkit** — install that on Day 5 before pulling the Isaac ROS container.
- **VRAM watch:** the A3000 laptop GPU has 6 GB (some SKUs 12 GB). Gazebo rendering + cuVSLAM + nvblox together can pressure 6 GB. If you hit OOM, drop the sim camera resolution and the nvblox voxel resolution (e.g. 5 cm → 10 cm). Not a blocker, just a knob.
- Ampere (compute capability 8.6) is well within Isaac ROS's supported range.

*(CPU-fallback path removed — not needed with the A3000.)*

---

## Day 1 — ROS 2 Jazzy

```bash
# Locale
sudo apt update && sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

# Enable universe + add the ROS 2 apt source
sudo apt install -y software-properties-common curl
sudo add-apt-repository universe
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt install -y ros-jazzy-desktop ros-dev-tools
```

Add to `~/.bashrc` (or source per-shell):
```bash
source /opt/ros/jazzy/setup.bash
```

**✅ Success check:** in two terminals, `ros2 run demo_nodes_cpp talker` and `ros2 run demo_nodes_py listener` exchange messages.

---

## Day 2 — PX4 SITL + Gazebo, fly it manually first

```bash
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
bash ./PX4-Autopilot/Tools/setup/ubuntu.sh      # installs sim deps; reboot after
cd PX4-Autopilot
make px4_sitl gz_x500                            # plain X500 in Gazebo
```

Install **QGroundControl** (AppImage) separately and launch it — it should auto-connect to SITL on UDP.

**✅ Success check:** in QGroundControl, arm and command takeoff; the X500 lifts off in Gazebo. (Or in the `pxh>` console: `commander takeoff`.) This proves SITL + Gazebo + GCS before any ROS or autonomy.

---

## Day 3 — Bridge PX4 ↔ ROS 2 and fly an offboard waypoint

**3a. Micro XRCE-DDS Agent** (the PX4↔ROS 2 link):
```bash
git clone https://github.com/eProsima/Micro-XRCE-DDS-Agent.git
cd Micro-XRCE-DDS-Agent && mkdir build && cd build
cmake .. && make && sudo make install && sudo ldconfig /usr/local/lib/
MicroXRCEAgent udp4 -p 8888                       # leave running
```
Restart SITL (`make px4_sitl gz_x500`); the PX4 client connects to the agent automatically.

**3b. ROS 2 messages + examples:**
```bash
mkdir -p ~/ws_px4/src && cd ~/ws_px4/src
git clone https://github.com/PX4/px4_msgs.git
git clone https://github.com/PX4/px4_ros_com.git
cd ~/ws_px4
source /opt/ros/jazzy/setup.bash
colcon build && source install/setup.bash
```

**✅ Success check (incremental):**
1. `ros2 topic list` shows `/fmu/out/...` topics → bridge works.
2. Run the offboard control example from `px4_ros_com` (or the PX4 ROS 2 offboard tutorial node) → the vehicle arms, switches to Offboard mode, and holds/flies a setpoint **commanded from ROS 2**.

This is the single most important milestone of the week: **ROS 2 is now flying the drone.** Everything else feeds setpoints into this.

---

## Day 4 — Depth camera into ROS 2

PX4 ships an X500 variant with a depth camera:
```bash
cd PX4-Autopilot
make px4_sitl gz_x500_depth                        # X500 + simulated depth camera
```

Bridge Gazebo's camera topics into ROS 2 (Jazzy pairs with Gazebo Harmonic via `ros_gz`):
```bash
sudo apt install -y ros-jazzy-ros-gz
# Bridge the depth image / camera_info / point cloud topics (names from `gz topic -l`)
ros2 run ros_gz_image image_bridge /depth_camera
ros2 run ros_gz_bridge parameter_bridge /camera_info@sensor_msgs/msg/CameraInfo@gz.msgs.CameraInfo
```

**✅ Success check:** in RViz2, visualize the depth image / point cloud and confirm it updates as the drone moves and as you place an object in the Gazebo world.

---

## Day 5 — Occupancy map from depth

**GPU path (Isaac ROS nvblox):** run via NVIDIA's `isaac_ros_common` Docker container (simplest, avoids host dependency hell). Feed it the bridged depth + camera_info + a pose source.
```bash
# Use the Isaac ROS Docker container per NVIDIA's "Getting Started" + nvblox quickstart.
# Inside the container, launch nvblox subscribed to the sim depth topics.
```

Configure nvblox to also publish a **2D ESDF/distance-map slice at flight altitude** (`nav_msgs/OccupancyGrid` or `nvblox_msgs/DistanceMapSlice`) — that 2D slice is what the Day-7 planner consumes.

**✅ Success check:** the 3D voxel map *and* a 2D slice appear in RViz, and an obstacle you drop into the Gazebo world shows up as occupied/high-cost cells.

---

## Day 6 — VIO with cuVSLAM

Launch **Isaac ROS cuVSLAM** on the sim stereo/depth + IMU; confirm it publishes an odometry estimate (`/visual_slam/tracking/odometry` or via TF). Compare its track against PX4 sim ground-truth pose over a 30–60 s flight and write down a first **drift number** (e.g. cm of position error after a loop). This is your baseline before the sensor ever flies.

> Wiring note: cuVSLAM's odometry is what feeds PX4 EKF2 on hardware (via `VehicleVisualOdometry`/`VISION_POSITION_ESTIMATE`). In **sim**, PX4 already has perfect state, so don't fight it — let SITL use its own state for flight control, and run cuVSLAM **in parallel** purely to validate the VIO pipeline and quantify drift. The hardware swap (cuVSLAM → EKF2) comes later on the bench.

**✅ Success check:** cuVSLAM publishes continuous odometry and you have a drift number vs. ground truth.

---

## Day 7 — Close the loop (detailed)

### Architecture — three small nodes plus nvblox

```
                 nvblox (Day 5)                          your code
 depth ─▶ ┌──────────────────────┐   /map (2D slice)  ┌───────────────┐  waypoints  ┌────────────────────┐
          │ nvblox: 3D voxels +   │ ──OccupancyGrid──▶ │ planner_node  │ ──Path────▶ │ offboard_manager   │ ──▶ PX4
 pose  ─▶ │ 2D ESDF/cost slice    │                    │ (A* on grid)  │             │ (TrajectorySetpoint)│   (Day 3 link)
          └──────────────────────┘                     └───────────────┘             └────────────────────┘
                                                              ▲                              ▲
                                                         goal pose                   /fmu/out/vehicle_local_position
```

You write **two** nodes; nvblox and the Day-3 bridge already exist:
1. `planner_node` — turns the 2D occupancy slice + a goal into a collision-free `nav_msgs/Path`.
2. `offboard_manager` — arms, holds Offboard mode, and streams the path as PX4 setpoints.

**Stay 2D for week 1.** Fly at a fixed altitude and plan on nvblox's 2D slice. Full 3D planning is a later upgrade (see end).

### Frame discipline (this is where loops die)
- PX4 local position & `TrajectorySetpoint` are **NED** (x=North, y=East, **z=Down**, so 2 m altitude = `z = -2.0`).
- ROS / nvblox / RViz are **ENU** (z=Up).
- Pick **one** planning frame (use the ENU `map` frame nvblox publishes), plan there, then convert to NED only at the moment you fill `TrajectorySetpoint`. `px4_ros_com` has helpers; if you hand-roll it: `north=y_enu, east=x_enu, down=-z_enu`.

### Node A — `planner_node` (skeleton, rclpy)
```python
# subscribes: /nvblox_node/static_map_slice  (nav_msgs/OccupancyGrid)
#             /goal_pose                      (geometry_msgs/PoseStamped, set from RViz "2D Goal Pose")
#             current pose from TF (map -> base_link)
# publishes : /planned_path                   (nav_msgs/Path, in map/ENU frame)
class PlannerNode(Node):
    def on_map(self, grid):           self.grid = grid           # store latest occupancy slice
    def on_goal(self, goal):          self.goal = goal; self.replan()
    def replan(self):
        start = self.current_xy_from_tf()
        # 1. inflate occupied cells by drone radius + margin (e.g. 0.35 m) -> safety buffer
        # 2. A* over free cells (8-connected) from start cell to goal cell
        # 3. simplify path (line-of-sight shortcut), publish as nav_msgs/Path
        ...
```
Start with stock A* on the inflated grid — don't write minimum-snap yet. **Obstacle inflation is the single most important line:** inflate by drone radius + a margin so the *center-point* path keeps the whole airframe clear. Re-run `replan()` on every new map message so newly-seen obstacles trigger replanning (that's your "mid-flight replanning").

### Node B — `offboard_manager` (skeleton, rclpy + px4_msgs)
```python
# pub: /fmu/in/offboard_control_mode (OffboardControlMode)  @ >=10 Hz  (position=True)
#      /fmu/in/trajectory_setpoint   (TrajectorySetpoint)   @ >=10 Hz
#      /fmu/in/vehicle_command       (VehicleCommand)        (arm, set Offboard)
# sub: /fmu/out/vehicle_local_position (VehicleLocalPosition)  -> current NED pos
#      /planned_path                   (nav_msgs/Path)
class OffboardManager(Node):
    def tick(self):  # 50 ms timer = 20 Hz
        self.pub_offboard_mode()                      # MUST keep streaming or PX4 exits Offboard
        wp = self.current_waypoint()                  # next path point, converted ENU->NED
        self.pub_trajectory_setpoint(x, y, z=-ALT, yaw=face_travel_dir)
        if self.reached(wp, tol=0.25): self.advance()  # 0.25 m acceptance radius
    # startup sequence: stream setpoints for ~1 s FIRST, THEN arm, THEN switch to Offboard
```
**Order matters:** publish setpoints *before* commanding Offboard, or PX4 rejects the mode switch. Keep `offboard_control_mode` flowing at ≥2 Hz the entire flight (use 10–20 Hz).

### Build & run order (terminals)
```bash
# T1: Micro-XRCE agent
MicroXRCEAgent udp4 -p 8888
# T2: PX4 SITL with depth cam
cd PX4-Autopilot && make px4_sitl gz_x500_depth
# T3: ros_gz bridges (depth + camera_info)   [Day 4]
# T4: Isaac ROS container -> cuVSLAM + nvblox  [Days 5-6]
# T5: your nodes
cd ~/ws_px4 && colcon build && source install/setup.bash
ros2 launch my_autonomy bringup.launch.py
# T6: RViz2 — add OccupancyGrid, Path, TF, PointCloud2; use "2D Goal Pose" to set the goal
```

### Test procedure
1. Empty world: confirm the drone flies start → goal in a straight line and holds. (Validates B alone.)
2. Drop **one box** in Gazebo between start and goal. Confirm nvblox marks it occupied and the planner routes around it.
3. Move the box *while flying* → confirm replanning kicks in.
4. Record per run: reached goal? closest-approach distance to the obstacle? replan latency? Note every failure mode (fell out of Offboard, clipped the obstacle, path oscillated, etc.).

### ✅ Week-1 done when
Simulated X500 autonomously flies start → goal, avoiding one inserted obstacle, with the map built live from depth and cuVSLAM producing a validated odometry track. That's the §6 Step-1 gate in BUILD.md — **only after this do you start buying hardware.**

### Upgrade path (don't do these in week 1)
- **3D planning:** plan over the nvblox ESDF in 3D (e.g. a sampling planner, or `ego-planner` / MAVROS-style local planners) instead of a 2D slice.
- **Smooth trajectories:** replace raw waypoints with minimum-snap / polynomial trajectories for smooth, fast flight.
- **Nav2 route:** nvblox ships a Nav2 costmap plugin (`nvblox_nav2`); usable if you adapt Nav2's 2D output to multirotor setpoints — heavier, more "standard," optional.

---

## Gotchas to expect

- **Gazebo version mismatch** is the #1 time-sink: PX4 main + ROS 2 Jazzy expect **Gazebo Harmonic**. Don't mix in old `gazebo-classic`.
- **Two ROS distros on one machine** (if you ever added Humble) → only ever source one `setup.bash` per shell.
- **Isaac ROS = Docker.** Don't try to apt-install it onto a bare host; use NVIDIA's container. It needs the NVIDIA Container Toolkit.
- **Offboard mode safety timeout:** PX4 drops out of Offboard if setpoints stop arriving at a high enough rate (~>2 Hz). If the drone "falls out" of autonomy, check your publish rate first.
- **Frames/TF:** PX4 is NED, ROS is ENU. The px4_ros_com helpers handle conversions — respect them or your waypoints go sideways.

---

## Official docs to cross-check against
- PX4 ROS 2 User Guide: https://docs.px4.io/main/en/ros2/user_guide
- PX4 uXRCE-DDS bridge: https://docs.px4.io/main/en/middleware/uxrce_dds
- PX4 Gazebo SITL: https://docs.px4.io/main/en/sim_gazebo_gz/
- ROS 2 Jazzy install: https://docs.ros.org/en/jazzy/Installation.html
- Isaac ROS Getting Started (Docker): https://nvidia-isaac-ros.github.io/getting_started/index.html
- Isaac ROS Visual SLAM (cuVSLAM): https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_visual_slam/index.html
- Isaac ROS Nvblox: https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/index.html
