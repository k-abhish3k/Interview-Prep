# 02 — Building the Service with FastAPI and SQLAlchemy

> **Grounding note.** The path-operation and dependency-injection patterns below are how the real
> service is built. The `Document`/`DocumentOut` worked example later in this chapter is a **teaching
> model** used to demonstrate the mechanics cleanly — the real service has no generic `documents` table
> like it (see chapter 01's grounding note). The section "The Real Models" replaces it with the actual
> two tables that exist in the codebase today: `ApproverMapping` and `IWPBDocumentWorkflow`.

## Why FastAPI for a service like this

This service was built with **FastAPI**, an ASGI (Asynchronous Server Gateway Interface) web
framework built on top of **Starlette** (routing/request handling) and **Pydantic** (data validation).
Three things make it a strong fit for a document-ingest microservice specifically:

- **Native async support.** A document upload endpoint spends most of its time *waiting* — on a blob
  storage write, a SQL insert, a Key Vault secret fetch, a Microsoft Graph API call to resolve the
  caller's identity. Those are all I/O-bound operations, not CPU-bound ones. An `async def` path
  operation that `await`s each of those calls lets a single worker process handle many concurrent
  requests while they're all waiting on I/O, instead of one thread being blocked per in-flight request.
  For a service whose entire job is "accept a file, then talk to three or four other services about
  it," this matters a lot more than it would for a CPU-bound endpoint.
- **Pydantic models as the validation layer, built in.** Rather than bolting on a schema library
  (`marshmallow`) as a separate step, FastAPI uses Pydantic models directly as function type
  annotations — the framework validates the request body against the model before your code runs, and
  serializes the response against a second model on the way out. Invalid input never reaches your
  business logic; it's rejected with a structured `422 Unprocessable Entity` automatically.
- **Automatic OpenAPI documentation.** Because path operations, Pydantic models, and dependencies are
  all typed, FastAPI generates a live, interactive OpenAPI schema (`/docs`, `/redoc`) with zero extra
  work. For an internal service consumed by other systems inside a bank — an admin UI, the chatbot's
  indexing pipeline, potentially other backend teams — that's a real integration accelerant: a client
  team can explore and test the API without a hand-written integration doc going stale.
- **A dependency-injection system that fits this service's shape exactly.** `Depends(...)` is how
  FastAPI wires shared logic — a database session, the current authenticated user, a role check — into
  path operations declaratively. Chapter 06 leans on this heavily for RBAC; this chapter introduces the
  pattern with the more familiar case of injecting a database session.

## Path operations: the FastAPI equivalent of a Flask route

FastAPI maps a URL and HTTP method to a Python function via a decorator on an `APIRouter` (the rough
equivalent of a Flask blueprint):

```python
from fastapi import APIRouter, HTTPException

router = APIRouter(prefix="/v1/documents", tags=["documents"])

@router.get("/{doc_id}")
async def get_document(doc_id: int):
    doc = await find_document(doc_id)
    if doc is None:
        raise HTTPException(status_code=404, detail="not found")
    return doc
```

Two differences from the Flask shape worth naming explicitly in an interview: the path parameter's
type (`doc_id: int`) is a plain Python type annotation, not a URL converter syntax — FastAPI validates
and coerces it the same way it validates request bodies, and an invalid value produces a `422`
automatically. And errors are raised as exceptions (`HTTPException`), not returned as
`(body, status_code)` tuples — FastAPI's exception handling middleware converts them to the right JSON
response, which keeps a route's "happy path" return type clean (it only ever returns the success shape).

## Pydantic models: request and response validation

A Pydantic model declares the *shape* of data — field names, types, which fields are required, which
have defaults or constraints — and FastAPI uses it in both directions:

```python
from pydantic import BaseModel, Field
from typing import Optional
from datetime import datetime

class DocumentCreate(BaseModel):
    filename: str = Field(min_length=1, max_length=260)
    tags: list[str] = Field(default_factory=list)

class DocumentOut(BaseModel):
    id: int
    filename: str
    status: str
    size_bytes: int
    created_by: str
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}  # lets this model read straight off a SQLAlchemy object

@router.post("", response_model=DocumentOut, status_code=201)
async def create_document(metadata: DocumentCreate):
    ...
```

`metadata: DocumentCreate` as a parameter type is the whole validation layer — a request body that's
missing `filename`, or has a `tags` field that isn't a list of strings, never reaches the function body;
FastAPI returns `422` with a structured error listing exactly which field failed and why. On the way
out, `response_model=DocumentOut` filters and validates the return value against that shape — so even
if the underlying SQLAlchemy object has extra internal fields (an `idempotency_key`, an ORM
relationship), only the fields declared on `DocumentOut` are ever serialized to the client. That's a
meaningful security property, not just a convenience: it's structurally impossible to accidentally leak
a field you forgot to explicitly exclude, because the response model is an allowlist, not a denylist.

## Dependency injection: the pattern this service leans on constantly

`Depends()` tells FastAPI "before running this path operation, call this function (or class) and pass
its return value in as an argument." It's used for anything a path operation needs but shouldn't
construct itself — a database session, the current authenticated caller, a pagination object:

```python
from fastapi import Depends
from sqlalchemy.orm import Session

def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.get("/{doc_id}", response_model=DocumentOut)
async def get_document(doc_id: int, db: Session = Depends(get_db)):
    doc = db.get(Document, doc_id)
    if doc is None or doc.is_deleted:
        raise HTTPException(status_code=404, detail="not found")
    return doc
```

`get_db` is a **generator dependency**: the code before `yield` runs before the path operation, the
`yield`ed value is what gets injected, and the code after `yield` runs after the response is sent —
which is exactly the right place to close the session whether the request succeeded or raised. This
same mechanism is what chapter 06 uses for authentication and RBAC: `current_user: User =
Depends(get_current_user)` and `_: None = Depends(require_role("DocumentUploader.Write"))` are the same
pattern applied to "who is calling" and "are they allowed to do this," composed independently of the
database-session dependency and of each other.

## Why SQLAlchemy ORM instead of raw SQL cursor calls

This service's data-access layer uses **SQLAlchemy**, not `pyodbc`/`cursor.execute()` calls directly
against Azure SQL. Three concrete reasons, not just "ORMs are more modern":

- **Maintainability.** A `Document` model declared once, in one place, with typed columns, is the
  single source of truth for the table's shape — every place in the codebase that touches a document
  works against that model, instead of every hand-written query independently getting the column list
  and types right (and silently drifting out of sync with the actual schema over time).
- **Migration support via Alembic.** SQLAlchemy's companion migration tool, **Alembic**, diffs the
  declared models against the database's current state and generates a versioned migration script —
  `alembic upgrade head` applies it, `alembic downgrade -1` reverses it. Without an ORM, schema changes
  are hand-written `ALTER TABLE` scripts with no tooling enforcing that dev, staging, and production
  databases actually converge on the same schema history.
- **Reduced injection surface.** SQLAlchemy's query API (`select(Document).where(Document.tenant_id ==
  tenant_id)`) builds parameterized SQL under the hood by construction — there is no string-formatting
  step where a caller-supplied value could end up concatenated into SQL text. Raw cursor code is *safe*
  when every query correctly uses `?`/`@param` placeholders, but that safety depends on every developer
  getting every query right, every time; the ORM's query-building API removes the vulnerable pattern
  from the space of things you can accidentally write.

The honest caveat, worth stating in an interview rather than glossing over: an ORM is not free. A
hand-tuned raw query can still outperform the SQL an ORM generates for the hottest, most
performance-critical paths, and SQLAlchemy's `text()` construct is the escape hatch for exactly that
case — you drop to raw parameterized SQL (still parameterized, still safe) for a specific query without
giving up the ORM everywhere else. For this service's access patterns (CRUD on a metadata table,
filtered/paginated lists), the ORM's generated SQL is not the bottleneck; the bottleneck, if any, is
almost always missing indexes, which chapter 05 covers independent of ORM vs. raw SQL.

## SQLAlchemy fundamentals: declarative models, `Session`, relationships

A **declarative model** maps a Python class to a database table:

```python
from sqlalchemy import String, BigInteger, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from datetime import datetime, timezone

