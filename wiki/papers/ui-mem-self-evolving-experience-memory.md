---
title: "UI-Mem: Self-Evolving Experience Memory for Online Reinforcement Learning in Mobile GUI Agents"
slug: ui-mem-self-evolving-experience-memory
arxiv: "2602.05832"
venue: arXiv.org
year: 2026
tags:
  - gui-agents
  - online-reinforcement-learning
  - experience-memory
  - procedural-memory
  - skill-evolution
  - grpo
  - mobile-agents
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: b0973b59d344facb9efdf3d4f108fd2b35d198e3
tldr: "Augments online GRPO for mobile GUI agents with a hierarchical, self-evolving experience memory (workflows, subtask skills, failure patterns) and stratified guidance sampling, lifting AndroidWorld 8B SR from 47.6% to 71.1% with cross-app transfer."
contribution_type:
  - method
  - system
datasets:
  - AndroidWorld
  - AndroidLab
  - AMEX
  - UI-Genie
code_url: "https://ui-mem.github.io/"
cited_by: []
---

## Problem & Context

Online Reinforcement Learning for GUI agents trained on raw mobile screenshots and action sequences has emerged as a way past the ceiling of pure supervised fine-tuning, with GRPO-style group-relative advantage estimation as the current workhorse optimizer (DigiRL, MobileRL, ARPO, InfiGUI-R1, UI-R1). The setting is brutally hard for two reasons that prior work only partially addresses:

1. **Inefficient credit assignment over long horizons.** GUI tasks routinely span >10 atomic steps. Outcome-only rewards collapse all intermediate progress into a single terminal signal, so a trajectory that navigates correctly to the right page but mistypes a single field is indistinguishable from a trajectory that never made it past the launcher.
2. **Repetitive errors across tasks with no transfer.** The same micro-failures (dismissing a confirmation popup, scrolling to find a hidden button, accepting a permission dialog) recur across totally different high-level tasks. Standard RL has no mechanism to retain the lesson once it is paid for in one task.

Prior art splits into two camps that each pick up half of this. Experience-replay methods (MobileRL, ARPO) inject past successful trajectories into the on-policy batch to stabilize training under sparse rewards, but the replayed trajectories are raw and task-specific — they do not help when the agent meets a genuinely novel task. Reward-shaping methods (MathShepherd, group-PRM variants) add step-level dense rewards but stay confined to the current rollout: they do not produce reusable artifacts that survive across tasks. The bottleneck the paper targets is the absence of a transferable, evolving experience layer that *both* densifies the reward signal *and* carries forward across tasks and applications.

## Key idea

GUI online RL should not learn from raw trajectories alone — it should **accumulate, reuse, and evolve** structured experience that transfers across tasks. UI-Mem operationalises this with three coupled mechanisms:

- A [[hierarchical-experience-memory]] storing parameterised templates at three abstraction levels (high-level workflows, mid-level subtask skills, failure patterns), retrieved by embedding similarity plus a UCB-style exploration bonus.
- A [[stratified-group-sampling]] mechanism that partitions each GRPO rollout group into strong-guidance / weak-guidance / no-guidance subgroups, with proportions controlled by a success-rate-driven curriculum. This keeps intra-group reward variance non-zero (so advantage estimation has a usable signal) while pushing the unguided policy to internalise the behaviours of the guided subgroup.
- A [[self-evolving-memory-loop]] that extracts new workflows from successful trajectories and new failure patterns from failed ones at the end of every training iteration, abstracts them into placeholder templates, and merges them into the memory with running success/usage statistics.

Together these turn the memory into a co-evolving artefact: the policy improves, the memory absorbs the policy's new strategies and failure modes, the next rollout group is shaped by an upgraded memory.

## Method

UI-Mem builds on GRPO ([[grpo]] not yet in the wiki) as its base optimizer. Given a task instruction $q$, the framework runs as follows.

**Hierarchical memory representation.** Three stores:

- High-Level Workflows $\mathcal{W}$: ordered subtask sequences for a task family (e.g. *Send Email* → [Open Mail, Select Recipient, Type Content, Send]).
- Mid-Level Subtask Skills $\Sigma$: natural-language action recipes for atomic capabilities ("Tap search icon, type {{query}}, select top result").
- Failure Patterns $\mathcal{F}$: error-diagnosis templates ("Avoid clicking Save before entering filename").

