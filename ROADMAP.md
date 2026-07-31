# Learning + Build Roadmap

> Philosophy: **learn just-in-time.** Don't pre-study six months of theory. Learn each concept right before the build step that forces you to use it — you'll retain 10× more. Each phase below pairs *what you build* with *what to learn first* and *where to learn it*.
>
> Companion docs: [README.md](./README.md) (plan going forward) · [BUILD.md](./BUILD.md) (decisions/BOM) · [SIM_WEEK1.md](./SIM_WEEK1.md) (week-1 commands) · [ROLES.md](./ROLES.md) (team) · [SUMMER.md](./SUMMER.md) (pre-work).
>
> Legend: ⭐ = do this one, it's the highest-leverage resource for the step. 📚 = deeper dive when you have time.

---

## How to use this roadmap
- **Keep a lab notebook** (a git repo or a Markdown journal). Write down every command that worked, every failure mode, every tuning value. Senior design is graded partly on this; future-you needs it too.
- **Do the core math by hand once.** Derive a 1D Kalman filter on paper before you trust a library. Plot an A* expansion. You only need to do it once to stop treating these as magic.
- **Build → break → understand → fix.** When something fails, resist copy-pasting a fix. Form a hypothesis first.

---

## ★ Your personal development track — VIO → autonomy engineer

Your goal isn't "do the project," it's **broaden from CV into an autonomy/localization engineer.** Your owned area is **State Estimation / VIO lead + autonomy-loop integration** (see [ROLES.md](./ROLES.md)). The key principle that makes this safe:

> **Learn VIO deeply on one clock; deliver VIO on another.**

