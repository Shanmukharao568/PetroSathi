# PetroSathi — Implementation Plan

Status legend: ☐ not started · ◐ in progress · ☑ done (implemented **and** tested)

## 0. Workspace summary (inspected 2026-11-08)

- `gnani_voice/` was an empty git repo (branch `main`, no commits). No existing code to reuse.
- Local tooling: Node v23.10.0, npm 10.9.2, MongoDB 8.0.10 (service installed, **stopped**),
  Python 3.12.5. Not installed: Docker, ChromaDB server, `mongosh`.

## 1. Architecture

Modular monolith, two npm projects in one repo:

```
gnani_voice/
├── backend/                 Express + TypeScript + Mongoose + ws
│   └── src/
│       ├── config/          env validation (zod), db connection, language registry
│       ├── middleware/      auth, rbac, validate, rate-limit, upload, errorHandler
│       ├── modules/<name>/  model · schema(zod) · service · controller · routes
│       │                    (auth, users, equipment, complaints, tickets, documents,
│       │                     knowledge, conversations, notifications, analytics, audit)
│       ├── providers/
│       │   ├── gnani/       gnaniPrismaService · gnaniEvonService · gnaniTimbreService
│       │   │                (each = interface + real impl + clearly labelled mock impl)
│       │   └── embeddings/  EmbeddingProvider interface + candidates
│       ├── rag/             pdf extraction, chunking, chroma client, ingestion pipeline
│       ├── ws/              /api/v1/voice/stream (auth, PCM framing, upstream proxy)
│       ├── jobs/            temp-audio sweeper, draft expiry
│       └── utils/           logger (redacting), ApiError, response helpers
├── frontend/                React + Vite + TS + Tailwind + React Router + Zustand
│   └── src/ api/ · components/ · features/<role>/ · hooks/ · audio/(AudioWorklet) · store/
├── knowledge-base/          manuals/ safety/ troubleshooting/ inspection/ (approved docs only)
└── docs/                    this plan, API notes, embedding benchmark results
```

Provider mode is chosen by `AI_PROVIDER_MODE=mock|real`. No silent fallback: in `real`
mode, missing config fails startup. Every mock response carries `provider: "mock"` and the UI
shows a "MOCK" badge.

## 2. Phases & checklist (each phase ends with a stop for your confirmation)

### Phase 1 — Foundation ☑ (backend 9/9 tests, both apps verified running)
- ☑ Repo layout, `.gitignore`, root README
- ☑ Backend: TS config, Express app, helmet/cors/rate-limit, env validation, Mongo connection,
  redacting logger, `ApiError` + centralized error handler, consistent JSON envelope,
  `/api/v1/health`, Swagger UI at `/api/docs`
- ☑ Frontend: Vite + React + TS + Tailwind, router skeleton, Axios client, design tokens
  (navy/slate + teal), layout shell, login page placeholder
- ☑ Tests: Vitest + Supertest health/404/error-envelope tests
- **Milestone:** both apps build and run locally.

### Phase 2 — Authentication & roles ☑ (backend 41/41 tests, flow verified in browser + API)
- ☑ User model, bcrypt, JWT (HS256, issuer/audience pinned), `/auth/login`, `/auth/me`,
  `/auth/logout` (server-side revocation via `tokenVersion`), `mustChangePassword` gate,
  admin-only user creation with one-time temporary password, activate/deactivate, role change,
  self password change (returns new token), admin password reset (new temporary password)
- ☑ `authenticate` (re-reads user each request) + `authorize(...roles)`; login rate limit;
  timing-equalised login; last-active-admin and self-demotion guards; AuditLog model + service
- ☑ Seed script (admin from env, demo users on `.test` domain, refuses production)
- ☑ Frontend: Zustand auth store, token verify on load, 401/expiry handling, forced password
  change, role-based redirects and guards, role layouts, code-split role areas, account page,
  admin Users page (search/filter/paginate, create, edit, reset password)
- ☑ Tests: valid/invalid/unknown/inactive login, malformed/forged/expired JWT, logout revocation,
  deactivation revocation, forced change, RBAC, self-escalation, cross-user access, no register route

### Phase 3 — Equipment & tickets ☑ (backend 73/73 tests, flow verified in browser)
- ☑ Equipment CRUD (admin write, all-roles read), code normalisation, delete blocked when referenced
- ☑ Ticket model, atomic `TKT-YYYY-NNNNN` numbering, transition table in `ticket.workflow.ts`,
  per-user `permissions` returned with every ticket, optimistic-concurrency updates
