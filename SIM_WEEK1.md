# Week-1 Simulation Setup Checklist

> Goal for the week: a simulated X500 takes off and flies a waypoint **autonomously via offboard control**, with a depth camera streaming into ROS 2, and an occupancy map building from that depth. No hardware, no crash risk.
>
> Companion doc: [BUILD.md](./BUILD.md). Target stack: PX4 SITL + Gazebo + ROS 2 **Jazzy** + Isaac ROS (cuVSLAM/nvblox).
>
> ⚠️ **These are starting-point commands.** Package names, repo paths, and versions drift — cross-check each against the official PX4 / ROS 2 / Isaac ROS docs (linked at the bottom) as you go. Don't paste blindly.

---

## Progress (as of 2026-07-30) — **~75% through the gate**

| Day | Milestone | Status | Evidence / blocker |
|---|---|---|---|
| 0 | Machine check | ✅ | Ubuntu 24.04, RTX A3000 **6 GB**, 16 cores, 31 GB RAM, 333 GB free |
| 1 | ROS 2 Jazzy | ✅ | `/opt/ros/jazzy` |
| 2 | PX4 SITL + Gazebo, manual flight | ✅ | gz-harmonic 8.12 + `~/PX4-Autopilot` builds and flies |
| 3 | Offboard waypoint from ROS 2 | ✅ | `~/ws_px4` builds `px4_msgs` + `px4_ros_com` + `gps_denied_autonomy` |
| 4 | Depth camera into ROS 2 | ✅ **2026-07-30** | 4 topics at rate, in flight, real TF — `DEPTH_SIM.md` §2, `results/depth_bridge.png` |
| 5 | Occupancy map from depth | ✅ **2026-07-30** | `octomap_server` on `/depth_camera/points`; 3/3 obstacles mapped at 21–37× map density, open ground **and** the area behind the aircraft 0.0% occupied — `MAPPING.md`, `results/octomap_day5.png`, `results/octomap_3view.png` |
| 6 | VIO + drift number | 🟡 **pipeline done, number invalid** | `icp_odometry` runs at 28 Hz off the depth cloud, scored against PX4 — but the aircraft left the commanded square at **16.4 m/s** and the estimator tracked only 47% of the distance, so the drift figure is not a benchmark. `VIO.md` §3 |
| 7 | Close the loop | 🟡 **partly done** | flew autonomously 2026-07-13 — but on `fake_world`'s **synthetic** map, with PX4's own sim pose |

**What's actually done:** Days 1–5 fully, and Day 7's *own code* — `planner_node`,
`offboard_manager`, `astar`, `fake_world` all built, and the closed loop flew
end-to-end in SITL (`~/ws_px4/src/gps_denied_autonomy/SITL_FLIGHT.md`).

**Day 4 closed on 2026-07-30.** `/depth_camera/image_raw` (28.8 Hz),
`/camera_info` (29.5 Hz), `/depth_camera/points` (24.5 Hz) and `/imu` (241 Hz) all
deliver, captured *during* an autonomous square with `px4_tf_publisher` supplying a
real moving `map -> base_link -> camera_link`. Runbook + numbers:
`~/ws_px4/src/gps_denied_autonomy/DEPTH_SIM.md`.

**Day 5 closed on 2026-07-30.** `octomap_server` builds a live map from
`/depth_camera/points`: three deliberately asymmetric obstacles all map to their true
positions, open ground inside the flight square comes back **0.0% occupied**, and the
sim holds ~500 MB for the whole run. Details, including two silent-failure traps
(wrong octomap parameter spellings; a sim-time/wall-time clock mismatch that drops
every cloud), are in `MAPPING.md`.

