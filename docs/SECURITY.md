# Security review (Phase 8, 2026-11-08)

A manual review of the whole codebase, backed by automated tests (`backend/tests/security.test.ts`
plus the auth, users, tickets, voice, complaints and RAG suites). It covers the code. Your
deployment (TLS, network, secrets management) is listed under *Deployment responsibilities*.

## Findings fixed in Phase 8

| # | Issue | Risk | Fix | Test |
|---|---|---|---|---|
| 1 | `trust proxy` hard-coded to 1 | Without a reverse proxy, clients could forge `X-Forwarded-For` and change their rate-limit identity | `TRUST_PROXY` setting, default **0**. The WebSocket limiter uses the same hop rule | `security.test.ts` (header ignored at 0 hops, correct at 1 and 2) |
| 2 | Swagger UI always public | Full API map exposed in production | `ENABLE_API_DOCS`: on outside production, **off in production** unless explicitly enabled | `apiDocsEnabled` cases |
| 3 | Unbounded PDF page count | A 25 MB PDF with thousands of pages could tie up the single ingestion worker | `MAX_PDF_PAGES` (default 2,000). pdf.js loads with font loading and XFA disabled (bundled pdf.js 6.4 no longer has the eval font path behind CVE-2024-4367) | page-cap test |
| 4 | No limit on WebSocket upgrade attempts | Each attempt costs a JWT check and a DB lookup, so a flood was possible | 30 upgrade attempts per minute per client IP, rejected with 429 before any work | 32 attempts → 30×401, then 429, 429 |

Bugs fixed in earlier phases that also had security impact: an IPv6 rate-limit bypass in the voice
limiter (Phase 4), the analytics date-range bug (Phase 7), and dependency advisories in the test
runner (Phase 4).

## Controls verified

**Authentication and sessions**
- bcrypt (cost 12). Login takes the same time for unknown emails, and returns the same message for an
  unknown email or a wrong password. The login rate limit is 20 attempts per 15 minutes per IP.
- JWT HS256 with issuer and audience pinned. The secret must be ≥ 32 characters, and the placeholder
  secret is refused in production. The user is re-read on every request, so deactivation, role
  changes, sign-out and password changes revoke existing tokens immediately (`tokenVersion`).
- Public sign-up (`POST /auth/register`, rate-limited, `SELF_REGISTRATION_ENABLED`) only ever creates Field Operators —
  the body is `.strict()` and has no role field — and the account is locked behind a one-time email code until verified.
- Admin-created and seeded accounts must change their temporary password
  before any other API call works. An admin cannot demote or deactivate themselves or the last active admin.

**Authorisation**
- Role checks happen on the server for every route; frontend guards are for navigation only.
- Object-level checks: operators only reach their own tickets, drafts and conversations (others
  return 404, so IDs can't be probed). Notifications are scoped to their owner.
- The OpenAPI coverage test checks that every route except login and health requires a bearer token.

**Input handling**
- Every body, query and parameter is validated with Zod. Schemas are `.strict()`, which blocks mass
  assignment. Query and body objects carrying `$`-operators are rejected (NoSQL injection is tested).
- Regexes built from user input are escaped. There is no `eval`, no `$where`, and no raw HTML anywhere
  in the frontend, where React escapes all text.
- Uploads: random server-side file names. The audio or PDF file signature is checked; the client's
  MIME type and extension aren't trusted. Size limits apply, and files are never served from a public
  static route. PDFs are downloaded through an authorised endpoint only.
- JSON bodies are limited to 1 MB. WebSocket frames are limited to 32 KB, must have even length, and
  sessions are capped in duration with an idle timeout.

**Secrets and privacy**
- Gnani keys stay on the server. The browser WebSocket authenticates with a subprotocol token, never
  a URL query string, and the server never echoes it back.
- Logs redact authorization headers, passwords, tokens and audio. Request logging records only the
  method, URL and status. Error responses never include stack traces in production.
- Audio is deleted when each request finishes, on success or failure. A sweeper removes files left
  behind, and audio is never stored in MongoDB. Verified in the acceptance run: 0 files and 0 binary fields.

**AI and RAG safety**
- Mock providers are always labelled. Real mode never silently falls back to mock (Evon must be set to
  `mock` explicitly).
- The extraction prompt treats operator text as untrusted data, and model output is validated with a
  strict schema. Severity and equipment are never invented. Retrieved passages are displayed as text,
  and PetroSathi never authorises shutdown, isolation, repair or restart.

## Known limitations and accepted risks

| Item | Notes / recommendation |
|---|---|
| JWT stored in `localStorage` | Readable by injected script if an XSS bug ever existed. No XSS sink was found. For production, serve the frontend with a strict CSP (below), or move to httpOnly cookies with CSRF protection |
| Rate-limit counters are in memory | They are per process and reset on restart. If you run several API instances, use a shared store (e.g. Redis) |
| No per-account lockout | IP-based limits only; a distributed guessing attack is slowed but not blocked. Add an account lockout or CAPTCHA if accounts face the internet |
| WebSocket origin check | Requests without an `Origin` header (non-browser clients) are allowed, but still need a valid JWT |
| ChromaDB has no authentication | The local launcher binds it to 127.0.0.1. Never expose it directly; put it on a private network |
| Audit log IP and email | Personal data; set a retention policy that suits your organisation |

## Deployment responsibilities (outside this codebase)

- Serve both apps over **HTTPS/WSS only**. Set `CLIENT_URL` to the exact frontend origin, and
  `TRUST_PROXY` to the number of proxies in front of the API.
- Frontend response headers, for example:
  `Content-Security-Policy: default-src 'self'; connect-src 'self' https://<api-host> wss://<api-host>; img-src 'self' data: blob:; media-src 'self' blob:; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; frame-ancestors 'none'`
  and `Permissions-Policy: microphone=(self)`.
- MongoDB and ChromaDB with authentication, on private networks only. Use a strong unique `JWT_SECRET`
  (rotating it signs everyone out). Keep `ENABLE_API_DOCS` unset in production.
- Back up MongoDB, including audit logs. ChromaDB can be rebuilt by reprocessing documents.
