---
title: "Memento-Skills: Let Agents Design Agents"
slug: memento-skills-let-agents-design-agents
arxiv: "2603.18743"
venue: arXiv
year: 2026
tags:
  - agent-skills
  - skill-evolution
  - procedural-memory
  - continual-learning
  - read-write-loop
  - skill-router
  - offline-rl
  - llm-agents
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: 2859af8b7e04397e65e78bf77ad7651e02a8726a
tldr: "An agent-designing agent that uses an executable skill library as growable, write-back memory; a behaviour-aligned InfoNCE skill router and a reflective skill-evolution loop drive +26.2% GAIA and +116.2% HLE accuracy without any LLM parameter updates."
contribution_type:
  - system
  - method
datasets:
  - GAIA
  - HLE
code_url: "https://github.com/Memento-Teams/Memento-Skills"
cited_by: []
---

## Problem & Context

Modern LLM agents are typically deployed as *frozen* models: parameters stay fixed after pre-training, so every adaptation must come from the input (prompt, context, retrieved memory). This makes the agent **stateless across deployments** — it cannot learn from its own execution experience, and falls back to whatever knowledge already lives in $\theta$ or the context window. Continual fine-tuning would close the gap but is impractical at deployment scale: cost, data hunger, and the brittleness of out-of-distribution generalisation make it a poor fit for live agents.

Prior work on agentic memory tries to recover continual learning by augmenting the agent with an external store: token-level memories (retrieve traces into the prompt), latent-level memories (learned state vectors), or parameter-level adapters. These are fragmented across representations and typically log raw episodic transitions; the policy that consumes them remains a static prompt over a frozen LLM. Memento~2 [[memento-2]] formalised this as the Stateful Reflective Decision Process (SRDP), where an episodic memory $\mathcal{M}_t$ is updated by Read–Write Reflective Learning and proved KL-regularised soft policy iteration converges to an optimal retrieval policy. But Memento~2 stored episodic traces; the memory unit did not carry *executable* know-how, and skill-shaped procedural memory remained an open problem.

Recent automatic skill-learning systems (GEPA-Skill, Letta Skill Learning, ProcMem) move toward procedural memory but stop at text-only `skill.md` guides — effectively prompt optimisation — and tend to overfit to single-task trajectories. The Claude Skills format packages instructions+scripts+resources into reusable folders but offers no learning mechanism: skills are authored by humans, not synthesised and refined from experience.

Memento-Skills attacks the gap between these two camps: it instantiates SRDP with *executable* skill folders as the memory unit (not raw traces, not text-only prompts) and adds a Read–Write loop that rewrites the policy materialised inside each skill folder after every failure.

## Key idea

Treat **executable skill folders as the unit of external memory**, and treat the **Read–Write loop over those folders as policy iteration**. A skill is not a logged trace and not a prompt-only guide — it is a directory with a declarative `SKILL.md`, helper scripts, and prompts that together encode a reusable, multi-step workflow. *Reading* a skill (selecting it for the current goal) is policy improvement; *writing* a skill (mutating its files based on post-hoc reflection on the execution trace and judge verdict) is joint policy evaluation + improvement at the skill level. Because each write rewrites the prompt or program that will run next, the policy materialised inside the skill is the thing that gets optimised — no LLM parameter updates required.

Two technical commitments make this work:

1. **A behaviour-aligned router, not a semantic one.** Pure cosine similarity over skill text picks lexically nearby skills that nevertheless execute the wrong workflow. The router is trained with single-step offline RL on synthetic positive/hard-negative pairs (multi-positive InfoNCE on a Qwen3-Embedding backbone), making retrieval optimise for behavioural utility — whether running this skill produces the right trajectory — rather than surface similarity.
2. **Skill evolution under a utility threshold with a unit-test gate.** After a failure, an LLM failure-attribution selector identifies the responsible skill (credit assignment); a skill rewriter patches it; if utility drops below threshold $\delta$, the system escalates to skill discovery (restructure or synthesise a new skill). All mutations pass through an automatic unit-test gate that rolls back regressions.

## Method

**Skill memory.** A skill memory $\mathcal{M}_t = \{c_i\}_{i=1}^{N_t}$ is a finite, growing set of skill folders. Each $c_i$ carries `SKILL.md` (declarative spec), helper scripts, and prompt fragments. The skill space evolves with $\mathrm{Write}(\mathcal{M}, s, a, r)$, which is not an append but a file-level rewrite of the targeted skill.

