# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI-powered daily stock analysis system for A-shares (CN), HK stocks, and US stocks. Fetches market data, runs multi-source news searches, calls LLMs via LiteLLM, generates decision dashboards, and pushes reports to notification channels (WeChat Work, Feishu, Telegram, Discord, email, etc.). Supports a ReAct-loop Agent mode for multi-turn strategy Q&A, a FastAPI web backend, and a React/TypeScript frontend.

## Commands

### Backend

```bash
# Install dependencies
pip install -r requirements.txt
pip install flake8 pytest

# Run the full CI gate (syntax + lint + offline tests)
./scripts/ci_gate.sh

# Minimum check when environment is restricted
python -m py_compile <changed_python_files>

# Flake8 fatal-only check (what CI enforces)
flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

# Run offline test suite (no external network)
python -m pytest -m "not network"

# Run a single test file
python -m pytest tests/test_config_manager.py -v

# Run with network tests (optional, adds external API calls)
python -m pytest -m network

# Run the analysis (requires .env configured)
cp .env.example .env  # then fill in keys
python main.py
python main.py --debug
python main.py --dry-run
python main.py --stocks 600519,AAPL
python main.py --market-review

# Start web server + run scheduled analysis
python main.py --webui
# Web server only (no scheduled runs)
python main.py --webui-only
```

### Frontend (apps/dsa-web)

```bash
cd apps/dsa-web
npm ci
npm run lint
npm run build
# Dev server
npm run dev
```

### Manual end-to-end scenarios

```bash
./test.sh quick       # single stock fast test
./test.sh dry-run     # data fetch only, no AI call
./test.sh market      # market review only
./test.sh us-stock    # US stocks (Apple, Tesla)
./test.sh a-stock     # A-shares
```

## Architecture

### Entry points

| File | Role |
|------|------|
| `main.py` | CLI entry point; parses args, sets up logging, launches `StockAnalysisPipeline`, `run_market_review`, and optionally starts the FastAPI server |
| `server.py` | Thin wrapper: `uvicorn` launcher for the API |
| `analyzer_service.py` | Standalone service for async analysis jobs |
| `webui.py` | Legacy WebUI entry (now superseded by `main.py --webui`) |

### Core analysis flow (`src/core/pipeline.py`)

`StockAnalysisPipeline` orchestrates the per-stock flow:

1. **Data fetch** — `DataFetcherManager` (in `data_provider/`) selects the appropriate fetcher (AkShare, YFinance, Tushare, Baostock, pytdx) based on market and config.
2. **Technical analysis** — `StockTrendAnalyzer` (`src/stock_analyzer.py`) computes MA, MACD, chip distribution.
3. **Fundamental pipeline** — `fundamental_adapter.py` aggregates valuation, earnings, boards (sector rankings), capital flow; fail-open, timeout-gated.
4. **News search** — `SearchService` (`src/search_service.py`) queries Tavily, SerpAPI, Bocha, Brave, MiniMax, or SearXNG.
5. **AI analysis** — `GeminiAnalyzer` (`src/analyzer.py`) calls LiteLLM with all context; parses structured `AnalysisResult`.
6. **Persist** — `DatabaseManager` (`src/storage.py`) writes to SQLite.
7. **Notify** — `NotificationService` (`src/notification.py`) dispatches to configured channels via senders in `src/notification_sender/`.

Market review runs independently via `src/core/market_review.py`.

### Agent mode (`src/agent/`)

When `AGENT_MODE=true`, `AgentExecutor` runs a ReAct loop:
- Sends LLM a system prompt with tool schemas
- LLM calls tools (`data_tools`, `analysis_tools`, `market_tools`, `search_tools`, `backtest_tools`)
- Loop continues until a final answer or `AGENT_MAX_STEPS`
- Strategies loaded from `strategies/*.yaml` are injected as natural-language instructions
- Multi-agent variant: `AGENT_ARCH=multi` enables orchestrator → specialist agents cascade (`src/agent/orchestrator.py`)

### API layer (`api/`)

FastAPI app created by `api/app.py:create_app()`. All routes are versioned under `api/v1/endpoints/`:

| Endpoint module | Path prefix |
|-----------------|-------------|
| `analysis.py` | `/api/v1/analysis` |
| `history.py` | `/api/v1/history` |
| `stocks.py` | `/api/v1/stocks` |
| `agent.py` | `/api/v1/agent` |
| `portfolio.py` | `/api/v1/portfolio` |
| `backtest.py` | `/api/v1/backtest` |
| `auth.py` | `/api/v1/auth` |
| `system_config.py` | `/api/v1/system-config` |
| `usage.py` | `/api/v1/usage` |

Static frontend files are served from the built output of `apps/dsa-web`.

### Frontend (`apps/dsa-web/`)

React + TypeScript + Vite + Tailwind. Source in `apps/dsa-web/src/`; the build output is what the FastAPI server serves. Auto-built on `python main.py --webui` unless `WEBUI_AUTO_BUILD=false`.

### Configuration (`src/config.py`)

Singleton `Config` loaded from `.env` (via `dotenv`). All features are controlled by environment variables — see `.env.example` for the full list. LLM routing uses LiteLLM with optional `LLM_CHANNELS` multi-channel config or a `LITELLM_CONFIG` YAML file. When `LITELLM_CONFIG` is set, it takes priority over channel env vars for model selection.

### Data providers (`data_provider/`)

`DataFetcherManager` (base class in `base.py`) selects per-market fetchers. US stocks and indices use YFinance exclusively for consistent adjusted-price history. A-share real-time data goes through AkShare/efinance with pytdx as fallback.

### Strategies (`strategies/`)

Plain YAML files. Each file defines `name`, `display_name`, `description`, `category`, `required_tools`, and `instructions` (free-form natural language). No code required. Add files here to extend agent strategies without touching Python. Custom strategy dir: `AGENT_STRATEGY_DIR`.

## Key Conventions

### Directory boundaries

- Backend logic: `src/`, `data_provider/`, `api/`, `bot/`
- Frontend: `apps/dsa-web/`
- CI/deployment: `scripts/`, `.github/workflows/`, `docker/`

### Configuration discipline

- Every new env var must be added to `.env.example` and documented in README / `docs/full-guide.md`.
- User-visible feature changes (CLI, API behavior, report format, notification channels) must also update `docs/CHANGELOG.md`.
- Prefer `fail-open` / graceful degradation for data source failures — do not break existing fallback chains.
- New config should default to "not enabled"; use `parse_env_bool()` from `src/config.py` for boolean flags.

### Commits and releases

- Commit messages in English.
- Version bumps are triggered automatically when a commit title contains `#patch`, `#minor`, or `#major`.
- Manual tags must be annotated tags.

### Testing markers

- `unit` — fast offline
- `integration` — no external network
- `network` — requires live external services (not a CI gate blocker)