class Base(DeclarativeBase):
    pass

class Document(Base):
    __tablename__ = "documents"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    tenant_id: Mapped[str] = mapped_column(String(32), nullable=False)
    filename: Mapped[str] = mapped_column(String(260), nullable=False)
    blob_path: Mapped[str] = mapped_column(String(1024), nullable=False)
    content_type: Mapped[str] = mapped_column(String(128), nullable=False)
    size_bytes: Mapped[int] = mapped_column(BigInteger, nullable=False)
    status: Mapped[str] = mapped_column(String(32), nullable=False, default="uploaded")

    created_by: Mapped[str] = mapped_column(String(128), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=lambda: datetime.now(timezone.utc))
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=lambda: datetime.now(timezone.utc),
                                                  onupdate=lambda: datetime.now(timezone.utc))

    is_deleted: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    deleted_by: Mapped[str | None] = mapped_column(String(128), nullable=True)
```

`Mapped[int]` / `mapped_column(...)` is SQLAlchemy 2.0's typed declarative style — the Python type
annotation and the column definition live together, so a type checker and the ORM agree on what a
`Document.id` actually is. This model maps onto exactly the same `documents` table schema covered in
chapter 05 (including `tenant_id`, soft-delete, and audit columns) — the ORM doesn't replace that
schema design, it's the Python-side representation of it.

**`Session` and `sessionmaker`.** A `Session` is the ORM's unit-of-work object — it tracks objects
you've loaded or modified and flushes changes to the database as a transaction. `sessionmaker` is a
factory that produces configured `Session` instances:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(sql_connection_string, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)
```

