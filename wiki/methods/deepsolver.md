---
name: DeepSolver
slug: deepsolver
type: system
tags:
  - multi-agent-system
  - llm-agent
  - self-reflection
  - continuous-learning
  - mcp
  - skill-evolution
source_papers:
  - cascade-cumulative-agentic-skill-creation-through
parent_methods: []
child_methods: []
code_repo: "https://github.com/CederGroupHub/CASCADE"
date_updated: 2026-05-18
---

## Problem setting

DeepSolver is the heavyweight problem-solving engine inside CASCADE. It activates when the Orchestrator routes a query as complex, or when SimpleSolver's one-shot execution fails. The problem it tries to solve: produce correct, executable code for an open-ended scientific-research query whose required tools, packages, and APIs are not specified up front and may include freshly-released libraries or undocumented proprietary code. It must (a) discover and read external documentation, (b) write and execute code in a managed environment, (c) recover from runtime errors via diverse debugging strategies, and (d) emit a final processed answer with traceable provenance.

## Mechanism

DeepSolver coordinates **four specialized agents** in a sequential workflow with conditional parallel debugging. Each agent has narrowly scoped tool access through CASCADE's MCP servers (Tavily, Memory, Research, Workspace) so that responsibilities don't leak:

- **Solution Researcher** — initiates the workflow. Has access to memory search, web search, code extraction from URLs, vector RAG over extracted code, and optional `quick_introspect`. Plans and decomposes for complex queries, selects required packages, and emits an initial complete code solution.
- **Code Agent** — receives the initial solution and executes exactly once (no retries, no self-debugging). Manages packages via `check_installed_packages` / `install_dependencies`. Decides via a boolean flag whether debugging is needed.
- **Debug Agents (x3, in parallel)** — invoked only when the Code Agent flags failure. The three instances run concurrently and are encouraged to employ different strategies from a documented toolbox: **Direct Fix** (obvious errors), **Introspection/Probe Fix** (`quick_introspect`, `runtime_probe_snippet`), **Knowledge-Graph Fix** (`parse_local_package` + `query_knowledge_graph` Cypher), **Local Package Fix** (shell exploration + `read_file`), **Research Fix** (Tavily search + code extraction), **Diagnostic Fix** (probe scripts), **Result Processing Fix** (clean-up of malformed output).
- **Output Processor Agent** — receives all results, evaluates each on execution success / required-data presence / quality, and selects the best. Preference is given to identical outputs from multiple debug agents (majority vote increases confidence).

The two **meta-skills** are emergent from this architecture rather than hard-coded:

- **Continuous learning** — Solution Researcher and Debug Agents acquire targeted external knowledge in real time (web search, URL code extraction) and adaptively leverage prior skills via `search_memory`.
- **Self-reflection** — Debug Agents apply Introspection/Probe / Knowledge-Graph / Diagnostic Fix strategies that go beyond simple stack-trace-driven self-debugging; they probe runtime state, traverse local-package knowledge graphs, and write diagnostic code to investigate causes.

## Procedure

1. Orchestrator deems a query complex (or SimpleSolver has failed) and dispatches to DeepSolver with the user query, identifier, and a memory pointer.
2. Solution Researcher: search memory → web search → identify required software → extract code from URLs → optional quick introspect → synthesize initial code solution.
3. Code Agent: optionally search memory → verify with research tools if needed → check / install packages → execute code exactly once → set `needs_debug` flag → emit execution output.
4. If `needs_debug` is false, the Output Processor returns the result directly.
5. If `needs_debug` is true, three Debug Agents launch concurrently, each:
   - Analyzes error.
   - Selects 1+ strategies from the seven-item toolbox.
   - Tests fix candidates with `execute_code`.
   - Returns a candidate result.
6. Output Processor Agent compares the three candidates plus context, picks the best (or none-of-the-above if all failed), formats the final response, and returns to Orchestrator.

For observability, MLflow records the full execution trace: agent phases, tool calls with inputs/outputs/timings, memory operations, and intermediate code artifacts.

## Assumptions

- The underlying LLM is capable enough to plan, write Python, and follow tool-use prompts. (Low-capability models like GPT-4.1-nano show near-zero rescue benefit on difficult tasks.)
- The workspace permits arbitrary package installation and shell command execution within a scoped directory (no benchmark-task leakage).
- Web search and external code repositories are reachable. Memory tools are optional during evaluation but assumed available in production.
- Tasks have outcome-checkable answers (the benchmark evaluation depends on this; production use does not strictly require it but loses an evaluation channel).

## Limitations

- **Latency.** Parallel debug agents plus full Solution-Researcher discovery push wall time higher than Native or Search+Debug baselines (GPT-5: 588s vs 240s/504s).
- **Floor-not-ceiling.** Improvements over Native are large but DeepSolver cannot lift a model whose prompt-following or coding is fundamentally insufficient (GPT-4.1-nano remains near 0% on difficult tasks).
- **Single-shot Code Agent.** The Code Agent's strict no-retry rule keeps responsibilities clean but means every error path goes through three full Debug Agents — efficient on hard problems, wasteful on easy ones.
- **Memory ablation in benchmark.** Headline numbers are reported with memory disabled, so they understate (and don't quantify) the contribution of consolidated memory.
- **Outcome-based grading.** Tolerance-bounded final-answer match doesn't validate intermediate-step correctness.

## Tradeoff profile

Compared to a single-pass Search+Debug agent: DeepSolver trades **latency and cost** for **success rate and robustness on difficult tasks**. Compared to a long-running ReAct-style self-debugging loop: DeepSolver trades **a deep linear retry tree** for **a wide-then-vote pattern** (three parallel debug agents, output selection), which is more amenable to majority-vote confidence and bounded latency. Compared to a Claude-Code-style monolithic agent: DeepSolver achieves higher pass@3 with much cheaper LLMs (e.g., GPT-4.1-mini at ~1/8 the input cost still beats Claude-Sonnet-4.5 on the benchmark), at the cost of orchestration complexity.
