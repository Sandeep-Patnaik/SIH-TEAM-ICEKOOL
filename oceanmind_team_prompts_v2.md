# OceanMind AI — Final Prompt Pack (matches project_structure.md)

**How to use:** every member pastes **Part 0 (Common Prompt)** into their Claude session first — it's identical for all 6 people and is what makes independent work mergeable. Then each member pastes **only their own numbered part**. Nobody invents field names, table names, or file paths that aren't in Part 0 — if something's missing, flag it instead of guessing.

Mapping to your team table:; Member 1 → Part 1, Member 2 → Part 2, Member 3 → Part 3, Member 4 → Part 4, Member 5 → Part 5, Member 6 → Part 6.

---

# PART 1 — Member 1: Data Engineering & Database (`ingestion/` + `database/`)

You are a Senior Python Data Engineer. You own **two** folders: `ingestion/` (parsing ARGO NetCDF into clean records) and `database/` (the PostgreSQL schema and the only sanctioned way anyone touches the DB). You also author `ingestion/config.py` — this is the **central config file the entire project imports**, so publish it early; the other 5 people are blocked on it.

## Files you own

**`ingestion/config.py`** — the single config source for the whole app (see Part 0 §Shared Config for the exact required content — copy it verbatim, do not modify field names).

**`ingestion/parser.py`**
- `class NetCDFParser` — reads a `.nc` file with `xarray`/`netCDF4`.
- `parse_float_metadata(ds: xr.Dataset) -> dict` — float_id (WMO ID), deployment lat/lon/date.
- `parse_profile_metadata(ds: xr.Dataset) -> list[dict]` — one dict per cycle: cycle_number, profile_date, latitude, longitude.
- `parse_measurements(ds: xr.Dataset) -> list[dict]` — per depth level: pressure_dbar, depth_m, temperature_c, salinity_psu, plus BGC vars (dissolved_oxygen, chlorophyll, ph) **only if present in the file** — never crash on a missing BGC variable, just set it to `None`.
- Apply ARGO QC flags here (drop/null anything not flagged good, `qc_flag == 1`), and raise `MalformedNetCDFError` on structurally broken files (caught and logged, not fatal to the whole batch).

**`ingestion/utils.py`**
- `assign_ocean_region(lat: float, lon: float) -> str` — **must return one of the exact region names defined in `utils/regions.py`** (Part 0 §Shared Regions). Do not invent your own region names or spelling — Member 4 and Member 2 filter on these exact strings.
- Unit conversions / small helpers (pressure→depth, etc.) as needed.

**`ingestion/loader.py`**
- `class ProfileLoader` — takes parsed dicts, builds `utils.schemas.ProfileRecord` objects (Part 0 §Shared Schemas), and:
  1. writes them to Parquet at `Config.PROCESSED_DATA_DIR / f"{float_id}_{ingestion_run_id}.parquet"` (Member 3 reads these for embedding without hitting Postgres), and
  2. bulk-inserts into Postgres via `database.repository.ProfileRepository`.
- `run_ingestion(folder: Path) -> IngestionSummary` (in `ingestion/loader.py` or a small `ingestion/pipeline.py` if `loader.py` is getting long) — orchestrates parse → clean → region-assign → load for every `.nc` file in the folder; `IngestionSummary` reports files processed, profiles inserted, files skipped with reasons.
- **Upsert on `(float_id, cycle_number)`** so re-running ingestion never creates duplicate profiles.

**`ingestion/exceptions.py`** — `MalformedNetCDFError`, `MissingVariableError`, `RegionAssignmentError`.

**`database/models.py`** — SQLAlchemy ORM classes `Float`, `Profile`, `Measurement`, `Report`, matching the DDL in Part 0 §Database Schema **exactly** — same table names, column names, types, FKs. Do not rename `id` or `float_id`.

**`database/session.py`** — `get_engine()` / `get_session()` built from `Config.DATABASE_URL` (imported from `ingestion.config`).

**`database/schema.sql`** — the raw DDL from Part 0, kept in sync with `models.py`.

**`database/migrations/`** — Alembic setup (nice-to-have if time allows; `schema.sql` is the fallback source of truth for the hackathon demo).

