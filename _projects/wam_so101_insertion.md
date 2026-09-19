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

  .wam-stats {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
    gap: 0.75rem;
    margin: 1.2rem 0;
  }

  .wam-stats > div {
    border: 1px solid var(--global-divider-color);
    border-radius: 8px;
    padding: 0.9rem;
  }

  .wam-stats strong {
    display: block;
    font-size: 1.35rem;
  }

  .wam-figure {
    margin: 1.5rem 0 1.8rem;
  }

  .wam-figure a {
    display: block;
    border: 1px solid var(--global-divider-color);
    border-radius: 8px;
    overflow: hidden;
    background: #fff;
  }

  .wam-figure img {
    display: block;
    width: 100%;
    height: auto;
  }

  .wam-figure figcaption {
    margin-top: 0.6rem;
    color: var(--global-text-color-light);
    font-size: 0.88rem;
    line-height: 1.5;
  }

  .wam-equation {
    margin: 1.2rem 0;
    padding: 0.35rem 0.75rem;
    overflow-x: auto;
    border-left: 2px solid var(--global-divider-color);
  }

  .wam-results {
    width: 100%;
    margin: 1.1rem 0 0.5rem;
    font-size: 0.92rem;
  }

  .wam-results th,
  .wam-results td {
    padding: 0.65rem 0.4rem;
    border-bottom: 1px solid var(--global-divider-color);
    text-align: left;
  }

  .wam-comparison summary {
    cursor: pointer;
    color: var(--global-theme-color);
    margin: 0.8rem 0;
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

## System Overview

AHA-WAM combines a video model with an action model. I adapted this backbone to the SO101 and built the surrounding data collection, training, and deployment pipeline. The figure below shows how demonstrations and human corrections feed back into a policy that runs on the physical robot.

<figure class="wam-figure">
  <a href="{{ '/assets/img/projects/wam-interactive-learning.jpg' | relative_url }}" target="_blank" rel="noopener" aria-label="Open the interactive learning framework at full resolution">
    <img src="{{ '/assets/img/projects/wam-interactive-learning.jpg' | relative_url }}" width="1644" height="850" loading="lazy" alt="Real-robot learning pipeline: dual-camera demonstrations train a video–action model; robot rollouts and human takeovers supply correction data for further training.">
  </a>
  <figcaption>Demonstrate, deploy, correct, and retrain. The video model provides visual features to the action model, while human interventions add examples of how to recover. Click the figure to enlarge.</figcaption>
</figure>

**On the robot**, fixed and wrist cameras provide complementary views of the object and insertion opening. The policy combines these images with joint state, a task instruction, and recent visual history to predict short action sequences. After executing a chunk, it receives fresh observations and continues in a closed loop.

**My implementation** connects the multi-camera input pipeline, calibrated joint-state and action interfaces, action chunking, and persistent visual memory. It also records policy execution and operator takeovers in a shared timeline for training and evaluation.

## Learning from Human Corrections

Demonstrations teach the basic task, but deployment exposes missed grasps and misaligned approaches. I built a takeover-and-handback workflow: an operator briefly corrects the robot, then lets the policy resume. The key training decision is to distinguish **what the robot observed** from **which actions it should imitate**.

<figure class="wam-figure">
  <a href="{{ '/assets/pdf/wam-method-overview.pdf' | relative_url }}" target="_blank" rel="noopener" aria-label="Open the temporal supervision diagram as a vector PDF">
    <img src="{{ '/assets/img/projects/wam-method-overview.png' | relative_url }}" width="2055" height="815" loading="lazy" alt="Correction timeline: past observations provide memory, only human-controlled actions provide correction labels, and eligible recorded future frames provide video targets. Video and action branches are jointly trained, then evaluated through video fidelity and robot task success.">
  </a>
  <figcaption>One intervention provides three kinds of information: past visual context, human action labels, and an observed visual continuation. The action labels stay within human control even when the visual context extends beyond it. Click for the vector PDF.</figcaption>
</figure>

The model retains its joint action–video training objective. For a minibatch $B$, the loss is:

<div class="wam-equation" markdown="1">

$$
\mathcal{L}(B)
=
\frac{1}{|B|}\sum_{i\in B}\ell_i^{\mathrm{action}}
+
\lambda_v
\frac{\sum_{i\in B}m_i^v\,\ell_i^{\mathrm{video}}}
{\max\!\left(1,\sum_{i\in B}m_i^v\right)}.
$$

</div>

Here, $\ell_i^{\mathrm{action}}$ supervises valid actions—**human-controlled actions only** for correction samples. The mask $m_i^v$ includes a video target only when a complete eligible continuation is available; $\lambda_v$ balances the two losses. Both branches use flow matching. In practice, this lets short interventions teach corrective motion while surrounding images supply temporal context.

I mix original demonstrations and correction samples at a **1:1 sampling ratio**, updating the video and action branches together. This retains practice on the original skill while adding recovery behavior.

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

## Evaluation and Takeaways

Controlled, unassisted evaluations showed higher task completion after corrective training:

<table class="wam-results">
  <thead>
    <tr><th scope="col">Task</th><th scope="col">Demonstration only</th><th scope="col">Best corrected policy</th></tr>
  </thead>
  <tbody>
    <tr><th scope="row">Cylinder insertion</th><td>1 / 13 · 7.7%</td><td><strong>6 / 13 · 46.2%</strong></td></tr>
    <tr><th scope="row">Power-adapter insertion</th><td>10 / 20 · 50.0%</td><td><strong>15 / 20 · 75.0%</strong></td></tr>
  </tbody>
</table>
<p class="wam-note">Best results come from different correction configurations: extended history and future video for the cylinder, and intervention-local context for the adapter. Counts reflect a small evaluation set with one training seed per configuration; the selected demo videos above do not necessarily show the best-scoring checkpoint.</p>

**The main lesson:** better video prediction does not automatically mean better physical control. Temporal context helped differently across the two tasks, so I evaluated both held-out video predictions and actual robot completion. The project brought together model adaptation, human-in-the-loop data collection, and real-robot evaluation in one working system.
