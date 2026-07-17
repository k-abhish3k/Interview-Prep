# 05 — Secrets Management with Key Vault, and SQL Integration

## The problem secrets management solves

A document-ingest service needs several credentials to function: a SQL connection string, a blob
storage account key (or connection string), possibly an API key for Azure Cognitive Services if OCR
enrichment is wired in, and possibly Microsoft Graph API credentials for org-directory lookups. The
naive approach — put them in environment variables set directly in App Service configuration, or worse,
in a `.env` file committed to the repo — creates real risk: anyone with read access to the App Service
resource (or the git history) can read plaintext secrets, rotating a leaked key means redeploying every
service that used it, and there's no audit trail of which secret was accessed when.

**Azure Key Vault** centralizes secret, key, and certificate storage behind Azure's access-control and
auditing model, so secrets are managed, rotated, and audited in one place instead of scattered across
every app's configuration.

## Key Vault references and managed identity

The naive-but-common mistake is to fetch a secret from Key Vault at app startup using *another*
secret (a Key Vault access key or service principal password) to authenticate to Key Vault — which
just moves the "where do I store the credential" problem one level up without solving it.

The actual solution is **managed identity**: Azure App Service (and Functions) can be assigned an
identity in Azure AD/Entra ID with no credential the developer ever sees or manages. That identity is
granted an **access policy** (or RBAC role) on the Key Vault, and the app authenticates to Key Vault
using that identity automatically — Azure's SDKs (`DefaultAzureCredential` in Python) handle the token
acquisition transparently.

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://doc-uploader-kv.vault.azure.net/", credential=credential)

