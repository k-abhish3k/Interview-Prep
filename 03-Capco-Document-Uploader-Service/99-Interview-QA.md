# 99 — Interview Q&A: Document Uploader Service

Read this last, once chapters 01–05 and the notebooks are fresh. Questions are grouped by type;
in a real interview they'll be mixed together, often as follow-ups to each other.

---

## Behavioral

### Q1. Tell me about the Document Uploader Service you built at Capco.

Use the STAR summary from `00-README.md` as your skeleton, under 90 seconds:

*Situation* — documents feeding the client's internal knowledge base were being collected manually,
with no audit trail or status tracking. *Task* — build an end-to-end uploader/ingest service with full
CRUD functionality, deployed on the client's Azure environment. *Action* — designed a REST API around
a `documents` resource, built it in **FastAPI** with Pydantic validation and a dependency-injection
system for database sessions and auth, used **SQLAlchemy** as the ORM for the data layer, enforced
**RBAC via MSAL/OAuth 2.0** against Azure AD on every endpoint, containerized it with a multi-stage
Docker build, deployed to Azure App Service with an Azure Function handling async post-upload
processing, and moved secrets into Key Vault with managed identity access.
*Result (illustrative)* — cut manual onboarding time by roughly 70% and gave the team a reliable,
auditable ingestion path feeding the chatbot's knowledge base.

Keep the "illustrative" numbers labeled as such if asked directly what you measured versus estimated —
honesty about which numbers are real versus defensible estimates reads as more credible than a
suspiciously precise made-up figure.

### Q2. Describe a technical decision on this project you'd defend, and one you'd reconsider.

**Defend:** choosing to split the workload between Azure App Service (synchronous upload API) and
Azure Functions (async post-upload processing) rather than doing everything synchronously in one
FastAPI request handler. This kept upload latency low and predictable regardless of how slow downstream enrichment (OCR,
indexing) got, and let each piece scale independently.

**Reconsider:** if the service started as a single synchronous endpoint that did validation, storage,
*and* text extraction inline (a reasonable MVP shortcut), that's worth naming as something you'd
split out earlier next time — it's a realistic version of "moved fast initially, paid a small
refactor cost later," which is a more credible answer than claiming everything was architected
perfectly from day one.

### Q3. How did you handle a production issue or bug in this service?

