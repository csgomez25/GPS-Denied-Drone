# Autonomous GPS-Denied Quadrotor — Design Review, Decisions & Build Plan

> Working consolidation of the proposal critique + the build path forward.
> Last updated: 2026-06-02
> Companion docs: [SIM_WEEK1.md](./SIM_WEEK1.md) (week-1 sim commands) · [ROADMAP.md](./ROADMAP.md) (learning + build roadmap) · [ROLES.md](./ROLES.md) (team split) · [SUMMER.md](./SUMMER.md) (get-ahead pre-work plan)

---

## 0. North Star vs. Deliverable

**Vision (motivation):** a drone that flies autonomously *anywhere* — GPS-denied, fully offline (no internet), indoor or outdoor.

**Deliverable (testable spec):**
> Autonomous waypoint navigation + obstacle avoidance across GPS-denied **indoor**, **outdoor**, and **transition (under-structure / urban-canyon)** environments, validated on a defined set of representative courses, with all perception/estimation/planning running **onboard with no network dependency**.

Keep the dream in the intro; keep the graded thing measurable.

---

## 0.5 Decisions Locked (2026-06-02)

| Area | Choice | Rationale |
|---|---|---|
| **Companion computer** | **Jetson Orin Nano (Super)** — *NOT the old 2019 Nano* | CUDA enables the Isaac ROS autonomy stack; deletes the need for a separate AI accelerator. |
| **AI accelerator** | **None (Hailo dropped)** | Jetson GPU + TensorRT does inference. One pipeline instead of Pi+Hailo+separate VIO. |
| **Depth/VIO sensor** | **RealSense D435i** | NVIDIA's reference sensor for cuVSLAM; best Isaac ROS integration; hardware-synced IMU. OAK-D's onboard-AI edge is moot once a Jetson is present. |
| **ROS distro** | **ROS 2 Jazzy** (already in use) | Isaac ROS now targets Jazzy and runs in Docker containers on Jetson, decoupling it from JetPack's Ubuntu 22.04 host. |
| **Autopilot** | **PX4 (lean)** — *can finalize in sim* | Best offboard + VIO docs. Most decoupled choice: it just receives vision odometry over uXRCE-DDS. ArduPilot equally viable. |
| **Frame** | **Holybro X500 V2** | PX4 reference platform; carries the payload; custom airframe is a stretch goal. |

> **Status 2026-07-31.** Both choices are **~90% confirmed** (Orin Nano Super +
> RealSense D435i), but the **budget does not unlock until the course starts**, so
> nothing can be ordered this summer. That does *not* excuse item 2 below: the Isaac
> ROS × Orin Nano × Jazzy support matrix is a **documentation check, not a purchase**,
> and if it comes back "Humble only" it invalidates a lot of Jazzy-specific work. Do it
> now, folded into the Isaac ROS container de-risk in SUMMER.md. Item 1 (confirming the
> board is an *Orin* Nano) can wait for the order.

### ⚠️ Two things to verify BEFORE buying
1. **Confirm the Jetson is an *Orin* Nano.** The original 2019 Jetson Nano maxes at Ubuntu 20.04, is EOL, has no Isaac ROS support, and is too weak for VIO + mapping. Dead end.
2. **Pin the Isaac ROS release that supports Orin Nano + Jazzy.** Newer Isaac ROS releases prioritize Jetson Thor / x86; Orin Nano is supported but verify the exact version matrix, or you may be forced back to Humble (which conflicts with the Jazzy preference).

### Resulting stack
`RealSense D435i → Isaac ROS cuVSLAM (VIO) + nvblox (occupancy map) + TensorRT (any NN) on Jetson Orin Nano (Jazzy, containerized) → PX4 vision-aided EKF2 → offboard control, on a Holybro X500 V2.`

### 0.6 Amendment (2026-07-30) — **octomap in sim, Isaac ROS on the Jetson**

**Decision:** the laptop sim path uses **`octomap_server`** for occupancy mapping.
Isaac ROS (nvblox + cuVSLAM) stays the **flight** stack on the Jetson, unchanged.
The §0.5 hardware decisions above are **not** affected.

**Why.** Isaac ROS ships only as NVIDIA Docker containers — nvblox and cuVSLAM are
pinned to specific CUDA/TensorRT/cuDNN versions and are not published as apt debs.
Standing that up on the dev laptop means installing Docker + the NVIDIA Container
Toolkit (neither present) and running a GPU mapper on a **6 GB A3000 that already
got the Gazebo GUI OOM-killed** during the Day-3 flight. That is days of yak-shaving
and an unresolved VRAM risk, spent on tooling that gets thrown away the moment the
Jetson arrives. `octomap_server` is one `apt install`, CPU-only, and needs no Docker
at all.

