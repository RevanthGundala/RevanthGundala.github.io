---
title: "Pouring a Liquid"
draft: true
date: 2026-03-05T00:00:00.000Z
description: "A fixed 5-DoF arm learns a pouring task. Early pretraining shows weak grasp generalization, but the policy already separates grasping from pouring."
categories:
  - Hello
---

## Goal

How much can online RL improve a robot's performance on a pouring task?

## Why this task

I chose pouring water from a bottle into a cup as a base task because it is surprisingly rich for a fixed 5-DoF arm.

The robot has to solve several subtasks in sequence:

1. Pick up the bottle.
2. Tilt it at the right angle.
3. Understand when to stop pouring.

The grasp itself is also interesting. Because the arm is fixed in place, the grap point on the bottle changes the entire geometry of the task. A high grasp and a low grasp lead to different pouring arcs, and only some of those arcs will line up with the cup. If the grasp is bad, the robot may need to reposition the cup before trying again.

## Pretraining setup

For the pretraining run, I used a set of 50 successful demonstrations that complete the full task end to end.

(insert vid 2x)

## What happened

Even after training on those 50 demonstrations, the robot still struggles to pick up the bottle consistently. It also does not generalize well to changes in object position, which is not too surprising because all 50 training samples place the bottle and cup in nearly the same location.

## One encouraging sign

The encouraging result is that the policy seems to understand some task structure. When the bottle has not been picked up, the robot usually keeps trying to grasp it instead of immediately moving into a pouring motion. That suggests the model has learned at least a rough ordering of subtasks, even if it still fails at execution.

## Eval

Looking at the eval, we also decided that realtime chunkign was a necessary in order to get mooth actions. Doing remote inference on modal caused a ~1s round trip. 

We saw that this drastically improved smoothness and quality of the robots execution. However, 
We also had to increase chunk size, since our inference delay between when enxt chunk was created, was around 33 actions, leaving about 17 chunks left over of the new chunk. Increasing our chunk size to 100 left enough of a transition for the new chunk to come in, so we never had any period of waiting before our action queue was full. interesting finding: we traingined to predict 50 actions, but doing 100 actions at inference still improved jitteryness and smoothjness. 

4th time it does everything and 80% of walter success before failing, 
5th epside it picks it up, and 80%. This one is super interesting because it still tries to pour the rest of the water in the bottle, and it is adjusting in realtime to find the right angle!
6th episode, picks up water bottle, grasp angle is wrong and it just completely misses.
7th epsiode gets around 50% of the water, but tilting angle is off
0,1,2,3, 8th, 10th episode fails to pick up water bottle
9th episode executes perfectly


 Great data! Here's the summary:

  ┌──────────────────────────┬────────────────┬────────────────┐
  │ Result                   │ Episodes       │ Rate           │
  ├──────────────────────────┼────────────────┼────────────────┤
  │ Good (≥80% water)        │ 4, 5, 9        │ 3/11 = 27%     │
  ├──────────────────────────┼────────────────┼────────────────┤
  │ Partial (some water)     │ 7              │ 1/11 = 9%      │
  ├──────────────────────────┼────────────────┼────────────────┤
  │ Failed grasp             │ 0,1,2,3,6,8,10 │ 7/11 = 64%     │
  └──────────────────────────┴────────────────┴────────────────┘

  The bottleneck is grasping — 64% of episodes fail before even attempting to pour. When it grasps
  successfully (4/11 times), it pours well (3 of 4 got ≥80%).

  The episode 5 observation — adjusting pour angle in real-time — is a great sign that the model HAS learned
  the task. The problem is consistency, especially grasp approach angle.


  ## stage 2 - pretraining on new 50 episodes

  decided to retrain on new 50 episodes focusing on more divertsity in the start state distribution, and also varying grsp angles. 
  Also focused on extra infra, using RTC and QUIC + NAT hole for direct commuivate

starting at    POST / -> 200 OK  (duration: 1.03 s, execution: 856.1 ms)
  QUIC predict -> OK  (in=113.2KB out=6.1KB exec=587.7ms)

much msoother 

