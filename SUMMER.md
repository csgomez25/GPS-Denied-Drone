# Summer Pre-Work Plan (get-ahead, before the fall timeline)

> Situation (June 2026): no course deadline yet. **2-person summer crew** — you (all software/autonomy) + a friend (structural + electrical). 1–2 more teammates expected when the course starts in fall.
>
> Goal of the summer: **de-risk + learn + secure long-lead parts** so that when the course starts you're past the slow, painful early phase instead of rushing for parts and fighting toolchains. Not "have a flying autonomous drone by August" — that's not the goal and would be a trap.
>
> Companion docs: [BUILD.md](./BUILD.md) · [SIM_WEEK1.md](./SIM_WEEK1.md) · [ROADMAP.md](./ROADMAP.md) · [ROLES.md](./ROLES.md)

---

## Why this timing is good
- **Summer load is fine even though you're doing all the software solo** — because it's *sim + learning + bench work*, not a graded deliverable on a clock. Doing the whole software stack yourself first is the **best possible autonomy-engineer education**: you'll understand every box before you hand some off in the fall.
- The slowest, most frustrating parts (toolchain setup, VIO learning curve, long-lead/low-stock parts) are exactly the things worth front-loading into a no-pressure summer.

---

## YOU this summer — software + VIO ramp
Run these roughly in parallel:

1. **Sim bring-up** — work through [SIM_WEEK1.md](./SIM_WEEK1.md). Target: the Day-7 autonomy loop (depth → nvblox → A* → offboard) flying around an obstacle in sim. *This is the Phase-1 gate.*
2. **VIO learning track** (your priority skill) — start the offline ladder from ROADMAP:
   - Labbe's Kalman/Bayesian Filters → run **OpenVINS on the EuRoC dataset** → implement a **toy VI-EKF on EuRoC** offline.
   - Get **cuVSLAM running in sim**, then on the **bench** once the RealSense + Jetson arrive.
   - Learn **Kalibr** (camera–IMU calibration) — you'll use it for real in the fall.
3. **Start the lab-notebook git repo** (this folder is a good seed) — log every working command + failure.

**Your summer "done":** sim autonomy loop works; you can explain VIO and have cuVSLAM running in sim + on the bench; EuRoC toy filter implemented.

---

## YOUR FRIEND this summer — structural + electrical
Maps to the Hardware role in ROLES.md (Teammate C). Concrete tasks:

1. **Assemble the Holybro X500 V2** — frame, motors, ESC, props, power module.
2. **Flash PX4, calibrate, and get stable MANUAL flight** — props on, open/safe area. This proves the airframe independently of any autonomy.
3. **Design + print the payload mounts** — camera mount, Jetson tray, battery cradle, prop guards. **Vibration isolation for the camera/IMU is the make-or-break detail** (bad damping silently wrecks your VIO — flag this to him as a first-class requirement, not an afterthought).
4. **Wiring + power budget** — clean cabling, connectors, kill-switch wiring, telemetry + ELRS RC link, sensor power/data routing.
5. **Learn:** Oscar Liang (hardware/soldering/ELRS), PX4 assembly docs, Holybro X500 build guide.

**Friend's summer "done":** X500 flies stably under manual control, with mounts + vibration isolation ready to accept the camera/Jetson.

---

## What to buy NOW vs. wait for fall

> ### ✅ RESOLVED 2026-07-31 — and it went the other way
>
> **Parts choices are ~90% confirmed (Orin Nano Super + RealSense D435i), but the
> budget does not unlock until the course starts.** Nothing can be ordered this summer.
>
> That inverts the plan below, and it changes what the rest of the summer is *for*.
> The binding constraint is no longer money — it is **your time in the fall**, which
> will be consumed by assembly, sensor bring-up, calibration, cuVSLAM integration,
> EKF2 fusion, flight tests, *and* mentoring two new teammates. There is no room in
> that for learning curves.
>
> **So the remaining summer has exactly one job: arrive in the fall with the learning
> curves already climbed.** Three things, all of which are offline and cannot be
> blocked by parts — see "Revised priorities" below.
>
> Note this also means **the Phase-1 sim gate no longer gates anything.** It existed to
> decide "is it safe to buy hardware?", and budget already decided that. Finish it for
> the milestone and the report, but do not let it crowd out the three items below.