Every entry stores concrete values as semantic placeholders (e.g. `{{filename}}`, `{{confirm_button}}`), so the same template covers many surface instantiations. Storage is a vector DB indexed by task/subtask embeddings. Retrieval scores a candidate plan $p$ with a UCB-style mix of historical success and usage-bonus exploration:

$$S(p) = \frac{N_{succ}(p)}{\sum_{p'} N_{succ}(p')} + \lambda_{ucb}\sqrt{\frac{\ln \sum_{p'} N_{used}(p')}{N_{used}(p)+1}}$$

Failure patterns use a recency bias to keep diagnoses aligned with the current policy.

**Memory-guided exploration.** For each task $q$, retrieve the best $(\mathcal{W}_q, \Sigma_q, \mathcal{F}_q)$ and partition the GRPO group of $G$ trajectories into three subgroups with proportions $(\lambda_{\text{strong}}, \lambda_{\text{weak}}, \lambda_{\text{none}})$:

- **Strong**: prompt with full $(q, \mathcal{W}_q, \Sigma_q, \mathcal{F}_q)$ — high-success positive anchors.
- **Weak**: prompt with only the workflow $(q, \mathcal{W}_q)$ — forces the agent to fill in low-level execution.
- **None**: prompt with $q$ alone — unbiased measurement of internalised policy.

A **dynamic dropout curriculum** schedules $\lambda_{\text{strong}}$ and $\lambda_{\text{none}}$ as linear functions of the EMA success rate $S_t$ via $\phi(S_t) = \frac{S_t - \theta_{\text{start}}}{\theta_{\text{end}} - \theta_{\text{start}}}$, so guidance fades as competence rises.

**Guidance-aware reward shaping** augments outcome reward with (i) a per-subtask progress reward $r_{progress} = |\mathcal{S}_{completed}|/|\mathcal{S}_{total}|$ relative to the retrieved workflow, and (ii) an internalisation bonus that adds $\alpha \cdot r_{outcome}$ only for the unguided subgroup, devaluing "easy" guided successes:

$$\mathcal{R}(\tau) = \lambda_o r_{outcome} + \lambda_p r_{progress} + \alpha \cdot \mathbb{I}[\text{guidance}(\tau)=\text{None}] \cdot r_{outcome}$$

**Self-evolving loop.** After each iteration, a reward model (two-stage: Qwen2.5-VL-72B converts screen+action to text → DeepSeek-V3 rule-verifies — achieves 0.900 judge accuracy vs 0.724 for direct MLLM scoring) labels trajectories. An LLM extractor (Seed1.8) then distils successful trajectories into new $(\mathcal{W}, \Sigma)$ entries and failed ones into $\mathcal{F}$ entries (focused on the first failed subtask). Each new extraction is abstracted into placeholders and merged into the memory: if cosine similarity to an existing entry is high, statistics are updated ($N_{succ}{+}{=}1$ or refreshed timestamp); otherwise a new entry is created with $(N_{succ}{=}1, N_{used}{=}0)$.

See the framework method page [[ui-mem-framework]] for the end-to-end procedure.

## Experiment & Results

**Setup.** Trained on top of Qwen3-VL-4B and Qwen3-VL-8B (Qwen3-VL family). Distributed Android emulator instances asynchronously generate rollouts. GRPO with $G=4$ samples per query, 5 epochs at learning rate $1\times 10^{-6}$. Training data: 256 queries built from AMEX, AndroidLab, and UI-Genie task instructions, GPT-4o-augmented for diversity. Subtask definitions seeded from annotated trajectories then refined by the self-evolving loop.

**AndroidWorld (116 programmatic tasks, 20 apps, $>10$ steps typical).**

| Model | Params | SR |
|---|---|---|
| Qwen3-VL-8B (baseline) | 8B | 47.6% |
| Vanilla GRPO | 8B | 53.0% |
| GRPO + Progress Reward | 8B | 57.3% |
| GRPO + Experience Replay | 8B | 56.5% |
| GRPO + Inference-time Prompting | 8B | 55.2% |
| **UI-Mem-8B** | 8B | **66.8%** |
| **UI-Mem-8B$^\star$** (with inference-time retrieval) | 8B | **71.1%** |
| Gemini-2.5-Pro | — | 69.7% |
| Seed1.8 | — | 70.7% |

UI-Mem-8B$^\star$ surpasses closed-source Gemini-2.5-Pro (69.7) and Seed1.8 (70.7). UI-Mem-4B reaches 58.2 (62.5 with retrieval), beating the much larger UI-Venus-7B (49.1).

