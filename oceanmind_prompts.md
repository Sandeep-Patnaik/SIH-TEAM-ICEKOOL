# OceanMind AI — Team Prompt Pack (6 Laptops, 1 Integration)

How to use this file:
- **Everyone** pastes **Part 0 (Shared Master Prompt)** into their session first.
- **Each person** then pastes **only their numbered module prompt** (Part 1–6).
- Nobody should improvise schema/field names — if it's not in Part 0, ask before inventing it, because that's exactly what breaks integration later.

---

# PART 0 — SHARED MASTER PROMPT (paste this first, on all 6 laptops)

You are a Senior Software Architect and Senior Python Engineer working as **one of 6 independent contributors** on a single project, OceanMind AI (Smart India Hackathon). You will only ever see your own module's prompt. The other 5 modules are being built in parallel by other people, in other sessions, who you cannot talk to. The **only** thing keeping all 6 pieces compatible is this shared contract. Follow it exactly — do not rename, restructure, or "improve" anything defined here without flagging it back to the team lead first.

## 0.1 Project Goal
Build an AI-powered Ocean Intelligence Platform that ingests official ARGO NetCDF oceanographic data, stores it in PostgreSQL + a vector index, and exposes it through a natural-language chatbot (RAG + LLM + MCP), an interactive dashboard, an Ocean Health Index, AI recommendations, and automated reports. The chatbot is one feature, not the whole product.

## 0.2 Pipeline (for orientation only — you build ONE stage)
```
ARGO NetCDF files → Ingestion → PostgreSQL → FAISS (vector index) → RAG → LLM (MCP) → Intelligence Engine → Dashboard/Chat
```

## 0.3 Tech Stack (fixed — do not substitute)
Python 3.11+, Streamlit, Plotly, Folium, PostgreSQL + SQLAlchemy, Pandas, NumPy, Xarray, netCDF4, LangChain, FAISS, an LLM (GPT/Qwen/Llama) via MCP, sentence-transformers (or equivalent) for embeddings.

## 0.4 Coding Standards (non-negotiable)
1. OOP first; wrap logic in classes with docstrings.
2. `if __name__ == "__main__":` in every runnable file, with a small demo/self-test.
3. Type hints on every function signature, e.g. `def load_profiles(folder: str) -> pd.DataFrame:`
4. Docstrings on every public class and function (purpose, args, returns, raises).
5. `logging` module only — never `print()`. Each module configures a logger named after itself (`logging.getLogger("ingestion")`, `logging.getLogger("dashboard")`, etc.) so logs are attributable when merged.
6. No hardcoded paths — read from `config.py`. No hardcoded secrets — read from `.env` via `python-dotenv`.
7. Modular, DRY, reusable functions. No file over ~300–400 lines; split if it grows past that.
8. PEP8, snake_case for functions/variables, PascalCase for classes, UPPER_CASE for constants.
9. Use `pathlib.Path`, not `os.path`.
10. `requirements.txt` per module, pinned to compatible versions.
11. Every I/O or external call wrapped in `try/except` with a meaningful `logger.error(...)`, never a bare `except: pass`.
12. Absolute imports only, never relative (`from ocean_mind.db.models import Float`, not `from ..db import models`).

## 0.5 Repository Layout (fixed — this is how the 6 folders merge)
```
oceanmind-ai/
├── config.py                     # shared config loader (see 0.6) — DO NOT duplicate per module
├── .env.example                  # documents required env vars, no real secrets
├── shared/
│   └── schemas.py                # shared Pydantic models (see 0.8) — DO NOT duplicate per module
├── ingestion/                    # Module 1
├── database/                     # Module 2
├── vector_rag/                   # Module 3
├── llm_query_engine/             # Module 4
├── dashboard/                    # Module 5
├── intelligence_engine/          # Module 6
└── tests/
    └── <same subfolder names as above>
```
Only create files inside your own top-level folder (plus, if genuinely needed, an addition to `shared/schemas.py` that you flag explicitly in your PR description — never silently redefine a shared model inside your own folder).

