---
title: "CASCADE: Cumulative Agentic Skill Creation through Autonomous Development and Evolution"
slug: cascade-cumulative-agentic-skill-creation-through
arxiv: "2512.23880"
venue: arXiv
year: 2025
tags:
  - skill-evolution
  - agentic-memory
  - self-reflection
  - continuous-learning
  - llm-agent
  - tool-use
  - mcp
  - scientific-discovery
  - materials-science
  - benchmark
importance: 4
date_added: 2026-05-18
source_type: tex
code_url: "https://github.com/CederGroupHub/CASCADE"
tldr: "CASCADE is a self-evolving multi-agent framework where DeepSolver cultivates two meta-skills (continuous learning and self-reflection) plus consolidated memory, raising GPT-5 success on the 116-task SciSkillBench from 35.4% (no evolution) to 93.3%."
contribution_type:
  - system
  - benchmark
  - method
datasets:
  - SciSkillBench
cited_by: []
---

## Problem & Context

The prevailing "LLM + tool use" paradigm wires agents up with a fixed catalog of human-curated tools and task-specific prompts. This scales poorly: domain experts must hand-author tool wrappers, and the agent's adaptability stops where the catalog stops. Earlier autonomous tool-generation work (e.g., LATM / CRAFT) generates Python utilities but is typically limited to built-in libraries and lacks mechanisms for sustained skill mastery, human–agent collaboration, or memory-based consolidation. In the scientific-discovery setting specifically, existing agents excel at literature search, hypothesis generation, and shallow data analysis, but stumble when complex long-horizon experiments require chaining unfamiliar packages, calling APIs in correct sequence, and recovering from runtime errors. The paper frames this as an inflection point analogous to the human transition from opportunistic tool use to cumulative skill acquisition, and asks how an agent can autonomously master external tools and codify the experience as reusable, transferable skills.

## Key idea

Reframe the paradigm from "LLM + tool use" to "LLM + skill acquisition". Instead of giving the agent a domain-specific tool list, equip a multi-agent system with two **meta-skills**: (1) continuous learning via web search, on-the-fly code extraction, and memory utilization; and (2) self-reflection that goes beyond self-debugging — covering code introspection, runtime probing, knowledge-graph exploration of local packages, and diagnostic scripting. Wrap these in a multi-agent solver (DeepSolver) inside a larger system (CASCADE) that adds session-wise + consolidated memory and natural-language human–agent collaboration. The agent's tools are intentionally domain-agnostic; specialization comes from accumulated skills, not hard-coded wrappers.

## Method

CASCADE's architecture (see [[deepsolver]] for the inner solver and [[sciskillbench]] for the benchmark) is built around an **Orchestrator agent** that mediates multi-turn dialogue, retrieves from consolidated memory (skills, preferences, credentials, experience), and routes each query to one of two pathways:

- **SimpleSolver** — a quick path inside the Orchestrator for tasks solvable by adapting memory matches; writes code, checks/installs packages, executes once. On failure, escalates to DeepSolver.
- **DeepSolver** — a four-step sequential workflow with conditional parallel debugging: (1) **Solution Researcher** searches web, extracts code from URLs, and drafts initial code; (2) **Code Agent** installs missing dependencies and executes once, deciding whether debugging is needed; (3) on failure, **three Debug Agents** run concurrently with different strategies (Direct Fix / Introspection-Probe Fix / Knowledge-Graph Fix / Local-Package Fix / Research Fix / Diagnostic Fix / Result-Processing Fix); (4) **Output Processor Agent** selects the best result and formats the response, preferring outputs that multiple debug agents agreed on.

Tools are exposed through four **MCP servers** plus equivalent direct functions:

- **Tavily MCP** — `tavily-search` for real-time web search.
- **Memory server** (built on mem0) — `save_to_memory` and `search_memory` over a hybrid vector (Supabase/PostgreSQL) + graph (Neo4j) store; uses GPT-4o-mini for memory extraction. Dual-path writes: LLM-curated diffs plus verbatim original text.
- **Research server** — `extract_code_from_url`, `retrieve_extracted_code` (vector RAG over extracted code), `quick_introspect` (Jedi-based static analysis), `runtime_probe_snippet` (KeyError/AttributeError probes), `parse_local_package` (AST → Neo4j), `query_knowledge_graph` (Cypher).
- **Workspace server** — `check_installed_packages`, `install_dependencies`, `check_package_version`, `execute_code`, `execute_shell_command`, `create_and_execute_script`, `read_file`, plus directory-scoping to prevent benchmark-task leakage.

The conversational front-end is a Streamlit app with Supabase Auth; session memory uses SQLiteSession from the OpenAI Agents SDK, and MLflow records full execution traces. Consolidated memory is user-scoped.

## Experiment & Results