- **Deliver (project clock):** integrate, calibrate, tune, and characterize **cuVSLAM** on the real sensor → feed PX4 EKF2 → measure drift. This *is* real localization engineering and it's what ships.
- **Learn the internals (your clock, offline, can't block the project):** datasets, not the aircraft.

**VIO learning ladder (do in order):**
1. ⭐ *Kalman & Bayesian Filters in Python* (Labbe) — work the EKF notebooks; derive a 1D filter by hand once.
2. ⭐ Run **OpenVINS** (or VINS-Fusion) on the **EuRoC MAV dataset** — watch a real estimator work; read its docs.
3. ⭐ **Implement a toy visual-inertial EKF / MSCKF-lite on EuRoC yourself**, offline. This is where deep understanding happens — and a buggy filter on a dataset crashes nothing.
4. Read the **MSCKF** and **IMU-preintegration (Forster et al.)** papers; 📚 Barfoot *State Estimation for Robotics* for rigor.
5. Learn **Kalibr** (camera–IMU calibration) — you'll use it for real on the RealSense.
6. **cuVSLAM** in sim → on the bench with the real RealSense → into EKF2 → drift number.

**Resume line this builds:** "CV engineer who built the VIO-based localization and integrated the full autonomy loop for a GPS-denied drone." That's a localization/autonomy story anchored in your existing CV strength.

> Don't solo the riskiest box blind: VIO is the schedule-dominating risk (BUILD.md §4). Pair on hard debugging, validate early/offline, and keep a fiducial/mocap localization fallback for flight tests so VIO tuning can't block the demo.

---

## Phase 0 — Foundations + Sim bring-up  (Weeks 1–2) — ✅ **done**
**Build:** everything in [SIM_WEEK1.md](./SIM_WEEK1.md) — ROS 2 Jazzy, PX4 SITL, fly an offboard waypoint, depth into ROS.

> **Status 2026-07-30: ✅ complete.** You write ROS 2 nodes from scratch
> (`planner_node`, `offboard_manager`, `fake_world`, `px4_tf_publisher`), the sim drone
> flies waypoints commanded from ROS, and **depth is now in ROS 2** — the last Phase-0
> item, closed 2026-07-30 (SIM_WEEK1 Day 4). Everything in Phase 0 is done.

**Learn first:**
- **ROS 2 core** — nodes, topics, pub/sub, services, parameters, `tf2`, launch files, `colcon`, workspaces.
  - ⭐ Articulated Robotics (Josh Newans) — YouTube channel + articulatedrobotics.xyz. The best practical ROS 2 teaching anywhere; he builds a real robot.
  - ⭐ Official ROS 2 Jazzy tutorials — https://docs.ros.org/en/jazzy/Tutorials.html (do the Beginner: CLI + Client Libraries tracks).
  - 📚 The Construct — https://www.theconstruct.ai (interactive, browser-based ROS courses).
- **Linux + Python/C++ comfort** — terminal, `apt`, virtualenvs, basic `rclpy`. If shaky on Python, fix that first; you'll write nodes in it.
- **PX4 mental model** — what SITL is, what offboard mode is, NED vs ENU frames.
  - ⭐ PX4 ROS 2 User Guide — https://docs.px4.io/main/en/ros2/user_guide

**Phase-0 done when:** you can write a ROS 2 node from scratch that subscribes to one topic and publishes another, and the sim drone flies a waypoint you command from ROS.

---

## Phase 1 — Full autonomy stack in simulation  (Months 1–2) — 🔄 **~50%**
**Build:** Day-7 loop fully fleshed out — cuVSLAM drift study, nvblox map, A* planner, offboard manager. Obstacle avoidance working in sim. (This is the §6 gate before buying hardware.)

> **Status 2026-07-30:** ✅ **A\* planner** (`astar.py`, validated headlessly and on real
> nuScenes HD-map rasters) and ✅ **offboard manager** are done and flew the closed loop
> autonomously in SITL on 2026-07-13. ✅ **Depth into ROS 2** closed 2026-07-30. ⬜ **the
> map** and ⬜ **the drift study** are not started — that's the missing half. The loop
> currently plans on a synthetic grid, so "obstacle avoidance in sim" is proven for
> *planning→flight*, not for *perception→flight*.
>
> **Scope change (BUILD.md §0.6):** the sim map is now **octomap**, not nvblox, and the
> VIO front-end is undecided. Isaac ROS remains the Jetson flight stack. This makes
> Phase 1 reachable on the laptop, but it means Phase-1 tuning numbers **will not
> transfer** to nvblox — budget re-tuning in Phase 2, and don't describe this phase as
> validating nvblox or cuVSLAM.
>
> **Track-4 note (quadrotor dynamics/control):** untouched so far. That's fine — the
> offboard position-setpoint interface let you skip it. It becomes real when you tune PID
> for actual all-up weight in Phase 3.

**Learn first — four tracks, in this order:**

**1. State estimation (the heart of GPS-denied).**
- ⭐ *Kalman and Bayesian Filters in Python* — Roger Labbe, free Jupyter book: https://github.com/rlabbe/Kalman-and-Bayesian-Filters-in-Python. Work the notebooks; this is the single best hands-on KF/EKF resource.
- ⭐ Cyrill Stachniss — YouTube ("Mobile Sensing and Robotics", SLAM, EKF lectures). University-quality, free.
- 📚 *State Estimation for Robotics* — Tim Barfoot, free PDF (asrl.utias.utoronto.ca). The rigorous VIO math (Lie groups, factor graphs) when you're ready.
- 📚 *Probabilistic Robotics* — Thrun/Burgard/Fox. The reference text for filters + SLAM.

**2. Visual-Inertial Odometry specifically.**
- ⭐ Read the **VINS-Mono** and **OpenVINS** papers — even if you use cuVSLAM, these explain *why* VIO works (feature tracking, IMU preintegration, initialization, scale).
- ⭐ First Principles of Computer Vision — Shree Nayar (Columbia), YouTube. Camera model, features, stereo — the vision half of VIO.
- NVIDIA Isaac ROS Visual SLAM docs — https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_visual_slam/

**3. Mapping + planning.**
- ⭐ *Planning Algorithms* — Steve LaValle, free book: http://lavalle.pl/planning/. Read the A*, Dijkstra, and sampling (RRT/PRM) sections.
- nvblox docs — https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/ (understand occupancy vs ESDF/TSDF).
- 📚 Cyrill Stachniss has occupancy-grid-mapping lectures too.

**4. Quadrotor dynamics, control & trajectories.**
- ⭐ **Coursera "Robotics: Aerial Robotics"** — Vijay Kumar, UPenn. *The* course for quadrotor dynamics, PD control, and **minimum-snap trajectories** (the Mellinger & Kumar method you'll use). Audit it free.
- ⭐ *Underactuated Robotics* — Russ Tedrake (MIT), free video + notes: https://underactuated.mit.edu. For understanding why quadrotors are controlled the way they are.
- 📚 *Modern Robotics* — Lynch & Park, free book + Coursera + YouTube. Rigid-body kinematics/dynamics foundation.
- 📚 Mellinger & Kumar 2011, "Minimum Snap Trajectory Generation and Control for Quadrotors" — the paper.

**Math backfill (only if these feel shaky — don't over-invest):**
- ⭐ 3Blue1Brown "Essence of Linear Algebra" (YouTube) — rotations, vectors, eigenstuff intuition.
- Probability basics (any intro) — you need Gaussians, covariance, Bayes' rule for the filters.

**Phase-1 done when:** sim drone autonomously avoids an inserted obstacle, you can *explain* how cuVSLAM estimates pose and how A* found the path, and you have a measured VIO drift number.

---

## Phase 2 — Hardware bring-up  (Months 2–4)
**Build:** assemble the X500, PX4 config + calibration, manual flight, then Jetson Orin Nano + RealSense bring-up, cuVSLAM on the *real* sensor on the bench.

**Learn first:**
- **Drone hardware** — frames, motors (KV, sizing), ESCs, props, LiPo/Li-ion safety, power budget, soldering, ELRS RC link.
  - ⭐ Oscar Liang's blog — https://oscarliang.com. The best practical FPV/drone hardware reference (motor/ESC/prop selection, soldering, ELRS setup, thrust-to-weight).
- **PX4 hardware setup** — firmware flashing, sensor calibration, flight modes, failsafes, geofence, kill switch.
  - ⭐ PX4 Basic Assembly + Standard Configuration docs — https://docs.px4.io/main/en/config/
  - ⭐ Holybro X500 V2 build guide — https://docs.holybro.com/drone-development-kit/px4-development-kit-x500v2
- **Jetson Orin Nano setup** — JetPack flashing, NVIDIA Container Toolkit, running Isaac ROS containers on the Jetson.
  - ⭐ NVIDIA Jetson "Getting Started" + Isaac ROS Getting Started — https://nvidia-isaac-ros.github.io/getting_started/
  - ⭐ The Orin Nano Super + Isaac ROS practical guide (from the BUILD.md sources) — verify your release supports Orin Nano + Jazzy *before* relying on it.
- **RealSense on Jetson** — librealsense + the ROS 2 wrapper.

**Safety (non-negotiable before any powered prop test):**
- Props OFF for all bench/first-power tests. Build the kill switch + geofence in PX4 *before* the first flight. Net/enclosure for indoor.

**Phase-2 done when:** the X500 flies stably under manual control, and the Jetson runs cuVSLAM + nvblox on the real RealSense feed on the bench (off the aircraft).

---

## Phase 3 — Integration & flight test  (Months 4–6)
**Build:** cuVSLAM → PX4 EKF2 (vision-aided, GPS denied) → offboard autonomy. Closed-loop indoor obstacle avoidance, then outdoor / under-structure.

**Learn first:**
- **Vision-aided EKF2 / GPS-denied config** — feeding external odometry into PX4, EKF2 parameter tuning.
  - ⭐ PX4 "Using Vision or Motion Capture Systems for Position Estimation" + EKF2 docs — https://docs.px4.io/main/en/computer_vision/
- **Offboard control robustness** — setpoint rate, mode-switch safety, failsafe behavior on VIO loss.
- **Flight-test methodology** — incremental envelope expansion, logging (PX4 ulog + ROS bags), `PlotJuggler` / Flight Review for analysis.
  - ⭐ PX4 Flight Review + log analysis docs.
- **Controller tuning** — PID tuning for your actual all-up weight.
  - ⭐ PX4 Multicopter PID Tuning Guide.

**Phase-3 done when:** indoor autonomous obstacle avoidance demo passes your success metric (e.g. 3/5 trials, ≥0.5 m margin), then repeated outdoors / under structure.

---

## Phase 4 — Final demo + documentation  (Month 6+)
**Build:** final mission, evaluation report, presentation. **Stretch:** outdoor robustness, then the custom STM32H7 flight-controller PCB (only if Phase 3 finished early — see BUILD.md scope ladder).

**Learn (only if pursuing the PCB stretch):**
- KiCad, STM32 + the ardupilot/PX4 hardware bring-up process, hardware bring-up debugging. This is a whole second project — gate it hard.

---

## Resource library (grouped, for reference)

**ROS 2**
- Articulated Robotics (YouTube + site) ⭐ · ROS 2 Jazzy docs ⭐ · The Construct · *A Gentle Introduction to ROS* (O'Kane, free, ROS1 concepts)

**PX4 / drones / hardware**
- PX4 docs (docs.px4.io) ⭐ · Oscar Liang (oscarliang.com) ⭐ · Holybro docs · ArduPilot docs (good background)

**State estimation / VIO**
- Kalman & Bayesian Filters in Python (Labbe) ⭐ · Cyrill Stachniss (YouTube) ⭐ · State Estimation for Robotics (Barfoot, free) · Probabilistic Robotics (Thrun) · VINS-Mono / OpenVINS papers · First Principles of Computer Vision (Nayar, YouTube)

**Planning / control / dynamics**
- UPenn Aerial Robotics (Coursera, Kumar) ⭐ · Underactuated Robotics (Tedrake, free) ⭐ · Planning Algorithms (LaValle, free) ⭐ · Modern Robotics (Lynch & Park, free) · Mellinger & Kumar min-snap paper

**NVIDIA / Isaac ROS / sim**
- Isaac ROS docs (nvidia-isaac-ros.github.io) ⭐ · NVIDIA DLI courses · Gazebo docs · PX4 SITL docs

**Math foundations (only if needed)**
- 3Blue1Brown Essence of Linear Algebra ⭐ · any intro probability (Gaussians, Bayes, covariance)

---

## Start RIGHT NOW (updated 2026-07-30, after Day 4)
1. ⬜ **SIM_WEEK1 Day 5 — occupancy map from depth** via `octomap_server`, then swap `fake_world` out for `/projected_map`. That swap *is* the Phase-1 gate. Both Day-4 blockers are now cleared — the TF tree, and the Gazebo RAM leak (root-caused to the unused 1920×1080 RGB camera, fixed by an overlay model that deletes it) — so the sim holds a flat ~490 MB and a long mapping run is finally practical. Drop an obstacle into the world first; every depth capture so far is over bare ground.
2. ⬜ **Decide the Day-6 VIO front-end** — `rtabmap_ros`, DPVO, or fold it into the offline OpenVINS ladder (item 6). Doesn't block Day 5; `px4_tf_publisher` is the interface contract.
3. ⬜ Verify the two ⚠️ items in BUILD.md §0.5 (Orin Nano not old Nano; Isaac ROS × Orin Nano × Jazzy version). *Open since June — and now **more** urgent, since dropping Isaac ROS from sim means nothing else will force the question before hardware.*
4. ⬜ Order the **RealSense D435i** (supply is thin; long lead item, bench-testable alone). *Open since June.*
5. ⬜ Audit-enroll in the **UPenn Aerial Robotics** Coursera course — it underpins Phase 1.
6. ⬜ **VIO ladder is behind.** You've used DPVO as a black box for the research track but haven't done steps 1–3 (Labbe's filters → OpenVINS on EuRoC → your own toy VI-EKF). That ladder *is* the career deliverable; the sim work won't produce it for you — and now that cuVSLAM is out of the sim path, even less of it comes for free.

*(Done: all code under git and **pushed** — lab notebook to `GPS-Denied-Drone`, both working trees merged into the private `GPS_Denied` repo; SIM_WEEK1 Days 1–4; the Day-5 perception-path decision, recorded as BUILD.md §0.6; both Day-4 blockers — the TF conflict and the Gazebo RAM leak — found and fixed.)*

> Don't buy the rest of the BOM until the Phase-1 sim gate passes.