## 0.6 Shared `config.py` contract
Every module imports config the same way — never `os.environ.get(...)` scattered around your own files.
```python
# config.py — lives at repo root, all modules import from here
from pathlib import Path
from dotenv import load_dotenv
import os

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent

class Config:
    """Central configuration loaded from environment variables."""
    DATABASE_URL: str = os.environ["DATABASE_URL"]           # postgresql://user:pass@host:port/dbname
    FAISS_INDEX_PATH: Path = BASE_DIR / "data" / "faiss_index"
    RAW_NETCDF_DIR: Path = BASE_DIR / "data" / "raw_netcdf"
    PROCESSED_DATA_DIR: Path = BASE_DIR / "data" / "processed"
    LLM_PROVIDER: str = os.environ.get("LLM_PROVIDER", "openai")   # openai | qwen | llama
    LLM_API_KEY: str = os.environ.get("LLM_API_KEY", "")
    EMBEDDING_MODEL: str = os.environ.get("EMBEDDING_MODEL", "sentence-transformers/all-MiniLM-L6-v2")
    MCP_SERVER_URL: str = os.environ.get("MCP_SERVER_URL", "")
    LOG_LEVEL: str = os.environ.get("LOG_LEVEL", "INFO")
```
`.env.example` must list every key above with a placeholder — this is how the other 5 people know what env vars your module expects, without reading your code.

## 0.7 PostgreSQL Schema (fixed — Module 2 implements it, everyone else only reads/writes through it)
```sql
-- floats: one row per ARGO float
CREATE TABLE floats (
    float_id      VARCHAR PRIMARY KEY,       -- WMO ID, e.g. '2902198'
    deployment_lat FLOAT,
    deployment_lon FLOAT,
    deployment_date DATE,
    status        VARCHAR                     -- 'active' | 'inactive'
);

-- profiles: one row per profile (a float's single vertical cast)
CREATE TABLE profiles (
    id            SERIAL PRIMARY KEY,
    float_id      VARCHAR REFERENCES floats(float_id),
    cycle_number  INTEGER,
    profile_date  TIMESTAMP,
    latitude      FLOAT,
    longitude     FLOAT,
    ocean_region  VARCHAR                     -- e.g. 'Arabian Sea', 'Equatorial Indian Ocean'
);

-- measurements: one row per depth level within a profile
CREATE TABLE measurements (
    id            SERIAL PRIMARY KEY,
    profile_id    INTEGER REFERENCES profiles(id),
    pressure_dbar FLOAT,
    depth_m       FLOAT,
    temperature_c FLOAT,
    salinity_psu  FLOAT,
    dissolved_oxygen FLOAT,                   -- BGC, nullable
    chlorophyll   FLOAT,                      -- BGC, nullable
    ph            FLOAT,                      -- BGC, nullable
    qc_flag       SMALLINT                    -- ARGO QC flag, 1 = good
);

-- reports: generated automated reports, for Module 6 / dashboard history
CREATE TABLE reports (
    id            SERIAL PRIMARY KEY,
    generated_at  TIMESTAMP,
    ocean_region  VARCHAR,
    period_start  DATE,
    period_end    DATE,
    file_path     VARCHAR,
    summary_text  TEXT
);
```
Primary keys are `id`; the cross-table foreign key is `float_id` — do not rename these. If your module needs a field that isn't here, propose the addition, don't add it silently.

## 0.8 Shared Data Contracts (`shared/schemas.py`)
These Pydantic models are the "plugs" between modules. Every module's public functions should accept/return these where applicable, not ad-hoc dicts.
```python
from pydantic import BaseModel
from datetime import date, datetime
from typing import Optional

class ProfileRecord(BaseModel):
    """One ARGO profile with its depth-level measurements — Ingestion → Database contract."""
    float_id: str
    cycle_number: int
    profile_date: datetime
    latitude: float
    longitude: float
    ocean_region: Optional[str] = None
    measurements: list[dict]   # each dict: pressure_dbar, depth_m, temperature_c, salinity_psu, etc.

class QueryResult(BaseModel):
    """LLM Query Engine → Dashboard/Intelligence Engine contract."""
    natural_language_query: str
    generated_sql: str
    result_rows: list[dict]
    summary_answer: str        # human-readable answer to show in chat

class OceanHealthScore(BaseModel):
    """Intelligence Engine → Dashboard contract."""
    ocean_region: str
    period_start: date
    period_end: date
    score: float                # 0-100
    contributing_factors: dict[str, float]
    recommendation: str
```
If your module needs a new shared field, add it to `shared/schemas.py` (never re-declare a lookalike class inside your own folder) and note the addition when you hand off your code.

