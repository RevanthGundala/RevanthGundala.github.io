---
title: "Debugging ACT and DAgger"
draft: false
date: 2026-08-21T09:16:45.000Z
description: "Getting a custom ACT policy working on a low-cost arm, and measuring DAgger corrections under matched placements."
---

*Code: [omx-act-recap](https://github.com/RevanthGundala/omx-act-recap) (policy & evaluation) · [omx-training](https://github.com/RevanthGundala/omx-training) (DAgger & robot control)*

I wanted a small, easy-to-modify imitation-learning policy that runs on low-cost hardware. This post covers how I got a custom ACT implementation working on a real robot arm (the 5-DOF [OpenMANIPULATOR-X](https://emanual.robotis.com/docs/en/platform/openmanipulator_x/overview/)), and how I measured whether DAgger corrections actually helped.


## Debugging a custom ACT implementation

Getting a policy working on a physical robot is hard. I wanted a minimal ACT implementation that would be easy to modify while still using LeRobot's training and evaluation infrastructure.

### Simplifying the task

{{< img src="setup.jpg" alt="Top-camera view of the OpenMANIPULATOR-X sponge task" caption="The final task setup: move the blue sponge onto the fixed trivet." >}}

I started by trying to place pasta into a bowl, but a single piece of pasta was small, low-contrast, and took up only a tiny fraction of the wide overhead camera. Cropping the camera down to the workspace likely would have resolved this, but rather than tuning camera crops, I simplified the setup instead: switching to a bright, large sponge and a fixed trivet. This isolated the sponge's position and orientation as the only variables so I could verify the policy architecture itself first.

{{< img src="pasta-training-demo.jpg" alt="Overhead view of a single piece of pasta beside a white bowl" caption="The single-piece pasta task from the overhead camera. I used this rotated camera orientation by accident, but kept it consistent across every episode. The pasta still occupied only a small part of the image, making it a poor target for visual localization." >}}

### Running the policy asynchronously

The arm needs motor commands sent 30 times a second (30 Hz) to move smoothly, but my computer could only compute a new batch of actions about 10 times a second (~100 ms). Standard ACT uses temporal ensembling, which queries the model to re-plan at every single motor step, which was too fast for my local computer to run in real time.

Instead, I ran planning and execution in parallel with a 30-action chunk size (exactly 1 second of motion at 30 Hz). Predicting 30 steps gave the arm plenty of buffer to keep moving while the next plan was generated in the background (~100 ms), without locking the robot into an overly long open-loop trajectory like 50 or 100 steps. Whenever a new plan finished, it blended into the queue with slight smoothing to keep the arm from jerking, while leaving the gripper commands instant so it could snap shut without delay.

{{< svgfig src="async-execution.svg" caption="Parallel planning and execution: the model generates 30-step action chunks in the background while the arm continuously moves at 30 Hz, blending in new commands without stuttering." >}}

### Evaluating coverage

Earlier sponge experiments showed that my dataset overrepresented one orientation, and the policy performed less consistently across the rest of the grid. Before training the base policy for the trivet task, I recollected the demonstrations using an explicit coverage plan with six locations and eight orientations, filling all 48 planned buckets.

{{< img src="coverage-grid-detection.jpg" alt="Top-camera coverage check with a position grid and detected sponge orientation" caption="The OpenCV collection gate detected the blue sponge, assigned its center to one of six reachable table regions, and estimated its orientation from the sponge's long axis. I grouped that angle into one of eight 22.5-degree bins and collected one demonstration for each position-orientation pair." >}}

To improve the base policy, I used human-gated DAgger (HG-DAgger): the autonomous policy ran until it began to fail, at which point I intervened with the leader arm to steer out of mistakes and aggregated those correction episodes into the dataset ([Ross et al., 2011](https://arxiv.org/abs/1011.0686); [Kelly et al., 2019](https://arxiv.org/abs/1806.03960)). I collected 18 correction episodes, combined them with the original 48 demonstrations, and fine-tuned on all 66 episodes.

{{< video src="dagger-takeover-continuous.mp4" poster="dagger-takeover-continuous.jpg" caption="An on-policy DAgger correction: the policy runs autonomously until I intervene around 8s with the leader arm to guide the grasp and placement on the trivet. (The physical handoff pause isn't captured because recording halts while the leader aligns.)" >}}


## Evaluation

### Protocol

I evaluated five sponge layouts repeated six times (30 trials total), shuffled into a fixed order. Before each rollout, I aligned the sponge by hand against an outline on the live camera preview, and recorded success or failure after each attempt.

For a second round of DAgger, I collected 12 more on-policy corrections focused on the specific layouts where the first policy struggled, bringing the dataset to 78 episodes.

### Results

{{< svgfig src="results-chart.svg" caption="Live success on 30 matched placements across the base model and two rounds of DAgger (with 95% Wilson intervals)." >}}

{{< svgfig src="per-layout.svg" caption="Successes out of 6 trials for each layout. Each row shows the base policy, DAgger round 1 and DAgger round 2, and the label under the layout name is how many round-two corrections that layout received." >}}

The layouts that received the most corrections improved most, while layouts with few or no corrections got worse.

### Why trials failed

Classifying trials by where the sponge started and finished in recorded video showed where the policy broke down:

{{< svgfig src="failure-modes.svg" caption="Outcome of each policy's 30 trials from top-camera video tracking." >}}

- **Every failure was a pick-up failure.** In 29 of the 36 failures the sponge never moved at all, and in the other 7 it was nudged slightly without being lifted. No failed trial ever reached the trivet.
- **Targeted corrections worked where spent.** Round two eliminated misses on the two layouts it targeted (layouts 1 and 4), but failures remained on the layouts that didn't receive new corrections.

Since failures happened at the grasp, future corrections needed to target the approach and grasp, not the carry or the placement. Standard imitation policies also struggle when an object slips, continuing along their trajectory in empty air. In an evaluation rollout, the fine-tuned policy successfully recovered after dropping the sponge shortly after lifting:

{{< video src="dagger-recovery.mp4" poster="dagger-recovery.jpg" caption="Recovering after a slip: when the sponge drops shortly after lifting, the policy retries the grasp autonomously and completes the placement on the trivet." >}}


## Conclusion

Two things stood out. First, offline training loss meant very little: low training loss and even successful single-episode overfitting proved the pipeline ran, but didn't translate to reliable closed-loop execution on the robot.

Second, evaluation design mattered as much as the training algorithm. Testing on uncontrolled or arbitrary placements creates a false impression of progress; under 30 matched placements, success only rose from 50% to 67%. Targeted corrections cleaned up the exact scenes they were trained on, but didn't meaningfully lift general performance.

Building on this means evaluating strictly on held-out, unseen layouts with interleaved rollouts to prevent lighting bias, as well as benchmarking recovery quantitatively. More fundamentally, collecting physical demonstrations and manual interventions is bottlenecked by human collection time—a limitation that led directly to testing [sim-and-real co-training](/posts/sim-cotrain-act/) to generate diverse demonstrations in simulation without manual teleoperation.