### Revised priorities (2026-07-31, hardware budget-locked)

1. ⭐ **The VIO ladder** (ROADMAP steps 1–3: Labbé's filters → OpenVINS on EuRoC → a
   toy VI-EKF). This is the career deliverable, it is entirely offline, and **the fall
   has no room for it**. Partial infrastructure already exists — `~/DPVO` has
   `evaluate_euroc.py` and EuRoC logs. Highest-value hours available.
2. ⭐ **De-risk Isaac ROS now, on the laptop.** Every decision so far has deliberately
   deferred it (octomap instead of nvblox, rtabmap instead of cuVSLAM), and BUILD §0.6
   warns it is "unverified for longer." The dev box can run it: **RTX A3000 6 GB,
   driver 595.71, 308 GB free**; Docker and the NVIDIA Container Toolkit are *not*
   installed. Install them, pull the container, run the cuVSLAM quickstart on a
   dataset. That validates the workflow, the ROS interfaces and the tuning surface —
   most of which transfers to the Jetson. **If the Isaac ROS × Jazzy × Orin Nano matrix
   turns out to need Humble, finding out in July is cheap and finding out in October
   with parts on the bench is not.**
3. **Practice Kalibr on a dataset.** Camera–IMU calibration is the one thing sim
   structurally cannot teach — in sim the extrinsic is exact and free; on hardware it
   is vibration, thermal drift and a fiddly toolchain. EuRoC ships calibration data.
4. Close the Phase-1 gate (an afternoon: point `planner_node` at `/projected_map`).
5. Research track: the 2-DOF along-track + heading matcher.

**Stop:** tuning rtabmap ICP. It does not ship (cuVSLAM does), it cannot test the hard
part, and it has already produced its honest finding (`VIO.md` §3b).

---

> ⚠️ **Original buy guidance, kept for the fall.** Confirm the money source; if the
> budget reimburses only from course start, don't float ~$1,100 out of pocket.

**Buy now (long-lead / unblocks summer work):**
- ⭐ **RealSense D435i** — thin stock post-spinout, long lead, and you can bench-test it independently. **Highest priority, buy first regardless.**
- **Jetson Orin Nano (Super)** — unblocks your cuVSLAM bench work. *(Re-verify the Isaac ROS × Orin Nano × Jazzy support matrix before ordering — BUILD.md §0.5.)*
- **Holybro X500 kit + Pixhawk 6C + battery + charger + RC** — unblocks your friend's assembly + manual flight. The platform is committed, so this is low-risk to buy early.

**Wait for fall (cheap, or depends on findings):**
- Spares/contingency, prop guards, netting (likely lab-provided), and anything you'd only spec after the sim phase.

---

## The fall handoff (plan for the team growing to 3–4)
By fall you'll know the whole stack cold, which makes you the natural **software/autonomy lead**. When the 1–2 new teammates join:

- **You keep:** State Estimation / **VIO lead** + autonomy-loop **integration** (your growth target).
- **Hand off:** **planning + obstacle avoidance** to a new teammate (you mentor them — you'll have built it in sim), and **flight control / PX4 tuning** to another.
- **Friend continues:** hardware (mechanical + electrical), now adding the real camera/Jetson integration.

This sets up the exact ROLES.md split, but with you having a summer head-start that lets you lead instead of scramble.

---

## Summer milestones (rough)
*(Original plan — superseded 2026-07-31 by the budget lock; kept for comparison.)*
- **Weeks 1–3:** toolchain + SIM_WEEK1 through Day 4; friend starts X500 assembly; RealSense + Jetson ordered.
- **Weeks 4–6:** sim autonomy loop closed (Phase-1 gate); friend reaches manual flight; OpenVINS-on-EuRoC running.
- **Weeks 7–10:** cuVSLAM in sim + on bench with real RealSense; toy VI-EKF; friend finishes mounts + vibration isolation.
- **End of summer:** sim demo + bench VIO working; airframe flight-ready.

**Revised end-of-summer target (no hardware available):** SIM_WEEK1 Days 1–5 done ✅,
Phase-1 gate closed, **OpenVINS running on EuRoC with a drift number you understand**,
**Isaac ROS + cuVSLAM demonstrated in a container on the laptop**, and Kalibr practised
on a dataset. Bench work moves to the fall, but arrives with no learning curve attached.