- ☑ Manual ticket creation with required `Idempotency-Key` (replay → same ticket, concurrent-safe)
- ☑ Accept / release (engineer), assign / reassign (admin), status changes with resolution
  and reopen-reason rules, comments, history endpoint
- ☑ Notifications (create/assign/status/comment; never to the actor), mark one/all read
- ☑ Audit records for equipment and ticket actions; demo equipment seed (P-101 left for the admin)
- ☑ Frontend: admin Equipment page, ticket list (scope tabs, filters, severity sort) and detail
  (complaint, transcript, resolution, comments, actions, equipment, history), operator manual
  report with review → confirm, notification bell
- Note: `POST /tickets` (manual complaint) is an addition to the spec's endpoint list; Phase 5's
  draft confirmation reuses the same `createTicket` service and idempotency mechanism.

<details><summary>Original Phase 3 scope</summary>

- Equipment CRUD; MaintenanceTicket model with status state machine
  `OPEN → UNDER_INSPECTION → IN_PROGRESS → RESOLVED → CLOSED` (+ allowed reopen paths),
  history entries (actor, timestamp, from, to, comment), comments, assignment,
  atomic ticket-number counter, notifications, audit log
- Frontend: admin equipment page, engineer ticket list/detail, operator ticket list/detail
- Tests: creation, assignment, valid/invalid transitions, comments, authz

</details>

### Phase 4 — Voice processing ☑ (backend 111/111 tests, browser-verified in mock mode)
- ☑ Prisma docs verified → [GNANI_INTEGRATION.md](GNANI_INTEGRATION.md). Only `X-API-Key-ID` is
  needed (`GNANI_API_KEY_SECRET` is no longer required); `model=prisma-v3.0` must be explicit.
- ☑ Language capability registry (10 verified Prisma languages; Evon/Timbre flags false until verified)
- ☑ Adapter interface + real `gnaniPrismaService.ts` + clearly-labelled `mockPrismaService.ts`
  (`[MOCK TRANSCRIPT]` prefix, provider `mock`); no fallback between modes
- ☑ `POST /voice/transcribe`: magic-byte check, size limit, random names, file deleted in `finally`;
  5-min sweeper for abandoned files; `GET /voice/languages`; Conversation model + `/conversations`
- ☑ `WS /api/v1/voice/stream`: subprotocol JWT auth, origin check, 1 session/user, 32 KB frames,
  even-length PCM check, max duration, idle timeout, upstream re-framing to 1,024-byte frames, commit
- ☑ Voice → ticket: `conversationId` on `POST /tickets`; server copies the ORIGINAL transcript
- ☑ Frontend: AudioWorklet PCM16LE/16 kHz capture (resampler verified with test tones),
  push-to-talk (hold/Space/Enter + Stop), level ring, timer, MP3 upload with progress,
  language picker with capability flags, transcript panel, MOCK labelling
- ☑ Tests: REST (valid/invalid/oversized/timeout/rejected/auth/deletion), adapter vs documented-protocol
  stub (headers, fields, 1,024-byte frames, commit, retries, timeout), WebSocket (auth, origin, concurrency,
  happy path, no-speech, NOT_STARTED, odd frames, 1009, unsupported language, disconnect cleanup)
- Not done: live Gnani verification (needs your key); real-microphone test in a browser with a mic

<details><summary>Original Phase 4 scope</summary>

- **First: verify current Gnani Prisma docs** (REST `POST /stt/v3`, `audio_file`,
  `X-API-Key-ID`; streaming `wss://…/stt/v3/stream`). Anything unverifiable stays mock-only.
- MP3 upload (Multer → temp dir → Prisma → always delete in `finally`) + sweeper for orphans
- Push-to-Talk: AudioWorklet → PCM16LE mono 16 kHz → app WS `/api/v1/voice/stream`
  (`start` / binary / `stop` ↔ `session_ready` / `transcript` / `completed` / `error`)
  → backend proxies to upstream protocol; JWT auth on upgrade, frame-size limits, max duration
- Language capability registry (from verified docs only)
- Frontend: PTT button, level meter, timer, connection state, MP3 upload w/ progress
- Tests: valid/invalid/oversized audio, timeout, missing creds, WS disconnect, bad frames,
  audio deletion

</details>

