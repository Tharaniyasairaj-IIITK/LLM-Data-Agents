# Semantic Contracts for LLM Data Agents

**Metric definitions are committed, and ambiguity is measured, in multiagent text-to-SQL analytics.**

Code, data generator, tests, classroom demo for blue-sky research direction on
**LLM data agents**: treating business-metric definitions as *social commitments* of a steward agent,
carried by every answer as a versioned, hashed **semantic contract**, and quantifying residual ambiguity with an
**Interpretation Divergence Index (IDI)** so that agents escalate instead of guessing.

---

## Why this matters

LLM data agents write and run SQL against warehouses and are judged by *execution accuracy*, which assumes a business
question determines the metric. It does not: "revenue" admits dozens of defensible readings (include GST? subtract promos?
how are refunds treated?). A wrong reading produces a **valid, plausible, wrong** answer that execution checks,
self-consistency and confidence estimates do not expose.

This repo makes that failure reproducible and measurable, and demonstrates the proposed remedy.

## Core ideas

| Idea | One line | Where |
|---|---|---|
| **Semantic contract** | An answer carries `(concept, version, hash)` of the definition it used; stale answers are detectable. | `sca/semantic_layer.py` |
| **Interpretation Divergence Index** | `IDI = (max - min) / |median|` of a question's value over all admissible interpretations. | `sca/divergence.py` |
| **Steward + escalation** | Uncovered concepts with high IDI are escalated to a steward; clarified commitments are memoised and shared. | `sca/agents.py` |
| **One-at-a-time (OAT) estimator** | A cheaper lower bound on the interpretation range (6 vs 24 evaluations for revenue). | `sca/divergence.py` |

## Headline results (seed 7, synthetic data; see [`results/summary.md`](results/summary.md))

| Experiment | Result |
|---|---|
| **E0** naive default on Mumbai September revenue | Rs 24,98,756 vs committed Rs 22,99,961 (+8.6%); 24 readings span Rs 22.99-29.85 lakh (IDI 0.262) |
| **E1** 48 benchmark questions | default agent silently wrong on **75%**; IDI-based escalation with a steward leaves **0 silent errors** after **4** steward calls |
| **E2** partial semantic-layer coverage (0 to 4 of 4 concepts) | context-only silent-error rate falls 0.75 to 0; divergence-aware agent stays at 0 |
| **E3** definition drift (revenue v1 to v2) | contracts flag **6/6** stale answers; plain numbers flag 0/6 |
| **E4** OAT estimator | 91% of the full range on average; no false ambiguity alarms, some missed flags where axes interact |

## Quickstart (about 2 minutes, no API key needed)

```bash
git clone https://github.com/Tharaniyasairaj-IIITK/LLM-Data-Agents.git
cd LLM-Data-Agents
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python scripts/build_warehouse.py        # seeded synthetic DuckDB warehouse (~170k orders)
python scripts/run_demo.py --pause       # the 6-act classroom demo
python scripts/run_experiments.py        # reproduces every number in the paper
python scripts/make_figures.py           # regenerates paper/figures/*.pdf
pytest -q tests                          # 9 tests
```

Requires Python 3.10+. `anthropic` is only needed for the optional live-LLM script.

## Classroom demo (about 12 minutes)

`python scripts/run_demo.py --pause` walks through six acts:

1. A SQL agent can explain *why* Mumbai orders dropped (riders unavailable).
2. The same agent on four business metrics: three of four are **silently wrong**.
3. 24 defensible readings of "revenue" and the **IDI**.
4. A **semantic contract**: committed, versioned, hashed; the context agent is exact.
5. No semantic layer: the divergence-aware agent **escalates to a steward** and remembers the commitment.
6. **Definition drift**: contracts detect stale answers; bare numbers cannot.

Discussion prompts: Is the interpretation set ever complete? Who should the steward be? What should an agent do when two
departments disagree?

## Repository layout

```
sca/                    core library
  warehouse.py            synthetic food-delivery warehouse (6 tables, DuckDB)
  concepts.py             concepts, interpretation axes, SQL builders, evaluator
  divergence.py           IDI and the OAT estimator
  semantic_layer.py       definitions, contract hashes, validation, Steward
  agents.py               naive / context / divergence-aware policies
  benchmark.py            48-question benchmark and scoring
semantic/metrics.yaml   committed definitions (plays the role of dbt / MetricFlow)
scripts/                build_warehouse, run_demo, run_experiments, make_figures, llm_agent
tests/test_core.py      invariants, hand-computed revenue check, numerical checks of Props. 2-4
results/                results.json, summary.md (generated)
paper/                  main.tex, references.bib, AAMAS 2027 template files, figures, PDF, Word version
```

## Mapping the paper to the code

| Paper | Code |
|---|---|
| Eq. (1)-(2) interpretation set and value | `sca/concepts.py` (`Concept.interpretations`, `Evaluator`) |
| Eq. (5) IDI | `sca/divergence.py::interpretation_spread` |
| Eq. (6) contract hash; Prop. 4.6 | `sca/semantic_layer.py` (`Definition.hash`, `SemanticLayer.validate`) |
| Eq. (8) OAT neighbourhood; Prop. 4.3 | `sca/divergence.py::oat_interpretations` |
| Algorithm 1 `Answer` | `sca/agents.py::DivergenceAwareAgent` |
| Algorithm 2 `ValidateAndRefresh` | `SemanticLayer.validate` and `scripts/run_experiments.py` (E3) |
| Algorithm 3 `DiscoverAxes` | not implemented (proposed future work) |
| Props. 4.2-4.4 | numerical checks in `tests/test_core.py` |
| Experiments E0-E4 | `scripts/run_experiments.py` |

## Optional: live LLM agent

`scripts/llm_agent.py` runs the same policies with Claude in the loop (tool use over the warehouse):

```bash
export ANTHROPIC_API_KEY=...            # optional: ANTHROPIC_MODEL (default: claude-sonnet-5-5)
python scripts/llm_agent.py --mode chatbot    "Why did our orders drop in Mumbai last week?"
python scripts/llm_agent.py --mode agent      "What was our revenue in Mumbai last month?"
python scripts/llm_agent.py --mode context    "What was our revenue in Mumbai last month?"
python scripts/llm_agent.py --mode divergence "What was our revenue in Mumbai last month?"
```

Modes: `chatbot` (no tools), `agent` (list/describe/run SQL), `context` (+ metric definitions),
`divergence` (+ `interpretation_spread`, instructed to report ranges when IDI is high).
This script has **not** been validated against the live API and was **not** used for the paper's numbers. Running it on
the 48 questions and comparing with the simulation is a natural first extension (roadmap milestone M1).


## Scope and limitations

- The experimental agents are **simulated policies**, not LLMs: they demonstrate the mechanism, not LLM behaviour.
- Data are **synthetic**; the committed definition lies in the interpretation set by construction (assumption A1).
- IDI is a **triage signal**: it says a question is ambiguous, not that an agent's default is wrong.

## Roadmap

| Milestone | Goal | Status |
|---|---|---|
| M0 | Framework, prototype, tests, experiments E0-E4 | done |
| M1 | Live-LLM study (H1, H2) on realistic warehouses | harness drafted |
| M2 | Automatic discovery of interpretation axes (H3) | designed |
| M3 | Stakes-aware escalation with learned priors (H4) | theory done |
| M4 | Multi-steward negotiation and role-scoped contracts | open |
| M5 | Public benchmark scoring definitional correctness and abstention | open |

## License

MIT, see [LICENSE](LICENSE).
