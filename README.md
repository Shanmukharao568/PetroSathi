# PetroSathi

Multilingual, voice-based maintenance complaint management for oil & gas field operations.
Operators report equipment problems by Push-to-Talk or MP3 upload; Gnani Prisma transcribes,
Gnani Evon extracts a structured complaint, the operator confirms, and a maintenance ticket is
created for engineers. PetroSathi is a **reporting and documentation assistant** — it never
authorizes equipment shutdown, isolation, repair or restart.

> **Status: all 8 phases complete. The 19-step acceptance scenario passed** (mock Gnani, real
> ChromaDB and embeddings) — see [docs/ACCEPTANCE.md](docs/ACCEPTANCE.md).
>
> | Doc | What it covers |
> |---|---|
> | [IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md) | Phase-by-phase checklist and decisions |
> | [GNANI_INTEGRATION.md](docs/GNANI_INTEGRATION.md) | What was verified in Gnani's docs; what needs your account (Evon) |
> | [EMBEDDING_BENCHMARK.md](docs/EMBEDDING_BENCHMARK.md) | Why `multilingual-e5-small` was chosen |
> | [SECURITY.md](docs/SECURITY.md) | Security review, controls, known limits, deployment duties |
> | [ACCEPTANCE.md](docs/ACCEPTANCE.md) | End-to-end acceptance run, step by step |

## Repository layout

| Path | Contents |
|---|---|
| `backend/` | Express + TypeScript API, MongoDB (Mongoose), WebSocket voice proxy, Gnani adapters |
| `frontend/` | React + Vite + TypeScript + Tailwind web app |
| `knowledge-base/` | Approved technical PDFs for RAG (`manuals/`, `safety/`, `troubleshooting/`, `inspection/`) |
| `docs/` | Plan, Gnani integration notes, embedding benchmark, security review, acceptance run |

## Prerequisites

- Node.js ≥ 20 (developed on 23.x)
- MongoDB ≥ 7 running locally (or a connection string)
- Python ≥ 3.10 for the local ChromaDB server (installed in an isolated venv, see below)

## Setup

```bash
# Backend
cd backend
cp .env.example .env      # then edit .env — see "Configuration" below
npm install
npm run dev               # http://localhost:5000  ·  API docs: http://localhost:5000/api/docs

# Frontend (second terminal)
cd frontend
cp .env.example .env
npm install
npm run dev               # http://localhost:5173
```

On Windows, if MongoDB is installed as a service but stopped, start it with
`net start MongoDB` from an elevated terminal.

### Document search (ChromaDB + local embeddings)

```bash
cd backend
python -m venv tools/chroma-venv                                   # once
tools/chroma-venv/Scripts/python -m pip install chromadb==1.5.9    # once (macOS/Linux: tools/chroma-venv/bin/python)
npm run chroma        # ChromaDB on CHROMA_URL (default :8000), data in backend/chroma-data
npm run kb:demo       # optional: (re)generate the 4 labelled DEMO PDFs in knowledge-base/
npm run kb:ingest     # ingest every PDF in knowledge-base/{manuals,safety,troubleshooting,inspection}
```

- Place your **approved** PDFs in `knowledge-base/<folder>/`. Optional sidecar `<file>.pdf.meta.json`:
  `{ "title": "...", "equipmentCodes": ["P-101"], "equipmentTypes": ["pump"] }`. Admins can also upload
  PDFs in *Admin → Documents*. Both paths use the same ingestion pipeline.
- Embeddings run locally with `Xenova/multilingual-e5-small`, chosen by `npm run rag:benchmark` (see
  [docs/EMBEDDING_BENCHMARK.md](docs/EMBEDDING_BENCHMARK.md)). The first run downloads about 120 MB into
  `backend/.models`; after that it works offline, and no document text leaves the server.
- Only text-based PDFs are supported. Scanned image PDFs are rejected with a clear OCR message.
- Answers are **extractive**: retrieved passages with document, page and section. Nothing is generated,
  so nothing can be fabricated.

### First sign-in (development)

1. In `backend/.env` set `SEED_ADMIN_EMAIL` and `SEED_ADMIN_PASSWORD` (≥ 10 chars, upper, lower, digit).
   Optionally set `SEED_DEMO_USER_PASSWORD` for the demo operators/engineers.
2. `cd backend && npm run seed`. This is idempotent and never overwrites existing users. If
   `SEED_DEMO_USER_PASSWORD` is empty, a random demo password is printed **once**.
3. Sign in at http://localhost:5173. Every seeded or admin-created account must set its own
   password at first sign-in.

Demo accounts (all on the reserved `.test` domain): `operator1@`, `operator2@`, `engineer1@`,
`engineer2@petrosathi.test`. The seed also adds eight demonstration equipment records
(P-102, P-201, C-301, HX-401, V-501, FV-601, M-701, PT-801). **P-101 is deliberately not seeded**
so the acceptance scenario can add it through *Admin → Equipment*.

