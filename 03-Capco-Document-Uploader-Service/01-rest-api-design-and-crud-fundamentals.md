# 01 — REST API Design and CRUD Fundamentals

> **Grounding note, read this first.** The concepts in this chapter (resources, verbs, status codes,
> idempotency, pagination, versioning) are general REST/CRUD fundamentals and are taught here the way
> any solid backend course would teach them. The **real service is not a classic per-resource CRUD
> API**, though — it's closer to a **use-case-oriented proxy in front of an external ingestion API**.
> Concretely: `/upload-files/{use_case}`, `/search-ingested-documents/{use_case}`, and
> `/remove-ingested-document/{use_case}` are thin, mostly-synchronous relays that call an external
> `INGEST_API`'s `batch-initialize`/`ingest`/`GET ingest/HEXA`/`DELETE ingest/.../HEXA` endpoints and
> pass the response straight back — there's no local `documents` table, no `PUT`/`PATCH`, and no
> soft-delete flag for most departments. The one exception is **IWPB**, which has real local state (a
> `IWPBDocumentWorkflow` SQL row per document, with actual create/read/update transitions) — that's the
> part of this service closest to genuine CRUD, and it's covered in chapter 02. Keep that distinction in
> mind while reading the general material below: it's the right mental model for REST API design in
> general, but map it onto *this* service as "proxy + one real stateful workflow," not "five CRUD
> resources."

## Why REST, and why it matters beyond "it's the standard"

Before writing a line of FastAPI, it's worth being able to explain *why* REST is the right shape for a
document uploader service, because interviewers will probe whether you understand the principles or
just memorized HTTP verbs. REST (Representational State Transfer) is an architectural style built
around a small set of ideas:

- **Resources, not actions.** You don't design an endpoint called `/uploadDocument` or
  `/getDocumentById` — you design a *resource* (`/documents`, `/documents/{id}`) and let the HTTP verb
  express the action. This keeps the API surface small and predictable: once a client knows the
  `documents` resource exists, it already knows the shape of every operation on it.
- **Statelessness.** Each request carries everything the server needs to process it (auth token,
  parameters, body) — the server doesn't hold session state between requests. This is exactly what
  lets you run five identical FastAPI containers behind a load balancer on Azure App Service with no
  sticky-session hacks: any instance can handle any request.
- **Uniform interface.** The same verb (`GET`, `POST`, `PUT`, `DELETE`) means the same thing across
  every resource in the system. A client that understands how to `GET` a document already understands
  how to `GET` a user, without reading new documentation.
- **Representations.** The client doesn't get "the document" — it gets a JSON *representation* of the
  document's current state (metadata, status, links), while the actual bytes live separately in blob
  storage. This separation is central to how the uploader service is actually built (see chapter 05).

## CRUD mapped to HTTP verbs and status codes

CRUD (Create, Read, Update, Delete) is the operation set; REST is how you expose it over HTTP. For a
`documents` resource:

| Operation | HTTP Verb | Endpoint | Success Code | Notes |
|---|---|---|---|---|
| Create | `POST` | `/documents` | `201 Created` | Returns the new resource + `Location` header |
| Read (one) | `GET` | `/documents/{id}` | `200 OK` | `404` if not found |
| Read (many) | `GET` | `/documents?status=ready&page=2` | `200 OK` | Filtering + pagination via query params |
| Update (full) | `PUT` | `/documents/{id}` | `200 OK` | Replaces the resource; **idempotent** |
| Update (partial) | `PATCH` | `/documents/{id}` | `200 OK` | Merges fields; not required to be idempotent |
| Delete | `DELETE` | `/documents/{id}` | `204 No Content` | Soft-delete in practice — see chapter 05 |

Getting status codes right is a cheap, high-signal way to demonstrate REST fluency in an interview.
The categories that matter most for this service:

- **2xx** — success. `200` for a body, `201` for "I created something and here it is," `204` for
  "succeeded, nothing to return."
- **4xx** — client's fault. `400` bad request body, `401` unauthenticated, `403` authenticated but not
  allowed, `404` doesn't exist, `409` conflict (e.g., duplicate upload — see idempotency below),
  `413` payload too large (very relevant for file uploads), `422` well-formed but semantically invalid.
- **5xx** — server's fault. `500` unhandled error, `503` the service (or a dependency like blob
  storage) is unavailable.

A common interview trap: returning `200` for everything and stuffing an error message in the JSON
body. That defeats the purpose of HTTP status codes — clients (and monitoring/alerting) rely on the
status code to decide behavior *before* parsing the body.

## Idempotency: the concept that matters most for uploads