> **⚠️ Day-6 result must not be quoted — and the first explanation for it was wrong.**
> The run produced `ATE 4.32 m, final drift 2.16 m (3.38%)`, which looks respectable
> and is not. I first blamed the aircraft (a 16 m excursion at 16.4 m/s ⇒ "tree
> collision"). Chasing it ruled that out — the nearest tree is 4.38 m *outside* the
> square — and found two better answers, either of which alone voids the number:
>
> 1. **The reference is not ground truth.** `/fmu/out/vehicle_odometry` is **EKF2's
>    estimate**; PX4 exports no groundtruth topic over uXRCE-DDS at all. The comparison
>    was estimator-vs-estimator, so an EKF position jump reads as estimator "error".
>    Path length varied **36–64 m across runs of the identical commanded 5 m square**,
>    while `offboard_manager` logged orderly waypoint progression throughout.
> 2. **ICP never registered anything.** Every frame logs `ratio = 0.000000` at ~0.1 ms
>    per update, where real ICP on this cloud takes tens of ms. `/odom` is not a tracked
>    trajectory, so no statistic derived from it means anything.
>
> Also ruled out with evidence: NED/FRD frame mixing (all NED), timebase artifacts,
> a starved cloud (197 955 points at 27.5 Hz), and bad normals. Full write-up:
> `VIO.md` §3b.

**What's left is Day 6, plus one swap.** `planner_node` already consumes
`nav_msgs/OccupancyGrid` and `/projected_map` is one — pointing it there instead of
`fake_world` and flying that mission *is* the Phase-1 gate. That is now unblocked: the
map contains only real obstacles.

> **✅ Day-5 defect found and fixed the same day.** The first working map had ~18% of
> the cells *behind* the aircraft occupied, where nothing exists. Plotting the occupied
> **voxel centres** in plan + two elevations (`viz_octomap3d.py`) identified them
> instantly as a thin sheet at **Up ≈ 0.2 m** — ground returns that survived the plane
> filter, not phantom structure. `/projected_map` is 2D, so a ground return and a 4 m
> tower look identical in it; the 3-view render is what made the difference.
> Raising `occupancy_min_z` 0.25 → 0.45 took the artifacts to **0.0%** and *sharpened*
> obstacle contrast from 4.6–9.5× to **21–37×**. Cost: obstacles under ~0.45 m are
> invisible — fine for a drone at 3 m, wrong for a ground robot.
>
> Still worth chasing separately: the free space carved is a near-complete **disc**,
> which a vehicle holding the commanded `yaw = 0` could not produce. That was the
> *exposure* mechanism, not the cause, but it suggests PX4 isn't tracking commanded
> yaw — a real bug in its own right.

> **Days 5–6 decided (BUILD.md §0.6):** `octomap_server` for occupancy in sim; Isaac
> ROS (nvblox + cuVSLAM) stays the *flight* stack on the Jetson. Isaac ROS ships only
> as NVIDIA Docker containers, and standing that up means Docker + the NVIDIA Container
> Toolkit (neither installed) plus a GPU mapper on 6 GB of VRAM — days of yak-shaving
> on tooling that gets discarded when the Jetson arrives. Read §0.6 for what this
> costs: Phase-1 tuning numbers will **not** transfer to nvblox, and this does not
> count as "nvblox validated."
>
> **Day-6 front-end decided (2026-07-30): `rtabmap_ros`.** Installed and available as
> `rtabmap_odom icp_odometry`. Chosen because it is apt-installable with no Docker, runs
> on the depth cloud we already publish, and needs no RGB — which matters, since the
> overlay model deletes the RGB camera to stop the 180 MB/s leak. It must run with
> **loop closure OFF**: rtabmap is SLAM, and loop closure corrects exactly the drift
> Day 6 exists to measure. DPVO stays where it earns its place, on the research track;
> the OpenVINS/EuRoC ladder stays the offline career deliverable.

> **Two blockers found during Day 4 — both now fixed (2026-07-30).**
> 1. ✅ **Invalid TF tree.** `depth_bridge.launch.py` defaulted `static_tf:=true`,
>    publishing `map -> camera_link` while `px4_tf_publisher` publishes
>    `base_link -> camera_link`. Both ran together, giving `camera_link` two parents —
>    and near the origin it looked fine. The default is now `false`, so bridge +
>    `px4_tf_publisher` is correct with no extra flags. Don't re-enable it.
> 2. ✅ **Gazebo's RAM leak — root-caused to the unused RGB camera.** `gz sim` grew at
>    a flat **180 MB/s** and was OOM-killed twice (15:16, 15:25 — the second took the
>    desktop session with it). It is the OakD-Lite's **1920×1080 RGB camera at 30 Hz**,
>    matching `1920×1080×3×30 = 178 MiB/s`. The **depth** camera does not leak at all.
>    Fixed with an overlay model that deletes the RGB sensor; the sim now holds ~490 MB
>    indefinitely and Day 4 still passes. **Requires one `export`** — see
>    `gps_denied_autonomy/sim/README.md` and `DEPTH_SIM.md` §4b.
>
> Ruled out along the way: `always_on=0` (doesn't gate rendering on subscribers) and
> the Gazebo version (system 8.14.0 leaks identically to the ROS-vendored 8.11.0 —
> note `.bashrc` makes 8.11.0 the default via `GZ_CONFIG_PATH`).

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

## Day 4 — Depth camera into ROS 2 — ✅ **done 2026-07-30**

> Full runbook, verified topic names and results:
> **`~/ws_px4/src/gps_denied_autonomy/DEPTH_SIM.md`**. What follows is the summary.

PX4 ships an X500 variant with a depth camera (an OakD-Lite):
```bash
cd PX4-Autopilot
HEADLESS=1 make px4_sitl gz_x500_depth   # HEADLESS: the GUI + sensor rendering
                                         # does not fit 6 GB of VRAM
```

Bridge Gazebo's camera topics into ROS 2 (Jazzy pairs with Gazebo Harmonic via
`ros_gz`, installed at `1.0.22-1noble`):
```bash
sudo apt install -y ros-jazzy-ros-gz
ros2 launch gps_denied_autonomy depth_bridge.launch.py
ros2 run gps_denied_autonomy px4_tf_publisher    # the real map->base_link->camera_link
```

**Three things this cost time to learn, recorded so Day 5 doesn't repeat them:**

- **`gz topic -l` first, always.** Gazebo auto-scopes any sensor topic without an
  explicit `<topic>` tag by `world/model/link/sensor`, so the paths change if the world
  or model is renamed. This is the same class of bug as the Day-3 topic drift.
- **There is no camera-rigid IMU.** A `camera_imu` topic *name* appears in
  `gz topic -l` with nothing publishing on it. The only live IMU is the flight
  controller's on `base_link`. Fine in sim (the camera joint is fixed, extrinsic known
  exactly) — but on hardware the camera–IMU calibration is real work, and this sim
  passing does not retire it.
