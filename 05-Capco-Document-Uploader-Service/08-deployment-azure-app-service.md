# 08 — Deployment: Azure App Service (No Docker in Production)

## The real deployment model, stated plainly up front

**Production deployment of this service does not use Docker anywhere.** `FULL_ARCHITECTURE.md` §11 and
`DEPLOYMENT_REQUIREMENTS.md` both describe the same thing: an **Azure App Service (native Python
runtime)** running `uvicorn`/`gunicorn` directly against `app.py`, configured via Application Settings
(environment variables) — not a container, not Web App for Containers, not an image pulled from a
registry. `.env` is explicitly local-development-only (`load_dotenv()` only runs when
`IS_LOCAL_MACHINE` is true; chapter 07). If a prior version of this course described a multi-stage
Dockerfile and Azure Container Registry as this service's real deployment path, that was inaccurate —
correct the record on this before repeating it in an interview: **this service runs on App Service's
native Python runtime, no container involved.**

## What actually runs, per `FULL_ARCHITECTURE.md` §11 and §12

| # | Component | Role | Confirmed detail |
|---|---|---|---|
| 1 | **Azure App Service** | Hosts the FastAPI app | `uvicorn`/`gunicorn` running `app.py` directly; can scale to multiple instances on one App Service Plan; Application Settings = env vars |
| 2 | **Azure Active Directory** | Sign-in, OBO, client-credentials, role claims | See chapter 04 for the full layered picture |
| 3 | **Managed Identity** | Passwordless auth to Blob Storage | `DefaultAzureCredential`, RBAC role Storage Blob Data Contributor |
| 4 | **Azure Blob Storage** | IWPB staging only | Container `pending-iwpb-uploads`, multi-instance-safe by design |
| 5 | **Azure SQL Database** | `approver_mapping`, `iwpb_document_workflow` | Schema `uploader`, via `oaal.io.database.DBSession`/SQLAlchemy |
| 6 | **HSBC INM-AI Config Service** | Feature flags | Called via on-behalf-of token; endpoint URL varies by `ENVIRONMENT` (`PREPOC`/`POC`/prod) |
| 7 | **External Ingestion API (`INGEST_API`)** | System of record | Every department's documents ultimately land here |
| 8 | **Corporate SMTP Relay** | IWPB workflow emails only | Plain `smtplib`, no Microsoft Graph |
| 9 | **NAS source** | Upstream file source, FEMA-linked | Pulled into HEXA via `INGEST_API`'s `ingest/nas` endpoint |

There is no VNet/Private Endpoint claim, no Front Door/WAF claim, and no separate Azure Function app in
this real architecture — the background maintenance work (reminders, auto-removal, purge) runs as an
in-process `asyncio` task inside the same App Service process, not as a separate Azure Function (chapter
06 covers why, and its real not-a-singleton caveat). Don't describe this service as "App Service plus
Azure Functions for async post-upload processing" — that's a plausible architecture for a *different*
kind of document-ingest service, but it is not what this one actually does.

## App Service Plan scaling: real, and directly relevant to the background loop

`FULL_ARCHITECTURE.md` §12's deployment view shows an App Service Plan that **can** scale to multiple
instances (`I1`, `I2`, ...), each running its own copy of `app.py` and its own copy of
`_iwpb_maintenance_loop`. Two components behave differently under that scale-out, worth being precise
about (both covered in depth in chapter 06):

- **Blob Storage and Azure SQL are shared and multi-instance-safe by design** — any instance can serve
  an approve/decline request for a document staged by any other instance, because the staged bytes live
  in Blob Storage (not local disk) and the workflow state lives in Azure SQL (not in-process memory).
- **`memory_cache` (the bearer-token cache) is per-process and not shared** — a session's cached token
  only exists on the instance that handled that session's sign-in.

## The App Service Python runtime, without a container

Since there's no Dockerfile `CMD` dictating the startup command, App Service's native Python runtime
needs an explicit **startup command** telling it how to launch the ASGI app — the real equivalent of
what a `CMD` line would do inside a container:

```bash
# Set directly via the CLI...
az webapp config set \
  --name upload-service \
  --resource-group my-rg \
  --startup-file "gunicorn --bind 0.0.0.0:8000 -k uvicorn.workers.UvicornWorker app:app"

# ...configuring the Python version and runtime stack (no image to pull):
az webapp config set \
  --name upload-service \
  --resource-group my-rg \
  --linux-fx-version "PYTHON|3.11"
```

`gunicorn` as a process manager, with `uvicorn.workers.UvicornWorker` as the worker class, is the
production-appropriate choice discussed generically elsewhere in this curriculum — mature process
management (worker health checks, graceful restarts) on top of full ASGI/async support — and applies
here unchanged; the only real difference from a containerized deployment is *where* that startup command
lives (App Service configuration, not a Dockerfile `CMD` baked into an image).

Application Settings (chapter 07) supply every environment variable `app.py` reads via `os.environ`;
there's no image layer to rebuild when a config value changes — only an App Service restart.

## `/health` and the warm-up gate: the real health-check story

```python
@app.get("/health")
def health(request: Request):
    """Health endpoint with statistics for regular reporting to Azure."""
    ...
    return {
        "version": version_number,
        "environment": environment,
        "runtimeDuration": time_taken,
        "subscription": "INM",
    }
```

This is what Azure's health probes actually hit — a real endpoint reporting build version, environment,
and uptime, not a placeholder. Paired with it, `read_root()`'s warm-up gate (chapter 06) returns a plain
"still loading" response instead of an OAuth redirect for the first 180 seconds after process start,
specifically so a probe hitting `/` right after a cold start or a restart doesn't get redirected into an
auth flow it can't complete. Both of these are real, already-shipped patterns worth citing positively —
"how do you make a deployment health-check-friendly" has a concrete, source-confirmed answer here.

## Containerizing this as an alternative — a proposal, not reality

This is genuinely useful, transferable knowledge, and worth keeping — just clearly labeled as **a
proposal you could make, not a description of what runs today.** If asked "how would you containerize
this," or if you want to demonstrate Docker fluency in an interview using this project as the vehicle,
this is a legitimate answer to give, framed correctly: *"the real system runs on App Service's native
Python runtime with no container — but if I were proposing a change, here's how I'd containerize it and
why it might be worth doing."*

**Why you might propose it:** environment consistency (dev laptop, CI, and App Service running the exact
same image instead of three independently-configured Python runtimes), and a clean path to Web App for
Containers or, later, a fully different orchestrator (AKS, Container Apps) without re-architecting the
app itself.

**A multi-stage Dockerfile for this service, as a proposal:**

```dockerfile
# ---- Stage 1: install dependencies ----
FROM python:3.11-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ---- Stage 2: slim runtime image ----
FROM python:3.11-slim
WORKDIR /app

COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

COPY . .

RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "-k", "uvicorn.workers.UvicornWorker", "app:app"]
```

**What would need to change to actually adopt this:** move from `az webapp config set --startup-file`
to `az webapp config container set --container-image-name ...` (Web App for Containers instead of the
native runtime stack), stand up an Azure Container Registry, and wire a build/push step into whatever CI
pipeline deploys this service today. None of the application code changes — `app.py`'s reliance on
`os.environ` for configuration (chapter 07) already works identically whether those variables come from
a container's environment or App Service's native-runtime Application Settings, which is exactly the
property that makes this a low-risk proposal rather than a rewrite.

**The honest trade-off to name if asked "why isn't this containerized already":** for a single service
with no unusual OS-level dependencies and one deployment target (App Service), the native runtime stack
is simply less operational surface — no registry to manage, no image build step in CI, no image
vulnerability scanning pipeline to stand up. Containerizing pays off when you need environment
portability across multiple compute targets or a heterogeneous OS-dependency stack; this service, as
built, needed neither.

## Tying It Back

The resume bullet says "Azure App Service" — the real system takes that literally: a native Python
runtime process, no container, no separate Functions app for background work, configured entirely
through Application Settings. That's a simpler, more concrete deployment story than a containerized
microservices picture would be, and being able to state it plainly — while still being able to describe
containerizing it as a competent, clearly-labeled proposal — is a stronger interview answer than
defaulting to "we used Docker" because that's the more commonly expected answer for a FastAPI service.
