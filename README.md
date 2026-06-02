# Autonomous GPS-Denied Quadrotor — Project Hub

A small autonomous quadrotor that navigates and avoids obstacles **without GPS and fully offline** (all compute onboard), across indoor, outdoor, and under-structure environments. Senior design project; this folder is the planning + lab-notebook home.

> Status: **planning / summer pre-work** (June 2026). No flying hardware yet — by design.

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

---

## Phases (scope ladder — each is a defensible deliverable)

1. **Sim** — obstacle avoidance working in SITL. ← summer gate, before buying the rest
2. **Hardware bring-up** — X500 manual flight; cuVSLAM + nvblox on the bench with the real RealSense
3. **Integration** — closed-loop indoor autonomous obstacle avoidance
4. **Outdoor / under-structure** — the GPS-denied "anywhere" demo
5. *(stretch)* custom 3D-printed frame · *(stretch)* custom STM32H7 flight-controller PCB

---

## Do next (immediate)

1. ⬜ `git init` this folder as the lab-notebook repo.
2. ⬜ Verify the two ⚠️ items in [BUILD.md §0.5](./BUILD.md) (Orin Nano, not old Nano; Isaac ROS × Orin Nano × Jazzy support).
3. ⬜ Confirm the **funding source** (reimbursable over summer or only in fall?) → finalizes the buy list.
4. ⬜ Order the **RealSense D435i** (thin stock, long lead, bench-testable alone).
5. ⬜ Start [SIM_WEEK1.md](./SIM_WEEK1.md) Day 1; friend starts the X500 build ([SUMMER.md](./SUMMER.md)).
6. ⬜ Audit-enroll in **UPenn Aerial Robotics** (Coursera) + begin the VIO ladder in [ROADMAP.md](./ROADMAP.md).

> Don't buy the rest of the BOM until the Phase-1 sim gate passes (except the long-lead RealSense/Jetson).
