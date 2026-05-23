---
title: Agent Workflow Memory
slug: agent-workflow-memory
arxiv: "2409.07429"
venue: ICLR 2025
year: 2024
tags:
  - procedural-memory
  - agent-skills
  - workflow-induction
  - web-navigation
  - lifelong-learning
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: Induces reusable, abstract workflows from agent trajectories and integrates them into agent memory, enabling LM-based web agents to learn from past experiences in both offline and online (supervision-free) regimes.
contribution_type:
  - method
  - system
datasets:
  - WebArena
  - Mind2Web
code_url: https://github.com/zorazrw/agent-workflow-memory
cited_by: []
---

## Problem & Context

LM-based agents have demonstrated promise on digital tasks like web navigation (WebArena, Mind2Web) and mobile app control, yet current methods integrate a *fixed* set of demonstrations through either training or in-context prompting. This produces agents that are brittle to changes in task context, environment, or websites — they can mimic action sequences similar to the supplied examples but fail to **disentangle complex tasks into the reusable sub-routines that actually carry across tasks**. Furthermore, because each task is solved independently, agents do not accumulate procedural knowledge over a deployment lifetime: success and failure trajectories produce no durable artifact that improves later behavior.

Prior to AWM, the field clustered around three responses: (i) modifying the action space (e.g., constraining the search, allowing LM self-feedback, or hard-wiring human-designed actions [[daniel-fried]]'s SteP baseline); (ii) augmenting agent memory with raw demonstrations or retrieved examples (Synapse, AutoGuide); (iii) human experts writing explicit workflows by hand (SteP). All of these either depend on high-quality canonical examples that are not always available, or on human-engineered domain-specific procedures that do not transfer. The open gap was a mechanism that lets the agent *itself* extract reusable sub-routines from its own experience, with or without supervision, and grow them over the deployment lifetime.

## Key idea

Build a **second tier of agent memory** that stores *workflows* — abstract, named sub-routines extracted from past trajectories — alongside the default tool/action memory. Workflows are inferred (not hand-written), stored as `(description, step_sequence)` pairs with environment-specific values abstracted to placeholders (e.g., `"dry cat food"` becomes `"{product-name}"`), and merged back into the agent's prompt context as additional procedural knowledge. The same mechanism applies in two operating regimes: *offline* when annotated training experiences exist, and *online* when only a stream of test queries is available and successes are judged by an LM evaluator. The crucial bet is that **abstract sub-routines reuse better than concrete demonstrations**: an induced "search for a product on Amazon" workflow generalizes across product types, where retrieving the literal "buy dry cat food" example does not.

## Method

AWM consists of an **induction module** I that maps a set of past experiences E = {(q, P)} to a workflow set W = {(d, P)}, plus a **memory integration** step that augments base memory M with W to form M_w = M + W. The agent thereafter generates actions via L(q, M_w, o) → a.

**Workflow representation.** Each workflow is (1) an NL description d (a short summary of the workflow's function) and (2) a step sequence (p_1, ..., p_k) where each step p contains an NL environment-state description, an explicit reasoning thought, and an executable program action (e.g., `stop()`, `CLICK(submit-id)`).

**LM-based induction.** I is implemented by prompting the agent LM to extract *common sub-routines* from one or more experiences. Two abstraction levers matter: (a) *granularity* — induce fine-grained sub-tasks (`"search for a product"`) rather than copy whole task instructions (`"buy dry cat food on Amazon and deliver to my address"`); (b) *value abstraction* — replace example-specific entities with placeholder names (`{product-name}`) so the workflow transfers across instances. Workflows are segmented on double-line breaks and stored separately in the workflow memory. The paper also explores rule-based induction (action-sequence dedup followed by environment-invalid-step filtering) as an ablation.

**Offline regime.** When canonical training experiences E_train exist (e.g., human-annotated demonstrations), AWM concatenates them into a single prompt and runs induction once: I(E_train) → W_offline. At inference, the workflow memory is fixed and used to solve every test query: L(q, M + W_offline, o) → a.

**Online regime.** When no training set is available, AWM processes test queries in streaming fashion. For each test instruction q^t, the agent solves it under memory M^t, producing experience e^t. An LM-based binary success judge L_eval (from [Pan et al., 2024]) labels e^t. Successful experiences are passed through I to produce w^t, which is appended to memory: M^{t+1} = M^t + {w^t}. This induces a **snowball effect** — once "find a place by its name" is in memory, it serves as a sub-component for inducing "get the zip code of a place," and workflow complexity grows monotonically with experience.

**Operating axes.** AWM is orthogonal to: the base agent LM (tested with GPT-4 and GPT-3.5-turbo), the underlying action space (works with default WebArena actions and with BrowserGym's modified action space), and the website domain (per-website workflow grouping keeps the memory targeted).

## Experiment & Results

Benchmarks: **WebArena** (812 execution-based tasks on five websites: Shopping, CMS, Reddit, GitLab, Maps) and **Mind2Web** (cross-task / cross-website / cross-domain element-and-action evaluation with element accuracy, action F1, step success rate, task success rate). Base model: `gpt-4-0613` (also `gpt-3.5-turbo` on Mind2Web), temperature 0.0.

**WebArena (online setting only; no training set).** AWM reaches **35.5% total success rate**, vs. BrowserGym 23.5% (the prior top autonomous method) — a **+12.0 absolute / +51.1% relative** improvement. AWM also beats SteP (33.0%), which uses 14 human-written workflows, by +7.6% relative despite using *no* human supervision. Per-website breakdown shows +11.8 to +30.7 absolute gains over BrowserGym across all five sites. Average steps-to-solve drops from 7.9 (BrowserGym) and 46.7 (AutoEval) to **5.9**, so improvement is not just from longer trajectories.

**WebArena cross-template.** On a cross-template subset (one example per task template, removing in-template overlap), AWM still leads at **33.2%** total SR vs. BrowserGym 20.5% and SteP 32.1%. Improvements are not artifacts of template overlap.

**Mind2Web offline (cross-task).** With GPT-4: AWM hits **50.6% element accuracy / 45.1% step SR / 4.8% task SR** vs. MindAct GPT-4 (41.6 / 36.2 / 2.0). +24.6% relative step SR. With GPT-3.5: AWM 39.0 / 34.6 / 2.8 vs. Synapse 34.0 / 30.6 / 2.4. Gains come predominantly from element selection (+5–9 EA), while action F1 is slightly lower than MindAct (workflows occasionally push agents toward workflow-aligned actions misaligned with the current state).

**Mind2Web generalization (cross-website, cross-domain).** AWM offline / online gain +8.9 / +7.4 (cross-task step SR), +3.6 / +3.8 (cross-website), +14.0 / +16.9 (cross-domain) absolute step-SR points over MindAct. **The margin widens with the train-test gap**: online AWM dominates as the domain shift grows because it sources workflows directly from test-distribution queries; offline AWM suffers when training and test distributions diverge.

**Ablations.** Rule- vs. LM-based induction performs comparably on WebArena (35.6 vs 35.5 SR) but LM-based wins by +2.8 on Mind2Web because abstract placeholders reduce element-selection bias. Code- vs. text-format workflows are roughly comparable. NL state descriptions beat HTML in workflow steps; combining them is *worse* than either alone (longer context + filter-induced contradictions).

## Limitations

- **Failure modes in online induction**: when an LM evaluator falsely judges a failed trajectory as successful, the induced workflow can be incorrect and degrade subsequent performance; AWM has no mechanism to revise or retract workflows once stored.
- **Workflow-action divergence**: even when workflows guide accurate element selection, action F1 slightly drops vs. MindAct on Mind2Web — agents have trouble identifying when to *diverge* from a workflow when the current state demands a different action.
- **Per-website grouping is heuristic**: AWM partitions workflow memory by website, which keeps memory small but does not scale to many domains; there is no learned retrieval over a unified workflow store.
- **GPT-4 dependence**: the LM-based induction module relies on a strong base model; how AWM behaves with smaller / open-source backbones is not studied.
- **No skill retirement / pruning**: the workflow memory only grows. Long horizons may accumulate stale or contradictory workflows.

## Open questions

- Can workflows be **revised** post-hoc when later evidence contradicts them (the LM evaluator was wrong, or a website changed)?
- What is the right **retrieval policy** for a unified workflow library spanning many websites and tasks? Per-website grouping is a stopgap.
- How does AWM compose with **action-space modifications** like SteP's hand-written skills — are induced workflows and authored skills additive or redundant?
- What does the **scaling behavior** look like? AWM's curve plateaus on WebArena after ~40 queries; is that an artifact of test-set size or a real saturation?
- Is there a principled **cross-domain transfer** signal for workflows, or does cross-domain success require always re-inducing from test queries (the online setting)?

## My take

AWM is the cleanest existing instantiation of the *bottom-up procedural memory* hypothesis: don't hand-author workflows, induce them from trajectories and abstract away the example specifics. The fact that the online regime — which sees only test queries and self-judges success — substantially outperforms even SteP's 14 human-written workflows on WebArena is the load-bearing result. It demonstrates that procedural memory does not need a curated training set or an extraction operator hand-crafted by a designer; an LM can perform induction on its own trajectories well enough to drive real-world performance.

Where I think the work invites follow-up: the workflow store is currently monotone (append-only, per-website silo). The interesting next moves are (a) learned retrieval over a shared store, (b) revision / retirement of workflows that turn out to be wrong, and (c) explicit modeling of when to *deviate* from a workflow — the slight action-F1 drop on Mind2Web suggests workflows can over-commit the agent. These are exactly the failure modes that distinguish AWM's current "skill library that grows" from a true *evolving* procedural memory.

## Related

- [[workflow-induction]]
- [[reusable-workflow]]
- [[zora-zhiruo-wang]]
- [[jiayuan-mao]]
- [[daniel-fried]]
- [[graham-neubig]]