Idempotency means calling an operation once has the same effect as calling it many times.
`GET`, `PUT`, and `DELETE` are idempotent by definition in REST — calling `DELETE /documents/42` three
times leaves the same end state as calling it once (deleted). `POST` is **not** idempotent by
default — calling `POST /documents` twice with the same file naively creates two documents.

This matters concretely for a document uploader: a flaky mobile connection or a retrying HTTP client
can resend the same upload request after a timeout, even though the first attempt actually succeeded
server-side. Without protection, that produces duplicate documents in blob storage and duplicate rows
in SQL. The standard fix is an **idempotency key**:

1. The client generates a unique key (a UUID) for a given upload attempt and sends it in a header,
   e.g. `Idempotency-Key: 3fa85f64-5717-4562-b3fc-2c963f66afa6`.
2. The server checks whether it has already processed that key. If yes, it returns the *original*
   response (same document ID, same status code) without redoing the work. If no, it processes the
   upload and records the key alongside the result.
3. Keys are typically stored with a TTL (e.g., 24 hours) — long enough to cover realistic retry
   windows, short enough not to grow the table unbounded.

This is a strong interview answer because it shows you understand that "idempotent" isn't just
textbook trivia — it's the specific mechanism that prevents a document-ingest service from silently
duplicating a client's data under retry, which is a realistic failure mode for any file-upload API.

## Pagination and versioning

**Pagination.** Once the `documents` table has thousands of rows, `GET /documents` can't return
everything. Two common approaches:

- **Offset/limit** (`?page=2&page_size=50` or `?offset=100&limit=50`) — simple, but can skip or
  duplicate rows if data changes between page requests, and gets slow on large offsets.
- **Cursor-based** (`?after=<opaque_token>&limit=50`) — the token encodes "where I left off" (often
  the last row's ID or timestamp), stays fast at any depth, and is stable under concurrent inserts.
  For a document list that's actively being written to (new uploads arriving constantly), cursor
  pagination is the more defensible design choice, though offset/limit is fine for smaller or
  read-mostly datasets and is faster to implement.

Either way, the response should include pagination metadata (`next_cursor`, `has_more`, or
`total_count`/`page`/`page_size`) so the client doesn't have to guess.

**Versioning.** APIs change; clients can't all upgrade simultaneously. The two dominant strategies:

- **URI versioning** — `/v1/documents`, `/v2/documents`. Explicit, cacheable, easy to route at the
  load-balancer or App Service level, and easy for a client to see in logs which version they're
  hitting. This is the pragmatic default for most internal microservices, including a service like
  this one sitting inside a larger client platform.
- **Header versioning** — `Accept: application/vnd.myapp.v2+json`. Keeps URIs clean but is harder to
  test with a browser or curl and easy for clients to get wrong.

For an internal document-ingest service consumed by a small number of known internal clients (the
chatbot's indexing pipeline, an admin UI), URI versioning is usually the right call — it's boring,
debuggable, and doesn't require content-negotiation logic on either side.

## Upload-specific API design: multipart and async processing

A document upload endpoint has two design questions a generic CRUD tutorial doesn't cover:

**Multipart uploads.** File bytes plus metadata (filename, content type, tags) are typically sent as
`multipart/form-data` rather than JSON, because JSON can't efficiently carry binary data without
base64-inflating it by ~33%. `POST /documents` accepts a multipart body with a `file` part and a
`metadata` part (JSON string), and FastAPI parses each part separately via `UploadFile`/`Form` parameters
(see chapter 02).

**Async processing pattern.** Validating and storing a file is fast; extracting text, running OCR, or
chunking/embedding it for the search index (course 01's territory) is slow — sometimes tens of
seconds. Blocking the upload request until all of that finishes produces a bad client experience and
risks HTTP timeouts. The standard pattern is:

1. `POST /documents` validates the upload, stores the raw bytes, writes a metadata row with
   `status = "uploaded"`, and returns `201 Created` immediately — fast, synchronous, small surface
   area for bugs.
2. A separate event-driven process (an Azure Function triggered on blob creation — see chapter 04)
   picks up the file asynchronously, does the slow work, and updates the row's `status` to
   `"processing" -> "ready"` or `"failed"`.
3. The client polls `GET /documents/{id}` (or subscribes to a webhook/queue) to observe the status
   transition.

This "accept fast, process async, expose status" pattern is one of the most reusable ideas in backend
API design — it shows up anywhere an operation is too slow to fit inside a single request/response
cycle, and it's the natural bridge between this service's REST layer and the event-driven layer
covered in chapter 04.
