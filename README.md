# Blog Writing Agent

A multi-agent blog generation pipeline built with **LangGraph**, orchestrating research, planning, parallel content generation, and image creation into a single automated workflow — with a Streamlit UI for running and reviewing generations.

## Overview

Given a topic, the agent:

1. Decides whether the topic needs web research (evergreen vs. time-sensitive)
2. Researches it via Tavily if needed, with recency filtering
3. Plans a structured outline (title, audience, tone, sections)
4. Writes all sections **in parallel**, grounding claims in retrieved evidence
5. Merges the sections, decides if diagrams would help, generates them, and assembles a final Markdown blog post with images

## Architecture

```
START
  │
  ▼
Router ──(needs_research?)──► Research (Tavily) ──┐
  │                                                 │
  └──(no)────────────────────────────────────────► Orchestrator
                                                      │
                                                      ▼
                                            Fan-out (Send API)
                                                      │
                                        ┌─────────────┼─────────────┐
                                        ▼             ▼             ▼
                                     Worker 1      Worker 2      Worker N   (parallel)
                                        └─────────────┼─────────────┘
                                                      ▼
                                          Reducer (subgraph):
                                          merge_content
                                              │
                                              ▼
                                          decide_images
                                              │
                                              ▼
                                     generate_and_place_images
                                                      │
                                                      ▼
                                                     END
```

### Nodes

| Node | Responsibility |
|---|---|
| **Router** | Classifies the topic as `closed_book`, `hybrid`, or `open_book`, and sets a recency window (10 years / 45 days / 7 days respectively). Generates 3–10 search queries when research is needed. |
| **Research** | Runs Tavily searches for each query, then uses an LLM call with structured output to normalize results into typed `EvidenceItem` objects, deduplicate by URL, and filter by recency for time-sensitive topics. |
| **Orchestrator** | Produces a schema-validated `Plan` (Pydantic) — blog title, audience, tone, blog kind, and 5–9 `Task` objects, each with a goal, 3–6 bullets, target word count, and flags for research/citation/code requirements. |
| **Fan-out** | Uses LangGraph's `Send` API to dispatch one worker per task, running section generation **concurrently** rather than sequentially. |
| **Worker** | Writes a single section in Markdown, covering all assigned bullets, staying within ±15% of the target word count, and citing only the provided evidence URLs — refusing to fabricate unsupported claims in open-book mode. |
| **Reducer** *(subgraph)* | Three-step pipeline: merges all sections into one document, decides (via structured LLM output) whether up to 3 diagrams would materially help, then generates and places those images. |

### Models

- **Text generation** (routing, research synthesis, planning, section writing): `gpt-4.1-mini`
- **Image generation**: `gemini-2.5-flash-image`

## Key Design Details

- **Structured outputs everywhere** — every LLM call that produces data used downstream (routing decisions, evidence, plans, image specs) is constrained to a Pydantic schema, not free-form text, for reliable parsing.
- **Conditional research** — research is skipped entirely for evergreen topics, avoiding unnecessary API calls and latency.
- **Grounded, citation-enforced writing** — in `open_book` mode, workers must attach a source link for any specific claim or explicitly state it isn't supported by the provided evidence.
- **Graceful image-generation fallback** — if an image generation call fails, the pipeline doesn't crash; the placeholder is replaced with a visible "generation failed" note (alt text, caption, prompt, and error) so the document is still usable.
- **Nested subgraph composition** — the reducer (merge → decide images → generate images) is its own compiled `StateGraph`, plugged into the main graph as a single node.

## Frontend (Streamlit)

- Streams graph execution **node-by-node** in real time, showing live progress (mode, research queries, evidence count, task count, sections completed, images).
- Tabbed interface: **Plan**, **Evidence**, **Markdown Preview** (renders local images inline), **Images**, **Logs**.
- **Past blogs** sidebar — lists previously generated `.md` files in the working directory and reloads them into the UI.
- **Export** — download the final Markdown alone, or a zip bundle containing the Markdown plus all generated images.

## Setup

### Requirements

```bash
pip install langgraph langchain-openai langchain-community python-dotenv pydantic streamlit pandas google-genai
```

### Environment Variables

Create a `.env` file in the project root:

```
OPENAI_API_KEY=your_openai_key
TAVILY_API_KEY=your_tavily_key
GOOGLE_API_KEY=your_google_key
```

- `TAVILY_API_KEY` — optional; if absent, the research step returns no results and the agent proceeds without external evidence.
- `GOOGLE_API_KEY` — required only if image generation is used.

### Run

```bash
streamlit run bwa_frontend.py
```

Enter a topic and an "as-of" date in the sidebar, then click **Generate Blog**. Output is saved as a `.md` file (and an `images/` folder, if diagrams were generated) in the working directory.

## Project Structure

```
.
├── bwa_backend.py     # LangGraph pipeline: state, nodes, graph construction
├── bwa_frontend.py    # Streamlit UI: run pipeline, stream progress, review/export output
├── images/            # Generated diagrams (created at runtime)
└── *.md               # Generated blog posts (saved at runtime)
```

## Example Flow

1. User enters topic: *"Latest developments in vector databases"*
2. Router detects this needs recent info → mode = `hybrid`, recency = 45 days
3. Research node queries Tavily, extracts evidence with source URLs and dates
4. Orchestrator drafts a 6-section outline
5. 6 workers write their sections in parallel, citing evidence where required
6. Reducer merges sections, decides 2 diagrams would help, generates them via Gemini
7. Final Markdown + images are saved and shown in the Streamlit preview tab

## Limitations

- Image generation is capped at 3 per blog post by design (editorial constraint set in the `decide_images` prompt).
- Evidence recency filtering relies on Tavily's `published_date` field being present and parseable; sources without a reliable date are treated conservatively.
- No persistence layer beyond local `.md`/`images/` files — the "Past blogs" feature reads from the working directory, not a database.