Structure any real (or plausible, clearly-framed) incident with STAR: what broke (e.g., a retrying
client created duplicate document rows because uploads weren't idempotent), how you diagnosed it
(logs/Application Insights showing repeated `POST`s with identical file content in a short window),
what you changed (added idempotency-key support, described in chapter 01), and the outcome (duplicate
uploads eliminated, verified via monitoring over the following weeks).

---

## Technical Deep-Dive

### Q4. How do you handle large file uploads without blocking the request thread?

Two layers to this answer. First, within the request itself: FastAPI's `UploadFile` already spools to
disk past a size threshold rather than holding the whole upload in memory, and `await file.read()`/
streaming the underlying `SpooledTemporaryFile` straight to blob storage avoids ever materializing a
large file as one in-memory `bytes` object; a request-size guard (checked early via `Content-Length` or
a small ASGI middleware) rejects oversized payloads before the handler does real work, similar in
intent to Flask's `MAX_CONTENT_LENGTH` but implemented explicitly rather than via a framework config
key. Second, and more importantly: don't do slow *processing* (OCR, extraction, indexing) inside the
upload request at all — accept the file, persist it, write a metadata row with `status="uploaded"`, and
return `201` immediately. An Azure Function triggered on blob creation does the slow work
asynchronously, and the client polls or gets notified when `status` becomes `"ready"`. This is the
async pattern from chapter 01 combined with the App Service/Functions split from chapter 04 — the
request is never blocked on anything slower than a storage write, and because the path operation is
`async def`, the worker process isn't even blocked *waiting* on that write — it can serve other
requests while the `await` is in flight.

### Q5. How would you make this endpoint idempotent?

`POST /documents` isn't idempotent by default — a retried request naively creates a duplicate. Fix:
require an `Idempotency-Key` header from the client (a UUID generated per upload attempt, not per
retry). On receiving a request, check a lookup table (or a unique constraint) for that key; if it's
been seen, return the original response instead of reprocessing; if not, process normally and record
the key with the result, with a reasonable TTL (e.g., 24 hours) so the table doesn't grow unbounded.
Worth noting under follow-up: `PUT`, `GET`, and `DELETE` are idempotent *by design* in this API (chapter
01's REST verb table) — it's specifically `POST` (create) that needs this extra mechanism.

### Q6. Managed Identity vs. storing secrets in Key Vault directly — what's the difference?

Storing a secret *in* Key Vault solves where the secret's value lives (centralized, encrypted,
audited, rotatable) — but the app still needs to authenticate *to* Key Vault to retrieve it, and if
that authentication itself uses a hardcoded credential, you've just relocated the original problem one
level up. Managed Identity solves that second half: Azure assigns the App Service/Function an identity
in Entra ID with no credential a developer ever sees, grants that identity access to the vault, and the
Azure SDK (`DefaultAzureCredential`) acquires tokens transparently. The combination — secrets
centralized in Key Vault, retrieved via managed identity — is what removes hardcoded credentials from
the system end-to-end; either piece alone is incomplete.

### Q7. How would you version this API?

For an internal service with a small number of known consumers (an admin UI, the indexing pipeline),
URI versioning (`/v1/documents`, `/v2/documents`) is the pragmatic choice — explicit in logs, easy to
route at the App Service level, trivial to test with curl, no content-negotiation logic required on
either side. Header versioning (`Accept: application/vnd.app.v2+json`) keeps URLs cleaner but adds
complexity that isn't justified for a small internal consumer set. Whichever scheme, the discipline
that matters more than the choice itself: never make a breaking change to `v1` in place — ship `v2`
alongside it and give consumers a migration window.

### Q8. Walk me through what happens, end to end, when a client uploads a document.

Client sends `POST /v1/documents` as `multipart/form-data` with a file part and a JSON metadata part,
including an `Idempotency-Key` header. FastAPI validates the request through the path operation's typed
parameters — file present (`UploadFile`), metadata matching the `DocumentCreate` Pydantic model — before
any business logic runs, then `Depends(get_current_user)` validates the caller's Azure AD-issued token
and `Depends(require_role("DocumentUploader.Write"))` checks their RBAC role claim (chapter 06). Once
authenticated and authorized, it streams the file to Azure Blob Storage, inserts a metadata row via the
SQLAlchemy `DocumentRepository` (`status="uploaded"`, audit columns populated, `tenant_id` resolved from
the token, never client input), and returns `201 Created` with the new document's ID and a `Location`
header. Blob creation fires an event; an Azure Function with a blob trigger picks it up, runs enrichment
(e.g., Cognitive Services Document Intelligence for OCR), and updates the SQL row's `status` to
`"ready"` or `"failed"`. Downstream, an indexing process (course 01's territory) picks up `"ready"`
documents and feeds them into the chatbot's search index. All secrets used along the way (SQL connection
string, storage key) come from Key Vault via managed identity — nothing is hardcoded.

### Q9. Why FastAPI over Flask for this service?

Four concrete reasons, not just "it's newer": **async support** — the upload endpoint spends most of its
time waiting on I/O (blob storage, SQL, Key Vault, Graph API), and `async def` path operations let one
worker process handle many concurrent requests during those waits instead of blocking a thread per
request. **Pydantic validation built in** — request/response shapes are declared once as typed models
instead of a separate schema library bolted onto the framework, and invalid input is rejected with a
structured `422` before it reaches business logic, which measurably reduces the boilerplate and the
class of bugs that comes from manually checking `if "field" not in request.json`. **Automatic OpenAPI
docs** — for a service that other systems inside a bank need to integrate against (an admin UI, the
chatbot's indexing pipeline, potentially other backend teams), a live, always-accurate interactive schema
is a real integration accelerant over a hand-maintained API doc that drifts out of date. **The
dependency-injection system fits the auth/DB-session pattern cleanly** — `Depends(get_db)`,
`Depends(get_current_user)`, and `Depends(require_role(...))` compose independently and declaratively on
each path operation (chapters 02 and 06), instead of authentication/session logic being manually wired
into every route or handled by a decorator stack.

### Q9a. Why SQLAlchemy ORM instead of raw SQL here?

Maintainability — a `Document` model declared once is the single source of truth for the table's shape,
instead of every hand-written query independently getting the column list right and silently drifting as
the schema evolves. Migration support via **Alembic** — schema changes are versioned, diffable, and
applied consistently across dev/staging/prod instead of hand-written `ALTER TABLE` scripts with no
tooling enforcing convergence. Reduced injection surface — SQLAlchemy's query-building API
(`select(Document).where(...)`) generates parameterized SQL by construction, so there's no
string-concatenation step where a caller-supplied value could end up embedded in SQL text. The honest
caveat: a hand-tuned raw query can still outperform the ORM's generated SQL on the hottest, most
performance-critical paths, and that's not a reason to avoid the ORM everywhere — SQLAlchemy's `text()`
construct is a legitimate escape hatch for that specific query, still parameterized and safe, without
giving up the ORM for the other 95% of the data layer.

### Q10. How do you prevent SQL injection in the metadata queries?

Every value that comes from a caller (filter parameters, IDs, filenames) goes through SQLAlchemy's
query-building API (`select(Document).where(Document.created_by == user)`) rather than string-formatted
SQL — the ORM parameterizes by construction, so there's no code path where a value gets concatenated
into the SQL text. Where a raw query is used deliberately (a hand-tuned hot-path query via `text()`), it
still uses bound parameters (`text("... WHERE created_by = :user").bindparams(user=user)`), with no
exceptions — "just this one internal admin query" is exactly the exception that becomes an incident.
This is demonstrated concretely in `notebooks/02_sql_data_layer_demo.ipynb`, including a side-by-side of
the vulnerable string-formatted query versus the parameterized/ORM-built one.

---

## System Design

### Q11. Design this service to handle 10x the current document volume.

Work through each layer:

- **API layer (App Service):** already stateless, so scale out horizontally with autoscale rules on
  CPU/request queue length — no code changes needed since no session state lives in the app process.
- **Upload path:** keep the accept-fast/process-async split (chapter 01) strict — at 10x volume, any
  synchronous processing still in the request path becomes the first bottleneck.
- **Blob storage:** effectively unlimited horizontal scale already; make sure blob naming avoids hot
  partitions (e.g., avoid sequential timestamp-prefixed names if using a storage account with
  partition-key-sensitive throughput limits).
- **Azure Functions:** consumption plan autoscales with event volume, but check downstream
  dependencies it calls (Cognitive Services, the SQL database) for their own throughput limits —
  Functions scaling out doesn't help if it just moves the bottleneck downstream.
- **SQL:** this is usually the real constraint at 10x. Check the indexes from chapter 05 are actually
  serving the hot-path queries (status lookups, pagination), consider read replicas if reads
  (status polling, listing) dominate over writes, and confirm connection pooling is configured so 10x
  more Function/App Service instances don't exhaust the database's max connections.
- **Idempotency table:** make sure the TTL-based cleanup keeps this from growing linearly with volume
  forever.

The meta-point worth stating explicitly: because the architecture already separates concerns (API vs.
async processing vs. storage vs. metadata), scaling 10x is mostly "turn the dial on each piece
independently and find the new bottleneck," not "redesign the system" — that's the payoff of having
made the App Service/Functions split in the first place.

### Q12. How would you add multi-tenancy (multiple client organizations sharing this service)?

Add a `tenant_id` column to the `documents` table (and every other table), include it in every index
(`(tenant_id, status, created_at)` rather than just `(status, created_at)`), and enforce it in every
query at the repository layer — never trust a client-supplied tenant ID without validating it against
the authenticated identity (resolved via Microsoft Graph / Entra ID, chapter 05). At the storage layer,
either prefix blob paths with `tenant_id/` or use separate storage containers per tenant if isolation
requirements are stricter (some compliance regimes require physical, not just logical, separation).
Worth flagging as a follow-up question to ask the interviewer back: "what level of tenant isolation
does this need — logical (shared DB, tenant_id column) or physical (separate databases/containers per
tenant)?" — the two have very different cost and complexity implications, and asking shows you
understand the tradeoff exists.

### Q13. How would you monitor and alert on this service in production?

Application Insights (or equivalent) for request-level telemetry — latency percentiles, error rates by
status code, and custom events for the async pipeline's stage transitions (`uploaded -> processing ->
ready/failed`), since a rising count of documents stuck in `"processing"` for too long is a real signal
of a stuck or failing Function. Alert on: elevated 5xx rate on the API, elevated Function execution
failures, documents stuck in `"processing"` past a threshold (e.g., 15 minutes), and Key Vault access
failures (a strong signal of a misconfigured managed identity or an expiring access policy).

---

## "What Would You Change If You Rebuilt This Today?"

### Q14. What would you change if you rebuilt this today?

A strong answer names 2–3 specific, defensible changes rather than either "nothing" (reads as
incurious) or a vague "I'd use better technology" (reads as unprepared):

1. **Event Grid instead of a raw blob trigger** for the post-upload Function, once there's more than
   one downstream consumer of "a document was uploaded" (e.g., both an indexing pipeline and a
   notification service) — Event Grid lets multiple subscribers react to the same event independently,
   where a single blob trigger function tends to accumulate unrelated responsibilities over time.
2. **Structured, schema-validated status transitions** enforced at the database layer (a check
   constraint or an explicit state-machine table) rather than trusting application code to only ever
   set `status` to one of the expected values — cheap insurance against a future bug silently writing
   an invalid status that breaks downstream polling logic.
3. **Fine-grained, resource-scoped authorization** beyond flat App Roles — today's RBAC (chapter 06)
   answers "can this caller write documents at all," but doesn't express something like "this uploader
   may only manage documents in the `compliance/` category." Azure AD supports richer patterns for this
   (application-managed permissions checked against Graph-resolved group membership, or a policy engine
   like Open Policy Agent in front of the FastAPI dependency layer) that weren't necessary at this
   service's original scope but would be worth revisiting if the category/permission matrix grew more
   complex than a handful of App Roles could cleanly express.

The underlying point interviewers are listening for: comfort naming real tradeoffs and specific
technical debt, not defensiveness about the original design. Every real system has things you'd do
differently with hindsight — naming them credibly is a stronger signal than claiming there aren't any.

---

## Client, Production, and Multi-Tenancy (HSBC / Bank of America)

### Q15. This service ingested documents for two banking clients — how did you guarantee HSBC's documents could never be queried or exposed to a Bank of America session?

Isolation had to hold at every layer the data touched, not just one:

- **Identity, not client input, decides the tenant.** The authenticated caller's tenant (`hsbc` or
  `bofa`) was resolved server-side from their Azure AD identity — never accepted as a client-supplied
  header, query parameter, or body field. A client-supplied tenant ID is trivially spoofable; a
  server-resolved one isn't.
- **SQL: `tenant_id` on every table, enforced at one chokepoint.** Every table carries a `tenant_id`
  column, it leads every composite index, and every query passes through a repository layer that
  injects `WHERE tenant_id = @tenant_id` automatically rather than trusting each call site to remember
  it — backed by SQL Server Row-Level Security policies as a second line of defense, so even a bug in
  application code can't return cross-tenant rows.
- **Blob storage: tenant-scoped paths/containers.** Blob paths were prefixed per tenant
  (`hsbc/2026/07/15/<guid>.pdf`), so a storage-level access policy or SAS token scoped to one tenant's
  prefix structurally cannot reach the other's files.
- **Tested, not just designed.** A test suite explicitly asserted "a query scoped to tenant A never
  returns a row belonging to tenant B," regardless of what filters or edge-case inputs were passed —
  because this is exactly the kind of bug that's invisible in normal testing (both tenants' data looks
  identical in shape) and catastrophic if it ships.

The honest framing for an interviewer: a missing `tenant_id` filter here isn't an ordinary bug, it's a
cross-tenant confidentiality breach between two named banking clients — contractually and regulatorily
serious, not just an embarrassing ticket — which is why the isolation was enforced structurally
(one chokepoint, defense in depth) rather than left to developer discipline at every call site. See
chapter 05 for the schema-level detail.

### Q16. How would you design this to handle a customer-facing document-upload SLA in production, not just work correctly in a demo?

A demo only has to be *correct*; a production SLA has to be correct **and** predictable under real
load, real failure modes, and real deploys. The concrete differences:

- **Predictable latency, not just eventual correctness.** The accept-fast/process-async split
  (chapter 01, 04) keeps the synchronous upload response fast regardless of downstream processing time
  — but the async side needed its own latency guarantee too. That's why the post-upload Azure Function
  ran on a **Premium plan instead of consumption**: consumption-plan cold starts after an idle period
  are fine for a demo (nobody's timing it) but not for a client expectation that documents process
  promptly, every time.
- **Zero-downtime deploys.** Deployment slots (staging → smoke test → swap) meant releases were a
  non-event instead of a scheduled outage window — a real SLA can't have a standing "the API is down
  during Tuesday 2am deploys" carve-out.
- **Scaling sized to real traffic shape, not average load.** Two banks' daily upload volume spikes
  around business-day start/end rather than arriving smoothly; autoscale rules were tuned to that
  pattern rather than to a flat average that would under-provision exactly when it mattered.
- **Monitoring that catches degraded-but-not-down states.** Elevated latency, documents stuck in
  `"processing"` past a threshold, and rising Function failure rates all had alerts (chapter 04/05,
  Q13) — a customer-facing SLA is violated by "slow and stuck" long before it's violated by "fully
  down," and a demo has no equivalent failure mode to catch.
- **Defined rollback, not just forward-only deploys.** Every release had an instant-rollback path (slot
  swap-back) — an SLA commitment implicitly promises "if a release goes wrong, we recover fast," not
  just "we deploy carefully."

The one-sentence version: a demo needs to work once, correctly; a production SLA needs to keep working,
predictably, through deploys, load spikes, and partial failures — which is a different (and much
larger) design problem than the CRUD logic itself.

### Q17. What Azure networking decisions were non-negotiable for a bank client here, and why?

Three, in order of how often a banking security review would ask about them:

1. **No public network path to App Service, SQL, or Key Vault.** The App Service and Function app were
   VNet-integrated, and Azure SQL and Key Vault were exposed only via **Private Endpoints** inside that
   VNet — so there's no IP address on the public internet that reaches the database or the secrets
   store directly, regardless of firewall rules or access keys. "Password-protected but publicly
   reachable" is not an acceptable answer to a bank's cloud governance review; "not reachable at all
   except through the private backbone" is.
2. **A single, WAF-fronted public entry point.** Azure Front Door / Application Gateway with a Web
   Application Firewall was the only public-facing surface, terminating TLS and filtering malicious
   traffic (SQLi/XSS patterns, rate-limiting abusive clients) before anything reached the App Service.
   One controlled ingress point is easier to monitor, patch, and reason about than "several services
   each with their own public endpoint and their own security posture."
3. **Identity-based access over network-based trust alone.** Azure AD authentication on the API and
   managed identity for every service-to-service call (App Service/Function → Key Vault, App
   Service/Function → SQL where supported) meant that even *within* the VNet, access wasn't "anything
   inside the network is trusted" — each component's identity was checked, which matters because VNet
   integration prevents external access but doesn't by itself prevent lateral movement between
   components that shouldn't talk to each other.

Why these were non-negotiable rather than "nice to have": a bank's regulators and internal security
team evaluate the *network topology* independently of the application code — they will ask "show me
that the database has no public endpoint" as a literal configuration check, not take "we use
parameterized queries" as a substitute answer. Getting the networking decisions right was a prerequisite
for the engagement being approved to go to production at all, not an optimization layered on afterward.

---

## Authentication, RBAC, and MSAL (chapter 06)

### Q18. Walk me through what happens, request by request, when an HSBC employee calls this API — from token to authorized response.

Five stages, each independently checkable:

1. **Token acquisition (client side, before the request even reaches this service).** The HSBC
   employee's client application uses **MSAL** to authenticate against Azure AD — interactively via the
   Authorization Code flow with PKCE for a human user, or via Client Credentials if the caller is
   another backend system rather than a person. MSAL returns an access token, caching it so subsequent
   calls reuse it (or a refreshed version) without re-prompting Azure AD every time.
2. **The request arrives with `Authorization: Bearer <token>`.** FastAPI's `Depends(get_current_user)`
   dependency runs before any route logic — it's the first thing that touches the request.
3. **JWT validation.** The service fetches (and caches) Azure AD's JWKS signing keys and verifies the
   token's signature, then checks `aud` (issued for *this* API, not some other one), `iss` (issued by
   the expected tenant — HSBC's, specifically, given the multi-tenant context), and `exp` (not expired).
   Any failure here is a `401` before anything else runs.
4. **RBAC role check.** `Depends(require_role("DocumentUploader.Write"))` (or the appropriate role for
   the endpoint) inspects the validated token's `roles` claim — populated by Azure AD from the App Role
   assignment an admin granted this employee — and returns `403` if the required role isn't present.
5. **Tenant-scoped data access.** Only after both checks pass does the route touch data, and even then
   through the `DocumentRepository` (chapter 05), which resolves `tenant_id` from the *token*, not from
   any client-supplied value, and applies it to every query structurally.

The point worth making explicit: these are four independently testable layers (MSAL/token acquisition,
JWT validation, RBAC, tenant scoping), not one blob of "auth logic" — a bug or a test gap in any one of
them is contained to that layer, not a systemic hole across the whole request path.

### Q19. What's the difference between authentication and authorization here, concretely?

They answer three genuinely different questions, and this service keeps them as three separate,
layered mechanisms rather than one blurred check — conflating them is a common mistake candidates make
under interview pressure:

- **Authentication — "who is this?"** Answered by Azure AD, MSAL, and OAuth 2.0. The service validates
  a signed JWT and trusts the identity (`oid`, `email`) it asserts, because that assertion came from a
  trusted issuer, not from anything the client claimed directly.
- **Authorization (RBAC) — "what are they allowed to do?"** Answered by the token's `roles` claim,
  populated from Azure AD App Role assignments, checked via `Depends(require_role(...))`. A caller can
  be fully authenticated (a real, valid HSBC employee) and still be authorized to do nothing but read —
  authentication succeeding says nothing about what operations are permitted.
- **Tenant scoping — "whose data can they see?"** Answered by the `tenant_id` filter in the repository
  layer (chapter 05), resolved from the token's tenant claim. This is deliberately *not* a role — an
  HSBC employee with the `DocumentUploader.Write` role has that role regardless of tenant; it's the
  separately-enforced `tenant_id` filter that keeps them from ever seeing Bank of America's rows, not
  the role name.

Chapter 06 calls this out explicitly as an interviewer trap: naming a role like `HSBC.Uploader` and
treating the naming convention itself as the tenant-isolation mechanism is fragile — it depends on every
future role and every future query respecting a naming convention rather than a structural,
independently-enforced boundary. The correct framing is three layered, independently-enforced concerns:
authentication (Azure AD/MSAL/OAuth), authorization (RBAC role claims), and data scoping (`tenant_id`
enforcement) — not two.

### Q20. How would you test the RBAC logic without hitting real Azure AD in CI?

Never call real Azure AD from a unit or CI test — it's slow, requires network access and real
credentials, and makes tests flaky for reasons that have nothing to do with the code under test. Two
complementary techniques:

- **Mock/stub JWTs signed with a test key.** Generate a JWT locally, signed with a throwaway RSA
  key the test suite controls (not Azure AD's real signing key), containing whatever claims a test case
  needs (`roles: ["DocumentUploader.Write"]`, a specific `tenant_id`). Point the JWKS-fetching code at a
  local test JWKS endpoint (or monkeypatch the key lookup) so the *real* validation code path — signature
  check, `aud`/`iss`/`exp` checks — still runs against a token the test fully controls. This tests the
  validation logic itself, not just the role-check logic downstream of it.
- **FastAPI dependency overrides for pure unit tests of route logic.** FastAPI's `app.dependency_overrides`
  lets a test replace `get_current_user` with a function that returns a fixed `AuthenticatedUser` object
  directly — no token, no signature validation, no JWKS call at all — when the test's actual concern is
  "does this endpoint behave correctly for a user with role X," not "does JWT validation work." Using
  both: a small number of tests exercise the real token-validation path end-to-end with a locally-signed
  test token; the bulk of route-level tests use dependency overrides for a fast, focused check of
  `require_role`'s pass/fail behavior — a `200` when the required role is present, a `403` when it's not.
  `notebooks/01_fastapi_crud_api_demo.ipynb` demonstrates this pattern directly, with a mock role-check
  dependency exercised via `TestClient` for both outcomes.