sql_conn_string = client.get_secret("sql-connection-string").value
storage_conn_string = client.get_secret("blob-storage-connection-string").value
```

`DefaultAzureCredential` tries several authentication methods in order — managed identity first when
running in Azure, falling back to a developer's Azure CLI login when running locally — so the *same
code* works unmodified in a developer's laptop session and in production, with zero secrets in either
environment's source or config files.

An alternative, even more transparent pattern is a **Key Vault reference** directly in App Service
application settings:

```
SQL_CONNECTION_STRING = @Microsoft.KeyVault(SecretUri=https://doc-uploader-kv.vault.azure.net/secrets/sql-connection-string/)
```

App Service resolves this at runtime (again via managed identity) and injects the actual secret value
as a normal environment variable. This FastAPI service reads it through a Pydantic **`BaseSettings`**
class (from `pydantic-settings`), not a bare `os.environ[...]` lookup — settings become a typed,
validated object instead of untyped string access scattered through the codebase:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    sql_connection_string: str
    blob_storage_connection_string: str
    tenant_id: str
    client_id: str
    api_app_id_uri: str

    model_config = {"env_file": ".env"}  # local dev only — never present in the deployed image (chapter 03)

settings = Settings()  # reads SQL_CONNECTION_STRING etc. from the process environment
```

`Settings()` fails fast at startup — with a clear validation error naming the missing field — if any
required App Service application setting (Key-Vault-injected or otherwise) isn't present, instead of
the app booting successfully and then throwing an opaque `KeyError` the first time a request needs that
value. The application code doesn't even need the Key Vault SDK for this pattern, which is a simpler
mental model for a small service, at the cost of a little less flexibility than calling the SDK
directly (e.g., no easy mid-request secret refresh).

**Rotation.** Because nothing in the app hardcodes the secret's value, rotating it is a Key
Vault-side operation: update the secret (Key Vault versions it automatically), and either restart the
app (for the reference pattern) or let the next `get_secret` call pick up the new version (for the SDK
pattern, since `get_secret` without a version always returns the latest). No code change, no
redeploy — a meaningful operational win over hardcoded or environment-baked secrets, and a strong
answer if asked "how would you rotate a compromised credential in this system?"

### Managed Identity vs. "storing secrets in Key Vault directly" — the distinction interviewers probe

This is a common interview question because candidates often conflate the two. Storing a secret *in*
Key Vault solves "where does the secret live" (centralized, encrypted, audited) but doesn't by itself
solve "how does the app authenticate to Key Vault to retrieve it" — if that authentication itself
requires a hardcoded credential, you haven't eliminated the original problem, you've relocated it.
Managed identity solves the second half: the app's *identity* is what Azure manages, with no secret
material anywhere in application code or config. The combination — secrets centralized in Key Vault,
accessed via managed identity — is what actually removes hardcoded credentials from the system
end-to-end.

## SQL schema design for document metadata

The file bytes live in blob storage; SQL holds the metadata that makes those bytes discoverable,
queryable, and auditable. A realistic schema:

```sql
CREATE TABLE documents (
    id              BIGINT IDENTITY(1,1) PRIMARY KEY,
    filename        NVARCHAR(260)   NOT NULL,
    blob_path       NVARCHAR(1024)  NOT NULL,
    content_type    NVARCHAR(128)   NOT NULL,
    size_bytes      BIGINT          NOT NULL,
    status          NVARCHAR(32)    NOT NULL DEFAULT 'uploaded',
    -- 'uploaded' -> 'processing' -> 'ready' | 'failed'

    -- Audit columns
    created_by      NVARCHAR(128)   NOT NULL,
    created_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),

    -- Soft delete
    is_deleted      BIT             NOT NULL DEFAULT 0,
    deleted_at      DATETIME2       NULL,
    deleted_by      NVARCHAR(128)   NULL
);

-- Fast lookups for the common "list my active documents by status" query
CREATE INDEX idx_documents_status_created
    ON documents (status, created_at DESC)
    WHERE is_deleted = 0;

-- Fast lookup for a specific uploader's documents
CREATE INDEX idx_documents_created_by
    ON documents (created_by)
    WHERE is_deleted = 0;
```

**Why soft-delete, not `DELETE FROM documents WHERE id = ...`.** `DELETE /documents/{id}` in the REST
API (chapter 01) sets `is_deleted = 1` and `deleted_at = SYSUTCDATETIME()` rather than removing the
row. Two reasons this matters specifically for this service: regulated-industry clients (Capco's
typical engagement type) often require a retention/audit trail — "prove this document existed and was
deliberately removed by whom, on what date" is a real compliance question, and a hard-deleted row can't
answer it. Second, it makes recovery from accidental or erroneous deletes trivial (`UPDATE ... SET
is_deleted = 0`) instead of requiring a backup restore. Every `SELECT` in the application then filters
`WHERE is_deleted = 0` (enforced in the repository layer, not left to every call site to remember).

**Why the filtered/partial indexes.** `WHERE is_deleted = 0` on the indexes means the index only
covers active documents — which is almost every query the app actually runs (nobody lists deleted
documents in the hot path) — keeping the index smaller and faster than indexing the whole table
including rows nobody queries by status anymore.

**Audit columns.** `created_by`/`created_at`/`updated_at` (and the `deleted_*` equivalents) aren't
optional nice-to-haves in a service like this — they're what makes "who uploaded this and when" and
"who deleted this and when" answerable after the fact, which is exactly the audit trail this course's
`00-README.md` calls out as a client requirement for regulated engagements.

### Multi-tenancy: the `tenant_id` column, and why getting it wrong is a severe incident

This service didn't run for one client — it ran in production for **two** banking clients, HSBC and
Bank of America, on the same codebase and, for cost and operational-simplicity reasons, the same
underlying Azure SQL server. That means the schema above is incomplete as shown: every table needs a
`tenant_id` (or `client_id`) column, and every single query against those tables needs to filter on it.

```sql
CREATE TABLE documents (
    id              BIGINT IDENTITY(1,1) PRIMARY KEY,
    tenant_id       NVARCHAR(32)    NOT NULL,   -- 'hsbc' | 'bofa' — never client-supplied, always
                                                  -- resolved server-side from the authenticated identity
    filename        NVARCHAR(260)   NOT NULL,
    blob_path       NVARCHAR(1024)  NOT NULL,   -- tenant-scoped, e.g. 'hsbc/2026/07/15/<guid>.pdf'
    content_type    NVARCHAR(128)   NOT NULL,
    size_bytes      BIGINT          NOT NULL,
    status          NVARCHAR(32)    NOT NULL DEFAULT 'uploaded',

    created_by      NVARCHAR(128)   NOT NULL,
    created_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),

    is_deleted      BIT             NOT NULL DEFAULT 0,
    deleted_at      DATETIME2       NULL,
    deleted_by      NVARCHAR(128)   NULL
);

-- tenant_id leads every composite index — every real query filters by tenant first
CREATE INDEX idx_documents_tenant_status_created
    ON documents (tenant_id, status, created_at DESC)
    WHERE is_deleted = 0;

CREATE INDEX idx_documents_tenant_created_by
    ON documents (tenant_id, created_by)
    WHERE is_deleted = 0;
```

Two architecturally valid ways to get this isolation, with different cost/complexity trade-offs:

- **Shared database, `tenant_id` row-level enforcement** (the schema above). Cheaper to run and
  operate — one database, one set of migrations, one connection pool — but the isolation guarantee
  lives entirely in application code: every repository method must add `WHERE tenant_id = @tenant_id`,
  with no exceptions, and that discipline has to be enforced structurally (a base repository class that
  injects the filter automatically, or SQL Server **Row-Level Security** policies bound to the session
  context) rather than trusted to every developer remembering it on every query.
- **Separate databases or schemas per tenant** (`hsbc` schema, `bofa` schema, or fully separate Azure
  SQL databases). Stronger physical isolation — a bug in one query literally cannot cross tenants,
  because there's no shared table for it to leak across — at the cost of N times the schema migrations,
  connection management, and operational overhead as tenants grow. This is the choice a stricter
  compliance regime, or a client contract that explicitly demands physical separation, would push
  toward.

This service used the shared-database, `tenant_id`-enforced model, with the `tenant_id` always
resolved server-side from the caller's authenticated Azure AD identity — **never** accepted as a
client-supplied header or query parameter, since a client-supplied tenant ID is trivially spoofable and
would turn "isolation" into "isolation as long as nobody tries."

**Why this is a severe incident, not an ordinary bug, if it's wrong.** A missing `tenant_id` filter on
one query — a forgotten `WHERE` clause, a join that drops the filter, a cache key that doesn't include
it — means an HSBC employee's session can list, read, or search Bank of America's uploaded documents
(or vice versa). For most SaaS products a data-scoping bug is embarrassing; for two banking clients
sharing infrastructure, it's a **cross-tenant data leakage incident**: a confidentiality breach between
two named clients, almost certainly contractually and regulatorily reportable, and the kind of failure
that ends an engagement rather than generates a bug ticket. That severity is exactly why `tenant_id`
enforcement belongs at the repository layer (one chokepoint every query passes through) rather than
being re-implemented ad hoc at each call site, and why it's worth testing explicitly — a test suite
that asserts "a query scoped to tenant A never returns a row belonging to tenant B" regardless of the
filters passed in.

### The SQLAlchemy data-access layer: a `Document` model and a tenant-enforcing repository

Chapter 02 introduced SQLAlchemy as this service's ORM and why it was chosen over raw cursor calls; this
is where that choice pays off directly against the schema above. The `Document` model maps onto exactly
the DDL shown, including `tenant_id`, soft-delete, and audit columns:

```python
from sqlalchemy import String, BigInteger, Boolean, DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from datetime import datetime, timezone

class Base(DeclarativeBase):
    pass

class Document(Base):
    __tablename__ = "documents"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    tenant_id: Mapped[str] = mapped_column(String(32), nullable=False)  # 'hsbc' | 'bofa'
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

The point raised earlier — that `tenant_id` enforcement "has to be enforced structurally ... rather than
trusted to every developer remembering it on every query" — is exactly what a **base repository class**
gives you with SQLAlchemy: override the one method every query eventually calls, and inject the tenant
filter there instead of at every call site.

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

class TenantScopedRepository:
    """Base repository: every query built through `self.query()` is automatically
    scoped to the caller's tenant. Subclasses never need to remember the filter."""

    model = Document

    def __init__(self, db: Session, tenant_id: str):
        self.db = db
        self.tenant_id = tenant_id  # resolved server-side from the validated Azure AD token, chapter 06

    def query(self):
        # Every read/write path in this repository funnels through here —
        # a missing `tenant_id` filter becomes structurally impossible, not just discouraged.
        return select(self.model).where(
            self.model.tenant_id == self.tenant_id,
            self.model.is_deleted == False,
        )


class DocumentRepository(TenantScopedRepository):
    model = Document

    def get(self, doc_id: int) -> Document | None:
        return self.db.scalar(self.query().where(Document.id == doc_id))

    def list_page(self, status: str | None = None, cursor_created_at: datetime | None = None,
                  page_size: int = 20) -> list[Document]:
        stmt = self.query()
        if status:
            stmt = stmt.where(Document.status == status)
        if cursor_created_at:
            stmt = stmt.where(Document.created_at < cursor_created_at)
        stmt = stmt.order_by(Document.created_at.desc()).limit(page_size)
        return list(self.db.scalars(stmt).all())

    def soft_delete(self, doc_id: int, deleted_by: str) -> Document | None:
        doc = self.get(doc_id)
        if doc is None:
            return None
        doc.is_deleted = True
        doc.deleted_by = deleted_by
        doc.deleted_at = datetime.now(timezone.utc)
        self.db.commit()
        return doc
```

A path operation never calls `Document.__table__` or writes a `WHERE` clause itself — it constructs
`DocumentRepository(db, current_user.tenant_id)` (with `tenant_id` coming from the validated token,
chapter 06, never from client input) and calls `.get(...)`/`.list_page(...)`/`.soft_delete(...)`. The
test suite mentioned above — "a query scoped to tenant A never returns a row belonging to tenant B" —
becomes a test against this one class rather than against every route individually, which is exactly the
"one chokepoint" property that makes the isolation guarantee auditable in a code review instead of
merely aspirational.

### Parameterized queries — automatic with the ORM, non-negotiable either way

```python
# Never do this, ORM or not — string formatting into SQL is a direct SQL-injection vector,
# and SQLAlchemy's text() construct still supports naive interpolation if you go out of your way to misuse it
cursor.execute(f"SELECT * FROM documents WHERE created_by = '{user}'")

# SQLAlchemy's query-building API parameterizes by construction — there is no string-concatenation
# step for a caller-supplied value to hide in:
stmt = select(Document).where(Document.created_by == user)
db.scalars(stmt).all()
```

This is worth stating plainly and confidently in an interview: **every** query in this service that
incorporates a caller-supplied value (filter parameters, IDs, search terms) is built through
SQLAlchemy's query API rather than string-formatted SQL — no exceptions, no "just this one internal
admin query." That's the "reduced injection surface" argument from chapter 02 made concrete: it's not
that raw parameterized `cursor.execute("... = ?", (user,))` calls are unsafe (they aren't, done
correctly) — it's that the ORM's API removes the *unsafe pattern itself* from the space of things a
query can accidentally be written as. The escape hatch for a hand-tuned query on a hot path is
SQLAlchemy's `text()` with bound parameters (`text("... WHERE created_by = :user").bindparams(user=user)`)
— still parameterized, still safe, just closer to the metal when profiling shows the ORM-generated SQL
is actually the bottleneck.

### Pagination at the data-access layer

Chapter 01 covered cursor vs. offset pagination conceptually; expressed through the repository above
(`DocumentRepository.list_page`), cursor pagination is `.where(Document.created_at <
cursor_created_at).order_by(Document.created_at.desc()).limit(page_size)` — SQLAlchemy compiles that to
the same shape of SQL as the hand-written version:

```sql
SELECT TOP (@page_size) id, filename, status, created_at
FROM documents
WHERE tenant_id = @tenant_id AND is_deleted = 0
  AND created_at < @cursor_created_at   -- from the last row of the previous page
ORDER BY created_at DESC;
```

Because `idx_documents_tenant_status_created` already leads with `tenant_id` and orders by
`created_at DESC`, this query is an efficient index seek at any page depth, unlike
`OFFSET 100000 ROWS FETCH NEXT 50 ROWS ONLY`, which forces SQL Server to scan and discard the first
100,000 matching rows before returning anything — a property of the *index and query shape*, not
something the ORM changes one way or the other; SQLAlchemy generates the `WHERE`/`ORDER BY`/`LIMIT`
clauses, the query planner still does the same work it would against hand-written SQL.

## Microsoft Graph API: org-directory and permissions

A document-ingest service inside an enterprise client rarely operates in an identity vacuum — it
typically needs to know *who* the uploader is beyond a bare username, and *what* they're allowed to
do. **Microsoft Graph API** is Microsoft's unified REST API surface over Azure AD/Entra ID, Microsoft
365, and related services, and a plausible integration point here is resolving an authenticated user's
directory identity — their department, manager, or group memberships — to drive authorization
decisions (e.g., "only users in the `Compliance-Docs` security group may upload to the `compliance/`
category") or simply to enrich the `created_by` audit column with a display name rather than an opaque
object ID:

```python
# Simplified — real usage requires an acquired OAuth token with appropriate Graph scopes
import requests

resp = requests.get(
    "https://graph.microsoft.com/v1.0/me",
    headers={"Authorization": f"Bearer {graph_token}"},
)
profile = resp.json()  # displayName, department, mail, ...
```

This is the kind of integration that's plausible and common for an enterprise-internal service but
easy to over-claim in an interview — frame it as "a typical integration point for enforcing
org-based access control," not as a verified detail of what was built, unless you have a specific
memory of implementing it.

## How this maps back to the resume bullet

"CRUD functionality" implicitly promises a data layer that's correct under concurrent access
(parameterized queries, proper indexing) and defensible under a compliance review (soft-delete, audit
columns) — and "Azure App service" implicitly promises secrets aren't sitting in that app's
configuration in plaintext. This chapter is where those two implicit promises get made concrete, and
it's usually where the most pointed technical-deep-dive interview questions land, because it's where
real production services most often cut corners.