### Ticket lifecycle

```
OPEN ──▶ UNDER_INSPECTION ──▶ IN_PROGRESS ──▶ RESOLVED ──▶ CLOSED (terminal)
  └─────────────────────────────▲   ▲  │          │
                                │   └──┘          └──▶ IN_PROGRESS (reopen, reason required)
                                └── (back to inspection)
```

- Any status change requires an assigned engineer. Only that engineer or an admin can change it.
- Engineers accept unassigned tickets themselves and can release them while still OPEN.
  Admins assign or reassign to any active engineer.
- Resolving requires a resolution summary. Every transition is stored in the ticket history with
  the actor, the time, the old and new status, and an optional comment.
- `POST /tickets` requires an `Idempotency-Key` header. Retrying with the same key returns the
  original ticket instead of creating a duplicate.

### Accounts and roles

- Anyone can **sign up** at `/signup`. Self-registered accounts are always **Field Operators**
  (the request cannot carry a role) and must confirm their email with a 6-digit code before
  anything else works. Set `SELF_REGISTRATION_ENABLED=false` to turn sign-up off.
- Administrators can also create users in *Admin → Users* (any role). The server generates a
  one-time temporary password that is shown once and never stored in plain text.
- Roles: `operator`, `engineer`, `admin`. Every API route enforces roles on the server; the
  frontend route guards are only for navigation.
- Signing out, changing a password, a role change or deactivation revokes that user's existing
  tokens immediately.

## Configuration

All secrets live **only** in `backend/.env` (git-ignored). The frontend `.env` holds public
values only — every `VITE_*` variable is shipped to the browser.

| Variable | Required | Purpose |
|---|---|---|
| `JWT_SECRET` | always | ≥ 32 chars. Generate: `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"` |
| `MONGODB_URI` | always | Defaults to `mongodb://localhost:27017/petrosathi` |
| `AI_PROVIDER_MODE` | always | `mock` (default) or `real` |
| `GNANI_API_KEY_ID` | `real` mode | Gnani API key (sent as `X-API-Key-ID`) — **add yours here** |
| `GNANI_API_KEY_SECRET` | no | Not used by any documented Gnani endpoint |
| `GNANI_PRISMA_STT_URL`, `GNANI_PRISMA_WS_URL` | `real` mode | Prisma STT REST / streaming endpoints |
| `GNANI_PRISMA_MODEL` | recommended | `prisma-v3.0` (needed for auto-detect). Empty → Gnani default (v2.5) |
| `VOICE_MAX_STREAM_SECONDS` | no | Push-to-talk recording cap, default 120 |
| `EVON_PROVIDER_MODE` | **yes, if `AI_PROVIDER_MODE=real`** | Must be `mock`: Gnani publishes no Evon API, so `real` is refused at startup |
| `GNANI_TIMBRE_API_URL`, `_MODEL`, `_VOICE` | optional | Spoken follow-up prompts (real mode only). Leave empty to disable |
| `COMPLAINT_DRAFT_TTL_HOURS` | no | Unconfirmed drafts expire after this many hours (default 24) |
| `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD` | dev seed | Seeded admin must change password at first login |
| `TRUST_PROXY` | production | Number of reverse proxies in front of the API (default 0 = ignore `X-Forwarded-For`) |
| `ENABLE_API_DOCS` | no | Swagger UI; default on in development, **off in production** |
| `MAX_AUDIO_SIZE_MB`, `MAX_PDF_SIZE_MB`, `MAX_PDF_PAGES` | no | Upload limits (20 MB, 25 MB, 2,000 pages) |
| `ANALYTICS_TIMEZONE` | no | IANA zone for analytics months and date filters (default `Asia/Kolkata`) |
| `CHROMA_URL`, `EMBEDDING_PROVIDER`, `EMBEDDING_MODEL`, `MODEL_CACHE_DIR`, `RAG_*` | for document search | Vector store, local embedding model (default e5-small), retrieval settings |

### Provider modes

- **`mock`** — no external calls. Every mock response is tagged `provider: "mock"` and the UI
  shows a pink **AI: mock providers** badge. Use this for local development and automated tests.
- **`real`** — calls Gnani. If required Gnani variables are missing, the server **refuses to
  start** and lists what is missing. It never silently falls back to mock.

## Voice input

- **Push-to-talk:** press and hold the microphone (or hold Space/Enter), speak, then release. Audio is
  captured only while the button is held, converted in the browser to PCM 16-bit, 16 kHz, mono by an
  AudioWorklet, and streamed to `WS /api/v1/voice/stream`. The API proxies it to Gnani Prisma. The
  Gnani key never reaches the browser.