**What it buys immediately.** `octomap_server` publishes `/projected_map` as a
`nav_msgs/OccupancyGrid` — the exact type `planner_node` already consumes. So
`fake_world`'s synthetic grid can be unplugged and replaced by a map built from the
depth camera **with no planner changes**, which is the Phase-1 gate.

**What it costs — accept this consciously.**
- octomap is an occupancy octree, **not** nvblox: no ESDF/TSDF, no GPU, different
  parameters, different failure modes. Phase-1 tuning numbers will **not** transfer
  to the Jetson; expect to re-tune against nvblox in Phase 2.
- It proves the *architecture* (depth → map → plan → fly), not the *implementation*
  you will fly. That is the right thing to prove first, but don't oversell it in the
  report as "nvblox validated."
- Isaac ROS × Orin Nano × Jazzy (§0.5 ⚠️ item 2) is now **unverified for longer**,
  because sim no longer forces the question. **Verify it independently** — it is
  still a hard gate before hardware.

**Consequence for pose.** `octomap_server` does not estimate pose; it needs a TF from
the map frame to the sensor. nvblox had the same requirement, with cuVSLAM supplying
it. In sim, PX4's own state supplies it instead (`px4_tf_publisher`), which SIM_WEEK1
Day 6 explicitly sanctions ("let SITL use its own state, run VIO in parallel"). That
node is the **interface contract**: VIO later replaces PX4 state behind the same
`map → base_link → camera_link` TF, and nothing downstream changes.

**Day-6 VIO in sim — decided 2026-07-30: `rtabmap_ros`**, installed. Same reasoning as
the mapper: apt-installable, no Docker, no GPU contention. Two constraints shaped it.
First, the overlay model that fixes the RAM leak **deletes the RGB camera**, so no
feature-based front-end has an image to work with — `icp_odometry` on the depth cloud
does not need one. Second, rtabmap is a *SLAM* system, so it must run with **loop
closure off**: loop closure corrects exactly the drift Day 6 exists to measure.

**Timebox it (2026-07-31).** Day 6 produced a running pipeline and two honest blockers
(`VIO.md` §3b): the "ground truth" is EKF2's own estimate, and ICP registers nothing.
Worth one more session — real ground truth from a Gazebo `PosePublisher`, one attempt
at the `ratio=0` cause — then move on regardless of the outcome. rtabmap does **not**
ship; cuVSLAM does. Days spent tuning ICP buy nothing that transfers, and the sim
cannot test the part that actually matters (camera–IMU calibration). With hardware
budget-locked until the fall, that time is far better spent de-risking Isaac ROS itself
and climbing the VIO ladder — see SUMMER.md "Revised priorities".

Same caveat as octomap-vs-nvblox, and it should be stated the same way in the report:
this validates the **pipeline**, not cuVSLAM. It also validates less than it appears
to — the sim's camera-IMU extrinsic is exact and known, whereas on hardware calibrating
it (Kalibr, vibration, thermal drift) is the actual work. DPVO stays on the research
track; the offline OpenVINS/EuRoC ladder stays the career deliverable.

**Confirmed later the same day (2026-07-30).** SIM_WEEK1 Day 4 passed, and every input
octomap needs is live: `/depth_camera/points` at 24.5 Hz, `/clock`, and a real
`map -> base_link -> camera_link` from `px4_tf_publisher`. `ros-jazzy-octomap-server`
and `ros-jazzy-octomap-rviz-plugins` are installed. So this decision is not just
cheaper on paper — the whole input side of it is already working.

One caveat surfaced the same session and was then **resolved**: `gz sim` grew at a flat
180 MB/s and was OOM-killed twice, the second time taking the desktop session down.
Root cause was the OakD-Lite's **unused 1920×1080 RGB camera**, which Gazebo renders
unconditionally — not scene complexity and not VRAM. An overlay model deletes that one
sensor and the sim now holds ~490 MB indefinitely, or 653 MB in the 37-tree forest
world (`DEPTH_SIM.md` §4b). Note this was **system RAM**, a separate limit from the
6 GB VRAM argued above; the two have confusingly similar symptoms.

The general lesson is worth carrying to the Jetson, where headroom is far tighter:
**an unused sensor still costs full price.** Nothing subscribed to that RGB topic and
`always_on=0` did not stop it either — a declared sensor renders regardless. Delete
what you do not need from the model rather than leaving it unsubscribed.