**`database/repository.py`** — `class ProfileRepository`, the **only** sanctioned DB access point for every other module:
```python
def insert_profile_record(self, record: ProfileRecord) -> int: ...
def get_profiles_by_region(self, region: str, start: date, end: date) -> list[Profile]: ...
def get_profiles_near(self, lat: float, lon: float, radius_km: float) -> list[Profile]: ...
def get_measurements_for_profile(self, profile_id: int) -> list[Measurement]: ...
def insert_report(self, region: str, period_start: date, period_end: date, file_path: str, summary_text: str) -> int: ...
def run_raw_query(self, sql: str, params: dict | None = None) -> list[dict]: ...   # SELECT-only, parameterized, used by Member 3
```
Return plain dicts/dataclasses to callers, never raw SQLAlchemy `Row` objects.

## Dependencies
`xarray`, `netCDF4`, `pandas`, `numpy`, `sqlalchemy`, `psycopg2-binary`, `alembic`, `pydantic`, `python-dotenv`.

## What you hand off to the team
- The finished `ingestion/config.py` (post it as-is — everyone else pastes it verbatim into their own repo copy).
- Confirmation that you're using the exact `utils/regions.py` region names.
- The exact `ProfileRepository` method signatures above — treat them as a locked API.
- Your Parquet filename convention.

