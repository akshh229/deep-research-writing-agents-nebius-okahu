# Project Report — Deep Research & Writing Agents

**Submitted by:** Akshat Raj
**Institution:** Chandigarh University
**Course:** B.E. Computer Science & Engineering (CSE)

> This report explains the AI system in my own words: what it does, how it is
> built, and the engineering ideas behind it. The project is built on and
> adapted from prior open-source work, credited in the [References](#references)
> section and in the repository README.

---

## 1. Abstract

This project is a hybrid AI system made of two cooperating agents, each exposed
as a **Model Context Protocol (MCP)** server:

1. A **Deep Research Agent** that gathers real-time, cited information on a topic.
2. A **LinkedIn Writing Workflow** that turns that research into a polished,
   publish-ready social post plus a matching AI-generated image.

Both servers plug into an AI harness (Claude Code or Cursor), which acts as the
orchestrator. The system demonstrates several core patterns of modern AI
engineering — tool-use agents, an evaluator–optimizer refinement loop, grounded
search with citations, structured model output, and automated
LLM-as-judge evaluation with tracing.

---

## 2. Problem Statement

Large Language Models (LLMs) on their own have two well-known weaknesses:

- **They hallucinate** — they can state confident but false facts because they
  answer from memory instead of current, verifiable sources.
- **They produce uneven first drafts** — a single-shot generation is rarely as
  tight or on-brand as writing that has been reviewed and revised.

The goal of this project is to wrap an LLM in an **engineered system** (a
"harness") that fixes both problems: ground every claim in real search results,
and improve the writing through repeated automated review — then measure the
final quality objectively instead of by gut feeling.

---

## 3. System Architecture

The system follows a two-server design. The harness calls tools on each server
in sequence.

```
user topic
   │
   ▼
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   Deep Research Agent        │        │   LinkedIn Writing Workflow  │
│   (MCP server)               │        │   (MCP server)               │
│                              │        │                              │
│  deep_research  ──┐          │        │  generate_post ──┐           │
│  analyze_youtube  │          │  research.md              │           │
│  compile_research ┘──────────┼──────► │  edit_post  ◄────┘ (loop)    │
│                              │        │  generate_image              │
└─────────────────────────────┘        └─────────────────────────────┘
                                                     │
                                                     ▼
                                          post.md + post_image.png
```

### 3.1 Why MCP?

Instead of hard-wiring the two agents into one monolithic script, each is a
standalone **MCP server** that publishes its capabilities as **tools**,
**resources**, and **prompts**. Any MCP-compatible harness can then discover and
call them. This keeps the two concerns (researching vs. writing) decoupled and
independently testable, and it is the same interface pattern used by production
AI tools. The server bootstrap is small and uniform — `create_mcp_server()`
builds a `FastMCP` instance and registers the tools, resources, and prompts
(`src/research/server.py`, `src/writing/server.py`).

---

## 4. Component 1 — Deep Research Agent

This is a **tool-use agent**: the LLM decides which tools to call and how many
times, based on the topic.

| Tool | Job |
|------|-----|
| `deep_research` | Runs a real-time web search for one query and returns a cited answer. |
| `analyze_youtube_video` | Pulls a YouTube transcript and summarizes it (when the seed includes video links). |
| `compile_research` | Merges all findings into a single structured `research.md` with sources. |

### 4.1 Grounded search

The core of grounding is in `run_grounded_search()`
(`src/research/app/research_handler.py`). Each raw query is wrapped in a research
prompt template, sent to the **Exa** search API, and the response — an answer
plus its source citations — is parsed into typed `ResearchResult` and
`ResearchSource` objects. Because every answer carries its source URLs, the
research is verifiable rather than hallucinated.

### 4.2 Structured output

Search results are not left as loose text. They are validated into **Pydantic**
schemas, so the rest of the pipeline receives type-safe, predictable data. This
is a key reliability technique: the model's output is forced into a known shape.

---

## 5. Component 2 — LinkedIn Writing Workflow

This server converts `research.md` + a short guideline into a finished post. Its
most important idea is the **evaluator–optimizer loop**.

```
research.md + guideline
        │
        ▼
   generate_post   ──►  draft v0
        │
        ▼
 ┌──────────────────────────┐
 │  review  ──►  edit        │   × N cycles
 └──────────────────────────┘
        │
        ▼
   final post ──► generate_image
```

### 5.1 Generate → Review → Edit

- **Generate** — the writer model produces a first draft from the research and
  the guideline.
- **Review** — a separate reviewer step (`review_post()` in
  `src/writing/app/post_reviewer_handler.py`) grades the draft against three
  shipped **profiles** — *structure*, *terminology*, and *character* — plus the
  guideline, and returns a structured list of concrete critiques. Optional
  **human feedback** is given the highest priority.
- **Edit** — the critiques are fed back to rewrite the draft.

Repeating this cycle a few times tightens the hook, removes redundancy, and pulls
the post on-brand — the same "draft, get feedback, revise" loop a human editor
would run, but automated. The README shows a real before/after: a verbose v0
becoming a punchy v3 after three cycles.

### 5.2 Image generation

Once the text is final, `generate_image` calls **Gemini's image model**
(`gemini-2.5-flash-image`) to produce a matching visual, giving a complete,
ready-to-publish post.

---

## 6. Evaluation — LLM-as-Judge

A system that generates content also needs to **measure** content quality
without a human grading every output. This project uses an **LLM-as-judge**
pipeline (`src/writing/evals/`):

1. An LLM scores each generated post against quality criteria (structure, tone,
   accuracy).
2. Those scores are compared to expert labels, and an **F1 score** is computed on
   dev and test splits of a labeled dataset (`datasets/`).

This turns "does this post look good?" into a repeatable, numeric benchmark.

### 6.1 Observability

Both servers optionally emit traces through **Monocle**, which can export to
**Okahu Cloud**. This makes the whole run inspectable — every LLM call, tool
call, judge score, and critique — so failures can be debugged and quality tracked
over time. Tracing is wired in at server startup via `configure_okahu()` and is
fully optional (the pipeline runs without it).

---

## 7. Technology Stack

| Concern | Technology |
|---------|-----------|
| Text LLM calls | Nebius AI Studio (via LangChain) |
| MCP server framework | FastMCP |
| Real-time search | Exa |
| Image generation | Gemini `gemini-2.5-flash-image` |
| Data validation / schemas | Pydantic + Pydantic Settings |
| Observability & evals | Okahu Cloud + Monocle |
| Linting / formatting | Ruff |
| Packaging / environment | uv |

---

## 8. AI Engineering Concepts Demonstrated

- **Tool-use agents** — the model chooses which tools to call and when.
- **Evaluator–optimizer loop** — quality is built through iterative review, not a
  single generation.
- **Grounded search** — every fact is backed by a citation from live search.
- **Structured LLM output** — Pydantic schemas keep model responses type-safe.
- **MCP server design** — capabilities exposed as tools, resources, and prompts.
- **LLM-as-judge evaluation** — automated, numeric quality scoring with tracing.

---

## 9. How to Run

```bash
cp .env.example .env      # add NEBIUS_API_KEY, EXA_API_KEY, GEMINI_API_KEY
uv sync                   # install dependencies
make test-end-to-end      # run research + writing pipeline end-to-end
```

Other entry points: MCP servers (`make run-research-server`,
`make run-writing-server`), a Streamlit UI (`make run-ui`), and the evaluation
commands (`make eval-dev`, `make eval-test`). Full details are in the README.

---

## 10. Conclusion

This project shows that the value of an AI application comes less from the raw
model and more from the **system built around it** — grounding, iterative
refinement, structure, and measurement. By combining a grounded research agent
with an evaluator–optimizer writing workflow, both served over MCP and traced for
observability, the system reliably turns a bare topic into a verifiable,
polished, publishable result.

---

## References

This project is built on and adapted from prior open-source work. Full credit to
the original authors:

- **Original workshop:** *designing-real-world-ai-agents-workshop* by Paul
  Iusztin — https://github.com/iusztinpaul/designing-real-world-ai-agents-workshop
- **Original contributors:** Louis-François Bouchard, Paul Iusztin, Samridhi Vaid.
- **License / copyright:** © 2026 Paul Iusztin, Towards AI Inc — MIT License (see
  `LICENSE`).