---

## 1. Revised Technical Decisions (what changed after review)

### 1.1 Perception — depth-first, not detector-first  ⬅ biggest change
- **Obstacle avoidance must run on geometry (depth), not object class.** YOLO only detects trained classes; it cannot see a net post, wire, pole, or arbitrary inserted obstacle. The mission explicitly uses obstacles "without prior knowledge," so a class detector is the wrong primary tool.
- **Primary pipeline:** RealSense depth → **nvblox** (GPU occupancy/ESDF map on the Jetson) → planner. nvblox is purpose-built for this and runs on-GPU.
- **No separate accelerator** (Hailo dropped). If you want a neural net at all, run it via **TensorRT on the Jetson GPU** for a real job — semantic labeling of waypoint targets, or a "don't fly near people" safety layer. It is **not** the obstacle detector.

### 1.2 State estimation — one estimator owns flight-critical state
- **Drop the redundant "outer EKF."** VIO already fuses the IMU; re-fusing the same IMU downstream double-counts and over-confidences the estimate.
- **Canonical pattern:** `VIO (Isaac ROS cuVSLAM, GPU-accelerated) → MAVLink VISION_POSITION_ESTIMATE / ODOMETRY → PX4 EKF2 (configured GPS-denied / vision-aided)`. (CPU fallback if Isaac ROS is unavailable: OpenVINS / VINS-Fusion.)
- **Drift is real:** VIO is odometry, not SLAM (no loop closure). Position drifts over distance/time. Keep courses short, state a drift budget, or add fiducials (AprilTags at known spots) / mocap for ground truth.

### 1.3 Indoor ≠ Outdoor — this roughly doubles validation work
- **Exposure transitions** (sun → under-building) are a classic VIO failure point; auto-exposure lag drops feature tracking exactly at the transition.
- **D435i depth degrades outdoors** (>~2–3 m): the IR projector is washed out by sun, so you fall back to passive stereo (texture-dependent). Indoors the projector helps.
- **Feature distance/scale:** outdoor features are far → weak parallax → weaker VIO; indoor features are close → rich parallax. Tuning may not transfer between the two.
- **Wind:** outdoor disturbance the indoor tests never see; stresses controller + estimator together.
- **GPS is a confounder outdoors:** "GPS-denied" outdoors must be *simulated* by disabling the fix in EKF2 — verify it's actually off or you'll fool yourself.

### 1.4 "No internet" is free — claim it as a feature
- Onboard autonomy never needs a network. The only internet touch-points are bench-time package/model downloads and a *local* RF telemetry link. State explicitly: **"fully offline, all compute onboard."**

### 1.5 Scope ladder (so the project can't fully fail)
1. Sim demo of obstacle avoidance (SITL).
2. Hardware holds VIO position hover (no drift blowup).
3. Waypoint following, no obstacles.
4. Obstacle avoidance, indoor.
5. Obstacle avoidance, outdoor / under-structure.  ← stretch
Custom STM32H7 flight-controller PCB stays a stretch goal, gated on (4) being solid.

---

## 2. Platform Strategy — buy the de-risked reference, customize later

Your payload is **not small**: depth camera + Pi 5 + accelerator + battery is ~250–350 g of gear. That forces a real aircraft and kills the original 500–650 g / 4" target (see §5). Two honest realities:

1. A 4" custom 3D-printed frame **cannot** carry this payload with usable thrust margin or flight time.
2. The fastest path to a *working autonomy stack* is the platform PX4 already documents for exactly this — companion computer + depth camera + VIO + offboard.

**Recommendation:** build the main system on the **Holybro X500 V2** (500-class, carbon arms, Pi/Jetson mounting plate, optional depth-camera mount, PX4 reference docs). Treat **custom 3D-printed airframe** the same way you treat the custom flight controller: a *stretch* once the autonomy stack works on known-good hardware. Don't fight airframe bugs and VIO bugs at the same time.

### Frame options (trade space)

| Option | Props | Pros | Cons | Verdict |
|---|---|---|---|---|
| **Holybro X500 V2** (recommended) | 10" | PX4-documented, easy payload room, stable, good wind handling, depth-cam + companion mounts exist | 10" props dangerous indoors → need large net/cage + prop guards; ~610 mm | **Main build** |
| 7" custom (carbon arms + printed plates) | 7" | Lighter, better indoor safety than 10", decent payload | Cramped layout, more design work, less documented | Good middle ground if you want "custom" sooner |
| 5" build | 5" | Agile, cheapest, safest indoors | Marginal payload + very short flight time with this gear | Only if payload shrinks (e.g. OAK-D + no Hailo) |
| Full custom 3D-printed | — | Your original goal | Pure-PETG arms fail; needs carbon arms + PA-CF plates | **Stretch goal** |

