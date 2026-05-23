---
title: "UI-Voyager: A Self-Evolving GUI Agent Learning via Failed Experience"
slug: ui-voyager-self-evolving-gui-agent
arxiv: "2603.24533"
venue: ICLR 2025 (submitted)
year: 2026
tags:
  - gui-agent
  - mobile-agent
  - self-distillation
  - self-evolution
  - credit-assignment
  - reinforcement-learning
  - androidworld
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: ""
tldr: "Trains a 4B mobile-GUI agent that surpasses human-level on AndroidWorld (81.0% Pass@1) by combining iterative Rejection Fine-Tuning with Group Relative Self-Distillation (GRSD), which uses SSIM-based fork-point detection to convert sparse trajectory rewards into dense step-level supervision from successful peer rollouts."
contribution_type:
  - method
  - system
datasets:
  - AndroidWorld
code_url: "https://github.com/ui-voyager/UI-Voyager"
cited_by: []
---

## Problem & Context

Mobile GUI agents built on top of multimodal LLMs face two intertwined obstacles when trained with reinforcement learning on long-horizon tasks (often up to 30 interaction steps on AndroidWorld). First, **inefficient learning from failed trajectories**: on hard tasks, the vast majority of rollouts fail, and conventional pipelines discard them entirely or only use them through low-information trajectory-level rewards. Second, **ambiguous credit assignment under sparse rewards**: GRPO and PPO assign the same advantage to every token in a 30-step trajectory that received a single binary reward, so a wrong action at step 5 nullifies the reward for the 29 actions that were correct.

Prior work splits into static-dataset training (limited because it cannot react to UI dynamics) and interactive RL training (which encounters the credit-assignment wall above). Concurrent work such as EvoCUA tackles fork points but relies on external VLMs to synthesize correction traces. UI-Voyager is positioned in the interactive-RL line but explicitly self-contained: no external teacher model is required, and failed trajectories are reclaimed rather than discarded.

## Key idea

Within a group rollout of $G$ trajectories for the same task, successful trajectories often pass through the same screen state as failed ones but choose a different action there. The authors call these shared-state divergence points **fork points** and argue that the successful sibling acts as a free, self-generated teacher exactly at those steps. Group Relative Self-Distillation (GRSD) extracts the teacher's action at each fork point and reuses it as dense step-level supervision for the failed trajectory through standard SFT. This converts trajectory-level sparse rewards into per-step targets without any external annotation or external teacher model, and it works inside the same training loop as Rejection Fine-Tuning, which acts as an iterative "warm-start" stage.

## Method

UI-Voyager is a two-stage self-evolving pipeline that uses Qwen3-VL-4B-Instruct as the backbone. The task is formulated as a POMDP $(\mathcal{S},\mathcal{O},\mathcal{A},\mathcal{T})$ where observations are screen screenshots plus the language instruction, and the action space is a fixed set of mobile primitives (click, long_press, swipe, open_app, input_text, keyboard_enter, navigate_back, navigate_home, wait, status, answer). A rule-based verifier built on Android Debug Bridge issues the binary success reward.

**Stage 1 — Rejection Fine-Tuning (RFT)**. A seed task generator synthesizes novel tasks by perturbing AndroidWorld task parameters (times, quantities, file entities). Multi-scale Qwen3-VL models roll out trajectories; only those reaching the goal or passing the verifier are retained. The 4B model is then SFT'd on this filtered set. The updated model becomes the next iteration's generator, producing harder and higher-quality data — three iterations lifted Pass@1 from 37% to 73%.

**Stage 2 — Group Relative Self-Distillation (GRSD)**. For each task, $G$ trajectories are sampled, then paired into successful $\tau^+$ and failed $\tau^-$. Fork Point Detection uses **SSIM** on cropped, resized, grayscale screenshots (with a mean-hash pre-filter) to test screen-state equivalence $\textsc{Same}(o_a,o_b) = \mathbb{1}[\text{SSIM}(\phi(o_a),\phi(o_b))\ge\theta]$. A **transition-alignment** pass skips already-aligned segments, a **divergence check** $\textsc{Diverge}(i,j)$ ensures the next-step observations actually differ (else the two actions are equivalent), and a **teacher-step selection** picks the highest-SSIM candidate, breaking ties by preferring the smallest successful-step index. A **monotonicity constraint** forbids later failed steps from matching earlier successful steps. Each detected fork point $(j, i^*(j))$ generates a training sample whose prompt is the failed-context prompt at step $j$ and whose response is the teacher's response at $i^*(j)$. Standard autoregressive cross-entropy is computed over the response tokens only, completely replacing the GRPO/PPO objective.

