# Deep Research & Writing Agents
### A Hybrid AI System of Two Cooperating MCP Agents

---

**Submitted by:** Akshat Raj

**Institution:** Chandigarh University

**Course:** B.E. Computer Science & Engineering (CSE)

---

## Summary

A hybrid AI system built from two cooperating agents, each served as a **Model
Context Protocol (MCP)** server:

- **Deep Research Agent** — gathers real-time, source-cited information on a topic
  using live web search (Exa) and YouTube transcript analysis.
- **LinkedIn Writing Workflow** — turns that research into a polished, on-brand
  post through an **evaluator–optimizer loop** (generate → review → edit), then
  produces a matching AI-generated image.

Output quality is measured automatically with an **LLM-as-judge** pipeline (F1
against expert labels) and traced end-to-end via Monocle / Okahu.

## Key Concepts Demonstrated

- Tool-use agents · Evaluator–optimizer loop · Grounded search with citations
- Structured LLM output (Pydantic) · MCP server design · LLM-as-judge evaluation

## Tech Stack

Nebius AI Studio · FastMCP · Exa · Gemini · Pydantic · Okahu + Monocle · uv · Ruff

---

*Full technical writeup in `REPORT.md`. Built on and adapted from prior
open-source work — credited in the References section of `REPORT.md` and
`README.md`.*