`pool_pre_ping=True` matters specifically for this service's traffic pattern: Azure SQL can silently
drop idle connections, and without pre-ping, the first query on a stale connection fails outright
instead of the pool quietly reconnecting — a real production symptom on a service with bursty upload
traffic followed by idle periods (chapter 04's cold-start discussion covers the analogous issue on the
Functions side).

**Relationship mapping.** If this service tracked, say, per-document processing events as a separate
table, SQLAlchemy expresses the one-to-many relationship declaratively rather than requiring a manual
join in every query that needs both:

```python
class ProcessingEvent(Base):
    __tablename__ = "processing_events"
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    document_id: Mapped[int] = mapped_column(ForeignKey("documents.id"))
    stage: Mapped[str] = mapped_column(String(32))  # "uploaded" -> "processing" -> "ready"/"failed"
    occurred_at: Mapped[datetime] = mapped_column(DateTime, default=lambda: datetime.now(timezone.utc))

    document: Mapped["Document"] = relationship(back_populates="events")

# on Document:
# events: Mapped[list["ProcessingEvent"]] = relationship(back_populates="document")
```

`doc.events` then lazily (or eagerly, with `selectinload`/`joinedload` when you want to avoid N+1
queries) loads the related rows as Python objects — the join is generated by the ORM, not hand-written
at every call site that needs it.

## The Real Models: `ApproverMapping` and `IWPBDocumentWorkflow`

This is where the real codebase (`src/uploader/tables/uploader_table.py`, `src/uploader/tables/base.py`)
departs from a textbook `Document` model. Two things are worth naming immediately:

**1. A custom, schema-scoped `Base`, not the bare `DeclarativeBase` from the section above:**

```python
# src/uploader/tables/base.py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    schema = "uploader"
    __table_args__ = {"schema": schema}
```

Every table in this service lives inside a dedicated `uploader` schema inside the shared Azure SQL
database — not the default `dbo` schema — declared once on the shared `Base` rather than repeated on
every model. `app.py`'s startup (`lifespan()`) creates that schema if it doesn't exist
(`sqla.schema.CreateSchema(schema)`) before creating any tables in it.

**2. The two real tables, using the older `sqla.Column(...)` declarative style rather than the typed
`Mapped[...]`/`mapped_column(...)` style shown earlier in this chapter** (both are valid SQLAlchemy —
this codebase predates a migration to the newer style):