### Phase 5 — Complaint extraction (Evon) ☑ (backend 165/165 tests, browser-verified in mock mode)
- ☑ Evon: no public API → prompt and output validation built and tested; transport not implemented; startup
  refuses real Evon; explicit `EVON_PROVIDER_MODE=mock`; labelled keyword mock (en/hi/te/ta)
- ☑ `ComplaintExtraction` Zod schema (strict, normalised, ignores model-supplied ids)
- ☑ Drafts: DRAFT → NEEDS_INFORMATION ⇄ READY_FOR_CONFIRMATION → CONFIRMED, plus EXPIRED (lazy + 10-min job,
  TTL purge 7 days after expiry); equipment checked against MongoDB; localised follow-up questions;
  answers stored verbatim; confirm = explicit, idempotent (server-side key), concurrent-safe
- ☑ Endpoints: `/complaints/extract`, `/complaints/drafts` (+ list), `GET/PATCH/DELETE /drafts/:id`,
  `/drafts/:id/questions`, `/drafts/:id/confirm`, `/drafts/:id/speech`
- ☑ Timbre TTS adapter from documented API (serialised, cached, audio-only success); real-mode only
- ☑ Frontend: Continue → draft page (transcript, follow-ups with Listen, editable details,
  confirm panel with safety notice, discard), unfinished-reports list on the report page
- Not done: live Evon (no API); live Timbre test (needs your key)

<details><summary>Original Phase 5 scope</summary>

- Verify Evon docs; adapter + Zod-validated `ComplaintExtraction`
- Equipment validation against Mongo; missing-info questions; draft state machine
  `DRAFT → NEEDS_INFORMATION → READY_FOR_CONFIRMATION → CONFIRMED` (+ `EXPIRED` via TTL job)
- Explicit confirm with idempotency key → exactly one ticket
- Timbre TTS (optional, text fallback)
- Frontend: transcript edit, extraction preview, follow-up Q&A, review & confirm screen
- Tests: valid, missing equipment/symptoms, ambiguous, invalid severity, unsupported language,
  draft edit, confirm, duplicate confirm

</details>

### Phase 6 — RAG ☑ (backend 194/194 tests incl. live ChromaDB + real-model multilingual; browser-verified)
- ☑ ChromaDB 1.5.9 in an isolated project venv (`backend/tools/chroma-venv`, `npm run chroma`); one
  collection per embedding model, cosine space; array metadata filtered with `$contains`
- ☑ Embedding interface; local transformers.js provider (no API key, offline after first download);
  labelled mock (hashed bag-of-words) for automated tests only
- ☑ Benchmark (`npm run rag:benchmark`) → [EMBEDDING_BENCHMARK.md](EMBEDDING_BENCHMARK.md): **chosen
  `Xenova/multilingual-e5-small`** (Recall@5 0.982, MRR@5 0.917, cross-lingual Recall@5 0.977 vs MiniLM
  0.786 / 0.632 / 0.727). The first run exposed a chunker bug (title-only chunks); fixed, then re-run
- ☑ One ingestion pipeline (preloaded `npm run kb:ingest` + admin upload): PDF signature check,
  page-level text extraction, heading/section detection, page-aware chunking with overlap,
  boilerplate removal, embeddings, vector replace, status/error/attempts in MongoDB, restart resume
- ☑ Four clearly labelled DEMO PDFs (`npm run kb:demo`), no confidential/manufacturer content
- ☑ API: documents CRUD + status + reprocess + protected file; `/knowledge/search` (filters: equipment
  code/type, document type/id, source; topK; threshold; explicit no-result); `/knowledge/ask`
  (extractive, no fabricated text); `/tickets/:id/references` (on demand, widening scope)
- ☑ Frontend: admin Documents page (stats, upload with progress, live status, errors, reprocess,
  delete, open, test search); ticket "Manual references" panel with citations that open the PDF page
- Caveats: relevance threshold has narrow separation on the demo corpus — recalibrate on real manuals;
  scanned PDFs (OCR) not supported; Evon-generated answers unavailable (extractive only)

<details><summary>Original Phase 6 scope</summary>

- ChromaDB (run via `pip install chromadb && chroma run`), PDF extraction with page numbers,
  chunking, metadata, ingestion status tracking, reprocess/delete with vector cleanup
- Embedding benchmark: ≥2 multilingual candidates on petroleum-domain EN/HI/TE/TA/code-mixed
  queries → Recall@5, MRR@5, latency, cost → documented choice