**If/when you do go custom:** don't 3D-print the load path. Use carbon-fiber tube arms + carbon center plates; 3D-print *only* the camera mount, companion-computer tray, battery cradle, and prop guards — and print those in **PA-CF (carbon-filled nylon)** or at least PETG, not PLA.

---

## 3. Sensor Decision — RESOLVED: RealSense D435i

With a Jetson doing CUDA inference, the OAK-D's only real advantage (onboard AI) is moot. cuVSLAM and nvblox are tuned around the RealSense, and the D435i's hardware-synced IMU is what VIO wants.

| | RealSense D435i ✅ | Luxonis OAK-D Pro |
|---|---|---|
| VIO suitability | **Best** — synced IMU, NVIDIA's reference cuVSLAM sensor | Weaker IMU sync historically; VIO harder |
| Outdoor depth range | Degrades past ~2–3 m in sun | Better (longer usable range) |
| Onboard AI | N/A — Jetson does inference | Yes (but redundant once you have a Jetson) |
| Supply (2026) | **Thin** — post-Intel-spinout, low stock | Generally available |
| Cost | ~$330 | ~$250–400 |

**Action:** **Buy the D435i early** — supply is tight post-spinout. The outdoor depth-range limit is a known constraint to design the outdoor test course around (keep obstacles within usable range, lean on cuVSLAM tracking for state), not a reason to switch sensors.

---

## 4. Starter Bill of Materials (Phase 1–2)

> Ballpark USD, mid-2026. Single resolved path (Jetson + RealSense, no Hailo). Order-of-purchase notes below.

| Item | Example | ~$ | Notes |
|---|---|---|---|
| Frame kit | Holybro X500 V2 (frame + motors + ESC + props) | 220 | Or X500 V2 full kit incl. Pixhawk |
| Flight controller | Pixhawk 6C (or 6C Mini) | 200 | PX4 reference FC; far better for VIO/offboard than a racing F405 |
| Power module / PDB | Holybro PM02/PM03 | incl/30 | Voltage + current telemetry |
| Companion computer | **Jetson Orin Nano (Super) dev kit** + NVMe + active cooling | 250 | ⚠️ Orin Nano, **not** the old 2019 Nano. Carries cuVSLAM + nvblox + TensorRT. |
| ~~AI accelerator~~ | ~~Hailo~~ — **dropped** | 0 | Jetson GPU does inference |
| Depth/VIO camera | RealSense D435i | 330 | **Buy early — low stock** |
| Battery | 4S 5000 mAh Li-ion or 4S LiPo ×2 | 80 | Li-ion for endurance, LiPo for punch |
| Charger | ISDT / hota balance charger | 60 | |
| RC link | ELRS TX module + RX | 70 | Manual override is mandatory for testing |
| Telemetry | SiK 915 MHz pair or ELRS backpack | 40 | Local RF, not internet |
| GPS (safety only) | M10 GPS (return-to-home outdoors) | 30 | Not used for autonomy |
| Spares | props ×N, 1 motor, 1 ESC | 60 | |
| **Crash-replacement reserve** | **toward a 2nd camera/Jetson** | **150+** | The expensive parts ride a crashing aircraft |

**Rough total: ~$1,090–1,290.** (Dropping the Hailo roughly offsets the Orin Nano costing more than a Pi 5.)

### Not on your original list but needed
- **RC transmitter** (e.g. RadioMaster) if you don't own one — ~$80–200
- **Ground-station laptop** running QGroundControl (probably already have)
- **Indoor safety enclosure / net** — price/borrow this; 10" props need real containment
- **Prop guards** for indoor flight

---

## 5. Weight & Thrust Reality (do this math before buying motors)

Rough all-up estimate with the Path A payload:

| Group | ~grams |
|---|---|
| Frame (X500-class) | 350 |
| Motors ×4 + ESC | 200 |
| Battery (4S) | 250–350 |
| Depth camera | 75 |
| Jetson Orin Nano + carrier + cooling + mounts | 180 |
| FC + PDB + GPS + RX + wiring | 90 |
| **All-up** | **~1,100–1,300 g** |