```python
# src/uploader/tables/uploader_table.py
import sqlalchemy as sqla
from uploader.tables.base import Base

class ApproverMapping(Base):
    __tablename__ = "approver_mapping"

    approver_details_id = sqla.Column(sqla.String(36), primary_key=True)
    submitted_by = sqla.Column(sqla.String)
    business_line = sqla.Column(sqla.String)
    approver1_email_id = sqla.Column(sqla.String)
    approver2_email_id = sqla.Column(sqla.String)
    status = sqla.Column(sqla.String)          # "ACTIVE" / "INACTIVE"


class IWPBDocumentWorkflow(Base):
    """
    Per-document approval/ingestion lifecycle for IWPB only. Every other
    business line has no equivalent local row -- its documents live
    entirely inside the external INGEST_API/HEXA system of record.

    `status` is one of:
        PENDING_APPROVAL, APPROVED, DECLINED, AUTO_REMOVED, PURGED
    """
    __tablename__ = "iwpb_document_workflow"

    workflow_id = sqla.Column(sqla.String(36), primary_key=True)
    parent_document_id = sqla.Column(sqla.String, nullable=True)   # set only once approved

    document_title = sqla.Column(sqla.String)
    file_ext = sqla.Column(sqla.String)
    classification = sqla.Column(sqla.String)
    business_line = sqla.Column(sqla.String)

    uploader_email = sqla.Column(sqla.String)
    upload_time = sqla.Column(sqla.DateTime)
    expiry_date = sqla.Column(sqla.String)

    stored_file_path = sqla.Column(sqla.String, nullable=True)     # cleared once approved/declined/removed
    stored_file_name = sqla.Column(sqla.String, nullable=True)
    stored_content_type = sqla.Column(sqla.String, nullable=True)

    status = sqla.Column(sqla.String, default="PENDING_APPROVAL")
    email_sent = sqla.Column(sqla.Boolean, default=False)

    approver_name = sqla.Column(sqla.String, nullable=True)
    approval_time = sqla.Column(sqla.DateTime, nullable=True)
    decline_reason = sqla.Column(sqla.String, nullable=True)

    removed_time = sqla.Column(sqla.DateTime, nullable=True)
    purge_time = sqla.Column(sqla.DateTime, nullable=True)

    reminder_count = sqla.Column(sqla.Integer, default=0)
    last_reminder_time = sqla.Column(sqla.DateTime, nullable=True)

    created_at = sqla.Column(sqla.DateTime)
    updated_at = sqla.Column(sqla.DateTime)
```

`ApproverMapping` is not IWPB-exclusive as a *table* — every department's approver-admin panel
(`/save-approver-details`, `/fetch-approver-details`) reads and writes it. Only its use for gating
Approve/Decline actions is IWPB-specific. `IWPBDocumentWorkflow` is populated exclusively when
`use_case == "IWPB"`; no other department has a row anywhere in this schema for its documents.

### `Base.metadata.create_all(checkfirst=True)`: schema auto-creation, and the gap it leaves

`app.py`'s `lifespan()` startup hook runs this on every boot:

```python
Base.metadata.create_all(bind=DBSession.engine(), checkfirst=True)
```

`checkfirst=True` means "create any table that doesn't already exist; leave existing tables alone." This
is genuinely convenient — a fresh environment gets its schema for free on first boot, with **no manual
migration step, and no Alembic** anywhere in this codebase. The real, honest gap this leaves: `create_all`
only ever *adds* new tables. It has no concept of altering an existing table — add a column to
`IWPBDocumentWorkflow` in code, and `create_all` silently does nothing about the already-existing
database table on the next boot, because the table already exists and `checkfirst` skips it entirely.
In practice that means any real schema change to either table today would need a manual `ALTER TABLE`
run against the production database out-of-band, with nothing in the codebase enforcing that dev,
staging, and prod actually converge on the same shape. If asked "how would you evolve this schema
safely," the honest, defensible answer is: **introduce Alembic** — `alembic revision --autogenerate`
diffs the declared models against the live database and produces a versioned, reviewable migration
script (`alembic upgrade head` applies it, `alembic downgrade -1` reverses it) — rather than continuing
to rely on `create_all`'s "only ever additive, only for brand-new tables" behavior once the schema needs
to change under an existing, populated table.

## A worked example (teaching model): `Document` CRUD router

The `Document`/`DocumentOut` router below is **not from the real codebase** — it's a clean, generic
teaching example that demonstrates the FastAPI + Pydantic + SQLAlchemy mechanics (path operations,
dependency injection, response models) in the more familiar shape of a classic CRUD resource, since the
real `IWPBDocumentWorkflow` model above is entangled with the approver-workflow logic covered in full in
chapter 05. Use this to internalize the mechanics; use the section above when asked specifically about
this service's real data layer.

Putting the pieces together — the same shape as the old Flask blueprint, expressed with FastAPI path
operations, Pydantic request/response models, and a dependency-injected SQLAlchemy session (the full
runnable version, using an in-memory SQLite database so it needs no external dependencies, is in
`notebooks/01_fastapi_crud_api_demo.ipynb`). Note this teaching example uses a generic SaaS `tenant_id`
column to demonstrate row-scoping — that's a standard multi-tenant CRUD pattern worth knowing in
general, but it is **not** this real service's isolation axis. This service's real "who can see what"
boundary is **business_line/use_case** (IWPB vs. FEMA vs. TPMB vs. GTRM vs. generic), enforced via the
feature-flag-gated RBAC in chapter 04 and the department routing in chapter 03, not a `tenant_id`
column — there's no multi-bank or multi-client tenancy in this specific service's real design:

