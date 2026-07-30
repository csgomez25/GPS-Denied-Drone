# Autonomous GPS-Denied Quadrotor — Project Hub

A small autonomous quadrotor that navigates and avoids obstacles **without GPS and fully offline** (all compute onboard), across indoor, outdoor, and under-structure environments. Senior design project; this folder is the planning + lab-notebook home.

> Status (2026-07-30): **sim bring-up, ~60% through the Phase-1 gate.** The
> planning→flight half of the autonomy loop flew autonomously in PX4 SITL on
> 2026-07-13, and the depth camera now streams into ROS 2 (Day 4, 2026-07-30).
> What's left is mapping and VIO. Still no flying hardware — by design.
> See [Where things stand](#where-things-stand).

---

## The plan, in one paragraph

Use the summer (no deadline) to **de-risk and learn**: stand the full autonomy stack up in **simulation** first, learn **VIO** deeply offline, and secure the **long-lead parts** — so the fall course starts past the painful early phase instead of scrambling. Build on the **PX4-documented reference platform** (Holybro X500 V2 + Jetson Orin Nano + RealSense + ROS 2 Jazzy + Isaac ROS cuVSLAM/nvblox) rather than a risky custom airframe, keeping the custom frame + custom flight-controller PCB as gated stretch goals. Prove obstacle avoidance in sim → bring up hardware → close the loop indoors → then outdoors. Ship a defensible deliverable via a **scope ladder** so the project can't fully fail.

---

## Your path (career angle)

You're broadening from **CV → autonomy/localization engineer.** You lead **State Estimation / VIO** + own **autonomy-loop integration**, co-own planning, and hand off pure detection CV. Principle: **learn VIO deeply on your own clock (offline, EuRoC); deliver VIO on the project clock (cuVSLAM, calibrated + tuned).** See [ROADMAP.md](./ROADMAP.md) + [ROLES.md](./ROLES.md).

---

## Team

- **Summer (2):** you = all software/autonomy (in sim) · friend = structural + electrical (assembly → manual flight → mounts + vibration isolation).
- **Fall (3–4):** add a **planning** teammate (you mentor) + a **control** teammate; friend continues hardware. → [ROLES.md](./ROLES.md)

---

## Document map

| Doc | What it's for |
|---|---|
| [BUILD.md](./BUILD.md) | Locked technical decisions, frame/sensor/compute choices, BOM, weight math, risks |
| [SIM_WEEK1.md](./SIM_WEEK1.md) | Day-by-day commands to stand up sim (ROS 2 Jazzy + PX4 SITL + cuVSLAM/nvblox + the Day-7 autonomy loop) |
| [ROADMAP.md](./ROADMAP.md) | Learn-just-in-time roadmap + curated resources + your VIO development track |
| [ROLES.md](./ROLES.md) | Team split, the data flow, and the interface contracts to nail |
| [SUMMER.md](./SUMMER.md) | What you + your friend do *right now*, and what to buy now vs. fall |

### Where the actual work lives (outside this repo)

| Tree | What it is | Last worked |
|---|---|---|
| `~/ws_px4/src/gps_denied_autonomy/` | ROS 2 autonomy nodes — `offboard_manager`, `planner_node`, `astar`, `fake_world`. Runbooks: `README.md`, `SITL_FLIGHT.md` | 2026-07-13 |
| `~/bev_gps_denied/` | The **research** track — semantic-BEV map-matching localization on nuScenes. Runbook: `README.md`, thesis: `GOAL.md` | 2026-07-13 |
| `~/PX4-Autopilot/`, `~/Micro-XRCE-DDS-Agent/` | Upstream deps, built from source | — |
| `~/DPVO/` | Monocular VIO front-end used offline for the research track's real-drift study | 2026-07-07 |

> ✅ **Both working trees went under git on 2026-07-30** — `gps_denied_autonomy`
> at `5aa13a5`, `bev_gps_denied` at `08eda77`. Six weeks of results (the DPVO
> drift study, the causal experiments, the first autonomous SITL flight, the
> Day-4 depth bridge) are now tracked rather than loose on this disk.
> All three repos are **local only — no remotes configured yet.**

---

## Where things stand

### Phase-1 sim gate (SIM_WEEK1) — **~60%**

| Day | Milestone | Status |
|---|---|---|
| 1 | ROS 2 Jazzy | ✅ `/opt/ros/jazzy` |
| 2 | PX4 SITL + Gazebo Harmonic, manual flight | ✅ gz-harmonic 8.12, PX4 builds + flies |
| 3 | Offboard waypoint commanded from ROS 2 | ✅ 2026-07-13 |
| 4 | Depth camera into ROS 2 | ✅ 2026-07-30 — 4 topics at rate, in flight, real TF |
| 5 | Occupancy map from depth | ⬜ next — now **octomap**, not nvblox ([BUILD.md §0.6](./BUILD.md)) |
| 6 | VIO + a drift number | ⬜ not started in sim; front-end undecided |
| 7 | Close the loop | 🟡 **flown on a synthetic map**, not a perceived one |

**The Day-7 loop already flies.** On 2026-07-13 the X500 armed, took off, threaded a
doorway on an A\* path, and landed — 1596 telemetry samples, 32 s, ended at the goal
(NED −0.05, +7.87, −2.98). Evidence + the three integration bugs fixed getting there:
`~/ws_px4/src/gps_denied_autonomy/SITL_FLIGHT.md`.

**Depth is now in ROS 2 (Day 4, 2026-07-30).** Depth image, camera_info, point cloud
and IMU all deliver at rate, captured *during* an autonomous square with a real moving
`map -> base_link -> camera_link` TF. Runbook and numbers:
`~/ws_px4/src/gps_denied_autonomy/DEPTH_SIM.md`.

**What's still missing is mapping and odometry.** The map is `fake_world`'s synthetic
wall, and pose comes from PX4's own perfect sim state. The gate text ("map built live
from depth *and* cuVSLAM producing a validated odometry track") is not met — and per
[BUILD.md §0.6](./BUILD.md) it will be met with **octomap rather than nvblox** in sim,
so the eventual claim must be "architecture proven," not "nvblox validated." Day 5 is
otherwise unblocked: octomap consumes the point cloud the bridge already publishes and
emits the `OccupancyGrid` type `planner_node` already accepts.

**Two blockers found during Day 4.** (1) ✅ *Fixed* — `depth_bridge.launch.py` defaulted
to a placeholder TF that conflicted with `px4_tf_publisher`, giving `camera_link` two
parents; the default is now off, so the normal path is correct with no extra flags.
(2) ⬜ *Open* — `gz sim` leaked to ~30 GB RSS and was OOM-killed twice, the second time
taking the desktop session down. Mapping runs keep the sim alive much longer than a
60 s capture, so this one bites Day 5 harder than it bit Day 4.

### Research track — stage 1 BEV localization — **~70%**

Full pipeline runs end-to-end on all 10 nuScenes mini scenes with real DPVO drift.

- **Met on injected drift:** mean ATE 1.30 → 0.82 m (37%) over 10 scenes. That's the
  stage-1 "definition of done" figure.
- **Not met on real VIO drift** — and that's the honest finding. On real DPVO residuals
  the cross-track matcher *regresses* (0.74 → 0.82 m). A causal experiment refuted the
  first explanation (along-track gating) and isolated a bimodally-aliased cross-track
  measurement. The 2026-07-13 ambiguity gate fixed the alias (phantom floor 0.29 →
  0.03 m) but made the matcher **inert, not helpful** — the gated sweep never flips.
- **Conclusion recorded:** stop tuning cross-track; the path is **2-DOF (along-track +
  heading)**, since real drift is along-dominated (0.57 vs 0.40 m).
- Remaining ~30%: the 2-DOF matcher, then RPE + a paired significance test.

### Everything downstream — **0%**

Phases 2–5 have not begun. No hardware ordered, no funding confirmed, the two ⚠️
BUILD.md §0.5 items still unverified. Overall project ≈ **13%**.

---

## Phases (scope ladder — each is a defensible deliverable)

1. **Sim** — obstacle avoidance working in SITL. ← summer gate, before buying the rest · **~50%**
2. **Hardware bring-up** — X500 manual flight; cuVSLAM + nvblox on the bench with the real RealSense · **0%**
3. **Integration** — closed-loop indoor autonomous obstacle avoidance · **0%**
4. **Outdoor / under-structure** — the GPS-denied "anywhere" demo · **0%**
5. *(stretch)* custom 3D-printed frame · *(stretch)* custom STM32H7 flight-controller PCB · **0%**

---

## Do next (immediate)

1. ⬜ **Push all three repos somewhere off this laptop.** They're now under git but
   have no remotes — one disk failure still costs six weeks. This is the cheapest
   remaining item on this list.
2. ⬜ **Characterise Gazebo's RAM leak** (`DEPTH_SIM.md` §4b) before Day 5 — mapping
   keeps the sim alive far longer than a 60 s capture, and the current ceiling is
   10–20 min. Start by A/B-ing `gz_x500` against `gz_x500_depth`.
3. ⬜ **[SIM_WEEK1](./SIM_WEEK1.md) Day 5 — occupancy map from depth**, via
   `octomap_server` (installed). Drop an obstacle into the world first; every depth
   capture so far is over an empty ground plane, so nothing has been within 3 m of the
   camera. Then unplug `fake_world` and point `planner_node` at `/projected_map`.
4. ⬜ Verify the two ⚠️ items in [BUILD.md §0.5](./BUILD.md) (Orin Nano, not old Nano; Isaac ROS × Orin Nano × Jazzy support). *Still open since June.*
5. ⬜ Confirm the **funding source** (reimbursable over summer or only in fall?) → finalizes the buy list. *Still open since June.*
6. ⬜ Order the **RealSense D435i** (thin stock, long lead, bench-testable alone). *Still open since June.*
7. ⬜ **Research track:** build the 2-DOF (along-track + heading) matcher — the cross-track
   line is closed out with evidence. See `~/bev_gps_denied/README.md`.
8. ⬜ Audit-enroll in **UPenn Aerial Robotics** (Coursera) + begin the VIO ladder in [ROADMAP.md](./ROADMAP.md).

> Don't buy the rest of the BOM until the Phase-1 sim gate passes (except the long-lead RealSense/Jetson).

---

## Progress log

| Date | Track | What happened |
|---|---|---|
| 2026-07-13 | drone | **First fully autonomous end-to-end SITL flight** — synthetic map → A\* → offboard, no human input. Three integration bugs fixed (topic-name drift, periodic-replan stall, PX4 arming params). → `SITL_FLIGHT.md` |
| 2026-07-13 | research | Ambiguity gate fixed the one-lane alias but made cross-track matching **inert**. Cross-track-only line closed out; 2-DOF is the path. |
| 2026-07-11 | research | Causal experiment (`dpvo-cross` / `dpvo-along`) **refuted** the along-track-gating hypothesis; isolated a biased/aliased cross-track measurement. |
| 2026-07-11 | research | Stage 1b — DPVO wired in for **real** VIO drift on all 10 scenes, replacing the injected random walk. Negative result: matcher regresses (0.74 → 0.82 m). |
| 2026-07-08 | drone | `planner_node` + `astar` + `fake_world` built; A\* validated headlessly and on real nuScenes HD-map rasters. |
| 2026-07-07 | research | Semantic gap closed + filtered fusion → mean ATE 1.30 → 0.82 m over 10 scenes on injected drift. |
| 2026-06-02 | plan | Project plan written (this folder). |
