---
title: Continual Learning Evaluation
tags:
  - continual-learning
  - benchmark
  - evaluation
  - skill-usage
key_venues: []
related_topics:
  - procedural-memory
  - skill-evolution
  - agentic-memory
key_people: []
linked_ideas: []
---

## Overview

This topic tracks benchmarks and evaluation protocols designed specifically for *continual* learning in LLM agents: settings where an agent encounters a stream of tasks, must accumulate reusable knowledge across them, and is measured on whether that accumulation actually helps (or hurts) downstream tasks. Distinct from one-shot skill use benchmarks, the focus is on lifelong protocols — agents begin without skills, externalize lessons over time, and carry an updated library forward.

A recurring finding is that high skill *usage* does not imply high skill *utility*: agents may invoke their memory aggressively while gaining little, or use it sparingly while improving substantially. Untangling these phenomena is the methodological core of this topic.

## Timeline

## Seminal works

## SOTA tracker

## Key benchmarks

## Open problems

### Known gaps

- Most benchmarks fix the task family; cross-domain continual learning is largely uncovered.
- Negative transfer (skills that hurt new tasks) is rarely measured explicitly.
- Realistic deployment conditions (rate limits, latency, noisy environments) are usually absent from evaluation harnesses.

### Methodological gaps

- Metrics often reward *any* skill invocation rather than *correct* invocation; calibration of skill-use credit assignment is open.
- Reproducibility across LLM backbones is limited because closed models change over time.
