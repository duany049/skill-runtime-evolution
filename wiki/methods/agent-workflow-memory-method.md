---
name: Agent Workflow Memory (AWM)
slug: agent-workflow-memory-method
type: prompting
tags:
  - procedural-memory
  - agent-skills
  - workflow-induction
  - web-navigation
  - lifelong-learning
source_papers:
  - agent-workflow-memory
parent_methods: []
child_methods: []
code_repo: https://github.com/zorazrw/agent-workflow-memory
date_updated: 2026-05-18
---

## Problem setting

Boost LM-based agents (specifically web-navigation agents over benchmarks like WebArena and Mind2Web) on long-horizon, multi-step tasks where the base agent solves each task independently and fails to accumulate procedural knowledge across tasks. AWM is targeted at the gap left by existing approaches: (i) training-based methods (BAGEL, AutoGuide) that need annotated examples and do not generalize to new tasks/sites; (ii) in-context demonstration methods (Synapse) that bias element selection toward the specific examples retrieved; (iii) human-engineered workflows (SteP) that require domain experts. AWM supports both an *offline* regime (training examples available) and an *online* regime (test queries only, no supervision).

## Mechanism

AWM augments the agent's prompt context with a second tier of memory composed of *induced workflows*. Concretely, for an agent with LM backbone L, base memory M (containing default tool documentation), observation o, and task instruction q, the augmented memory M_w = M + W replaces M in the inference call: L(q, M_w, o) → a.

The workflow set W is produced by an **induction module** I that takes one or more past experiences E = {(q, P)} and outputs workflows W = {(d, P_d)} where d is an NL workflow description and P_d is an abstracted step sequence. I is implemented as an LM prompt that asks the agent LM to extract common sub-routines from the input experiences. The prompt enforces two abstraction levers: induce *fine-grained sub-tasks* rather than copy full instructions, and *replace example-specific entities with placeholder names* ({product-name}, {location}, etc.). A rule-based alternative for I — action-sequence deduplication plus environment-invalid-step filtering — is reported as an ablation and performs comparably on WebArena but worse on Mind2Web.

## Procedure

**Offline regime** (when canonical training experiences E_train exist):

1. Concatenate all training experiences into a single induction prompt.
2. Call I(E_train) → W_offline. Segment the LM output on double-line breaks to yield individual workflows.
3. Store W_offline in agent memory: M_w = M + W_offline.
4. At inference, every test query q is solved under the fixed memory M_w: a = L(q, M_w, o).

**Online regime** (when only test queries are available):

1. Initialize agent memory to base M; set M^0 = M.
2. For each incoming test instruction q^t (t = 1, 2, ...):
   - Solve under current memory: generate trajectory (p^t_1, ...) using L(q^t, M^t, ·).
   - Wrap the trajectory and instruction into experience e^t = (q^t, P^t).
   - Apply an LM-based binary success evaluator L_eval(e^t) ∈ {0, 1} (Pan et al., 2024).
   - If success: induce {w^t} = I(e^t); update memory M^{t+1} = M^t ∪ {w^t}.
   - If failure: M^{t+1} = M^t (no workflow induction from failures).
3. After processing all test queries, report average success rate over the predicted trajectories.

**Per-website grouping**: workflows are partitioned by associated website, so each agent invocation only sees the workflow subset relevant to the current site. Keeps memory small and targeted.

**Workflow composition** is implicit: an induced workflow can serve as a sub-routine inside a more complex later-induced workflow (e.g., "find a place by its name" becomes a sub-step of "get the zip code of a place"). No explicit composition operator is implemented; composition emerges from the LM's induction prompt seeing earlier workflows alongside new trajectories.

## Assumptions

- The base LM is strong enough to perform reliable induction (reported with GPT-4 and, in some settings, GPT-3.5-turbo; not validated for smaller open-source models).
- The LM evaluator L_eval has acceptable false-positive rate; otherwise wrong workflows accumulate.
- The action space is fixed during deployment (AWM does not modify which actions the agent has access to; it modifies the *guidance* the agent receives).
- Per-website partitioning is appropriate, i.e., workflows do not need to cross websites within a deployment.
- The workflow library can grow monotonically without retirement; no compression or pruning policy is required at the scales studied.

## Limitations

- **No revision**: once stored, a workflow cannot be retracted or edited when later trajectories reveal it was wrong.
- **Evaluator-bound**: the online regime's quality is upper-bounded by the LM evaluator's accuracy.
- **Workflow over-commitment**: AWM can pull agents toward workflow-aligned actions when the current state actually demands a divergent action (visible as a small action-F1 drop on Mind2Web).
- **Per-website silo**: cross-site reuse is not supported; the partitioning is a heuristic.
- **No learned retrieval**: every relevant workflow is included in context; no policy chooses which workflows to surface.
- **Monotone library**: the memory only grows; long-horizon deployments would need a retirement / dedup mechanism not provided by AWM.

## Tradeoff profile

- **+** Operates without any human-authored workflows or training annotations (online regime).
- **+** Generalizes across tasks, websites, and domains — gains *widen* as train-test distribution gap widens (online > offline as gap grows).
- **+** Reduces steps-per-task: AWM solves WebArena tasks in 5.9 steps vs. 7.9 (BrowserGym) and 46.7 (AutoEval).
- **+** Compatible with multiple base LMs and action spaces.
- **−** Adds an extra LM call per induction step in the online regime.
- **−** Requires an LM evaluator for online use; its errors compound.
- **−** No mechanism to recover from incorrect workflows once they enter memory.
- **−** Workflow granularity, abstraction level, and per-website partitioning are all prompt/heuristic choices that may not transfer to other deployment regimes.