- `/knowledge/search` with filters; Evon grounded answers with real citations only
- Tests: upload, extraction, page metadata, filters, no-result, deletion/vector cleanup

</details>

### Phase 7 — Dashboards & analytics ☑ (backend 208 tests: 204 pass, 4 Chroma-integration skipped without a server; 14 new analytics tests; browser-verified)
- ☑ `GET /dashboard`, role-specific from live data. Operator: push-to-talk / MP3 / language selector,
  own counts, recent tickets, unfinished reports, recent activity. Engineer: assigned / open / under
  inspection / in progress / resolved / critical, unassigned queue, severity-ordered work list,
  activity. Admin: system tiles, people, equipment, knowledge base, activity, latest audit
- ☑ `GET /analytics/overview|tickets|equipment` (admin): monthly created vs resolved (zero-filled,
  timezone-aware), status, severity, category, average resolution overall and by severity (null,
  never 0, when nothing resolved), complaints per equipment, by type, recurring issues (2+) with top symptoms
- ☑ `GET /audit-logs` (+ facets): action exact or prefix, entity, actor, date range, pagination
- ☑ Charts per the dataviz method: palette validated with the script (blue/orange CVD ΔE 24.7; ordinal
  severity ramp passes), thin bars with value at tip, 2px straight lines with crosshair and one tooltip,
  legend and end labels, table-view twin on every chart, a single filter row, previous render dimmed on refetch
- ☑ Bugs found and fixed: resolution average ignored the date range (`$match` spread overwrote
  `resolvedAt`); UTC date ranges produced a phantom month in IST; smoothed lines overshot below zero;
  clipped end and tip labels; horizontal page overflow (grids without a shrinkable base track, and
  `sr-only` header text escaping table scrollers). This also fixed the Users and Equipment pages on phones
- Not included: dark mode (the app is light-only); chart texture fill for print / forced-colors

<details><summary>Original Phase 7 scope</summary>

- Operator / engineer / admin dashboards, Recharts analytics from real aggregations,
  notifications, audit-log viewer

</details>

### Phase 8 — Hardening & docs ☑
- ☑ Security review → [SECURITY.md](SECURITY.md). Four fixes, each with tests: `TRUST_PROXY` (no
  spoofable X-Forwarded-For by default), API docs off in production, PDF page cap plus pdf.js
  hardening, WebSocket upgrade flood limit. NoSQL injection, mass assignment and error leakage are
  verified by tests
- ☑ OpenAPI coverage test: every served route is documented, none are stale, and every route except
  login and health requires auth
- ☑ Frontend UI tests (Vitest + Testing Library, 20 tests): login, role guards, forced password
  change, status actions, push-to-talk button, chart table views, date presets. Found and fixed a
  pointer-capture ordering bug in the push-to-talk button
- ☑ Full acceptance scenario, all 19 steps, with real ChromaDB and e5 and mock Gnani →
  [ACCEPTANCE.md](ACCEPTANCE.md)
- ☑ README: setup, configuration, production checklist, testing

<details><summary>Original Phase 8 scope</summary>

- Full test pass, security review, OpenAPI completeness, setup + env guides

</details>

## 3. External configuration you will need to supply (in `backend/.env`)

| Item | Needed from | Notes |
|---|---|---|
| `GNANI_API_KEY_ID` / `GNANI_API_KEY_SECRET` | Phase 4 | Prisma STT (REST + streaming) |
| `GNANI_EVON_API_URL`, `GNANI_EVON_MODEL` | Phase 5 | Exact endpoint/model enabled on your account |
| `GNANI_TIMBRE_API_URL`, `GNANI_TIMBRE_MODEL`, `GNANI_TIMBRE_VOICE` | Phase 5 | Optional |
| `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD` | Phase 2 | Dev seed only; forced change on first login |
| Running MongoDB | Phase 1+ | Local service is installed but stopped |
| Running ChromaDB | Phase 6 | `pip install chromadb` then `chroma run --path ./chroma-data` |
| Embedding provider/model/key | Phase 6 | Chosen after benchmark |
| Approved PDFs in `knowledge-base/` | Phase 6 | Only sample docs clearly marked "DEMO" will be generated |

Until credentials exist, everything runs in `AI_PROVIDER_MODE=mock` with visible MOCK labels.
No real Gnani test will be reported as passing unless run with your credentials.