**AndroidLab (138 tasks, 9 apps, fine-grained Sub-SR / RRR / ROR metrics).** UI-Mem-8B$^\star$: Sub-SR 56.0, ROR 94.9, SR 44.9 — beats MobileRL (SR 42.5, the strongest experience-replay baseline) and UI-TARS-1.5-7B (SR 40.6). Sub-SR improvement is the biggest signal that hierarchical workflows densify the credit signal.

**Ablations (AndroidWorld 8B, full = 66.8%).** Removing each memory level hurts (workflow-only, skill-only, failure-only all clearly underperform full). Replacing abstracted templates with raw experience drops to 58.2. Disabling self-evolving memory updates drops to 62.9. Removing stratified group sampling and giving uniform full guidance noticeably underperforms — overuse of guidance prevents internalisation.

**Cross-application generalisation.** On five held-out apps (Bluecoins, Cantook, Maps.me, Pi-Music, Zoom) entirely absent from training, zero-shot UI-Mem matches or beats baseline on all five; inference-time retrieval adds consistent further gains, with the largest jumps on Bluecoins and Maps.me.

**Training dynamics.** Standard GRPO shows oscillating SR with no upward trend and intra-group reward variance frequently collapsing to zero (all-failure groups, vanishing gradient). UI-Mem maintains healthy non-zero variance through the stratified mix, yielding smooth convergence.

## Limitations

- Evaluation is Android-only (AndroidWorld, AndroidLab + held-out apps from AndroidLab). No web or desktop GUI evidence; cross-platform transfer is not measured.
- Reward model relies on a Qwen2.5-VL-72B + DeepSeek-V3 cascade plus a Seed1.8 extractor — heavy inference dependencies that may not be reproducible without comparable model access, and not factored into reported costs.
- The hierarchical workflow level requires seed subtask definitions distilled from annotated trajectories; bootstrapping the memory in a regime with zero annotated trajectories is not addressed.
- Stratified sampling proportions, curriculum thresholds $(\theta_{\text{start}}, \theta_{\text{end}})$, UCB weight $\lambda_{ucb}$, and reward weights are not ablated with sensitivity curves; the chosen schedule may be brittle to task distribution.
- Failure-pattern store uses recency bias without explicit forgetting or compaction policy — long-running deployments could accumulate stale or contradicting patterns.
- No measurement of safety / unintended actions during exploration despite the Impact Statement flagging it as a real risk for live deployment.

## Open questions

- Does the abstracted-template representation transfer to non-mobile GUIs (web, desktop), or are mobile-specific UI primitives baked into the workflows?
- How does the memory behave under longer training horizons — does the failure-pattern store saturate? Does workflow drift hurt the agent once curriculum guidance is fully removed?
- Can the self-evolving loop run without a separate frontier-scale reward model (Qwen2.5-VL-72B + DeepSeek-V3), e.g. with a self-judged setup, without collapsing extraction quality?
- Is stratified group sampling beneficial outside the GUI setting (mathematical reasoning, code, tool use), or does it depend on the long-horizon + sparse-reward signature?
- What is the right unit for a "skill" in $\Sigma$ — token-level action recipes, sub-screen abstractions, or longer compound primitives? The paper picks one but does not justify it against alternatives.

## My take

UI-Mem is the strongest empirical demonstration to date that procedural-memory ideas (templated workflows, subtask skills, abstracted failure patterns) can be married to online RL rather than bolted onto inference time. The key conceptual contribution is not any single component but the realisation that **guidance must be heterogeneous within a rollout group** — otherwise advantage estimation breaks. That observation generalises beyond GUIs: any sparse-reward, long-horizon RL setting with a retrievable knowledge source faces the same intra-group variance collapse. I expect the stratified-guidance idea to migrate into agentic reasoning and code RL within a year.

The framework also offers a clean answer to the "skill library evolution" question that procedural-memory work has circled for years: extraction is delegated to an LLM, abstraction is template-based, and merge is a vector-similarity + statistics update. It is not the only possible design but it is concrete and works end-to-end. The dependency on closed/frontier-scale models for reward judgement and experience extraction is the main practical caveat.

## Related

- [[hierarchical-experience-memory]]
- [[stratified-group-sampling]]
- [[self-evolving-memory-loop]]
- [[ui-mem-framework]]
- [[han-xiao]]
- [[hongsheng-li]]
