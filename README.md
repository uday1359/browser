# Agentic Browser

A general-purpose, AI-powered browser for Windows 11. It combines a web browser workspace with an
autonomous agent: you give an instruction by chat or voice, and the agent plans and carries out
multi-step website tasks with Playwright-controlled Chromium. Local LLMs (Ollama, llama.cpp, vLLM) are
used first. Cloud LLMs are used only as a fallback that you have consented to. Every sensitive action
goes through a backend permission and approval system.

The specification is [`CLAUDE.md`](CLAUDE.md). This README describes what is **actually
implemented**. For gaps, see [`docs/KNOWN_LIMITATIONS.md`](docs/KNOWN_LIMITATIONS.md).

---

## Features (implemented)

| Area | What works |
|---|---|
| Browser workspace | Tabs, address bar, back/forward/reload/stop, home, bookmarks, downloads panel. The page view is a live CDP screencast of a host-side Chromium, and mouse, keyboard, wheel and IME input are forwarded to it (TD-002). |
| Identity | Guest mode (in-memory, TTL) and registered mode (email + password, Argon2id, JWT in an httpOnly cookie). Guest history is never imported into an account. |
| Agent | Goal → plan → OBSERVE/THINK/DECIDE/ACT/VERIFY loop, information-gap detection, clarification, duplicate-submission guard, bounded retries/replanning, result verification, pause/resume/cancel, checkpoint after every step. |
| Clarification | When the agent needs information, you can answer with the camera, an uploaded image (local OCR, confirmation required) or by typing. All three go through the same validation. |
| Safety | Risk levels (`low`, `user_permission`, `explicit_approval`, `restricted`). Standing grants, per-action approvals (10-minute expiry), and an audit log. CAPTCHA/login/OTP/blocked-page detection with user handoff. OTPs and passwords are kept in a per-task in-memory vault and never reach an LLM, storage or the logs. |
| LLMs | Ollama (default), llama.cpp, vLLM; cloud fallback to OpenAI, Anthropic and DeepSeek only with `CLOUD_LLM_ENABLED=true`, user consent and the `cloud_sharing` grant. A privacy filter is applied to cloud requests. |
| Voice | Wake phrase, stop/pause/resume/cancel commands, continuous conversation, interruption, language detection for the 11 priority languages (English + 10 Indian languages), voice/rate/volume/mic settings. STT is the Web Speech API or a server endpoint; TTS is `speechSynthesis` or a server endpoint. |
| OCR | Tesseract.js in the backend for camera/image input and the `vision.ocr_page` tool. Structured field extraction; sensitive fields are masked. |
| Plugins | Cordis 4 host with 7 bundles and 10 plugins. Bundles load on demand by capability (see [`docs/PLUGIN_DOCUMENTATION.md`](docs/PLUGIN_DOCUMENTATION.md)). |
| MCP | MCP **server** at `/mcp` (Streamable HTTP, bearer tokens). It has 9 tools, a `task://` resource and one prompt. MCP **client** for external servers from `config/mcp-servers.json`. |
| External resources | Folder/file scopes chosen by the user (realpath containment, revocable) and applications from an allowlist (`config/applications.json`). Uploads are validated by magic bytes. |
| Observability | Pino JSON logs with redaction, request ids, optional OpenTelemetry traces (OTLP/HTTP). |

## Requirements

| Component | Version / note |
|---|---|
| OS | Windows 11. This is the only validated platform. |
| Node.js | ≥ 22.12 (`engines`); tested on **24.22.1** with npm 11 |
| Chromium | Playwright's bundled Chromium (`npx playwright install chromium`) |
| Ollama | Running on `http://127.0.0.1:11434` with a pulled model |
| MySQL 8 | **Optional.** Without `DATABASE_URL` only guest mode is available. Validated with MySQL 8.0 (`backend/test/prisma.integration.test.ts`). |
| Docker/Kubernetes | Not used. Everything runs directly on the host. |

## Quick start

