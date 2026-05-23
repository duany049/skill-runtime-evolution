---
title: Stateful Reflective Decision Process
aliases:
  - SRDP
  - Reflected MDP
  - stateful prompts
tags:
  - reinforcement-learning
  - mdp
  - agentic-memory
  - policy-iteration
  - llm-agents
maturity: emerging
definition: An augmented MDP in which the agent state is paired with a growing skill (or case) memory, restoring the Markov property while the memory evolves; the LLM is treated as a decision kernel acting conditionally on retrieved memory.
key_papers:
  - memento-skills-let-agents-design-agents
first_introduced: "Memento-2 (Wang et al., 2025)"
date_updated: 2026-05-18
related_concepts:
  - read-write-reflective-learning
linked_ideas: []
---

## Definition

The Stateful Reflective Decision Process (SRDP) is a decision-process formalism for LLM agents whose external memory grows over time. It extends the standard MDP with an episodic / skill memory space $\mathfrak{M}$ and an LLM decision kernel $p_{\mathrm{LLM}}(a \mid s, c)$:

$$\mathcal{D}_{\mathrm{SRDP}} = \langle \mathcal{S}, \mathcal{A}, \mathcal{P}, R, \gamma, \mathfrak{M}, p_{\mathrm{LLM}} \rangle$$

By augmenting the state to $x_t = (s_t, \mathcal{M}_t)$, the SRDP can be re-cast as a standard MDP — the Reflected MDP $\mathcal{D}_{\mathrm{ReMDP}}$ — over augmented states, with transition kernel

$$\mathcal{P}^{\mathrm{LLM}}(x' \mid x, c) = \sum_{a} p_{\mathrm{LLM}}(a \mid s, c)\, \mathbf{1}\{x' = (s', \mathrm{Write}(\mathcal{M}, s, a, r))\}\, \mathcal{P}(s' \mid s, a).$$

The composite retrieval-conditioned policy is $\pi^{\mu}(a \mid s, \mathcal{M}_t) = \sum_{c \in \mathcal{M}_t} \mu(c \mid s, \mathcal{M}_t)\, p_{\mathrm{LLM}}(a \mid s, c)$, where $\mu$ is the retrieval policy over memory items.

Under bounded rewards $|r| \leq R_{\max}$ and discount $\gamma < 1$, KL-regularised soft policy iteration over the Reflected MDP converges to an optimal retrieval policy $\mu^{\star}$ (Memento~2, Theorem 8).

## Intuition

A vanilla LLM agent with a frozen $\theta$ violates the Markov property the moment its memory starts growing — the "true" state includes whatever the agent has stored, but the model sees only the current observation. SRDP fixes this by lifting the memory *into* the state, so transition probabilities are well-defined and standard RL machinery (Bellman operators, value functions, policy iteration) all apply.

The kernel $p_{\mathrm{LLM}}$ encapsulates the LLM as a stochastic policy: given a retrieved memory case $c$, it produces an action distribution over $\mathcal{A}$. Different retrievals yield different action distributions, which is why the retrieval policy $\mu$ is the real learning target — the LLM weights stay frozen, but the choice of *what to retrieve* is optimised end-to-end.

## Variants

- **Episodic-trace SRDP** (Memento-2): $c_i$ is a logged case (state, action, outcome). $\mathrm{Write}$ appends; no in-place mutation.
- **Skill-folder SRDP** (Memento-Skills): $c_i$ is an executable skill (declarative spec + scripts + prompts). $\mathrm{Write}$ rewrites the skill's files; mutations are gated by an automatic unit-test.
- **Hybrid memory SRDP** (Memento-Skills with tip memory): the agent maintains both a skill library $\mathcal{S}_t$ and a tip memory $\mathcal{T}_t$; the augmented input becomes $x_t = (q_t, \mathcal{T}_t)$ and the routing policy reads over $\mathcal{S}_t$.

## Comparison

| Axis | SRDP / Reflected MDP | Standard MDP | POMDP |
|---|---|---|---|
| State | $(s_t, \mathcal{M}_t)$ — augmented | $s_t$ | belief $b_t$ over hidden state |
| Memory | first-class, evolves under $\mathrm{Write}$ | implicit in $s_t$ | encoded in belief update |
| Policy unit optimised | retrieval policy $\mu$ over $\mathcal{M}_t$ | action policy $\pi$ | belief-conditioned policy |
| Convergence | KL-regularised soft policy iteration | classical | classical with belief updates |

## Known limitations

- The Markov property is restored only nominally — the memory is unbounded in principle, so practical implementations rely on a bounded-cardinality $\mathcal{M}_t$ via dedup, retirement, or compression heuristics not covered by the theory.
- The LLM decision kernel $p_{\mathrm{LLM}}$ is treated as fixed; convergence guarantees do not account for distribution shift induced by an upstream LLM swap.
- The Write operator is opaque: the theory assumes it is a measurable function but says nothing about whether the LLM-driven file rewrites are correct or even monotonic.

## Open problems

- Tight finite-sample bounds on the asymptotic value gap when $\mathcal{M}_t$ grows under a non-i.i.d. task distribution.
- Convergence behaviour when $\mathrm{Write}$ is approximate (i.e. an LLM-driven rewriter rather than an oracle update).
- Extension to multi-agent SRDP with shared memory and distributed credit assignment.

## Relationship to foundations

SRDP sits inside the classical Markov decision process family. The Reflected MDP construction is essentially state augmentation — a well-worn trick to recover the Markov property under history-dependent dynamics — combined with the standard soft-policy-iteration machinery from KL-regularised RL. Its novelty is *what* gets lifted into the state (an external skill / case memory) and how the Write operator is engineered (file-level rewrites, not just transition logs).

## My understanding

The framework's payoff is conceptual: it turns "agent with growing memory" into a well-posed RL problem, so that retrieval can be analysed and improved with the standard toolbox. Its practical limit is the gap between $\mathrm{Write}$-the-mathematical-symbol (any measurable function from $(\mathcal{M}, s, a, r)$ to a new memory) and $\mathrm{Write}$-the-implemented-system (an LLM that may produce incorrect, regressive, or destabilising edits). The convergence proof guarantees that *if* the writes are well-behaved, the retrieval policy converges — it does not guarantee the writes themselves are. Memento-Skills's unit-test gate is the engineering bridge that tries to enforce this assumption.