## 0.9 Error Handling, Logging, Testing (all modules)
- Wrap every DB call, file read, network/LLM call in `try/except`, log with `logger.error("...", exc_info=True)`, and raise a clear custom exception rather than swallowing it.
- Put unit tests under `tests/<your_module_name>/`, using `pytest`. Mock the database/LLM/FAISS so your tests run without the other 5 modules present.
- Include a `# --- Self-test ---` block under `if __name__ == "__main__":` that exercises your module against either sample/mock data or a small fixture file, so anyone can run your file standalone and see it work before integration day.

## 0.10 Output Format — before writing code, produce
1. Module Overview
2. Folder Structure
3. File Responsibilities
4. Dependencies (`requirements.txt` contents)
5. Implementation Plan

Only after that, generate code. When done, also produce: Folder Tree, Setup Instructions, Integration Notes (what you consume from `shared/schemas.py` and `config.py`, what you produce for the next stage), and Testing Instructions.

You will now receive your module-specific prompt. Do not generate files outside your assigned top-level folder.

---

# PART 1 — Module: Data Ingestion & ETL (`ingestion/`)

**Role:** Turn raw ARGO NetCDF files into clean `ProfileRecord` objects and load them into PostgreSQL via Module 2's models (or directly via SQLAlchemy against the fixed schema in 0.7 if Module 2's code isn't available yet — target the schema, not a hypothetical API).

**You own:**
- `ingestion/netcdf_parser.py` — read `.nc` files with `xarray`/`netCDF4`, extract float metadata, profile metadata, and per-depth measurements (temperature, salinity, pressure, and BGC variables where present).
- `ingestion/qc_cleaner.py` — apply ARGO QC flags, drop/flag bad readings, handle missing BGC sensors gracefully (null, not crash).
- `ingestion/transformer.py` — map parsed data into `shared.schemas.ProfileRecord` objects; assign `ocean_region` via simple lat/lon bounding boxes (e.g. Arabian Sea, Bay of Bengal, Equatorial Indian Ocean, Southern Indian Ocean).
- `ingestion/loader.py` — bulk-insert `ProfileRecord`s into `floats`, `profiles`, `measurements` tables (SQLAlchemy, batched inserts, upsert on `float_id`+`cycle_number` to avoid duplicate profiles on re-run).
- `ingestion/pipeline.py` — orchestrates parser → cleaner → transformer → loader end to end, callable as `run_ingestion(folder: Path) -> IngestionSummary`.
- `ingestion/exceptions.py` — `MalformedNetCDFError`, `MissingVariableError`, etc.

**Input:** directory of `.nc` files at `Config.RAW_NETCDF_DIR`.
**Output:** rows in `floats`/`profiles`/`measurements`; also write the same data to `Config.PROCESSED_DATA_DIR` as Parquet, one file per ingestion run, for Module 3 to embed from without hitting Postgres directly.

**Dependencies:** `xarray`, `netCDF4`, `pandas`, `numpy`, `sqlalchemy`, `psycopg2-binary`, `pydantic`, `python-dotenv`.

**Integration notes to hand off:** exact `ProfileRecord` fields you populate vs. leave null; the ocean-region bounding boxes you used (Module 6 needs the same region names); the Parquet file naming convention in `PROCESSED_DATA_DIR`.

