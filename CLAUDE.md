# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A LangGraph tool-calling agent ("FDE dispatcher") that answers cold-chain logistics questions by combining three sources: a legacy MSSQL telemetry table, a live weather API, and SOP documents in Pinecone. Front end is Streamlit. The step-by-step build and deploy runbook is [docs/instrutions.md](docs/instrutions.md) (Phases 0 to 6).

## Commands

Python 3.12. No test suite, linter, or build step exists.

```
pip install -r requirements.txt
python scripts/ingest_legacy_data.py      # CSV -> dbo.TBL_SC_FLEET_HIST_RAW (admin creds, replaces table)
python scripts/ingest_sop_pinecone.py     # data/policy/* -> Pinecone (incremental, hash-cached)
python src/agent_tools.py                 # smoke-test all 3 tools against live services
python src/orchestrator.py                # CLI chat loop with the agent ("exit" to quit)
streamlit run src/ui.py                   # main app
```

`scripts/setup_security_and_view.sql` is run manually as `sa` (it uses `GO`, so run it in a SQL tool, not pyodbc). The `FDE_VIEWS.AgentAuditLog` table and its `GRANT INSERT` live only in docs/instrutions.md Phase 4, not in a script.

Local MSSQL: `docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=..." -p 1433:1433 --name legacy-mssql -d mcr.microsoft.com/mssql/server:2022-latest`. All DB code needs **ODBC Driver 18 for SQL Server** installed on the host.

## Architecture

**Data path and security model.** The CSV is ingested with deliberately cryptic legacy column names (`V_LAT`, `IOT_TEMP_VAL_C`, ...). The SQL script builds a semantic view `FDE_VIEWS.VW_ACTIVE_FLEET` with clean names and a login `USR_FDE_RO` that can only SELECT that view (raw table denied). The agent always connects as `USR_FDE_RO`; admin creds are only for ingestion and the UI audit-log tab. If you change the view's columns, also update the `query_telemetry_db` docstring in [src/agent_tools.py](src/agent_tools.py), because the LLM reads that docstring as its schema.

**Agent** ([src/orchestrator.py](src/orchestrator.py)): a two-node graph, `reasoner` <-> `tools` (`ToolNode` + `tools_condition`), with `MemorySaver` checkpointing keyed by `thread_id`. The compiled `fde_agent` is a module-level object imported by the UI.

**Tools** ([src/agent_tools.py](src/agent_tools.py)): `query_telemetry_db` (SELECT-prefix check plus DB permissions, returns max 10 rows), `fetch_corridor_conditions` (Open-Meteo, no key), `search_compliance_sop` (Pinecone retriever, k=2). Tools return error strings instead of raising, so the LLM sees failures.

**Import-time side effects.** Importing `agent_tools` or `orchestrator` loads the embedding model, connects to Pinecone, and builds the LLM. A missing `PINECONE_API_KEY` raises at import. Under Streamlit, the HuggingFace model is cached with `st.cache_resource`; [.streamlit/config.toml](.streamlit/config.toml) disables the file watcher to avoid reload problems with torch.

**System prompt gap.** [src/prompts/system_prompt.txt](src/prompts/system_prompt.txt) is injected only in the orchestrator `__main__` loop. [src/ui.py](src/ui.py) sends only `HumanMessage`s, so the Streamlit agent runs without the system prompt.

**UI** ([src/ui.py](src/ui.py)): streams graph updates, shows tool inputs/outputs as expanders, and writes every tool call and final answer to `FDE_VIEWS.AgentAuditLog` (failures are swallowed and printed). The second tab reads that log with admin creds checked against `.env`.

## Configuration (`.env`, gitignored)

| Variable | Effect |
| :--- | :--- |
| `Agent_llm` | `OPENAI` (gpt-4o), `DEEPSEEK` (needs `DEEPSEEK_API_KEY`), anything else = local Ollama `qwen2.5:7b` |
| `Embeddings_model` | `OPENAI` -> index `fde-sop-index-openai` (1536 dim); else local HF -> `fde-sop-index-local` (1024 dim) |
| `Local_Embedding_Model` | HF model id, default `BAAI/bge-m3` |
| `PINECONE_API_KEY` | required |
| `SQL_SERVER_HOST`, `SQL_SERVER_PORT` | default `localhost`, `1433` |
| `SQL_AGENT_USER`, `SQL_AGENT_PASSWORD` | read-only agent login |
| `SQL_ADMIN_USER`, `SQL_ADMIN_PASSWORD` | ingestion + audit tab |

Ingestion and runtime must use the same `Embeddings_model`, otherwise the agent queries an empty or wrong-dimension index. The ingest script deletes and recreates an index whose dimension does not match.

## SOP ingestion details

`ingest_sop_pinecone.py` reads `.md/.txt/.pdf/.csv/.xlsx` from `data/policy/`. It tracks MD5 per file in `data/cache/ingestion_hash_cache.json` (committed). Changed files are re-chunked and upserted with IDs `<file>-chunk-<n>`; deleted files are purged by `source_file` metadata. The cache is not keyed by index, so after switching `Embeddings_model`, delete the cache file to force a full re-ingest.

## Deployment

[.github/workflows/deploy.yml](.github/workflows/deploy.yml) is manual (`workflow_dispatch`): SSH to EC2, `git pull origin main`, `sudo systemctl restart streamlit`. It does not reinstall requirements, so dependency changes need a manual `pip install` on the box. The systemd unit is documented in docs/instrutions.md Phase 6.