**Reflected MDP formulation.** Augment state to $x_t = (s_t, \mathcal{M}_t)$ to restore the Markov property. The transition kernel weaves the LLM decision kernel $p_{\mathrm{LLM}}(a \mid s, c)$ with the environment kernel $\mathcal{P}(s' \mid s, a)$ and the Write operator. Under bounded rewards and $\gamma < 1$, KL-regularised soft policy iteration converges to the optimal retrieval policy (Memento~2, Theorem 8).

**Read–Write loop.**
1. **Observe** task $q_t$; form augmented input $x_t = (q_t, \mathcal{T}_t)$ where $\mathcal{T}_t$ is tip memory.
2. **Read** (skill selection): $c_t \leftarrow \mathrm{Router}(x_t, \mathcal{S}_t)$. If $c_t = \varnothing$ and `CreateOnMiss` is enabled, synthesise a fresh skill.
3. **Execute** the multi-step workflow $a_t \leftarrow \mathrm{LLM}(x_t, c_t)$.
4. **Feedback**: judge returns reward $r_t \in \{\textsc{correct}, \textsc{incorrect}\}$.
5. **Write** if $r_t = \textsc{incorrect}$:
   - update utility $U_{t+1}(c_t) = n_{\text{succ}}/(n_{\text{succ}} + n_{\text{fail}})$;
   - extend tip memory with a generic tip;
   - run failure-attribution selector to pick $c^{\dagger}$;
   - if $U_t(c^{\dagger}) < \delta$ and $n(c^{\dagger}) \geq n_{\min}$, escalate to skill discovery; otherwise patch $c^{\dagger}$ in place;
   - run a synthetic unit-test through the updated skill; rollback on regression;
   - retry up to $K$ feedback rounds.

**Behaviour-aligned router.** Crawl ~8k public skill repos (filter `stars > 500`, SHA-256 dedup), then sample ~3k seeds and use an LLM to synthesise routing goals from each seed's skill name + description (judge filters quality). For minibatch $B$ skills $\{d_i\}$ with positives $\mathcal{Q}_i^+$ and hard negatives $\mathcal{Q}_i^-$, minimise multi-positive InfoNCE (temperature $\tau$). Cast routing as a one-step MDP with reward $r(q,d)$; the learned score is a soft $Q$-function $Q_\theta(q,d) \propto s(d,q)$, giving a Boltzmann routing policy that maximises a KL-regularised value objective with uniform prior — single-step offline policy improvement for retrieval. Backbone: Qwen3-Embedding-0.6B.

## Experiment & Results

**Setup.** All experiments use Gemini-3.1-Flash as the frozen LLM. Two benchmarks: GAIA (165 questions: 100 train / 65 test) and Humanity's Last Exam (HLE; 788 train / 342 test, evenly spread across 8 subjects). Baseline is *Read-Write ablation*: same loop and retrieval, but skill-level optimisation (failure attribution, rewriting, discovery) is disabled. Up to three reflective retries per question.

**Router evaluation (140 synthetic routing queries).** Recall@1 climbs from 0.32 (BM25) and 0.54 (Qwen3 baseline embedding) to **0.60** for Memento-Qwen; Recall@10 reaches 0.90. End-to-end route hit rate rises from 0.29 (BM25) / 0.53 (Qwen3) to **0.58**, and judge success rate from 0.50 / 0.79 to **0.80**. The large jump over BM25 confirms that lexical overlap is a poor proxy for behavioural utility; the smaller-but-consistent gain over Qwen3 shows offline RL fine-tuning injects behavioural signal that semantic embeddings miss.

**GAIA.** Training accuracy rises from 65.1% (round 1) to 91.6% (round 3). On the held-out test set, full Memento-Skills hits **66.0%** vs **52.3%** for the Read-Write ablation — a **+13.7 pp** absolute (+26.2% relative) gain attributable specifically to skill optimisation. The skill library grows to 41 skills.

**HLE.** Training accuracy rises from 30.8% (R0) to 54.5% (R3); Humanities and Biology gain the most (66.7%, 60.7% by R3), Engineering saturates earlier (42.1%). Test-set: Memento-Skills **38.7%** vs Read-Write **17.9%** — **+20.8 pp** absolute (+116.2% relative). The skill library grows to 235 skills, distributed across HLE's 8 subject clusters in t-SNE projection.

**Cross-task transfer.** GAIA's diverse questions show little train→test skill reuse (most optimised skills are never re-triggered at test time). HLE's structured subject taxonomy enables substantial reuse: a skill refined on one Biology training question is frequently re-fired for unseen Biology test items. **Skill transfer depends on domain alignment** — when the benchmark has structured categories, skill libraries transfer; when it is a grab-bag, they do not.

**Convergence.** Diminishing returns across rounds match the Memento~2 asymptotic value-gap bound: as the library grows, the memory coverage radius $r_\mathcal{M}$ shrinks, which reduces both $\varepsilon_{\mathrm{LLM}}(r_\mathcal{M})$ and the retrieval error $\delta_\mathcal{M}$.

## Limitations

- **Single-agent, discrete-episode.** The Write pipeline triggers on episode boundaries. Long-horizon / trajectory-level continual learning over near-infinite contexts is left as future work.
- **No multi-agent coordination.** "Memento Teams" with shared/partitioned skill memory and distributed credit assignment is sketched but not built; concurrent reflective writes risk race conditions on `SKILL.md`.
- **Skill transfer is benchmark-shape-dependent.** GAIA shows weak transfer because questions are too heterogeneous — there is no guarantee the curated skill library generalises off-benchmark.
- **One LLM, one embedding backbone.** All experiments use Gemini-3.1-Flash + Qwen3-Embedding-0.6B; generalisation across model backbones is not measured.
- **Skill catalogue curation costs.** The 8k → 3k synthetic-query pipeline requires LLM judging and is the main offline preprocessing cost.
- **Sandbox safety not in scope.** The judge measures task correctness, not whether execution destroyed environment state — production safety remains future work.

## Open questions

- **Convergence rate.** Does the empirical $O(n^{-1/d})$ memory-coverage decay generalise, and is the asymptotic value gap tight under the SRDP bound?
- **Scaling the skill library.** Does Parzen-kernel-style retrieval hold at 1M+ skills, or does another retrieval primitive become necessary?
- **Cross-benchmark, cross-LLM transfer.** When the underlying LLM is swapped (Gemini → Claude → Qwen), does the same skill library still help, or does the behavioural signal embedded by the router collapse?
- **Memory hygiene.** When does the library need pruning, retirement, or compaction? The paper has utility-driven discovery but no formal deprecation policy.
- **Multi-agent reflective writes.** How is distributed credit assignment performed when Agent A writes a skill that Agent B later fails with?
- **Trajectory-level reflection.** Without discrete episode boundaries, when should an agent pause to write?

## My take

The strongest contribution is the *commitment* to executable skill folders as the memory unit. Both Memento~2's episodic-trace memory and GEPA/Letta's text-only skill guides hit walls — traces don't carry executable structure, text-only skills can't be unit-tested and rolled back. By packaging `SKILL.md` + scripts + prompts as the atom, Memento-Skills inherits the Reflected MDP convergence story while gaining a real unit-test gate and a write operator that materially changes future-policy execution, not just future-prompt content.

The router story is the second key bet: explicitly framing skill routing as a one-step MDP and fitting an InfoNCE objective as soft $Q$-learning is clean, and the gap between Recall@1 of 0.32 (BM25) → 0.54 (Qwen3) → 0.60 (Memento-Qwen) shows that even strong semantic embeddings under-represent behavioural similarity. The downside: the synthetic query-generation pipeline is brittle, and Recall@1 of 0.60 still leaves substantial slack — production deployments will need either a stronger router or cheaper escalation paths when the top-1 pick is wrong.

The GAIA-vs-HLE divergence is the most policy-relevant finding for downstream readers: skill libraries transfer when benchmarks are domain-structured and barely transfer when they are not. This sharpens the question of *what* to ingest as candidate train tasks if one wants the skill library to be useful at deploy time. The paper is honest about this rather than burying it.

## Related

- Memory framework: [[stateful-reflective-decision-process]]
- Central learning loop: [[read-write-reflective-learning]]
- Behaviour-similarity routing: [[behaviour-aligned-skill-router]]
- System artefact: [[memento-skills-system]]
- Router training method: [[infonce-skill-routing]]
- Authors: [[huichi-zhou]], [[siyuan-guo]], [[jun-wang]]