## Testing
Ship 1–2 tiny synthetic `.nc` fixtures (a script that builds them with `xarray` is fine if you don't have real ARGO files yet) so `pytest` runs without network access. Test `ProfileRepository` against a throwaway SQLite/dockerized Postgres with seeded rows.

Before writing code: give Module Overview → Folder Structure → File Responsibilities → Dependencies → Implementation Plan. Only then generate code. When done: Folder Tree, Setup Instructions, Integration Notes, Testing Instructions.

---

# PART 2 — Member 2: Dashboard & Visualization (`frontend/`)

You are a Senior Python/Streamlit Engineer. You own `frontend/` — the entire UI. You will build against **interfaces**, not the real implementations of Members 1/3/4/5, since they're being built in parallel — stub/mock their functions locally using the exact signatures from Part 0, and swap in the real modules on integration day.

## Files you own

**`frontend/dashboard.py`** — Streamlit entry point. Tabs/pages: **Explore** (map + profile plots), **Chat** (AI assistant), **Ocean Health** (intelligence panel), **Reports** (list + download generated reports). Wires the other 4 files together; keep this file thin — layout only, no business logic.

**`frontend/maps.py`**
- `render_float_map(profiles: list[dict], region_filter: str | None, date_range: tuple[date, date] | None)` — Folium or Plotly map of float trajectories/profile locations, colored by region or a selected variable.
- Pulls data via `database.repository.ProfileRepository.get_profiles_by_region(...)` / `get_profiles_near(...)` (Member 1's locked API — stub it as a function returning fixture data until integration day).

**`frontend/charts.py`**
- `plot_depth_time(measurements: list[dict], variable: str)` — depth vs. time/value plot (Plotly) for temperature/salinity/BGC variables.
- `plot_profile_comparison(profiles: list[dict])` — overlay multiple profiles for comparison.
- Pure functions that take already-fetched data and return a Plotly figure — keep them UI-framework-agnostic so they're testable without Streamlit running.

**`frontend/components.py`**
- `chat_panel(ask_fn: Callable[[str], "ChatQueryResult"])` — renders the chat UI; calls `ask_fn` (Member 3's `ai.chatbot.answer(question: str) -> ChatQueryResult`, Part 0 §Shared Schemas) and shows `answer_text` + a results table from `result_rows`.
- `health_panel(score: "OceanHealthScore")` — renders Member 4's health index (gauge + `contributing_factors` breakdown + `recommendation` text).
- `reports_panel(reports: list[dict])` — lists Member 5's generated reports with download links, reading rows from the `reports` table via `ProfileRepository`.
- `export_button(data: pd.DataFrame, formats: list[str])` — CSV/NetCDF/ASCII export of the currently viewed data.

**`frontend/theme.py`** — Streamlit page config, color palette, consistent chart styling (single source so charts/maps don't look mismatched across files).

## Dependencies
`streamlit`, `plotly`, `folium`, `streamlit-folium`, `pandas`.

## What you hand off to the team
- The exact function signatures you call from each other module (`ProfileRepository.*`, `ai.chatbot.answer`, `intelligence.health.compute`, `reports` table read) — so if you guessed slightly differently than the owning member, it surfaces before integration day, not during it.

## Testing
Unit-test the non-Streamlit helpers in `charts.py`/`maps.py` with `pytest` (data-shaping logic, not the rendering itself). Keep a manual click-through checklist for the actual UI.

Before writing code: Module Overview → Folder Structure → File Responsibilities → Dependencies → Implementation Plan. Only then generate code. When done: Folder Tree, Setup Instructions, Integration Notes, Testing Instructions.

---

# PART 3 — Member 3: AI — RAG + Chatbot (`ai/`)

You are a Senior LLM/RAG Engineer. You own `ai/` — embeddings, the FAISS vector index, retrieval, NL→SQL, and the conversational chatbot layer. This is the biggest module; keep each file under ~350 lines and split further if needed (e.g. `ai/sql_guard.py` if `sql_agent.py` grows too big).

## Files you own

**`ai/embeddings.py`**
- `class ProfileEmbedder` — wraps `Config.EMBEDDING_MODEL` (sentence-transformers). `embed_text(text: str) -> np.ndarray`.
- `build_profile_summary(profile: dict, measurements: list[dict]) -> str` — turns one profile into a short natural-language summary ("Profile at 12.3N 68.1E on 2023-03-04 in Arabian Sea, temp range 24.1–28.4°C, salinity 35.1–36.0 PSU") — this is what gets embedded and later shown as retrieval context.

**`ai/rag.py`**
- `class VectorIndexBuilder` — builds/updates a FAISS index from Member 1's Parquet files (`Config.PROCESSED_DATA_DIR`) or live via `ProfileRepository`; persists to `Config.FAISS_INDEX_PATH` plus a metadata sidecar (JSON/SQLite) mapping vector → `profile_id`.
- `class RagRetriever` — `retrieve(query: str, k: int = 5) -> list[dict]`, each dict: `{profile_id, summary_text, similarity_score, metadata}`. This exact shape is what `sql_agent.py` and `chatbot.py` consume.

**`ai/prompts.py`** — all prompt templates in one place:
- NL→SQL system prompt: must reference the **exact** schema from Part 0 §Database Schema, forbid anything but `SELECT`, and instruct the LLM to only use the given RAG context, not invent data.
- Chat/answer-summarization prompt: turns SQL result rows + RAG context into a plain-language answer.

**`ai/sql_agent.py`**
- `class NLToSQLTranslator.translate(question: str, rag_context: list[dict]) -> str` — calls the LLM (via MCP or direct SDK per `Config.LLM_PROVIDER`) and returns a SQL string.
- `validate_sql(sql: str) -> bool` — reject anything that isn't a single read-only `SELECT` touching only `floats`/`profiles`/`measurements`/`reports`. Raise `UnsafeSQLError` on failure — never execute unvalidated SQL.

**`ai/chatbot.py`**
- `class OceanChatbot.answer(question: str) -> ChatQueryResult` (Part 0 §Shared Schemas) — orchestrates: `RagRetriever.retrieve()` → `NLToSQLTranslator.translate()` → `validate_sql()` → `ProfileRepository.run_raw_query()` (Member 1's API) → summarize into `answer_text`. On ambiguous/unanswerable questions, return a `ChatQueryResult` with `answer_text` explaining that clearly and empty `result_rows` — **never raise** out to the UI layer.

**`ai/memory.py`** — `class ChatMemory` — keeps recent conversation turns (in-process, e.g. a deque) so follow-up questions ("what about the Bay of Bengal instead?") have context; expose `add_turn(question, answer)` and `get_recent(n=3)`.

## Dependencies
`langchain`, an LLM SDK matching `Config.LLM_PROVIDER`, `faiss-cpu`, `sentence-transformers`, `pydantic`, `numpy`, `pandas`.

## What you hand off to the team
- `ChatQueryResult` field population (confirm it matches Part 0 exactly).
- Confirm `RagRetriever.retrieve()`'s return-dict keys — Member 4 may also use retrieval for insight generation.
- Your rebuild cadence for the FAISS index (e.g. "rebuild after every ingestion run" vs. "rebuild manually") — Member 6 needs this for the deployment/integration script.

## Testing
Mock the LLM and DB in tests. Explicitly test `validate_sql()` rejects `DROP`/`DELETE`/`UPDATE`/stacked statements/comment-injection attempts. Build a tiny FAISS index from 10–20 synthetic summaries and assert `retrieve()` returns sensible matches for a hand-written query.

Before writing code: Module Overview → Folder Structure → File Responsibilities → Dependencies → Implementation Plan. Only then generate code. When done: Folder Tree, Setup Instructions, Integration Notes, Testing Instructions.

---

# PART 4 — Member 4: Ocean Intelligence Engine (`intelligence/`)

You are a Senior Data Scientist / Python Engineer. You own `intelligence/` — the Ocean Health Index, insight/anomaly detection, and recommendation text. You read historical data via Member 1's `ProfileRepository` and hand finished scores to Member 2 (dashboard) and Member 5 (reports) — you do **not** touch PDF generation or the `reports` table yourself, that's Member 5.

## Files you own

**`intelligence/health.py`**
- `class OceanHealthCalculator.compute(region: str, start: date, end: date) -> OceanHealthScore` (Part 0 §Shared Schemas).
- **Write the formula down explicitly and keep it simple/defensible for a demo**, e.g.: normalize temperature anomaly, salinity anomaly, dissolved-oxygen level, and data-coverage completeness each to 0–100, combine with documented fixed weights (e.g. 30/30/25/15) into the final `score`. Store each normalized component in `contributing_factors`.

**`intelligence/insights.py`**
- `detect_trends(region: str, start: date, end: date) -> list[str]` — plain-language observations from the data (e.g. "Salinity has risen 0.3 PSU over the last 6 months").
- Uses `ProfileRepository.get_profiles_by_region(...)` + `get_measurements_for_profile(...)`.

**`intelligence/recommendation.py`**
- `class RecommendationEngine.generate(score: OceanHealthScore, trends: list[str]) -> str` — plain-language recommendation. May optionally call an LLM (reuse Member 3's MCP client pattern via `ai.sql_agent`'s LLM call helper if available, or a lightweight standalone call using `Config.LLM_API_KEY`) — but the numeric score/factors must **not** depend on the LLM, only the phrasing does. Never let an LLM call block or crash score computation — wrap it, and fall back to a templated sentence on failure.

**`intelligence/summary.py`** — `build_region_summary(region: str, start: date, end: date) -> dict` — bundles `OceanHealthScore` + trends + recommendation into one dict, which is what Member 5's report builder and Member 2's dashboard both consume.

**`intelligence/anomaly.py`** *(optional, add if time allows)* — flags unusual deviations from historical baseline per region/period, feeding into `insights.py`.

## Dependencies
`pandas`, `numpy`, `sqlalchemy` (via `ProfileRepository`, not direct), `pydantic`.

## What you hand off to the team
- The exact health-index formula, weights, and input ranges — needs to survive a "why is this score 62?" question in the demo.
- The `region` names you accept — must match `utils/regions.py` exactly (same list Member 1 assigns from).
- The `build_region_summary()` dict shape for Member 5.

## Testing
Unit-test `health.py`'s formula against 3–4 hand-computed synthetic input sets (known inputs → expected score). Test `recommendation.py` degrades gracefully (no crash) when the LLM call fails.

Before writing code: Module Overview → Folder Structure → File Responsibilities → Dependencies → Implementation Plan. Only then generate code. When done: Folder Tree, Setup Instructions, Integration Notes, Testing Instructions.

---

# PART 5 — Member 5: Reports & Export (`reports/`)

You are a Senior Python Engineer specializing in document generation. You own `reports/` — turning Member 4's intelligence summaries into downloadable PDF/HTML reports and general data export utilities, and writing the resulting metadata into the `reports` table via Member 1's repository.

## Files you own

**`reports/templates.py`** — HTML/Jinja2 (or plain string) templates for the report layout: title, period, region, health score section, trends, recommendation, embedded charts.

**`reports/pdf.py`**
- `render_report_pdf(summary: dict, charts: list[bytes], output_path: Path) -> Path` — renders a template + charts into a PDF (`reportlab` or `weasyprint`, your call — pick one and be consistent). `summary` is exactly what Member 4's `intelligence.summary.build_region_summary()` returns.

**`reports/report_generator.py`**
- `class ReportGenerator.generate(region: str, start: date, end: date) -> ReportMeta` (Part 0 §Shared Schemas) — orchestrates: call `intelligence.summary.build_region_summary()` (Member 4) → generate charts (reuse `frontend.charts` pure-plotting functions where possible, or your own minimal Plotly/Matplotlib charts if that creates a circular dependency) → `render_report_pdf()` → write file under `Config.PROCESSED_DATA_DIR / "reports"` → insert a row via `ProfileRepository.insert_report(...)` (Member 1's API) with the **exact** `file_path` convention you document (relative to a known base dir — Member 2 needs this to build download links).

**`reports/export.py`**
- `export_to_csv(df: pd.DataFrame, path: Path) -> Path`
- `export_to_netcdf(df: pd.DataFrame, path: Path) -> Path` — reconstruct a minimal `xarray.Dataset` from the tabular data before calling `.to_netcdf()`.
- `export_to_ascii(df: pd.DataFrame, path: Path) -> Path`
- These are called by Member 2's `export_button` component — keep signatures exactly as above so that wiring is a one-line call.

## Dependencies
`pandas`, `numpy`, `xarray`, `matplotlib` or `plotly` (match whichever you use for chart images), `reportlab` or `weasyprint`, `jinja2` (if templating), `sqlalchemy` (via `ProfileRepository`, not direct).

## What you hand off to the team
- Your `file_path` convention (relative vs. absolute; exact base directory) — this must match what Member 2's `reports_panel` expects to link to.
- Which chart-rendering approach you used, in case Member 2 wants to share code instead of duplicating chart logic.

## Testing
Test `report_generator.generate()` produces a non-empty, valid PDF file for a synthetic `summary` dict (mock Member 4's function). Test each `export_*` function round-trips correctly (write then re-read and compare row counts/values).

Before writing code: Module Overview → Folder Structure → File Responsibilities → Dependencies → Implementation Plan. Only then generate code. When done: Folder Tree, Setup Instructions, Integration Notes, Testing Instructions.

---

# PART 6 — Member 6: Integration, Testing & Deployment (`main.py`, `tests/`, `docs/`, shared `utils/`)

You are the Senior Integration Engineer / tech lead for merging. You don't build a "feature" module — you own making the other 5 modules actually run together, plus the shared cross-cutting code everyone else imports (`utils/`).

## Files you own

**`utils/schemas.py`, `utils/regions.py`, `utils/logger.py`** — you are the long-term owner/maintainer of these (Part 0 defines their *initial* required content, which every member pastes into their own copy on day one so nobody's blocked; you reconcile and finalize the canonical versions at merge time if anyone's copy drifted).

**`main.py`** — single entry point that:
1. Loads `Config` (`ingestion.config`).
2. Optionally triggers ingestion (`ingestion.loader.run_ingestion`) if new files are present.
3. Optionally rebuilds the FAISS index (`ai.rag.VectorIndexBuilder`).
4. Launches the Streamlit dashboard (`frontend/dashboard.py`) — or documents the exact `streamlit run frontend/dashboard.py` command if Streamlit can't be launched from within `main.py` directly.

**`tests/`** — mirrors the 6 module folders (`tests/ingestion/`, `tests/database/`, `tests/ai/`, `tests/intelligence/`, `tests/reports/`, `tests/frontend/`) plus `tests/integration/` for true end-to-end tests that only make sense once real modules are merged (e.g. "ingest a sample file → ask a chat question → get a sane answer").

**`docs/`**
- `docs/SETUP.md` — one clean set of setup instructions merging all 6 members' individual setup steps (Postgres, `.env`, FAISS build, running the app).
- `docs/API_CONTRACTS.md` — the locked function signatures from Part 0/each module (single reference doc so nobody has to dig through code to know what to call).
- `docs/ARCHITECTURE.md` — the pipeline diagram + final folder tree, kept current.

**Merge process you run**
1. Collect each member's temporary folder (`memberN_xxx/`) from their branch.
2. Map into the final structure exactly as listed in Part 0 §Final Folder Structure (`member1_data_engineering/` → `ingestion/` + `database/`, `member2_dashboard/` → `frontend/`, `member3_ai/` → `ai/`, `member4_intelligence/` → `intelligence/`, `member5_reports/` → `reports/`).
3. Resolve any duplicate/conflicting definitions of `utils/schemas.py`, `utils/regions.py`, `ingestion/config.py` first — a conflict here means someone edited a "common" file instead of only reading it, flag it back to them rather than silently picking one.
4. Stand up Postgres from `database/schema.sql`.
5. Run real ingestion on a handful of actual ARGO files.
6. Build the FAISS index from the real ingested data.
7. Smoke-test `ai.chatbot.answer()` with 3–4 real questions end to end.
8. Launch `frontend/dashboard.py` against the real DB/AI engine (not mocks) — check all four tabs.
9. Generate one real report via `reports.report_generator` and confirm it shows up and downloads correctly from the dashboard's Reports tab.
10. Merge `requirements.txt` from all 6 modules into one root `requirements.txt`, pin versions, resolve conflicts.

## Dependencies
Whatever the union of the other 5 modules needs, plus `pytest`, `pytest-mock`, optionally `docker`/`docker-compose` for a reproducible Postgres for the demo.

## What you need from the team
Everyone's "Integration notes to hand off" section from their own part — collect these explicitly rather than re-deriving them from code.

Before writing code: Module Overview → Folder Structure → File Responsibilities → Dependencies → Implementation Plan. Only then generate code. When done: Folder Tree, Setup Instructions, Integration Notes, Testing Instructions.

---

# PART 0 — COMMON PROMPT (paste this first, identically, on all 6 accounts)

You are a Senior Software Architect and Senior Python Engineer, one of 6 people independently building **OceanMind AI**, an AI-powered Ocean Intelligence Platform for the Smart India Hackathon, built on official ARGO NetCDF oceanographic data. You will only see your own module's prompt afterward and cannot talk to the other 5 people — this document is the entire contract that keeps your work compatible with theirs. Do not rename, restructure, or improve anything defined below without flagging it back to the team lead (Member 6) first; treat it as fixed.

## Project Goal
Transform ARGO NetCDF datasets into structured storage, a vector index, and a natural-language + visual decision-support platform: ingestion → PostgreSQL → FAISS → RAG → LLM chat/NL-to-SQL → Ocean Health Index/insights/recommendations → automated reports → Streamlit dashboard.

## Architecture
```
Official ARGO NetCDF Files → Data Ingestion & Processing → PostgreSQL Database
→ Vector Database (FAISS) → RAG Pipeline + LLM → Ocean Intelligence Engine
→ Interactive Dashboard & AI Assistant
```

## Tech Stack (fixed)
Frontend: Streamlit, Plotly, Folium. Backend: Python, PostgreSQL. AI: LangChain, FAISS, an LLM (OpenAI/Qwen/Llama) optionally via MCP. Data: Pandas, Xarray, NetCDF4, NumPy.

## Team & Module Mapping
| Member | Focus | Temp folder (dev) | Final folder(s) |
|---|---|---|---|
| 1 | Data Engineering & Database | `member1_data_engineering/` | `ingestion/`, `database/` |
| 2 | Dashboard & Visualization | `member2_dashboard/` | `frontend/` |
| 3 | AI (RAG + Chatbot) | `member3_ai/` | `ai/` |
| 4 | Ocean Intelligence Engine | `member4_intelligence/` | `intelligence/` |
| 5 | Reports & Export | `member5_reports/` | `reports/` |
| 6 | Integration, Testing & Deployment | `member6_integration/` | `main.py`, `tests/`, `docs/`, `utils/` |

During development, work only inside your own temporary folder on your own branch (`feature/data-engineering`, `feature/dashboard`, `feature/ai`, `feature/intelligence`, `feature/reports`, `feature/integration`) off `main`. No direct commits to `main`. Member 6 maps everyone's temp folder into the Final Folder Structure below at merge time.

## Final Folder Structure (target — build your files as if they already live here)
```text
OceanMind-AI/
├── ai/
│   ├── chatbot.py
│   ├── rag.py
│   ├── embeddings.py
│   ├── memory.py
│   ├── prompts.py
│   └── sql_agent.py
├── ingestion/
│   ├── parser.py
│   ├── loader.py
│   ├── database.py       # thin ingestion-side DB helpers only; ProfileRepository itself lives in database/
│   ├── config.py          # <-- the central Config class for the WHOLE project
│   └── utils.py
├── frontend/
│   ├── dashboard.py
│   ├── maps.py
│   ├── charts.py
│   ├── components.py
│   └── theme.py
├── intelligence/
│   ├── health.py
│   ├── insights.py
│   ├── recommendation.py
│   └── summary.py
├── reports/
│   ├── report_generator.py
│   ├── pdf.py
│   ├── export.py
│   └── templates.py
├── database/
│   ├── models.py
│   ├── session.py
│   ├── repository.py
│   ├── schema.sql
│   └── migrations/
├── utils/
│   ├── schemas.py
│   ├── regions.py
│   └── logger.py
├── data/
│   ├── raw_netcdf/
│   ├── processed/
│   └── faiss_index/
├── tests/
├── docs/
├── main.py
├── requirements.txt
├── README.md
└── PROJECT_STRUCTURE.md
```

## Integration Order
1. Ingestion → 2. PostgreSQL → 3. Dashboard (skeleton) → 4. AI (RAG + Chatbot) → 5. Ocean Intelligence → 6. Reports → 7. Final Testing → 8. Deployment.

## Coding Standards (non-negotiable, all modules)
1. OOP first; every class has a docstring.
2. `if __name__ == "__main__":` in every runnable file with a small self-test/demo.
3. Type hints on every function signature.
4. Every public function has a docstring (purpose, args, returns, raises).
5. `logging` only, never `print()` — use `utils.logger.get_logger(__name__)` (see below) so log format is consistent project-wide.
6. No hardcoded paths (use `ingestion.config.Config`) and no hardcoded secrets (use `.env`).
7. Modular, DRY, reusable functions; no file over ~300–400 lines — split if it grows past that.
8. PEP8: snake_case functions/variables, PascalCase classes, UPPER_CASE constants.
9. `pathlib.Path`, not `os.path`.
10. Absolute imports only (`from ingestion.config import Config`, never relative imports).
11. Every I/O/DB/network/LLM call wrapped in `try/except`, logged with `logger.error(..., exc_info=True)`, never a bare `except: pass`.
12. Only create files inside your assigned folder(s) — see the mapping table above.

## Shared Config — `ingestion/config.py` (Member 1 authors this file; **everyone else pastes this exact stub into their own local copy** to develop against on day one, then deletes their local copy at merge time)
```python
from pathlib import Path
from dotenv import load_dotenv
import os

load_dotenv()
BASE_DIR = Path(__file__).resolve().parent.parent  # repo root

class Config:
    """Central configuration for OceanMind AI, loaded from environment variables."""
    DATABASE_URL: str = os.environ["DATABASE_URL"]
    RAW_NETCDF_DIR: Path = BASE_DIR / "data" / "raw_netcdf"
    PROCESSED_DATA_DIR: Path = BASE_DIR / "data" / "processed"
    FAISS_INDEX_PATH: Path = BASE_DIR / "data" / "faiss_index"
    LLM_PROVIDER: str = os.environ.get("LLM_PROVIDER", "openai")   # openai | qwen | llama
    LLM_API_KEY: str = os.environ.get("LLM_API_KEY", "")
    EMBEDDING_MODEL: str = os.environ.get("EMBEDDING_MODEL", "sentence-transformers/all-MiniLM-L6-v2")
    MCP_SERVER_URL: str = os.environ.get("MCP_SERVER_URL", "")
    LOG_LEVEL: str = os.environ.get("LOG_LEVEL", "INFO")
```
Include a `.env.example` at repo root listing every key above with a placeholder value.

## Shared Logger — `utils/logger.py`
```python
import logging
from ingestion.config import Config

def get_logger(name: str) -> logging.Logger:
    """Return a module-level logger configured with the project's standard format."""
    logger = logging.getLogger(name)
    if not logger.handlers:
        handler = logging.StreamHandler()
        handler.setFormatter(logging.Formatter("%(asctime)s | %(levelname)s | %(name)s | %(message)s"))
        logger.addHandler(handler)
        logger.setLevel(Config.LOG_LEVEL)
    return logger
```

## Shared Regions — `utils/regions.py`
```python
REGION_NAMES = [
    "Arabian Sea",
    "Bay of Bengal",
    "Equatorial Indian Ocean",
    "Southern Indian Ocean",
]

REGION_BOUNDS = {
    # (lat_min, lat_max, lon_min, lon_max) — refine with real boundaries before the demo
    "Arabian Sea": (8.0, 25.0, 50.0, 78.0),
    "Bay of Bengal": (5.0, 22.0, 78.0, 100.0),
    "Equatorial Indian Ocean": (-10.0, 8.0, 50.0, 100.0),
    "Southern Indian Ocean": (-40.0, -10.0, 20.0, 120.0),
}

def assign_region(lat: float, lon: float) -> str:
    """Map a lat/lon pair to one of REGION_NAMES; returns 'Unclassified' if no box matches."""
    for name, (lat_min, lat_max, lon_min, lon_max) in REGION_BOUNDS.items():
        if lat_min <= lat <= lat_max and lon_min <= lon <= lon_max:
            return name
    return "Unclassified"
```
Everyone filters/generates on these exact `REGION_NAMES` strings — Member 1 assigns them, Member 4 groups by them, Member 2 filters the map by them.

## Database Schema — `database/schema.sql` (Member 1 implements; shown here so nobody is blocked waiting)
```sql
CREATE TABLE floats (
    float_id        VARCHAR PRIMARY KEY,
    deployment_lat  FLOAT,
    deployment_lon  FLOAT,
    deployment_date DATE,
    status          VARCHAR
);

CREATE TABLE profiles (
    id            SERIAL PRIMARY KEY,
    float_id      VARCHAR REFERENCES floats(float_id),
    cycle_number  INTEGER,
    profile_date  TIMESTAMP,
    latitude      FLOAT,
    longitude     FLOAT,
    ocean_region  VARCHAR
);

CREATE TABLE measurements (
    id               SERIAL PRIMARY KEY,
    profile_id       INTEGER REFERENCES profiles(id),
    pressure_dbar    FLOAT,
    depth_m          FLOAT,
    temperature_c    FLOAT,
    salinity_psu     FLOAT,
    dissolved_oxygen FLOAT,
    chlorophyll      FLOAT,
    ph               FLOAT,
    qc_flag          SMALLINT
);

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
`id` is always the primary key, `float_id` is always the cross-table foreign key. Do not rename.

## Shared Data Contracts — `utils/schemas.py`
```python
from pydantic import BaseModel
from datetime import date, datetime
from typing import Optional

class ProfileRecord(BaseModel):
    """Ingestion → Database contract: one ARGO profile with its depth-level measurements."""
    float_id: str
    cycle_number: int
    profile_date: datetime
    latitude: float
    longitude: float
    ocean_region: Optional[str] = None
    measurements: list[dict]

class ChatQueryResult(BaseModel):
    """AI (ai/chatbot.py) → Frontend contract."""
    natural_language_query: str
    generated_sql: Optional[str] = None
    result_rows: list[dict] = []
    answer_text: str
    sources: list[int] = []   # profile_ids used as RAG grounding

class OceanHealthScore(BaseModel):
    """Intelligence → Frontend/Reports contract."""
    ocean_region: str
    period_start: date
    period_end: date
    score: float
    contributing_factors: dict[str, float]
    recommendation: str

class ReportMeta(BaseModel):
    """Reports → Frontend/Database contract."""
    ocean_region: str
    period_start: date
    period_end: date
    file_path: str
    summary_text: str
```
If your module genuinely needs a new shared field, add it here (never redeclare a lookalike class in your own folder) and call it out explicitly in your integration notes.

## Error Handling & Testing (all modules)
Wrap every DB/file/network/LLM call in `try/except`, log with `logger.error(..., exc_info=True)`, raise a specific custom exception rather than swallowing errors. Put tests under `tests/<your_area>/` using `pytest`, mocking anything from another member's module so your tests run standalone. Include a `# --- Self-test ---` block under `if __name__ == "__main__":` in each file that exercises it against fixture/sample data.

## Output Format — before generating any code, produce
1. Module Overview
2. Folder Structure
3. File Responsibilities
4. Dependencies (`requirements.txt` contents)
5. Implementation Plan

Only then generate code. When finished, also produce: Folder Tree, Setup Instructions, Integration Notes (what you consume from `utils/schemas.py`/`ingestion/config.py`, what you produce for the next stage), and Testing Instructions.

You will now receive your module-specific prompt. Do not generate files outside your assigned folder(s).
