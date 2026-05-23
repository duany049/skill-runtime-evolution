---
name: Evolvable Skill Bank Controller
slug: evolvable-skill-bank-controller
type: training
tags:
  - skill-selection
  - reinforcement-learning
  - agentic-memory
  - embedding
  - top-k
source_papers:
  - memskill-learning-evolving-memory-skills-self
parent_methods: []
child_methods: []
code_repo: "https://github.com/ViktorAxelsen/MemSkill"
date_updated: 2026-05-18
---

## Problem setting

Train a Top-K skill-selection policy whose action space (a *skill bank*) can grow, shrink, or be re-specified between training cycles. Standard policy-gradient setups assume a fixed-dimensional discrete action space; that assumption breaks as soon as the bank evolves. The method must therefore (i) score an arbitrary current set of skills, (ii) sample an ordered K-subset for a per-span composition, and (iii) compute joint probabilities under without-replacement selection for stable PPO-style training.

## Mechanism

For each text span $x_t$ and retrieved memories $M_t$, encode the joint state with a shared embedding model: $h_t = f_\text{ctx}(x_t, M_t)$. For each skill $s_i$ in the current bank, encode its *description* (not full content) to get $u_i = f_\text{skill}(\text{desc}(s_i))$ with the same encoder, mapping state and skills into a shared space. Score skills by inner product $z_{t,i} = h_t^\top u_i$ and apply softmax to get $p_\theta(i \mid h_t)$. Because both axes of the score matrix are computed from embeddings, the policy automatically adapts when the bank changes size.

To draw an ordered Top-K subset without replacement, use **Gumbel-Top-K**. The joint probability of the selected ordered tuple $A_t = (a_{t,1}, \ldots, a_{t,K})$ is the standard sequential-without-replacement factorization:
$\pi_\theta(A_t \mid s_t) = \prod_{j=1}^{K} \frac{p_\theta(a_{t,j}\mid s_t)}{1-\sum_{\ell<j} p_\theta(a_{t,\ell}\mid s_t)}$
which is used inside a PPO objective with importance weighting and clipping.

## Procedure

1. For each trace at training time, walk through it span by span (span size 512 tokens default).
2. At each span $x_t$, retrieve up to 20 memory items from the trace's memory bank → $M_t$.
3. Compute $h_t$ via the shared encoder (Qwen3-Embedding-0.6B in MemSkill).
4. Compute or cache $u_i$ for each skill in the current bank.
5. Sample $A_t$ via Gumbel-Top-K from $p_\theta(\cdot \mid h_t)$.
6. Forward $(x_t, M_t, A_t)$ to the executor LLM, which emits structured memory updates; parse and apply to the memory bank.
7. After the full trace is built, evaluate the memory bank on the trace's memory-dependent queries; use task performance (F1 or success rate) as the controller reward.
8. Run PPO updates on the controller using the joint without-replacement log-prob and the trace reward.
9. After each designer-driven evolution step, briefly boost exploration toward newly added skills before resuming normal selection.

## Assumptions

- A shared text encoder is rich enough to score memory skills by their description against a state representation built from span + retrieved memories.
- Task performance on the trace's memory-dependent queries is an informative-enough signal for credit-assigning to span-level skill selections.
- The skill bank evolves slowly enough that the controller can re-stabilize between evolution steps (exploration boost helps).
- Skill descriptions are concise enough that embedding the description (rather than full content) captures applicability.

## Limitations

- Pure inner-product scoring may underutilize subtle interactions between selected skills; composition is captured only by what the executor LLM does with the K-tuple, not by the controller.
- Top-K with K fixed at train/eval time treats every span as needing the same number of skills; an adaptive K could help.
- Cross-trace generalization of the controller relies on the encoder generalizing; if traces have systematically different lexical surface forms (dialogue vs. document), the encoder choice may dominate.
- Reward is given per-trace, not per-span, so per-span credit assignment is implicit (mitigated by the joint K-set probability, but not solved).

## Tradeoff profile

- **Adaptable action space** — handles arbitrary additions, refinements, deletions of skills without re-architecting.
- **Lightweight controller** — a small MLP on top of frozen embeddings; cheap to train and to redeploy after each bank update.
- **PPO + joint K-set log-prob** — stable RL formulation, but inherits PPO's hyperparameter sensitivity (clip range, GAE, etc.).
- **Strong cross-base transfer** — because the controller scores against text embeddings rather than producing logits over fixed indices, the same controller works with different executor LLMs.