- **The point cloud is x-forward, not ROS's z-forward optical convention.** Assume
  wrong and Day 5's map is silently rotated 90°.

**✅ Success check — met.** All four topics deliver at rate, captured *during* an
autonomous square, with `max |first-last| = 17.2 m` across 409 frames proving the
stream tracks the world. Evidence: `results/depth_bridge.png`.

Two things the original check asked for that were **not** done, and why:
- *RViz2 visualisation* → replaced with a headless PNG checker. RViz alongside SITL's
  sensor rendering is the VRAM combination that OOM-killed the sim on Day 3. Same
  evidence, less budget. A laptop constraint, not a design choice.
- *"place an object in the Gazebo world"* → **still outstanding.** Every capture so far
  is over an empty ground plane, so nothing is within 3 m. Near-field depth is where a
  real D435i is trustworthy and where obstacle avoidance lives. Do this on Day 5.

---

## Day 5 — Occupancy map from depth — ✅ **done 2026-07-30**

> Full runbook, results and the two silent-failure traps:
> **`gps_denied_autonomy/MAPPING.md`**. Summary below.

> **Revised 2026-07-30: `octomap_server`, not nvblox.** Full rationale and accepted
> costs in [BUILD.md §0.6](./BUILD.md). nvblox stays the *flight* stack on the Jetson;
> this substitution is for the laptop sim only. The original nvblox route is kept below
> for Phase 2.

Everything octomap needs is already published by the Day-4 bridge:

| octomap input | supplied by |
|---|---|
| `PointCloud2` | `/depth_camera/points` |
| `map -> camera_link` TF | `px4_tf_publisher` (don't re-enable `static_tf` — see above) |
| `use_sim_time` | `/clock` |

```bash
python3 gps_denied_autonomy/sim/spawn_obstacles.py   # geometry to actually map
ros2 launch gps_denied_autonomy octomap.launch.py    # tuned params, see MAPPING.md
ros2 run gps_denied_autonomy px4_tf_publisher --ros-args -p use_sim_time:=true
python3 gps_denied_autonomy/check_octomap.py results/octomap_day5.png 20
```

**Two traps here, both of which fail *silently*** — nodes run, topics exist,
`ros2 topic hz` looks healthy, and the map stays empty:

1. **`use_sim_time:=true` on `px4_tf_publisher`.** The cloud is stamped in sim time
   (~68 s) while the TF publisher defaults to wall clock (~1.78e9), so octomap drops
   every cloud as *"earlier than all the data in the transform cache."*
2. **octomap's parameter names.** It ignores unknown ones without warning. The
   plausible-looking `filter_ground` and `pointcloud_min_z` are **wrong** — the real
   names are `filter_ground_plane` and `point_cloud_min_z`. Get them wrong and the
   ground plane is mapped as one solid obstacle. `ros2 param list /octomap_server`
   against a running node is the only reliable check.

`octomap_server` publishes **`/projected_map`** as a `nav_msgs/OccupancyGrid` — the
exact type `planner_node` already consumes. That is the whole point: `fake_world`'s
synthetic grid unplugs and a perceived map plugs in **with no planner changes**, which
is the Phase-1 gate.

**Both Day-4 blockers are cleared**, which is what makes this practical: the TF tree is
correct, and the sim now holds a flat ~490 MB instead of dying in minutes — so a long
mapping run is finally possible. Just remember the `GZ_SIM_RESOURCE_PATH` export, or
the RGB leak comes back.

**✅ Success check — met 2026-07-30.** Three asymmetric obstacles spawned north of the
flight square all map to their true positions (contrast 4.6–9.5× vs. the map as a
whole), open ground inside the square comes back **0.0% occupied**, and gz holds
492 → 501 MB across the run. Evidence: `results/octomap_day5.png`.

> Counting occupied cells near an obstacle is *not* a real check — on a map that is
> 90% occupied (what a broken ground filter produces) every window has hits and
> everything "passes". `check_octomap.py` also requires a **control patch** of open
> ground to come back free, and normalises each obstacle's density by what its own
> geometry allows. That is what caught the ground-filter bug.

<details>
<summary>Original nvblox route — deferred to Phase 2 on the Jetson</summary>

**GPU path (Isaac ROS nvblox):** run via NVIDIA's `isaac_ros_common` Docker container (simplest, avoids host dependency hell). Feed it the bridged depth + camera_info + a pose source.
```bash
# Use the Isaac ROS Docker container per NVIDIA's "Getting Started" + nvblox quickstart.
# Inside the container, launch nvblox subscribed to the sim depth topics.
```

Configure nvblox to also publish a **2D ESDF/distance-map slice at flight altitude** (`nav_msgs/OccupancyGrid` or `nvblox_msgs/DistanceMapSlice`) — that 2D slice is what the Day-7 planner consumes.

**✅ Success check:** the 3D voxel map *and* a 2D slice appear in RViz, and an obstacle you drop into the Gazebo world shows up as occupied/high-cost cells.
</details>

---

## Day 6 — VIO (rtabmap `icp_odometry`, not cuVSLAM)

> **Decided 2026-07-30: `rtabmap_ros`, installed.** cuVSLAM has the same Docker/VRAM
> problem that moved Day 5 to octomap (BUILD.md §0.6). `rtabmap_odom icp_odometry` is
> apt-installable, needs no Docker, and runs on the depth cloud already being published
> — which matters because the overlay model **deletes the RGB camera** to stop the
> 180 MB/s leak, so no feature-based front-end has an image to work with.
>
> **Run it with loop closure OFF.** rtabmap is a SLAM system; loop closure corrects
> precisely the drift this day exists to measure. A drift number from a loop-closing
> SLAM system is not a VIO drift number.
>
> **Use the forest world.** ICP on a flat plane with three boxes is *degenerate* — the
> ground constrains only 3 DOF and the estimate slides freely along it. Trees constrain
> all directions. This is not cosmetic; it is the difference between a drift number and
> a slow slide into nonsense.
>
> Whatever the front-end, it publishes odometry behind the same `map -> base_link` TF
> that `px4_tf_publisher` supplies today — that node is the interface contract.
>
> Note the sim's IMU is the **body** IMU on `base_link`, not a camera-rigid one
> (see Day 4). Usable here because the camera joint is fixed and the extrinsic exact.
> On hardware that calibration is real work; sim passing does not retire it.

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

> **Status 2026-07-30: not met.** The flight half is done (autonomous start → goal
> through a doorway, 2026-07-13) and the depth stream is live (Day 4). Missing: the map
> is still `fake_world`'s synthetic grid, and there is no odometry track or drift
> number. Per BUILD.md §0.6 the map will be built by **octomap, not nvblox**, and the
> VIO front-end is undecided — so when this gate is called, state plainly what was
> validated. "Architecture proven with octomap + PX4 sim pose" is an honest and
> defensible claim; "nvblox and cuVSLAM validated" would not be.

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
- **Gazebo *sensor* frames are x-forward**, while ROS's optical convention is z-forward. Confirmed on the depth cloud, Day 4. Different axis trap from the NED/ENU one above, and it bites the mapper rather than the controller.
- **One child frame, one parent.** Running two publishers that both parent `camera_link` produces an invalid TF tree that tf2 resolves nondeterministically — and near the origin it looks *fine*. Hit on Day 4; see DEPTH_SIM.md §4a.
- **Point the camera where the information is.** Mounted level at 3 m, half the depth frame is sky and grazing-incidence ground returns defeat octomap's plane filter — 18.4% of open ground came back *occupied*. A 20° downward tilt takes that to zero and lifts frame utilisation 52% → 78%. Tilt beats flying lower, because lowering the aircraft does not move the horizon.
- **A sensor's mounting angle lives in two places.** The model (what the sim renders) and the TF (what the stack believes). Disagree and every point is rotated about the camera, which looks like odometry drift rather than a mounting error.
- **Watch system RAM, not just VRAM.** `gz sim` with the depth airframe grew at 180 MB/s and was OOM-killed twice on 2026-07-30, the second time taking the desktop session down. The 6 GB VRAM limit is a *separate* constraint with a confusingly similar symptom. Root cause was the unused 1920×1080 RGB camera; fixed by an overlay model (`DEPTH_SIM.md` §4b).
- **An unused sensor still costs you everything.** Nothing subscribed to that RGB topic, and `always_on=0` didn't stop it either — Gazebo renders declared sensors regardless. If you don't need a sensor, delete it from the model rather than leaving it unsubscribed.
- **You have two Gazebo installs.** `.bashrc` sources ROS, which sets `GZ_CONFIG_PATH` to the ROS-vendored **8.11.0**; apt has **8.14.0**. Check `gz sim --version` before blaming (or reporting) a version-specific bug.
- **Don't redirect the `pxh>` console to an uncapped file.** Two SITL logs reached 7.8 GB, almost entirely terminal escape codes.

---

## Official docs to cross-check against
- PX4 ROS 2 User Guide: https://docs.px4.io/main/en/ros2/user_guide
- PX4 uXRCE-DDS bridge: https://docs.px4.io/main/en/middleware/uxrce_dds
- PX4 Gazebo SITL: https://docs.px4.io/main/en/sim_gazebo_gz/
- ROS 2 Jazzy install: https://docs.ros.org/en/jazzy/Installation.html
- Isaac ROS Getting Started (Docker): https://nvidia-isaac-ros.github.io/getting_started/index.html
- Isaac ROS Visual SLAM (cuVSLAM): https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_visual_slam/index.html
- Isaac ROS Nvblox: https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/index.html
