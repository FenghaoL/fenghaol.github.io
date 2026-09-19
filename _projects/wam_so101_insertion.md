---
layout: page
title: WAM Precision Manipulation on SO101
description: Adapting a world–action model for real-world insertion and learning recovery behavior from human corrections.
img: assets/img/projects/wam-cover.jpg
importance: 0
category: work
permalink: /projects/wam-so101-insertion/
---

<style>
  .wam-demo {
    margin: 1.1rem 0 1.6rem;
  }

  .wam-demo video {
    display: block;
    width: 100%;
    aspect-ratio: 8 / 3;
    border-radius: 8px;
    background: #111;
  }

  .wam-demo figcaption,
  .wam-note {
    margin-top: 0.45rem;
    color: var(--global-text-color-light);
    font-size: 0.88rem;
    line-height: 1.45;
  }

  .wam-stats,
  .wam-flow {
    display: grid;
    gap: 0.75rem;
    margin: 1.2rem 0;
  }

  .wam-stats {
    grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  }

  .wam-stats > div,
  .wam-flow > div {
    border: 1px solid var(--global-divider-color);
    border-radius: 8px;
    padding: 0.9rem;
  }

  .wam-stats strong {
    display: block;
    font-size: 1.35rem;
  }

  .wam-flow {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }

  .wam-flow strong,
  .wam-flow span {
    display: block;
  }

  .wam-flow strong {
    margin-bottom: 0.5rem;
    color: var(--global-theme-color);
  }

  .wam-flow span {
    font-size: 0.9rem;
    line-height: 1.5;
  }

  .wam-loop {
    padding: 0.7rem 1rem;
    border-left: 3px solid var(--global-theme-color);
    background: var(--global-code-bg-color);
    font-size: 0.9rem;
  }

  .wam-comparison summary {
    cursor: pointer;
    color: var(--global-theme-color);
    margin: 0.8rem 0;
  }

  @media (max-width: 640px) {
    .wam-flow {
      grid-template-columns: 1fr;
    }
  }
</style>

<figure class="wam-demo">
  <video controls autoplay loop muted playsinline preload="metadata" poster="{{ '/assets/img/projects/wam-cylinder-corrected.jpg' | relative_url }}" aria-label="Successful cylinder insertion, fixed and wrist camera views">
    <source src="{{ '/assets/video/wam-so101/cylinder-corrected.mp4' | relative_url }}" type="video/mp4">
  </video>
  <figcaption>Pick up a cylinder, align it with the matching hole, and insert it. Fixed camera on the left; wrist camera on the right.</figcaption>
</figure>

I adapted **AHA-WAM**, a pretrained world–action model, to a dual-camera **SO101** arm for precision insertion. The project connects visual observations, robot state, and language instructions to closed-loop robot control, then uses human corrections to teach the policy how to recover when an approach goes wrong.

The two tasks are cylinder insertion and power-adapter insertion. Both require more than reaching an approximate location: the robot must align the object with a small opening and respond to misalignment during execution.

<div class="wam-stats">
  <div><strong>2 cameras</strong>Fixed view + wrist view</div>
  <div><strong>2 tasks</strong>Real-robot insertion</div>
  <div><strong>149 rollouts</strong>Correction training data</div>
  <div><strong>231 interventions</strong>Human correction segments</div>
</div>

## From Observations to Robot Actions

AHA-WAM combines a video model with an action model. The video branch provides visual context for predicting short action sequences, while recent observations supply memory of how the robot reached its current state.

<div class="wam-flow" role="group" aria-label="Closed-loop deployment pipeline, steps one through four">
  <div><strong>01 · Observe →</strong><span>Fixed + wrist images<br>Joint state<br>Task instruction</span></div>
  <div><strong>02 · Add context →</strong><span>Recent visual history<br>Video-model features</span></div>
  <div><strong>03 · Predict actions →</strong><span>Action model generates a short sequence of joint targets.</span></div>
  <div><strong>04 · Execute ↻</strong><span>SO101 executes an action chunk, then receives fresh camera observations.</span></div>
</div>
<div class="wam-loop">↻ New observations return to step 01, keeping the policy in a closed loop with the physical robot.</div>

My implementation covers the multi-camera input pipeline, robot state and action interfaces, action chunking, and persistent visual memory for real-robot inference.

## Learning from Human Corrections

Demonstrations teach the basic task, but a deployed policy also encounters missed grasps and misaligned approaches. I built a takeover-and-handback workflow: an operator briefly corrects the robot, then lets the policy resume.

<div class="wam-flow" role="group" aria-label="Training workflow, steps one through four">
  <div><strong>01 · Demonstrate →</strong><span>Fine-tune the pretrained policy on teleoperated task demonstrations.</span></div>
  <div><strong>02 · Deploy & correct →</strong><span>Run the policy and record human takeovers when it needs help.</span></div>
  <div><strong>03 · Prepare data →</strong><span>Keep human action labels separate from the surrounding visual observations.</span></div>
  <div><strong>04 · Train & evaluate</strong><span>Mix corrections with original demonstrations, update the model, and test unassisted execution.</span></div>
</div>

Only human-controlled actions serve as correction labels. Recorded images around those corrections provide temporal context and eligible video-prediction targets. Training mixes original demonstrations and correction samples equally, so the policy practices recovery alongside the original skill.

## Real-Robot Demos

<figure class="wam-demo">
  <video controls loop muted playsinline preload="none" poster="{{ '/assets/img/projects/wam-adapter-corrected.jpg' | relative_url }}" aria-label="Successful power-adapter insertion, fixed and wrist camera views">
    <source src="{{ '/assets/video/wam-so101/adapter-corrected.mp4' | relative_url }}" type="video/mp4">
  </video>
  <figcaption>Remove the adapter from its holder and insert it into the receiving fixture. The socket housing has its internal components removed; this task evaluates geometric alignment and insertion.</figcaption>
</figure>

The successful rollouts above show the policy after corrective training. Expand the baseline clips below to see unsuccessful executions from the same recorded initialization IDs. These are selected examples, not a success-rate estimate; the clips are independently recorded and are not synchronized.

<details class="wam-comparison">
  <summary>Compare with the demonstration-only policy</summary>
  <figure class="wam-demo">
    <video controls loop muted playsinline preload="none" poster="{{ '/assets/img/projects/wam-cylinder-baseline.jpg' | relative_url }}" aria-label="Cylinder insertion failure before corrective training">
      <source src="{{ '/assets/video/wam-so101/cylinder-baseline.mp4' | relative_url }}" type="video/mp4">
    </video>
    <figcaption>Cylinder insertion · demonstration-only policy · unsuccessful rollout.</figcaption>
  </figure>
  <figure class="wam-demo">
    <video controls loop muted playsinline preload="none" poster="{{ '/assets/img/projects/wam-adapter-baseline.jpg' | relative_url }}" aria-label="Power-adapter insertion failure before corrective training">
      <source src="{{ '/assets/video/wam-so101/adapter-baseline.mp4' | relative_url }}" type="video/mp4">
    </video>
    <figcaption>Power-adapter insertion · demonstration-only policy · unsuccessful rollout.</figcaption>
  </figure>
</details>

## What I Learned

Human corrections improved task completion on both insertion tasks in controlled evaluations. The useful amount of visual history and future-video supervision differed by task, and better video predictions did not always translate into better physical execution. I evaluated the model through real-robot completion as well as video prediction, with particular attention to behavior after an unsuccessful approach.
