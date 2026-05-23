---
title: Environment-Learned Agent
aliases:
  - environment-learned CUA
  - environment-driven skill acquisition
  - pre-inference environment learning
  - bottom-up exploration agent
tags:
  - computer-use-agent
  - environment-learning
  - skill-library
  - procedural-memory
  - bottom-up-agent
maturity: emerging
definition: "A computer-use agent that, prior to deployment, runs a dedicated environment-learning phase to autonomously acquire a verified, environment-specific skill set, and at inference time executes those skills instead of trial-and-error exploring."
key_papers:
  - osexpert-computer-use-agents-learning-professional
first_introduced: "OSExpert (Liu et al., 2026)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

An *environment-learned agent* explicitly separates a *learning phase* (interact with the target digital environment, no user queries, no human annotation, produce a verified skill set) from an *inference phase* (load the skill set and execute verified procedures). The skill set carries environment-specific procedural knowledge, recorded failures, and primitive-instances for fine-grained control; it is meant to be loaded per-app rather than shared across all applications.

## Intuition

General CUAs trained on cross-environment human demonstrations behave like amateurs in any specific environment: they know how UIs *generally* work but not how *this* application's workflow is laid out. Environment-learned agents amortize the cost of figuring that out into an explicit pre-inference loop, so that at inference time the agent already knows which buttons do what, which composite tasks chain reliably, and which actions are out of reach. The split echoes how human experts develop application proficiency: spend time mastering the tool, then operate the tool quickly.

## Variants

- **Goal-free environment learning** (OSExpert): exploration is driven entirely by the environment's structure, with no user queries.
- **Query-driven environment learning** (AppAgent-v2, SEAgent, Mobile-Agent-E): exploration is conditioned on user-provided or tutorial-derived queries; the skill set inherits whatever distribution those queries impose.
- **Lifelong / continuous environment learning**: exploration continues at deployment time, with the skill set treated as a growing artifact (proposed as future work in OSExpert).

## Comparison

Distinct from inference-time-scaling agents (Agent-S3, CoAct-1, Computer-Use-Preview), which carry no per-environment memory and recover from unfamiliarity only by spending more test-time compute. Distinct from end-to-end CUA backbones (OpenCUA, UI-TARS) which absorb generic UI priors during pre-training but cannot adapt to a new app at inference time without re-training.

## Known limitations

- Quality of the skill set is bounded by the backbones used during exploration; weak planners / action modules produce noisy skills.
- Exploration is compute-heavy; the cost is paid once per environment but does not amortize across applications that share only generic patterns.
- Skills are tied to a specific UI version / theme; major UI changes can invalidate large portions of the skill set.

## Open problems

- Cross-environment skill transfer: a skill learned in LibreOffice Writer should partially seed Microsoft Word.
- Compact, queryable representations of large skill sets (the OSExpert version stores them as text + embeddings).
- Mixing autonomous exploration with sparse, opportunistic human supervision when failure modes are persistent.

## Relationship to foundations

The paradigm sits between *task-conditioned RL* (which trains on episodes labeled by reward) and *imitation learning from demonstrations* (which trains on annotated trajectories): it gathers its own training signal from environment interaction but uses verification rather than reward to consolidate trajectories into reusable units.

## My understanding

Environment-learned agents reframe "deployment" as "deployment after pre-inference training in this specific environment", which is a meaningfully different design point from today's general-purpose CUAs. The framing is more important than any one instantiation: it carves out a research direction (how to learn an application-specific skill library) that's orthogonal to the current race for larger and better generalist CUA backbones.