**Benchmark: SciSkillBench** — a new curated benchmark of 116 materials-science / chemistry research tasks across six categories: 76 data-oriented (22 retrieval, 24 analysis, 4 management, 26 processing) and 40 computation-oriented (32 simulation, 8 specialized-models/toolkits). Tasks are split into 58 Level-0 (function-level guidance) and 58 Level-1 (only high-level objective) items, and stratified by P-value (>=0.5 simple, <0.5 difficult) computed across 28 model configurations.

**Headline numbers (Level-0 + Level-1 combined success rate, GPT-5):**

- Native (no evolution): **35.36%**
- Search+Debug (S&D) baseline: 89.74%
- **DeepSolver: 93.26%** — pass@2 = 97.41%, pass@3 = 98.28%; Level-0 pass@2 = pass@3 = 100%.

**Robust across backbones.** Improvements over Native are large for every tested LLM: O3 23.05% -> 91.84% (~69-point lift), GPT-5-mini 22.09% -> 82.18%, GPT-4.1-mini 16.03% -> 72.78%, Qwen3-Coder-30B 21.41% -> 64.38%. DeepSolver also beats S&D by an average of 17.53 points across models. Notably, DeepSolver + GPT-4.1-mini reaches pass@3 = 89.66% in 190s — exceeding the Claude-Code (Sonnet-4.5) baseline at pass@3 = 87.93% / 239s while costing ~7.5–9.4× less.

**Ablations** isolate the two meta-skills (Appendix B.2): in most cases self-reflection alone outperforms the combined web-search + self-debugging conventional augmentation, indicating self-reflection is the dominant driver of robustness.

**Real-world demonstrations.** Four scenarios on the OpenAI O3 backbone: (i) piezoelectricity determination on a Materials-Project structure, solved first-attempt by selecting pymatgen and reasoning about point-group exclusions; (ii) predicting systematic GGA/GGA+U vs r²SCAN error patterns of CHGNet — CASCADE formulated and validated the hypothesis without the target article being online at test time; (iii) autonomous lab integration with AlabOS / proprietary Alab-GPSS to synthesize Li2Fe0.8Ni0.2Cl4 and fit EIS data via the `impedance` package — its fit was ~60x faster and slightly higher R^2 (0.991 vs 0.989) than the lab's hand-tuned baseline; (iv) reproducing Li_xCoO2 intercalation voltage curves from a published paper via memory transfer (LiCoO2 → CoO2 delithiation insight) across four SevenNet MLIPs.

## Limitations

- **Reasoning ceiling.** GPT-4.1-nano stays near zero on difficult tasks even with DeepSolver, indicating the framework cannot rescue insufficient model capability or prompt-following.
- **Cost vs accuracy.** DeepSolver takes longer than Native or S&D (e.g., GPT-5: 588s vs 240s / 504s) because parallel debug agents and multi-step research add overhead.
- **Memory disabled during benchmark.** The headline SciSkillBench numbers come with memory tools disabled, so they understate (but don't quantify) the contribution of consolidated memory in practice.
- **Outcome-based evaluation.** Tolerance-bounded final-answer match doesn't certify intermediate correctness — a wrong-but-fortunate execution could pass.
- **Lab-safety gating.** The autonomous lab demonstration required human verification of generated code before activating physical devices; CASCADE is not yet trusted to execute lab actions unattended.
- **Long-horizon iteration.** End-to-end autonomous discovery loops with iterative hypothesis refinement still require human seed steps and clarification turns.

## Open questions

- How to balance plasticity (continuous learning) against reliability (don't overwrite working skills) without manual schedules?
- What unit of procedural memory makes the strongest cross-task transfer signal — verbatim code snippets, summarized recipes, or graph-typed entities?
- Can the domain-agnostic design transfer to software engineering, biology, or robotics with the same tool surface but different memory seeds?
- How should the system price compute when difficulty is unknown — i.e., when to abort DeepSolver early versus continue debugging?
- Can self-reflection alone (without web search) be made strictly dominant on a wider range of tasks?

## My take

The paper's strongest contribution is reframing where the locus of specialization lives: not in the tool inventory, but in the meta-skill loop. The ablation showing self-reflection alone often outpaces search+debug is the most surprising and the most theoretically informative result — it argues that *what to do when execution fails* matters more than *what to read before execution starts*. The SciSkillBench gap between Level-0 and Level-1 questions is also useful evidence that current agents bottleneck on autonomy more than on knowledge. The autonomous-lab and reproduction demos are persuasive case studies but the headline benchmark numbers are still the load-bearing claim.

## Related

- Topics: [[skill-evolution]], [[agentic-memory]], [[procedural-memory]], [[continual-learning-evaluation]]
- Methods introduced: [[deepsolver]], [[sciskillbench]]
- Concepts introduced: [[llm-skill-acquisition-paradigm]]
- People: [[xu-huang]], [[junwu-chen]], [[philippe-schwaller]], [[gerbrand-ceder]]