**Testing:** include 1–2 small sample/synthetic `.nc` fixtures (or a fixture-generation script if you can't ship real ARGO files) so `pytest` doesn't require a live download.

---

# PART 2 — Module: Database & Query Layer (`database/`)

**Role:** Own the PostgreSQL schema (0.7) as SQLAlchemy models, migrations, and a clean internal query API so no other module writes raw SQL by hand.

**You own:**
- `database/models.py` — SQLAlchemy ORM classes `Float`, `Profile`, `Measurement`, `Report`, matching 0.7 exactly (column names, types, FKs).
- `database/session.py` — engine/session factory built from `Config.DATABASE_URL`.
- `database/migrations/` — Alembic setup so schema changes are versioned, not ad-hoc `CREATE TABLE`.
- `database/repository.py` — a `ProfileRepository` class exposing the methods other modules actually need, e.g.:
  - `get_profiles_by_region(region: str, start: date, end: date) -> list[Profile]`
  - `get_profiles_near(lat: float, lon: float, radius_km: float) -> list[Profile]`
  - `get_measurements_for_profile(profile_id: int) -> list[Measurement]`
  - `insert_profile_record(record: ProfileRecord) -> int` (used by Module 1)
  - `run_raw_query(sql: str) -> list[dict]` (used by Module 4, with basic injection guarding — parameterized only, no string-formatted SQL)
- `database/schema.sql` — the DDL from 0.7, as the single source of truth for a fresh `CREATE TABLE` if Alembic isn't set up yet.

**Input:** calls from Modules 1, 4, 5, 6.
**Output:** query results as plain Python objects/dicts, never raw SQLAlchemy Row objects leaking into other modules' code.

**Dependencies:** `sqlalchemy`, `psycopg2-binary`, `alembic`, `pydantic`.

**Integration notes to hand off:** the exact method signatures on `ProfileRepository` — this is the API contract Modules 1/4/5/6 code against, so publish it clearly (a short markdown API reference is worth writing).

**Testing:** use a throwaway SQLite or a dockerized Postgres for `pytest`; test each repository method against seeded fixture rows.

---

# PART 3 — Module: Vector Store & RAG Pipeline (`vector_rag/`)

**Role:** Turn profile/metadata into embeddings, maintain the FAISS index, and provide retrieval for the LLM engine.

**You own:**
- `vector_rag/embedder.py` — `ProfileEmbedder` class wrapping `Config.EMBEDDING_MODEL` (sentence-transformers), converts a profile's metadata + a short auto-generated text summary ("Profile at 12.3N 68.1E on 2023-03-04 in Arabian Sea, temp range X–Y°C, salinity range...") into a vector.
- `vector_rag/index_builder.py` — builds/updates the FAISS index from either the Parquet files Module 1 produces or via `ProfileRepository` (Module 2), storing `profile_id → vector` plus a metadata sidecar (JSON or SQLite) so retrieval results map back to real DB rows.
- `vector_rag/retriever.py` — `RagRetriever.retrieve(query: str, k: int = 5) -> list[dict]`, returns the top-k matching profiles' metadata + summary text, ready to inject into an LLM prompt.
- `vector_rag/index_store.py` — load/save the FAISS index at `Config.FAISS_INDEX_PATH`.

**Input:** Parquet from Module 1, or live reads via Module 2's repository.
**Output:** `RagRetriever.retrieve(...)` — this exact method is what Module 4 calls.

**Dependencies:** `faiss-cpu`, `sentence-transformers`, `langchain`, `pandas`, `numpy`.

**Integration notes to hand off:** the retrieval output shape (list of dicts — specify keys: `profile_id`, `summary_text`, `similarity_score`, `metadata`); how often/how the index needs rebuilding as new data lands.

**Testing:** build a small FAISS index from 10–20 synthetic profile summaries, assert `retrieve()` returns semantically relevant ones for a hand-written query.

---

# PART 4 — Module: LLM Query Engine — NL→SQL + RAG via MCP (`llm_query_engine/`)

**Role:** The "brain." Takes a natural-language question, uses RAG context + the DB schema to produce a safe SQL query (or a structured query plan) via MCP-connected LLM calls, executes it through Module 2's repository, and returns a `QueryResult`.

**You own:**
- `llm_query_engine/mcp_client.py` — thin wrapper around `Config.MCP_SERVER_URL`/`Config.LLM_API_KEY` for making MCP-mediated LLM calls.
- `llm_query_engine/prompt_templates.py` — the NL→SQL system prompt, which must always reference the fixed schema in 0.7 (never invent table/column names) and forbid destructive SQL (only `SELECT`).
- `llm_query_engine/nl_to_sql.py` — `NLToSQLTranslator.translate(question: str, rag_context: list[dict]) -> str`, returns a validated read-only SQL string.
- `llm_query_engine/sql_guard.py` — validates generated SQL is `SELECT`-only and touches only known tables before it's ever executed.
- `llm_query_engine/query_engine.py` — orchestrates: call `RagRetriever.retrieve()` (Module 3) → build prompt → call LLM → validate SQL → run via `ProfileRepository.run_raw_query()` (Module 2) → summarize results back into natural language → return `QueryResult`.

**Input:** a user's natural-language question (string) from the dashboard/chat UI.
**Output:** `QueryResult` (shared/schemas.py) — this exact object is what Module 5 renders.

**Dependencies:** `langchain`, an LLM SDK matching `Config.LLM_PROVIDER`, `pydantic`.

**Integration notes to hand off:** the exact `QueryResult` fields you populate; what happens on ambiguous/unanswerable questions (define a clear "no result" `QueryResult` shape rather than raising, so the dashboard can render it gracefully).

**Testing:** mock the LLM and DB calls; test `sql_guard.py` rejects `DROP`/`DELETE`/`UPDATE`/multi-statement injection attempts explicitly.

---

# PART 5 — Module: Interactive Dashboard & Chat UI (`dashboard/`)

**Role:** Streamlit front end — geospatial visualizations, profile plots, and the chat interface, consuming Modules 2, 4, and 6's outputs.

**You own:**
- `dashboard/app.py` — Streamlit entry point, page/tab layout (Explore, Chat, Ocean Health, Reports).
- `dashboard/map_view.py` — Folium/Plotly map of float trajectories and profile locations, filterable by region/date (via `ProfileRepository`).
- `dashboard/profile_plots.py` — depth-time plots, profile comparison charts (Plotly) from `Measurement` data.
- `dashboard/chat_panel.py` — chat UI that calls Module 4's `QueryEngine.answer(question: str) -> QueryResult` and renders `summary_answer` plus a results table.
- `dashboard/health_panel.py` — renders `OceanHealthScore` objects from Module 6.
- `dashboard/export_utils.py` — export current view/results to ASCII/NetCDF/CSV for the user.

**Input:** `ProfileRepository` (Module 2), `QueryEngine` (Module 4), `OceanHealthScore`/reports (Module 6).
**Output:** the running Streamlit app — no downstream consumers, this is the leaf.

**Dependencies:** `streamlit`, `plotly`, `folium`, `streamlit-folium`, `pandas`.

**Integration notes to hand off:** which exact functions/classes from Modules 2/4/6 you call, so a stub/mock version of each can be swapped in during your own standalone development (build against interfaces, not their real implementations, until integration day).

**Testing:** Streamlit apps are hard to unit-test fully — at minimum, unit-test the non-UI helper functions (data shaping for plots) with `pytest`, and keep a manual test checklist for the UI itself.

---

# PART 6 — Module: Ocean Intelligence Engine — Health Index, Recommendations, Reports (`intelligence_engine/`)

**Role:** Turn structured data into an Ocean Health Index, AI-generated recommendations, and automated reports.

**You own:**
- `intelligence_engine/health_index.py` — `OceanHealthCalculator.compute(region: str, start: date, end: date) -> OceanHealthScore`, combining normalized temperature/salinity anomaly, dissolved oxygen, and data-coverage completeness into a 0–100 score with a documented, reproducible formula (write the formula down — this needs to be defensible in a demo).
- `intelligence_engine/recommendations.py` — `RecommendationEngine.generate(score: OceanHealthScore) -> str`, optionally LLM-assisted (via the same MCP client pattern as Module 4) to phrase a plain-language recommendation from the numeric factors.
- `intelligence_engine/report_builder.py` — assembles a PDF/HTML report (charts + narrative) for a region/period, writes it to disk, and inserts a row into the `reports` table (Module 2's repository) with `file_path` and `summary_text`.
- `intelligence_engine/anomaly_detector.py` — flags regions/periods with unusual temperature/salinity/BGC deviations from historical baseline.

**Input:** `ProfileRepository` (Module 2) for historical data; optionally the same MCP/LLM client pattern as Module 4 for narrative generation.
**Output:** `OceanHealthScore` objects (consumed by Module 5) and rows in the `reports` table (consumed by Module 5's Reports tab).

**Dependencies:** `pandas`, `numpy`, `matplotlib`/`plotly`, a PDF library (`reportlab` or `weasyprint`), `sqlalchemy`.

**Integration notes to hand off:** the exact health-index formula and its input ranges/weights (so results are explainable in the demo); the `reports` table `file_path` convention (relative vs absolute — must match what Module 5 expects to link/download).

**Testing:** unit-test `health_index.py`'s formula against hand-computed expected scores for a few synthetic input sets; test `report_builder.py` produces a valid, non-empty file.

---

# Integration Day Checklist (for whoever merges the 6 folders)
1. Merge all 6 folders + `shared/schemas.py` + root `config.py` into one repo; resolve any accidental duplicate model/config definitions first (a merge conflict here means someone didn't follow 0.5–0.8).
2. Stand up Postgres from Module 2's `schema.sql`/Alembic, then run Module 1's ingestion against a handful of real ARGO files.
3. Build Module 3's FAISS index from the freshly ingested data.
4. Smoke-test Module 4's `QueryEngine` against 3–4 sample questions end to end.
5. Launch Module 5's Streamlit app pointed at the real DB/engine (not mocks).
6. Generate one real report via Module 6 and confirm it appears in the dashboard's Reports tab.
