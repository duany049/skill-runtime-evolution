---
name: Linxi "Jim" Fan
affiliation: NVIDIA
research_areas:
  - embodied-ai
  - foundation-models-for-agents
  - open-ended-environments
  - multimodal-agents
  - minecraft-agents
homepage: ""
scholar: ""
date_updated: 2026-05-20
type:
  kind: researcher
---

## Research areas

Foundation models for embodied agents. Built MineDojo (the open-source Minecraft benchmark Voyager runs on) and co-advised Voyager. Research thread spans MineDojo (data-and-simulation infrastructure) → VIMA (multimodal task specification) → Voyager (LLM-driven lifelong learning) — a deliberate stack from environment to embodiment to agent.

## Recent work

- [[voyager-open-ended-embodied-agent-large]] — equal-advising senior author and corresponding author; co-led the Voyager project that established the canonical "automatic curriculum + executable skill library + iterative prompting" architecture for LLM-driven lifelong embodied agents.

## My notes

Fan's MineDojo benchmark is the substrate that makes Voyager-style work possible, and his arc from benchmark → multimodal control → frozen-LLM agent is a textbook example of vertically integrated research infrastructure. The fact that Voyager runs entirely through black-box GPT-4 calls (no parameter access) reflects the NVIDIA Research stance of treating large frozen models as fixed substrates to be scaffolded around rather than trained.