- This is **~2× your original 500–650 g target.** The 4" frame was never going to work.
- Target **thrust-to-weight ≥ 2:1** (so ≥ ~2.6 kg total thrust). X500-class 10" props on 2216-class motors hit this comfortably; a 4–5" build does not with this payload.
- **Expect short flights** regardless — budget Li-ion packs for endurance and plan test sessions around battery swaps.

---

## 6. First Steps — what to actually do now

**Do not buy the full BOM yet.** Order of operations that de-risks spend:

0. **Verify the two ⚠️ items in §0.5 first** (Orin Nano not old Nano; Isaac ROS × Orin Nano × Jazzy version matrix). These gate everything below.

1. **Stand up simulation first (Week 1–2, ~$0).** → see **[SIM_WEEK1.md](./SIM_WEEK1.md)** for the day-by-day checklist.
   - Install PX4 + Gazebo SITL + ROS 2 **Jazzy**. (PX4↔ROS link is uXRCE-DDS, distro-agnostic; Jazzy works even though PX4 docs lean Humble.)
   - Bridge the simulated X500 + depth camera into ROS 2; get a depth → nvblox occupancy → A* → offboard loop flying a waypoint in sim.
   - This validates the whole stack with zero crash risk and proves the architecture before hardware.

2. **First hardware buy = camera + FC + Jetson (long-lead / low-stock items):**
   - RealSense D435i (supply risk — order first), Pixhawk 6C, Jetson Orin Nano. Bring up cuVSLAM + nvblox on the bench, off the aircraft.

3. **Second buy = airframe + power + RC** once the sim loop works and VIO tracks on the bench.

4. **Build order on hardware:** manual flight first (prove the airframe) → VIO position hold → waypoints → obstacle avoidance indoor → outdoor / under-structure.

---

## 7. Open Decisions (remaining)

**Resolved:** compute (Jetson Orin Nano), accelerator (none), sensor (D435i), ROS distro (Jazzy), frame (X500 V2). Autopilot leaning PX4, finalize in sim.

**Team (known):** Summer = 2 (you = software/autonomy, leading VIO + integration; friend = structural + electrical). Fall = 3–4 (add planning + control teammates). See [ROLES.md](./ROLES.md) and [SUMMER.md](./SUMMER.md).

Still open:
- [ ] **Verify** Orin Nano (not old Nano) + Isaac ROS Jazzy support matrix — see §0.5 ⚠️.
- [ ] **Confirm the funding source** — is the parts budget reimbursable over summer, or only once the course starts? (Drives buy-now vs wait — SUMMER.md.)
- [ ] Indoor test space: netted/enclosed area sized for a 500-class drone with 10" props? (Drives prop-guard / cage needs.)
- [ ] Do you already own an RC transmitter / charger / bench supply?
- [ ] Sim: Gazebo (light, standard) vs Isaac Sim (heavier, but matches the NVIDIA stack and gives photoreal depth — you already have a Jetson workflow).

---

## Sources
- [RealSense D435i product page (RealSense, Inc.)](https://www.realsenseai.com/products/depth-camera-d435i/)
- [Tangram Vision — keeping RealSense sensors alive / spinout context](https://www.tangramvision.com/blog/how-to-keep-your-realsense-sensors-alive)
- [Luxonis OAK vs RealSense comparison](https://docs.luxonis.com/hardware/platform/comparison/vs-realsense)
- [OAK-D Pro vs RealSense D435i](https://discuss.luxonis.com/blog/1303-oak-d-pro-vs-intel-realsense-d435i)
- [Holybro X500 V2 + Pixhawk 6C — PX4 Guide](https://docs.px4.io/main/en/frames_multicopter/holybro_x500v2_pixhawk6c.html)
- [PX4 Development Kit X500 v2 — Holybro](https://holybro.com/products/px4-development-kit-x500-v2)
- [ArduPilot — Intel RealSense depth camera docs](https://ardupilot.org/copter/docs/common-realsense-depth-camera.html)
- [Isaac ROS Visual SLAM (cuVSLAM) — supported platforms/distros](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_visual_slam/index.html)
- [Isaac ROS Nvblox](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/index.html)
- [Isaac ROS on Jetson Orin Nano Super — practical guide (JetPack/ROS notes)](https://thomasthelliez.com/blog/isaac-ros-on-nvidia-jetson-orin-nano-super/)
- [PX4 uXRCE-DDS (ROS 2 bridge)](https://docs.px4.io/main/en/middleware/uxrce_dds)
