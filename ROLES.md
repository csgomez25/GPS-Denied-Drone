# Team Roles & Interfaces

> **Your role decision:** you LEAD **State Estimation / VIO** and own the **autonomy-loop integration**, and you **co-own planning**. This broadens you from CV into a localization/autonomy engineer (see ROADMAP.md "personal development track"). You hand off pure object-detection CV — it's resume-redundant for you.
>
> **Team evolution:** **Summer = 2 people** (you = all software/autonomy · friend = structural + electrical). **Fall = 3–4** (add a planning teammate + a control teammate as the course starts).
>
> This doc pins the **interface contracts** between roles — the seams are where modular teams succeed or bleed time.
>
> Companion docs: [README.md](./README.md) · [BUILD.md](./BUILD.md) · [SUMMER.md](./SUMMER.md) · [ROADMAP.md](./ROADMAP.md)

---

## The data flow (who owns each box)

```
         [Friend: Hardware]            [YOU: VIO lead]            [Fall: Planning]        [Fall: Control]
 RealSense ───────────────▶ cam/IMU ─▶ cuVSLAM ─▶ EKF2 ─▶ pose/odom ─▶ nvblox map ─▶ A* ─▶ Path ─▶ traj+track ─▶ PX4
   (mounted, damped,          stream    (calib,    (vision-   + TF        (occupancy   (plan   (way-   (min-snap,
    powered, wired)                      tune)      aided)                  /ESDF)       around  points) tracking)
                                                                                         obstacles)
            └────────────────────────── YOU also own: integration = the message contracts, TF tree, launch, closing the loop ──────────────────────────┘
```

You sit at the **front of the autonomy chain** (localization) and you're the **integrator** who makes the whole chain work end-to-end.

---

## YOU — State Estimation / VIO lead + autonomy-loop integration  (co-own planning)

**Own (primary):**
- **VIO / localization:** integrate + calibrate (Kalibr) + tune **cuVSLAM** on the RealSense; configure PX4 **EKF2** for GPS-denied / vision-aided; feed VIO odometry in; **characterize drift**.
- **Autonomy-loop integration (software architect):** the ROS 2 workspace, node graph, TF tree, launch files, and — most importantly — **owning the message contracts** between every box. You're the person who makes localization → planning → control actually work as one loop on hardware.

**Co-own (shared, mentor the fall teammate):**
- **Planning + obstacle avoidance** — you build it in sim over the summer, then hand the lead to a new teammate in the fall while staying the integration owner.

**Learn:** ROADMAP "personal development track" (VIO ladder: Labbe → OpenVINS/EuRoC → toy VI-EKF → Kalibr → cuVSLAM).
**You produce:** reliable pose/odom + `map→base_link` TF. **You consume:** sensor stream (friend), and in the fall a `nav_msgs/Path` from the planning teammate.

---

## Friend — Hardware: Structural + Electrical  *(confirmed, summer)*

**Own — Mechanical:** X500 assembly; payload mounts (camera, Jetson tray, battery, prop guards); **vibration isolation for camera/IMU** — *the make-or-break detail; bad damping silently wrecks your VIO*; cooling; weight budget; CAD for custom mounts (and the eventual custom-frame stretch).

**Own — Electrical:** power distribution, wiring, soldering, ESC/motor, battery management, telemetry + ELRS RC link, sensor power/data cabling, kill-switch wiring.

**Delivers:** a powered, sensor-mounted, manually-flyable platform with clean vibration isolation. **Needs from you:** compute/sensor specs, mounting + vibration constraints, cabling needs.
**Learn:** Oscar Liang (hardware/soldering/ELRS), PX4 assembly docs, Holybro X500 build guide.

---

## Fall teammate 1 — Planning & Obstacle Avoidance  *(you mentor; co-owned → handed off)*

**Own:** nvblox occupancy/ESDF consumption → A*/D* planning → obstacle inflation → mid-flight replanning → `nav_msgs/Path` output. You'll have a working sim version to hand them as a starting point.
**Consumes:** your pose + TF + map. **Produces:** collision-free `Path` (to control).
**Learn:** ROADMAP Phase-1 planning track (LaValle, nvblox docs).

---

## Fall teammate 2 — Flight Control & Autopilot

**Own:** PX4 firmware config, flight modes, failsafes, **kill switch + geofence**; turning the planner's `Path` into **min-snap trajectories** + **tracking** (owns the `offboard_manager`); PID tuning for the real all-up weight; uXRCE-DDS / MAVLink link health.
**Consumes:** the `Path`. **Produces:** tracked flight.
**Learn:** ROADMAP Phase-1 control track (UPenn Aerial Robotics, min-snap paper, Tedrake), PX4 PID-tuning docs.

---

## Test / Sim / Integration  *(a 4th person, or shared)*

**Own:** Gazebo worlds + obstacle courses; evaluation metrics (success rate, closest-approach margin, replan latency, drift); logging/analysis (ulog + rosbag + PlotJuggler); flight-test safety procedures; documentation.
> If no dedicated person: sim worlds → you; metrics/logging → control teammate; safety procedures → friend.

---

## The seams to nail in writing

**1. Sensor stream (friend ↔ you).** You need tight camera–IMU time-sync + IMU for VIO; the (fall) planning teammate needs depth. Agree **one shared RealSense launch** (owned by you as integrator) and what topics/rates it publishes.

**2. Localization → Planning (you ↔ planning teammate).** You publish pose/odom + `map→base_link` TF in a defined frame (recommend ENU `map`). Pin the topic, frame, and update rate the planner can rely on, plus your **drift characterization** so the planner knows how much to trust the pose over distance.

**3. Planning → Control (planning ↔ control).** The `Path`/trajectory handoff: exact message (`nav_msgs/Path`), frame, waypoint acceptance radius, and mid-flight replan behavior. You arbitrate this as integrator even though you don't own either side.

> Write these contracts down before anyone codes in the fall. 90% of integration pain is an undocumented frame convention or topic mismatch at exactly these seams.

---

## Scaling the team

| Team size | Split |
|---|---|
| **Summer = 2** | **You:** all software (VIO + planning + integration, in sim) · **Friend:** hardware (assembly → manual flight → mounts/vibration). |
| **Fall = 3** | **You:** VIO lead + integration (+ co-own planning) · **B:** planning **+** control · **Friend/C:** hardware + test. |
| **Fall = 4** | **You:** VIO lead + integration · **B:** planning · **C:** control · **Friend/D:** hardware (+ test folded in or a 5th). |

Merge *roles*, never blur *interfaces* — the contracts above hold at every size. Your summer head-start means in the fall you **lead** the software instead of scrambling.
