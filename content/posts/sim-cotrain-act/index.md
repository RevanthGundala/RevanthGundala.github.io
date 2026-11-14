---
title: "Sim-and-Real Co-Training"
draft: false
date: 2026-11-12T10:00:00.000Z
description: "Generating 1,000 stitched demonstrations in MuJoCo and measuring across six seeds whether co-training improves a real ACT policy."
---

*Code: [omx-act](https://github.com/RevanthGundala/omx-act) (policy & evaluation) · [omx-training](https://github.com/RevanthGundala/omx-training) (robot control)*

Collecting new teleoperated demonstrations on a physical robot is slow. After [testing DAgger corrections](/posts/act-blog/) on a low-cost arm, I wanted to see whether demonstrations generated in a simulator could improve an ACT policy instead.


## Setting up the task

### Replacing the sponge with a rigid cylinder

In an earlier experiment, the robot picked up a soft sponge and placed it on a fixed trivet. Simulating a deformable sponge in MuJoCo was difficult: rigid CAD approximations couldn't match foam compliance, and Gaussian splat scans lacked clean contact boundaries. If a co-trained policy failed, I wouldn't know whether the method failed or the soft-body model did.

<div class="media-row">
{{< img src="sponge-splat.jpg" alt="Gaussian splat scan attempt of the sponge showing top, side, and texture projections" caption="A Gaussian splat scan attempt of the sponge. While it captured the curled surface and texture, missing underside geometry made it unusable for contact simulation." >}}
{{< img src="sponge-3d-model.jpg" alt="3D model views of the sponge design iterations" caption="Attempting to model the sponge contour and profile across multiple CAD iterations." >}}
</div>

{{< video src="sponge-sim-fail.mp4" poster="sponge-sim-fail.jpg" caption="Replaying a real demonstration in MuJoCo with the modeled sponge (left: real top camera; right: simulation). Rigid collision approximations caused the fingers to push or slip off the sponge rather than pinching it." >}}

I replaced the sponge with a rigid cylinder—a pill bottle—and made the woven hot pad movable as well. Using a rigid object kept contact physics simple so the experiment tested whether the underlying co-training technique helped. Throughout the study, I kept my real dataset fixed at 50 demonstrations and generated up to 1,000 simulated episodes in MuJoCo with no new teleoperation.


## Calibrating the simulator

Before generating simulated training data, I calibrated a plain-MuJoCo scene against the physical 5-DOF [OpenMANIPULATOR-X](https://emanual.robotis.com/docs/en/platform/openmanipulator_x/overview/) follower arm.

### Cameras and lens calibration

MuJoCo renders ideal pinhole images, while both real cameras—especially the wide-angle wrist camera—have noticeable barrel distortion. I swept a printed checkerboard across each camera to fit lens distortion models using [Zhang's method](https://en.wikipedia.org/wiki/Camera_resectioning#Zhang's_method) and applied them to the simulated renders so the camera views matched reality.

{{< video src="checkerboard-calibration.mp4" poster="checkerboard-calibration.jpg" caption="Sweeping a printed checkerboard across the overhead camera (left) and wrist camera (right) to calibrate lens distortion across different angles and distances." >}}

After lens calibration, I aligned camera positions: the overhead camera was anchored against the arm's base mounting holes, and the wrist camera was closely aligned so the gripper fingertips matched real camera frames.

### Joint offsets and motor tracking

Real motor encoders don't read zero at exact physical zero, and motors lag under load rather than snapping to commanded angles instantly. I calibrated the joint zero offsets by touching known points on the table, and tuned the simulated motor stiffness and damping with [CMA-ES](https://en.wikipedia.org/wiki/CMA-ES) directly against recorded teleop logs so the simulated arm tracked commands with realistic lag.

<div class="media-row">
{{< img src="arm-pose-calibration.jpg" alt="Physical robot arm clamped to the workbench with ruler measurements calibrating table-to-pivot height and mounting plate thickness" caption="Calibrating physical arm geometry and table offset: measuring base mounting plate thickness and table-to-shoulder pivot distance before aligning joint zeroes." >}}
{{< img src="setup-real-vs-sim.jpg" alt="Side-by-side comparison of real episode 12 at 5.00s versus simulated episode 12 for both top and wrist cameras" caption="Matched episode 12 at 5.00s in hardware (left) and follower-driven MuJoCo replay (right) across both top and wrist cameras." >}}
</div>


## Generating 1,000 simulated episodes

To vary where the objects started without recording new human demonstrations, I used the approach from the MimicGen paper ([Mandlekar et al., 2023](https://arxiv.org/abs/2310.17596)): take existing human demonstrations, split them into sub-tasks (like picking up and placing down), shift them to new object locations, and stitch them back together into new complete runs.

I split each real demonstration after the lift, shifted the grasp to a new bottle position and the release to a new pad position, and smoothed the transition between them. Replaying the stitched paths in MuJoCo let me check every candidate physically, keeping only the episodes where the robot stood the bottle upright on the pad.

{{< video src="sim-stitched-demo.mp4" poster="sim-stitched-demo.jpg" caption="One stitched demonstration (overhead and wrist views). A pick-up at a new bottle position and a release at a new pad position are joined mid-air and saved only if the bottle finishes upright." >}}


## Why simulator scores could not pick the policy

I originally hoped to evaluate checkpoints in simulation and deploy only the winner. Comparing six co-trained policies on both 100 simulated scenes and the 20 real placements showed why that didn't work:

{{< svgfig src="sim-vs-real.svg" caption="Success on 100 simulated scenes (open circles) versus 20 held-out real placements (blue dots) across six co-trained policies. Dashed orange lines mark runs where strong simulation scores dropped on hardware." >}}

- **Checkpoints were noisy.** Checkpoints from a single run swung between 41% and 80% success just a few thousand steps apart on the same simulated scenes.
- **Rankings flipped on hardware.** The policy that scored lowest in simulation (49%, Seed 1003) reached 65% live—beating runs that hit 85–94% in simulation—while the run that reached 94% in simulation landed at 60% live.
- **Visual realism didn't dictate transfer.** A classifier could separate real from simulated frames with near-100% accuracy throughout training. As [Maddukuri et al. (2025)](https://arxiv.org/abs/2503.24361) observed, policies can transfer despite obvious visual differences, so closing the appearance gap in simulation didn't guarantee physical success.

On the physical robot, estimating depth from a single 2D camera is imprecise. Being off by even a centimeter knocks the bottle over in real life, whereas simulation gives the arm perfectly known object shapes and forgiving contact.


## Evaluation

### Protocol

I tested each policy on 20 held-out scenes (9 far reaches and 11 near reaches), lining up the bottle and pad by hand before each rollout using target rings from the overhead camera.

To keep comparisons fair, each co-trained model ran in the same session as its matched baseline (trained only on the 50 human demos), alternating which policy went first so drift wouldn't favor either side. Because 20 trials per seed has noticeable variance, I used 6 seeds.

{{< img src="placement-guide-top.jpg" alt="Live top-camera view showing green and red placement rings for the bottle and hot pad" caption="Overhead placement guide for one of the held-out scenes (green: bottle; red: hot pad)." >}}

### Results

{{< svgfig src="six-seed-results.svg" caption="Live success on 20 matched placements across six seeds (240 trials). Top: across-seed means with 95% t-intervals. Bottom: each seed's human-data baseline (grey) paired with its co-trained policy (blue, or orange where lower)." >}}

Five seeds matched or beat the baseline, with three scoring higher than any model trained purely on human demonstrations ever reached (13, 15, and 16 out of 20). The biggest improvements came on distant placements where the 50 human demonstrations were sparse.

Seed 1002 went the other way: while its baseline reached 9/20, the co-trained policy collapsed to 4/20. That single failed run introduced so much spread that the overall comparison wasn't statistically conclusive (the 95% confidence interval stretches from a 13% drop to a 44% gain).

{{< video src="real-paired-comparison.mp4" poster="real-paired-comparison.jpg" caption="Paired rollouts on the same distant placement (Seed 1006). Top: co-trained ACT grasps cleanly and places the bottle upright. Bottom: baseline ACT (human data only) sweeps past without grasping." >}}

### Checking why Seed 1002 failed

Offline losses gave no warning: Seed 1002 had lower error than successful runs during both training and validation, yet collapsed live on nearby placements. With a small batch size of 8 and a 50/50 mix, training drew only 80,000 simulated samples—leaving over 75% of the simulated frames unsampled and making individual checkpoints noisy.

<div class="media-row">
{{< svgfig src="seed-loss-curves.svg" caption="Training error on real frames alongside held-out validation loss across seeds. Seed 1002 (orange) had the lowest offline error across both metrics, despite collapsing on the physical robot." >}}
{{< svgfig src="seed1002-diagnosis.svg" caption="Seed 1002 live scores split into far (orange) and near (blue) placements alongside each session's real-only baseline." >}}
</div>

Averaging the weights across the last four checkpoints smoothed out the step-to-step noise and fixed the near scenes immediately without any new training (jumping from 1/11 to 7/11). But far placements stayed stuck until I trained with a 90% simulation mix at a larger batch size.

### Why trials failed

Recorded 30 Hz gripper telemetry showed where rollouts broke down:

{{< svgfig src="failure-modes.svg" caption="Outcome breakdown of all 240 live trials classified from gripper stall telemetry." >}}

- **Almost every failure happened at the pick-up.** In 85% of misses, the robot never lifted the bottle—either hovering above the table without closing, or clamping down on empty air. Once the arm actually grabbed the bottle, it almost never dropped it on the way to the pad.
- **The robot hesitated less on distant bottles.** Without simulated data, the arm often swept right past far-away bottles without even trying to grab them. Adding simulated examples cut those hesitations in half.

### Comparing with the paper

Maddukuri et al. ([arXiv:2503.24361](https://arxiv.org/abs/2503.24361)) reported a very similar gain on Franka pick-and-place (44% → 58–67%). The difference is in evaluation: their paper trained a single seed and took the best of three checkpoints on the test set, which masks step-to-step noise. Evaluating a single fixed checkpoint across multiple seeds exposed the 1-in-6 collapse that wiped out statistical confidence.


## Conclusion

Two things stood out. First, simulation worked well for generating demonstrations, but scoring policies in simulation couldn't predict which ones would transfer to hardware. Simulation has forgiving contacts and exact object boundaries, whereas real 2D cameras easily misjudge distance and tip the bottle over. Second, training noise easily masquerades as policy failure: one run collapsed to 20% despite having the lowest training loss, while identical runs reached 80%. Evaluating a single seed or checkpoint makes it easy to draw the wrong conclusion.

Building on this means improving training stability. Training longer didn't help—without decaying the learning rate, the model bounced around instead of settling. Averaging weights over time and using larger batches gives the stability needed before tackling harder manipulation tasks.
