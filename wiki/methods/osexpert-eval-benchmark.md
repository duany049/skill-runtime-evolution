---
name: OSExpert-Eval Benchmark
slug: osexpert-eval-benchmark
type: benchmark
tags:
  - computer-use-agent
  - gui-agent
  - benchmark
  - long-horizon
  - fine-grained-action
  - latency-evaluation
  - generalization
source_papers:
  - osexpert-computer-use-agents-learning-professional
parent_methods: []
child_methods: []
code_repo: "https://github.com/Lumos-Jiateng/OSExpert"
date_updated: 2026-05-18
---

## Problem setting

Existing CUA benchmarks (OSWorld, OSUniverse, WinAgentArena, macOSWorld, Mind2Web, BEARCUBS) measure at most one or two of: long-horizon composite skills, generalization to unseen / creative UIs, fine-grained low-level actions, end-to-end latency. None measures all four together; OSWorld in particular emphasizes short, single-function tasks, which over-states current CUAs' real-world competence. OSExpert-Eval is built to evaluate professional expertise across all four dimensions simultaneously.

## Mechanism

113 evaluation tasks across six interactive GUI environments — LibreOffice Writer, LibreOffice Calc, LibreOffice Impress, GIMP, Tableau Public (web-based), and **MiniWord** (an in-house, creative-UI text editor designed specifically to minimize memorized UI conventions). Tasks are partitioned along three axes:

- **Long-horizon composite skills** (GIMP, LibreOffice subsets): require chaining multiple unit functions to reach a higher-level goal; longest-horizon task is 14 expert-steps.
- **Unseen-UI generalization** (Tableau, MiniWord subsets): Tableau is rarely represented in CUA training distributions; MiniWord uses redesigned icons and alternative layouts.
- **Fine-grained low-level actions** (GIMP, LibreOffice subsets): precise spatial control — text-span selection, drag-and-drop, object-boundary tracing, exact-angle rotation.
- **Execution efficiency** (cross-cutting, applied to every task): elapsed wall-clock seconds, success or failure.

Human expert reference times are reported for every subset (11–48 s across subsets).

## Procedure

- Cap each episode at 30 interaction steps for all methods (no Best-of-N).
- Base model temperature 1.0; OpenAI models via official API; open-source models via vLLM.
- Report avg success rate ± std over 3 independent runs per task, separately per environment.
- Report avg completion seconds ± std over 3 independent runs.

## Assumptions

- The six chosen environments are representative of professional desktop GUI workflows.
- The 30-step cap and single-run policy approximate practical deployment constraints.
- Human-expert reference times establish the lower bound; success-rate reference is taken as 100% by construction (humans solved each task during curation).

## Limitations

- Desktop-only; no mobile, no continuous real-time environments, no purely web-native automation beyond Tableau Public.
- Relatively small (113 tasks); coverage of any single environment is modest.
- Human-expert wall-clocks reflect *practiced* operators on these specific tasks, not first-encounter users; the efficiency gap should be interpreted accordingly.
- The benchmark is co-designed with the proposed framework — separating "good benchmark" from "benchmark that flatters this method" requires independent validation.

## Tradeoff profile

The four-axis split intentionally trades benchmark *size* for benchmark *coverage along under-served dimensions*. Compared with OSWorld (large, narrow) or OSUniverse (long-horizon + fine-grained but no latency evaluation), OSExpert-Eval offers the only joint measurement of long-horizon, unseen-UI, fine-grained, and latency, at the cost of fewer tasks per axis. The contrast table (Table 1 in the paper) is explicit about what each prior benchmark covers and what it skips.