```powershell
# 1. Install dependencies (from the repo root)
npm install
#    npm 11 may block native install scripts. If so, approve exactly these and re-run npm install:
npm install-scripts approve @swc/core esbuild argon2 prisma @prisma/engines protobufjs

# 2. Browser binary and a local model
npx playwright install chromium
ollama pull qwen3:1.7b

# 3. Configuration
copy .env.example backend\.env
#    then edit backend\.env, at least:  OLLAMA_MODEL=qwen3:1.7b
#    (the default qwen3.5:4b is too slow on CPU-only machines, see "Model guidance")

# 4. Generate the Prisma client (needed for the backend build, works without a database) and build
npm run db:generate
npm run build

# 5a. Production-style run: backend serves the built UI on http://localhost:3000
#     (requires NODE_ENV=production and JWT_SECRET in backend\.env, see docs/DEPLOYMENT_GUIDE.md)
npm start

# 5b. Development: backend on :3000 + Vite dev server on http://localhost:5173 (proxies /api, /mcp, /socket.io)
npm run dev
```

Open the UI, choose **Continue as guest** (or register if a database is configured), type a URL or
an instruction such as *"Open wikipedia.org and tell me today's featured article"*.

For registered mode, follow [`docs/DATABASE_DOCUMENTATION.md`](docs/DATABASE_DOCUMENTATION.md) (run
`database/setup.sql`, set `DATABASE_URL`, then run `npm run db:migrate`).

### Model guidance

The agent makes one structured JSON decision per step (output capped at 600 tokens). On CPU-only
machines this decision dominates latency. `tests/integration/bench-decide.mjs` benchmarked a real
~1,400-token decide prompt on an i5-8250U with 8 GB RAM and no GPU:

| Model | Decide latency | Chose the correct action |
|---|---|---|
| `qwen3:1.7b` (**recommended on CPU**) | 49 s cold / 12 s warm | yes |
| `qwen3.5:0.8b` | 36 s | **no** |
| `qwen3.5:2b` | 107 s cold | yes |
| `llama3.2:3b` | 97 s cold | yes |
| `qwen3.5:4b` (default in `env.ts`/`.env.example`) | 156 s cold | yes |

An earlier throughput measurement on the same machine: `qwen3.5:4b` ~21 tok/s prompt / ~3 tok/s
generation, `qwen3:1.7b` ~44 tok/s prompt / ~8 tok/s generation. In a live end-to-end run with
`qwen3:1.7b`, planning took about 43 s and each decide step took 30–55 s.

Recommended `backend/.env` for CPU-only hosts:

```ini
OLLAMA_MODEL=qwen3:1.7b
LLM_TIMEOUT_MS=300000
AGENT_CONTEXT_TOKENS=2000
```

The decide prompt uses compact one-line tool signatures and puts the stable sections first, so Ollama
can reuse the cached prompt prefix between steps ("warm" timings). With a GPU, larger models give
better planning and tool use. To reproduce the benchmark, run `npm run build`, start the fixture site
on port 4600 (`node tests/fixtures/serve.ts`), then run
`node tests/integration/bench-decide.mjs qwen3:1.7b qwen3.5:4b`.

## Configuration