## Experiment & Results

Evaluation is on **AndroidWorld** (116 tasks; success rate averaged over 64 randomized runs). The 4B UI-Voyager attains **81.0% Pass@1**, beating all baselines and the reported 80.0% human-level performance. By comparison: MAI-UI-235B-A22B 76.7%, UI-Tars-2 (230B) 73.3%, Gemini-2.5-Pro 69.7%, UI-Venus-1.5-30B-A3B 77.6%, Step-GUI-4B 63.9%, and Qwen3-VL-4B 45.3%. After three RFT iterations the model reaches 73.2%, used as the GRSD starting checkpoint; GRSD then lifts it to 81.0% while GRPO and PPO plateau at ~76% from the same checkpoint and take ~175 steps to merely match one RFT iteration (64.0%). On the ten lowest-success-rate tasks where GRPO/PPO show little movement, GRSD shows the largest gains, confirming its strength when successful samples are scarce. Two qualitative case studies (BrowserMaze fork at step 12, SystemBluetoothTurnOff fork at step 0) illustrate that fork points can sit anywhere along the trajectory.

## Limitations

- **SSIM as a state-equivalence proxy is imperfect under streaming observations.** Animations, keyboard transitions, loading screens, blinking cursors, toast notifications, and clock updates cause temporal misalignment or spurious mismatches. The discussion proposes time-aware matching over a short temporal window, masking high-variance UI regions, and combining SSIM with OCR/layout tokens — none of which are implemented in the paper.
- **Predefined high-level action space.** AndroidWorld's discrete actions abstract away low-level touch dynamics (gesture duration, trajectory shape, release timing). Policies may underperform when deployed with finer-grained controls or different action wrappers.
- **Single benchmark.** All results are on AndroidWorld; transfer to desktop OS, web, or in-the-wild mobile remains untested.
- **No comparison against contemporaneous on-policy distillation variants** beyond a textual reference; quantitative comparison with OPD and EvoCUA is absent.

## Open questions

- Does GRSD generalize to environments where successful sibling trajectories are extremely rare or where state-equivalence is harder to measure (e.g., dense web pages, OSWorld)?
- Can fork-point detection be made robust enough to remove the screen-state-as-image assumption (e.g., language-only or DOM-based environments) without losing the dense-supervision benefit?
- How does GRSD interact with model scale — would a 30B base see the same headroom over GRPO/PPO, or does the gap close as the policy gets stronger?
- What is the right way to combine GRSD's step-level SFT with a value-based RL signal so the agent can still explore beyond the convex hull of its successful peers?

## My take

The cleanest insight here is the framing: a group rollout already contains its own teacher whenever a successful and failed trajectory share a state. That observation turns sparse rewards into dense supervision without any external model, which is genuinely useful in the regime where state-of-the-art teachers are expensive or unavailable. The SSIM-based matching is a pragmatic choice that the authors are appropriately honest about — they flag temporal-misalignment failure modes in the discussion and gesture at fixes, though the paper does not implement them. The 4B-vs-235B comparison is striking, but worth contextualizing: the reported baselines are taken from prior papers and the AndroidWorld split was randomized over 64 seeds, while many baselines were not — so the magnitude of the gap is plausibly inflated by evaluation protocol. Still, the result is meaningful: the credit-assignment problem in long-horizon GUI tasks is real, and the GRSD recipe is a credible alternative to running PPO/GRPO from cold on a 30-step trajectory. The fact that the method works *because* RFT first lifts the policy to ~73% Pass@1 also matters — without that warm start, the supply of successful peers would be too thin for fork-point matching to fire. This is the most actionable lesson: dense self-distillation only kicks in after the data distribution shifts enough that successful siblings exist for failed ones to learn from.

## Related

- [[group-relative-self-distillation]]
- [[fork-point-detection]]
- [[fork-point]]
- [[self-evolving-gui-agent]]
- [[androidworld-benchmark]]
- [[rejection-fine-tuning]]
- [[zichuan-lin]]
- [[deheng-ye]]
- [[tencent-hunyuan]]
- [[skill-evolution]]