0-3 epsiode, robot doesnt pick up bottle
4 episode, picks it up and misses cup on pour
5 epsiode is success
6 episode gets around 25% success of the pour 
7 episode - drops model mid pour
8 epsiode is same as 4th
9 episode is same as 8th
10 epsiode fails to pick up bottle


tried evaling at later time in day, 5/6 failed to pick ujp bottle, 1/6 poured it wrong

idea was to collect slightly more data udner differnet lighting position and camera position 

0 - fails to pick up bottle 
1 - complete success
2 - complete success
(included an extra item here in the visual space - we put a scale in here, which was apart of all trianing episodes)
3 - mis-aims for the water bottle 
4 - fails to pick up water bottle, although retries. shows that it is following training data better
5 - mis-aims
6 - same as ep 4
7 - picks up, misses pour
8 - same as ep 7
9 - same as ep 8

potential conclusions here -
- despite training on 50% data, we seem to have exact smae failure modes as 50 episodse
- lighting + camera angle is extremely importnat. Training on 70 episodes with 50 from old distribution and 20 form new, and then evaling on the new environment showd 0% success rate or any signs of life. Need to follow up on what is the data distribution we should be training on if you want robust policies to work in different environments. 
- Is this a model issue? Seems like more data didnt solve our problem before. Should we try a different distribution of data? i.e. 50/50 from old and new datasets and eval on new? or maybe have 80% new and 20% old? Although like i said, seems like more data isnt the problem based on the fact we are getting similar results from the 50 episode eval. 

rerunning with notes, and checkign eval + training episodes to see ifth eym atch

0 - misses pour
1 - misaims. 
(see metadata)


seems our data distribution is not diverse enough and successful episodes during eval are working towards wat our training set contains, which is a good sign. Model seems to strugle partiall with less diversity .

0 - success
1 - fails to pick up bottle 
(see metadata)

(new targeted evals)
┌────────────┬───────────────────────┬──────────────┬──────────────────────────────────────────────────────┐
│ Run        │ OpenCV/YOLO label     │ Bucket count │ Visual result                                        │
├────────────┼───────────────────────┼──────────────┼──────────────────────────────────────────────────────┤
│ 17:46:49   │ thin_distribution     │ 2/4          │ Success                                              │
├────────────┼───────────────────────┼──────────────┼──────────────────────────────────────────────────────┤
│ 17:51:13   │ thin_distribution     │ 2/4          │ Success                                              │
├────────────┼───────────────────────┼──────────────┼──────────────────────────────────────────────────────┤
│ 17:53:02   │ out_of_distribution   │ 0/4          │ Failed pickup / bad interaction                      │
├────────────┼───────────────────────┼──────────────┼──────────────────────────────────────────────────────┤
│ 17:55:47   │ unknown               │ 0/4          │ Not a clean success; YOLO missed bottle at start     │
├────────────┼───────────────────────┼──────────────┼──────────────────────────────────────────────────────┤
│ 17:58:36   │ in_distribution       │ 4/4          │ Success                                              │
└────────────┴───────────────────────┴──────────────┴──────────────────────────────────────────

this is where we realized Dagger comes in. b/c it works on ood
o


-------------- NE EXPEIMENT ----------------------

trained act model, posterior collapse caused kl to go to 0
debugged by overfitting on one episode, realized it couldnt overfit
tried to zero out images to see if preprocessor was issue, 
trainged with mlp, and it worekd
tried to train with just decoder - worked
tried ot train with qpos_token and decoder - worked

=== NEW AGAIN ===
one batch overfit, we had 0.73 loss on our custom impl, and validated with lerobot at 0.81. Why is even lerobots loss bad?
trained until 20k, did eval step


 outputs/eval_videos/h264/top_20260616-182634_h264.mp4 - shows our eval 
robot is able to perform trajectory, but memorized it. looked at data, seemed like decent distribution
actually found that it just cant locate pasta. it can only find bowl


realized that we did not train with pretrained backbone, and initialized weights or andom


these show it understands vision better

outputs/eval_videos/top_20260617-111112.mp4
outputs/eval_videos/top_20260617-110914.mp4
 outputs/eval_videos/top_20260617-111440.mp4

 talk about all ablations
 use recommended-split.jpg to rerun

 