- **MP3 upload:** up to 30 seconds of speech (Gnani's REST limit) and `MAX_AUDIO_SIZE_MB`. The file is
  stored under a random name in `AUDIO_UPLOAD_DIR` and deleted when the request finishes, whether it
  succeeded or failed. A sweeper removes anything older than `AUDIO_RETENTION_MINUTES`.
- **Languages:** Bengali, English, Gujarati, Hindi, Kannada, Malayalam, Marathi, Punjabi, Tamil and
  Telugu (all `-IN`), plus auto-detect with Prisma v3.0. The UI shows what each language supports.
- The original transcript is stored unchanged. Operators edit the ticket description, not the transcript.
- Microphone access needs `localhost` or HTTPS in the browser.

## Complaint workflow

1. Record or upload → original transcript (stored unchanged).
2. **Continue** creates a complaint draft. The extractor finds equipment tag, symptoms and category. It
   reports severity only on clear evidence; otherwise severity stays "Not assessed".
3. The equipment tag is checked against the register. Missing or unknown facts become follow-up
   questions in the operator's language (spoken too, if Timbre is configured).
4. The operator edits details and presses **Confirm and create ticket**. Only then is a ticket created.
   Confirmation uses a server-side idempotency key, so retries or double clicks create one ticket.
5. Unconfirmed drafts expire after `COMPLAINT_DRAFT_TTL_HOURS` and are purged a week later.

## Scripts

| Where | Command | Does |
|---|---|---|
| backend | `npm run dev` | Start API with reload |
| backend | `npm run chroma` | Start local ChromaDB (project venv) |
| backend | `npm run kb:ingest` | Ingest knowledge-base PDFs (needs MongoDB + Chroma) |
| backend | `npm run kb:demo` | Regenerate the labelled demo PDFs |
| backend | `npm run rag:benchmark` | Embedding benchmark → docs/EMBEDDING_BENCHMARK.md |
| backend | `npm run seed` | Create dev admin + demo users (refuses in production) |
| backend | `npm test` | Vitest + Supertest suite with in-memory MongoDB (uses a local `mongod` if installed, otherwise downloads one). Chroma and real-model tests run only when Chroma is up or the model is cached, and are otherwise reported as skipped |
| backend | `npm run typecheck` / `npm run build` | Type-check / compile to `dist/` |
| frontend | `npm run dev` | Vite dev server |
| frontend | `npm test` | UI tests (Vitest + Testing Library + jsdom) |
| frontend | `npm run build` | Type-check and production build |

Both `backend/` and `frontend/` contain an `.npmrc` with `legacy-peer-deps=true`. It works around an
npm 10.9 resolver crash on Vitest's optional browser peers. Keep it when you run `npm install`.

## API conventions

- Base path `/api/v1`; OpenAPI UI at `/api/docs`, JSON at `/api/docs.json` (development; see `ENABLE_API_DOCS`).
  A test fails if any route is undocumented, or documented but not served
- Success: `{ "success": true, "data": …, "meta"?: … }`
- Error: `{ "success": false, "error": { "code", "message", "details"? } }`

## Testing

```bash
cd backend && npm test     # 213 tests; 4 more run only when ChromaDB is up (npm run chroma)
cd frontend && npm test    # 20 UI tests
```

- Automated tests use the **labelled mock** Gnani providers, an in-memory MongoDB and a mock embedding.
  Gnani adapters are tested against local stubs of the *documented* protocols. That is not proof that
  the live service behaves the same, so run it once with your key.
- `tests/multilingual.retrieval.test.ts` runs against the real e5 model once it is cached
  (`npm run rag:benchmark` downloads it). Tests never download models.

## Going to production — checklist

PetroSathi does not deploy itself; these are the settings you control. Details: [docs/SECURITY.md](docs/SECURITY.md).

1. `NODE_ENV=production`, a strong unique `JWT_SECRET`, and `CLIENT_URL` set to the exact frontend origin.
2. HTTPS/WSS only. Set `TRUST_PROXY` to the number of proxies in front of the API.
3. MongoDB and ChromaDB with authentication, on private networks only. Back up MongoDB (including audit logs).
4. Gnani: `AI_PROVIDER_MODE=real`, `GNANI_API_KEY_ID`, `GNANI_PRISMA_MODEL=prisma-v3.0`, and
   `EVON_PROVIDER_MODE=mock` until Gnani provides an Evon API contract. Timbre settings are optional.
5. Frontend: build with `VITE_API_BASE_URL` pointing at the API, and serve it with a strict
   Content-Security-Policy and `Permissions-Policy: microphone=(self)` (example in SECURITY.md).
6. Do **not** run `npm run seed` in production (it refuses anyway). Create the first admin with a
   one-off script or a database insert using a bcrypt hash, then sign in and change the password.
7. Load your approved manuals with `npm run kb:ingest` or *Admin → Documents*, and recalibrate
   `RAG_MIN_RELEVANCE_SCORE` on them (`npm run rag:benchmark` shows how).

