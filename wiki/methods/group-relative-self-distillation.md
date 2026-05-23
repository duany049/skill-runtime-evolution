---
name: Group Relative Self-Distillation
slug: group-relative-self-distillation
type: training
tags:
  - self-distillation
  - credit-assignment
  - group-rollout
  - step-level-supervision
  - gui-agent
source_papers:
  - ui-voyager-self-evolving-gui-agent
parent_methods: []
child_methods: []
code_repo: "https://github.com/ui-voyager/UI-Voyager"
date_updated: 2026-05-18
---

## Problem setting

Long-horizon agent training (e.g. mobile GUI tasks of up to ~30 interaction steps) where the environment provides only a terminal success/failure signal. Standard policy-gradient methods such as GRPO and PPO assign the same advantage to every token in a trajectory, so a single wrong step at, say, step 5 drags down the credit for the remaining correct actions. Failed trajectories — the majority on hard tasks — are typically discarded in conventional pipelines.

## Mechanism

Within a group rollout of $G$ trajectories for the same task $q$, successful trajectories $\tau^+$ and failed trajectories $\tau^-$ are paired. For each pair, [[fork-point-detection]] identifies steps where the two trajectories visit the same screen state but choose a different action. At each such fork point $(j, i^*(j))$, GRSD constructs a training sample: the prompt is the failed trajectory's context at step $j$, the response is the successful trajectory's response at $i^*(j)$. The objective is the standard autoregressive next-token cross-entropy computed only over the response tokens. This converts the sparse trajectory-level reward into a dense, per-step SFT signal. The shortest successful trajectory in the group serves as the "teacher," and the same successful step may teach multiple failed steps. GRSD entirely replaces the GRPO/PPO objective in the second training stage rather than being added on top of it.

## Procedure

1. Roll out $G$ trajectories per task; partition into successful and failed sets.
2. For each $(\tau^+, \tau^-)$ pair, run the fork-point detection algorithm (transition-alignment pre-pass, SSIM-based state-equivalence test with mean-hash pre-filter, divergence check, teacher-step selection with monotonicity).
3. For each detected fork point, splice the failed-side prompt with the successful-side response.
4. Fine-tune the policy with standard cross-entropy on the response tokens.
5. Iterate: the updated policy generates fresh group rollouts, and the cycle repeats.

## Assumptions

- Successful and failed trajectories within the same group exist in sufficient supply. In practice this requires a warm-started policy (e.g. via prior RFT) with a non-trivial success rate.
- Screen-state equivalence is well-approximated by a fast structural similarity measure (SSIM on cropped, resized, grayscale screenshots). This is plausible for static mobile UIs but breaks down under heavy animation or streaming observations.
- The action space is discrete and parsable as a structured tool call, so action equality is well-defined.
- The reward signal is binary and trajectory-level (success/failure); GRSD does not use any intermediate reward shaping.

## Limitations

- The supply of fork points is bounded by how often successful and failed siblings actually share states. Tasks where successful rollouts are extremely rare see thin supervision until enough RFT iterations bootstrap them.
- SSIM is sensitive to transient visual perturbations: animations, keyboard pop-ups, blinking cursors, clock changes, and progress indicators can lower SSIM below the threshold even when the underlying logical state is identical, or raise it spuriously when it differs.
- GRSD only learns from the convex hull of behaviors the successful peers already produce; it has no built-in exploration beyond rejection sampling.
- The paper validates GRSD only on AndroidWorld with a 4B Qwen3-VL-Instruct backbone; transfer to desktop, web, or larger models is untested.

## Tradeoff profile

- **Strength**: Converts sparse rewards into dense per-step supervision without any external teacher model, external annotation, or value network. Reuses failed trajectories that would otherwise be discarded. Compatible with standard SFT infrastructure.
- **Weakness**: Requires a sufficiently strong starting policy so that successful siblings exist; without RFT warm-up, fork-point yield collapses. Inherits all limitations of SSIM as a state-equivalence proxy.
- **vs. GRPO / PPO**: Empirically lifts AndroidWorld Pass@1 from 73.2% (RFT init) to 81.0% where GRPO/PPO plateau near 76% from the same init. Particularly strong on low-success tasks where GRPO/PPO stagnate.
- **vs. external-teacher distillation (OPD, EvoCUA)**: Drops the dependency on a high-quality external policy, at the cost of being limited to what the agent's own successful siblings can demonstrate.