All backend settings are environment variables. They are validated with zod in
`backend/src/config/env.ts`, and invalid values stop startup with a clear message. They are documented in
[`.env.example`](.env.example) and in [`docs/DEPLOYMENT_GUIDE.md`](docs/DEPLOYMENT_GUIDE.md#environment-variables).

| Group | Key variables |
|---|---|
| Server | `NODE_ENV`, `PORT` (3000), `FRONTEND_ORIGIN`, `LOG_LEVEL` |
| Auth | `JWT_SECRET` (≥ 32 chars, required in production), `JWT_TTL_MINUTES`, `GUEST_SESSION_TTL_MINUTES` |
| Database | `DATABASE_URL` (optional), `RETENTION_*` |
| Browser | `BROWSER_HEADLESS`, `BROWSER_ALLOW_PRIVATE_NETWORKS`, `BROWSER_ALLOWED_HOSTS`, `BROWSER_BLOCKED_HOSTS`, `BROWSER_MAX_TABS`, `BROWSER_MAX_DOWNLOAD_MB` |
| LLM | `OLLAMA_*`, `LLAMACPP_*`, `VLLM_*`, `CLOUD_LLM_ENABLED`, `OPENAI_*`, `ANTHROPIC_*`, `DEEPSEEK_*`, `LLM_TIMEOUT_MS` |
| Voice | `STT_*`, `TTS_*` (OpenAI-compatible endpoints) |
| MCP | `MCP_SERVER_ENABLED`, `MCP_SERVER_TOKEN` |
| Agent | `AGENT_MAX_STEPS`, `AGENT_MAX_RETRIES`, `AGENT_CONTEXT_TOKENS`, `OCR_LANGUAGES` |

File-based configuration (in `CONFIG_DIR`, default `config/`, git-ignored):
`applications.json` (application allowlist) and `mcp-servers.json` (external MCP servers). The
`*.example.json` files show the format.

User preferences (voice, cloud consent, homepage) and permission grants are set in the UI under **Settings**.

## Development

Root scripts (`package.json`):

| Script | Purpose |
|---|---|
| `npm run build:libs` | `tsc -b shared browser-engine agent-runtime` |
| `npm run build` | libs → backend (`tsc`) → frontend (`tsc -b && vite build`) |
| `npm run typecheck` | Type-check all packages |
| `npm run lint` | ESLint (flat config) |
| `npm run format` / `format:check` | Prettier |
| `npm run dev` | `concurrently` backend + frontend dev servers |
| `npm start` | Start the built backend (`node --env-file-if-exists=.env dist/main.js` in `backend/`) |
| `npm run db:generate` / `db:migrate` | `prisma generate` / `prisma migrate deploy` |
| `npm test` | Unit/integration (Vitest, projects `libs` + `backend`) and frontend component tests |
| `npm run test:e2e` | Playwright Test (`tests/e2e/playwright.config.ts`) |

Workspaces: `shared` (zod schemas, events, enums), `browser-engine`, `agent-runtime`, `backend`,
`frontend`. After you change `shared`, rebuild it (`npx tsc -b shared`), because the other packages consume its
build output.

## Testing

```powershell
npm test                     # libs + backend (Vitest) and frontend (Vitest + Testing Library, jsdom)
npx vitest run --project libs
npx vitest run --project backend
npm run test:e2e             # Playwright end-to-end tests
$env:LIVE_OLLAMA='1'; npx vitest run agent-runtime/src/llm/providers/ollama.live.test.ts   # live Ollama
```

- The live Ollama test runs only with `LIVE_OLLAMA=1`. It honours `OLLAMA_BASE_URL` and `OLLAMA_MODEL`
  (default `qwen3.5:4b`).
- `backend/test/prisma.integration.test.ts` runs only when `DATABASE_URL` points to a migrated
  database (use a dedicated test database such as `agentic_browser_test`). Otherwise it is skipped.
- Manual end-to-end driver: `node tests/integration/drive-task.mjs "<instruction>" [--approve|--reject] [--answer field=value]`
  (against a running backend; `AB_BASE` defaults to `http://127.0.0.1:3000`).
- Test evidence and results are recorded in `docs/TEST_REPORT.md` and `docs/REQUIREMENT_TRACEABILITY.md`.

## Architecture (summary)

One NestJS host process serves the REST API (`/api/v1`), Socket.IO (`/socket.io`) and the MCP endpoint
(`/mcp`), and in production also the built UI. The same process hosts the Browser Engine (Playwright
Chromium, one isolated `BrowserContext` per app session) and the Agent Runtime (Cordis plugins, tool
registry/dispatcher, LLM router, agent core). The backend is the permission source of truth: the
runtime asks through ports and can never grant itself anything.

| Document | Content |
|---|---|
| [`docs/FINAL_ARCHITECTURE.md`](docs/FINAL_ARCHITECTURE.md) | Packages, process model, component → file map, Cordis services, security boundaries |
| [`docs/DATA_FLOW.md`](docs/DATA_FLOW.md) | CLAUDE.md §5 steps 1–15 mapped to real code paths |
| [`docs/API_DOCUMENTATION.md`](docs/API_DOCUMENTATION.md) | REST, WebSocket and MCP reference |
| [`docs/DATABASE_DOCUMENTATION.md`](docs/DATABASE_DOCUMENTATION.md) | Prisma schema, migrations, retention, storage policy |
| [`docs/PLUGIN_DOCUMENTATION.md`](docs/PLUGIN_DOCUMENTATION.md) | Writing plugins/bundles/tools; every existing tool and its risk |
| [`docs/DEPLOYMENT_GUIDE.md`](docs/DEPLOYMENT_GUIDE.md) | Host deployment on Windows 11, env vars, HTTPS, backups, upgrades |
| [`docs/KNOWN_LIMITATIONS.md`](docs/KNOWN_LIMITATIONS.md) | What is missing, partial or not validated |
| [`docs/TECHNICAL_DECISIONS.md`](docs/TECHNICAL_DECISIONS.md) | ADR log (TD-001 …) |

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `npm install` succeeds but `argon2`/`prisma`/`esbuild` fail at runtime | npm 11 blocked their install scripts. Run the `npm install-scripts approve …` command above, then `npm install` again. |
| Backend build fails with missing `../generated/prisma/client.js` | Run `npm run db:generate` (the client is generated into `backend/src/generated/prisma`, which is git-ignored). |
| `/api/v1/health` shows `browser: "down"` | Chromium is not installed or failed to start. Run `npx playwright install chromium` and check the log line `Browser engine failed to start`. The API keeps working. |
| Agent says *No language model provider is available* | Ollama is not running, or `OLLAMA_MODEL` is not pulled (the health check reports `model "…" is not installed in Ollama`). Check **Settings → System status** or `GET /api/v1/llm/status`. |
| Steps time out on CPU | Use the model guidance above (`qwen3:1.7b`, `LLM_TIMEOUT_MS=300000`, `AGENT_CONTEXT_TOKENS=2000`). |
| Register/login answer `503 Database not configured` | `DATABASE_URL` is not set. Guest mode still works. |
| Navigation to `localhost`/LAN addresses is blocked | SSRF protection. Add the host to `BROWSER_ALLOWED_HOSTS` (for example `localhost`) or set `BROWSER_ALLOW_PRIVATE_NETWORKS=true` (development only). |
| `403 Missing or invalid X-AB-Client header` | Cookie-authenticated `POST/PUT/PATCH/DELETE` requests must send `X-AB-Client: web`. API clients should use `Authorization: Bearer`. |
| Login does not stick in production over plain HTTP | Cookies are `Secure` when `NODE_ENV=production`. Serve over HTTPS (reverse proxy) or use `http://localhost`. |
| `npm run dev`: backend never starts (only `tsc` watch output) | The backend `dev` script uses `tsc -w … & node --watch …`. With npm's default `cmd.exe` shell on Windows, `&` runs the commands one after another, so `node` never starts. Workaround: `npm run build`, then run `npx tsc -p backend/tsconfig.build.json -w` in one terminal and `npm start` (or `node --env-file-if-exists=.env --watch dist/main.js` in `backend/`) in another, plus `npm run dev -w @ab/frontend`. |
| Port 5173 in use | Vite uses `strictPort`. Free the port, or start Vite on another port and set `FRONTEND_ORIGIN` accordingly. |
| OCR is slow the first time / fails offline | Tesseract language data is downloaded on first use from `cdn.jsdelivr.net` into `storage/tessdata`. |

## Limitations

See [`docs/KNOWN_LIMITATIONS.md`](docs/KNOWN_LIMITATIONS.md). The most important ones:

- Cloud LLM providers (tested only against local fakes) and server STT/TTS are **NOT VALIDATED**
  against real services.
- The deployment is a single process (no horizontal scaling).
- On CPU-only hosts, local LLM steps take tens of seconds.
"# browser" 
