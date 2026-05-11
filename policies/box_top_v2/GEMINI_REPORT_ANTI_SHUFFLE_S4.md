# Gemini Report: S4 Anti-Shuffle Strategy & Experiments
**Date**: 2026-05-11
**Author**: Gemini CLI

## 1. Objective
The goal of this phase (Curriculum Stage S4) was to break a high-frequency "shuffle" exploit discovered in the V2.5 PoseGaitQuality seed (`model_648.pt`). The robot was satisfying forward velocity commands with ~7-8Hz micro-steps (approx. 5-7mm per step) instead of taking real walking steps.

## 2. Methodology: Structural Reward Overhaul
To break the basin, we moved beyond simple weight scaling to structural changes:

*   **Cadence-Gated Speed Reward**: Added `cadence_gated_track_lin_vel_xy_exp`. The velocity tracking reward is multiplied by a factor that drops to zero if cadence exceeds a target threshold (initially 4Hz). This makes the shuffle exploit economically "un-profitable."
*   **Continuous Swing Clearance**: Added `continuous_swing_clearance_reward` (target z = 0.005m). Provides a non-sparse, dense signal for lifting feet during the swing phase.
*   **Successive Gating Curriculum**: Instead of a hard 2Hz jump, we implemented a curriculum that nudges the gate lower (7Hz -> 5Hz -> 4Hz -> 3.5Hz) to provide a reachable learning gradient.
*   **Lateral Weight-Shift Oscillation**: Added a reward for lateral CoM oscillation (1.0 Hz) to force the robot to shift weight and lift legs.

## 3. Experimental Branches (Summary of Recent Progress)

| Branch | Strategy | Cadence (Hz) | Step Advance (m) | Result |
| :--- | :--- | :--- | :--- | :--- |
| **G** | 4.3Hz Nudge | 0.88 | -0.0028 | **BASIN BROKEN**: Shuffle stopped, but robot is "marching in place" (zero speed). |
| **H** | Oscillation + W:400 | ~1.50 | **+0.0030** | **DISCOVERY**: Real forward steps found, but 100% fall rate. |
| **I/J** | Survival/Zero Speed | N/A | N/A | **FAIL**: 100% Fall Rate. High instability when attempting to step. |
| **K/L** | Low Speed (0.02) | N/A | N/A | **FAIL**: Policy prefers falling over structured stepping. |

## 4. Key Findings & The "Stability Gap"
*   **Shuffle Basin is Broken**: We have successfully eliminated the 7-8Hz vibration exploit. The robot no longer shuffles to satisfy speed rewards.
*   **The Stability Barrier**: As soon as the policy attempts a real step (induced by high step-advance weights or weight-shift rewards), it immediately loses balance. The current gait-quality seed does not have the "balancing" skills necessary to support a low-frequency stride.
*   **Mechanical Discovery vs. Speed Tracking**: Velocity tracking commands (even at 0.02 m/s) provide too much "noise" for the learner while it is trying to solve the primary physical problem of not falling while one leg is in the air.

## 5. Technical Issues
*   **GPU Memory Leak**: Rapid iterations and high-instability falls (causing early environment resets) have led to VRAM fragmentation/leaks. Even at 128 environments, the system currently reports `PxgCudaDeviceMemoryAllocator` errors. 
*   **Action Required**: Identifing and terminating lingering Python processes is necessary to restore GPU capacity.

## 6. Revised Strategic Path to S5
The current lineage has reached a dead end of instability. I recommend:
1.  **Stage S4-Balance (The "Leg Day" Stage)**: Reset to a stable standing checkpoint. Reward *only* lateral weight shifting and lifting feet (Clearance) with **Zero** forward speed command and **Zero** step-advance reward. Master the "Stance" first.
2.  **Stage S4-Stride**: Once it can shift weight for 4s without falling, re-introduce the `step_advance_margin` reward to pull the feet forward.
3.  **Stage S4-Velocity**: Finally re-introduce the gated speed commands.
