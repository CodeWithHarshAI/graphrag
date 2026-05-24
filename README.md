# GraphLens

A GraphRAG demo application that answers natural-language questions about movies using graph traversal instead of vector embeddings. Ask a question, watch a force-directed graph animate in real time, inspect the generated Cypher query, and read an AI-synthesized answer — all without a vector database.

---

## What is GraphRAG?

Standard RAG splits documents into chunks, embeds them, and retrieves the most similar ones by cosine distance. This works well for simple factual lookups but structurally cannot answer relational questions such as:

- *Which actors worked with both Tom Hanks and Meg Ryan?* — requires set intersection across two traversal paths
- *How is Kevin Bacon connected to Tom Hanks?* — requires shortest-path traversal across 4+ hops
- *Which actors in Nolan films never co-starred together?* — requires exclusion reasoning over a graph complement

No chunk size, no re-ranking strategy, and no retrieval depth can fix this. The operations are absent from the vector paradigm. Graph traversal is the only answer.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11+, FastAPI |
| LLM | OpenAI GPT-4o |
| Graph database | Neo4j AuraDB (free tier) |
| Frontend | Streamlit |
| Graph visualization | streamlit-agraph |
| HTTP client | httpx |

100% Python. No Node.js, no Docker, no npm. A single `pip install -r requirements.txt` installs everything.

---

## Architecture

The system has three moving parts: a Streamlit frontend, a FastAPI backend, and Neo4j AuraDB as the graph store.

```
Browser (port 8501)
    |
    |-- HTTP --> Streamlit app.py
                    |
                    |-- POST /api/query --> FastAPI
                    |-- GET  /api/health
                    |-- GET  /api/schema
                    |-- GET  /api/examples
                                |
                                |-- generate_cypher()  --> OpenAI GPT-4o
                                |-- run_cypher()       --> Neo4j AuraDB
                                |-- synthesize_answer()--> OpenAI GPT-4o
```

**Request flow for a single question:**

1. User submits a question in the Streamlit UI
2. FastAPI sends the schema and question to GPT-4o, which returns a Cypher query
3. FastAPI executes the Cypher query asynchronously against Neo4j AuraDB
4. Raw records (nodes, relationships, paths) are parsed into a graph payload
5. GPT-4o synthesizes a conversational answer from the question, the Cypher, and the raw results
6. Streamlit renders the force-directed graph, the natural-language answer, and the Cypher side by side

---

## Graph Schema

```
(Person)-[:ACTED_IN {roles}]->(Movie)
(Person)-[:DIRECTED]-------->(Movie)
(Person)-[:WROTE]---------->(Movie)
(Movie)-[:IN_GENRE]-------->(Genre)
(Movie)-[:PRODUCED_BY]----->(Studio)

Node properties
  Person : name, born
  Movie  : title, year, tagline, revenue, rating
  Genre  : name
  Studio : name
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Python 3.11+ | Check with `python3 --version` |
| Neo4j AuraDB account | Free tier at neo4j.com/cloud/aura — no local install needed |
| OpenAI API key | GPT-4o access at platform.openai.com |
| TMDB API key | Free key at themoviedb.org/settings/api |

---

## Getting Started

**Step 1 — Clone and install**

```bash
git clone https://github.com/techwithprateek/graphrag
cd graphrag
pip install -r requirements.txt
```

**Step 2 — Configure credentials**

```bash
cp .env.example .env
```

Open `.env` and fill in your keys:

```
OPENAI_API_KEY=sk-...
NEO4J_URI=neo4j+s://xxxx.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-password
TMDB_API_KEY=your-tmdb-key
```

**Step 3 — Seed the database**

```bash
python3 seed/load_tmdb.py
```

Downloads approximately 200 popular movies from TMDB — cast, crew, genres, and production studios — and loads them into Neo4j as a property graph. Takes 2–3 minutes.

**Step 4 — Start the backend**

```bash
# Terminal 1
uvicorn backend.main:app --reload
```

Backend runs at http://localhost:8000

**Step 5 — Start the frontend**

```bash
# Terminal 2
streamlit run frontend/app.py
```

Open http://localhost:8501 in your browser.

---

## Project Structure

```
graphrag/
|
|-- backend/
|   |-- main.py                  # FastAPI app, CORS, route registration
|   |-- config.py                # Loads credentials from .env
|   |-- models.py                # Pydantic request and response models
|   |
|   |-- routes/
|   |   |-- query.py             # POST /api/query — orchestrates LLM + Neo4j
|   |   `-- schema.py            # GET /api/schema, /api/examples, /health
|   |
|   |-- services/
|   |   |-- neo4j_service.py     # Async Neo4j driver, Cypher execution, graph builder
|   |   `-- llm_service.py       # GPT-4o: generate_cypher() and synthesize_answer()
|   |
|   `-- prompts/
|       `-- cypher_prompt.py     # System prompts and few-shot Cypher examples
|
|-- frontend/
|   `-- app.py                   # Entire Streamlit UI in one file
|
|-- seed/
|   `-- load_tmdb.py             # Downloads TMDB data and loads into Neo4j
|
|-- requirements.txt
|-- .env.example
`-- README.md
```

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Liveness check |
| GET | `/api/schema` | Node labels, relationship types, property keys |
| GET | `/api/examples` | The 7 built-in demo questions |
| POST | `/api/query` | Main endpoint: question → Cypher → answer + graph |

---

## Demo Questions

The UI sidebar includes seven example questions that cover progressively harder graph operations:

| # | Question | Graph operation |
|---|---|---|
| 1 | Which movies did Tom Hanks act in? | 1-hop, single edge type |
| 2 | Which actors worked with both Tom Hanks and Meg Ryan? | 2-hop set intersection |
| 3 | Which directors have worked with Keanu Reeves more than once? | 2-hop edge-count aggregation |
| 4 | Which actor has appeared in the most genres? | 3-hop cross-entity aggregation |
| 5 | How is Kevin Bacon connected to Tom Hanks? | 4+ hop `shortestPath()` |
| 6 | Which movies share the director and genre of The Matrix? | 3-hop multi-condition join |
| 7 | Actors in Nolan films who never co-starred together? | 3-hop exclusion reasoning |

Questions 5 and 7 are the clearest demonstration of what graph traversal can do that vector RAG structurally cannot.

---

## Why Not Vector RAG?

**Shortest path (Q5):** The connection between Kevin Bacon and Tom Hanks is a path across 4+ relationship hops. No text chunk contains this path. Retrieving every chunk that mentions both actors gives a bag of facts, not a traversal route. Only `shortestPath()` in Cypher can answer this correctly.

**Exclusion reasoning (Q7):** Finding actors who *never* co-starred requires computing the complement of a relationship set — a structural graph operation with no vector equivalent. It cannot be approximated by any combination of embedding retrieval, re-ranking, or prompt engineering.

These are not edge cases. They represent the majority of interesting questions about connected data.

---

## Node Colors

| Color | Node type |
|---|---|
| Coral `#E8593C` | Movie |
| Purple `#7F77DD` | Person (actor, director, writer) |
| Teal `#1D9E75` | Genre |
| Amber `#BA7517` | Studio |