```python
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File
from sqlalchemy.orm import Session
from sqlalchemy import select

router = APIRouter(prefix="/v1/documents", tags=["documents"])

@router.post("", response_model=DocumentOut, status_code=201)
async def create_document(
    file: UploadFile = File(...),
    db: Session = Depends(get_db),
    current_user: "AuthenticatedUser" = Depends(get_current_user),   # chapter 06
    _: None = Depends(require_role("DocumentUploader.Write")),        # chapter 06
):
    content = await file.read()
    doc = Document(
        tenant_id=current_user.tenant_id,
        filename=file.filename,
        blob_path=await upload_to_blob_storage(content, file.filename, current_user.tenant_id),
        content_type=file.content_type,
        size_bytes=len(content),
        status="uploaded",
        created_by=current_user.email,
    )
    db.add(doc)
    db.commit()
    db.refresh(doc)
    return doc

@router.get("", response_model=list[DocumentOut])
async def list_documents(
    status: str | None = None,
    page: int = 1,
    page_size: int = 20,
    db: Session = Depends(get_db),
    current_user: "AuthenticatedUser" = Depends(get_current_user),
):
    stmt = select(Document).where(
        Document.tenant_id == current_user.tenant_id,   # tenant filter — see chapter 05's repository pattern
        Document.is_deleted == False,
    )
    if status:
        stmt = stmt.where(Document.status == status)
    stmt = stmt.order_by(Document.created_at.desc()).offset((page - 1) * page_size).limit(page_size)
    return db.scalars(stmt).all()

@router.get("/{doc_id}", response_model=DocumentOut)
async def get_document(doc_id: int, db: Session = Depends(get_db),
                        current_user: "AuthenticatedUser" = Depends(get_current_user)):
    doc = db.scalar(
        select(Document).where(Document.id == doc_id, Document.tenant_id == current_user.tenant_id,
                                Document.is_deleted == False)
    )
    if doc is None:
        raise HTTPException(status_code=404, detail="not found")
    return doc

@router.delete("/{doc_id}", status_code=204)
async def delete_document(
    doc_id: int,
    db: Session = Depends(get_db),
    current_user: "AuthenticatedUser" = Depends(get_current_user),
    _: None = Depends(require_role("DocumentUploader.Write")),
):
    doc = db.scalar(
        select(Document).where(Document.id == doc_id, Document.tenant_id == current_user.tenant_id,
                                Document.is_deleted == False)
    )
    if doc is None:
        raise HTTPException(status_code=404, detail="not found")
    doc.is_deleted = True
    doc.deleted_by = current_user.email
    doc.deleted_at = datetime.now(timezone.utc)
    db.commit()
```

This is the version worth being able to sketch on a whiteboard: five path operations, one `Document`
model, `Depends(get_db)` injecting a session instead of a global connection, `Depends(get_current_user)`
and `Depends(require_role(...))` injecting authentication and authorization independently of the
business logic, and the `tenant_id` filter present on every single query — which is exactly the
separation of concerns an interviewer is listening for when they ask "how would you test this without a
real database, and how do you know two tenants can't see each other's documents?"

## Tying It Back

The resume bullet's "CRUD functionality" is really satisfied by the **real** `IWPBDocumentWorkflow`
lifecycle (create on upload, read for the waiting/ingested lists, update on approve/decline/purge) — the
teaching `Document` router above demonstrates the same FastAPI/SQLAlchemy mechanics more generically.
"End to end" is satisfied by everything a real path operation composes via `Depends()` or direct calls
into `iwpb_workflow`: a database session (this chapter), an authenticated identity and role/approver
check (chapter 04), and, upstream of all of it, environment configuration read from App Service
Application Settings (chapter 07 — Key Vault is a proposed hardening step for this service today, not a
confirmed part of it). FastAPI's dependency-injection system is what makes that composition explicit and
independently testable rather than tangled together inside each route function — which is a stronger,
more concrete answer to "how is this code organized?" than "it's a
Flask app with some helper functions."
