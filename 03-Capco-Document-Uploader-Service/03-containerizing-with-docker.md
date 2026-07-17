# 03 — Containerizing with Docker

## Why containerize a FastAPI service at all

The resume bullet says "Built and Implemented an end to end document uploader-ingest **Azure App
service**" — and the practical path from a FastAPI app on a laptop to "running reliably on Azure App
Service" is almost always Docker. Containerizing solves a specific, concrete problem: "works on my
machine" failures caused by different Python versions, different OS-level dependencies, or missing
environment configuration between a developer's laptop, the CI pipeline, and the App Service instance
in Azure. A Docker image is a single artifact — build it once, and the exact same bytes run in every
environment, with no "but I have Python 3.11 and prod has 3.9" surprises.

## Images vs. containers

These two terms get conflated constantly, and an interviewer will notice if you use them
interchangeably:

- An **image** is a read-only, immutable template — a filesystem snapshot plus metadata (entrypoint,
  exposed ports, environment defaults) built from a `Dockerfile`. It's the equivalent of a class
  definition.
- A **container** is a running (or stopped) *instance* of an image — an isolated process with its own
  filesystem view, network namespace, and resource limits, created from that image. It's the
  equivalent of an object instantiated from that class.

You build one image (`docker build -t doc-uploader:1.4.0 .`) and can run many containers from it
simultaneously (`docker run doc-uploader:1.4.0`, three times, for three replicas) — each container has
its own writable layer on top of the shared read-only image layers, so they don't interfere with each
other.

## Layers and build caching

A Docker image is built as a stack of **layers**, one per instruction in the Dockerfile (roughly).
Each layer is content-addressed and cached — if a layer's inputs haven't changed since the last build,
Docker reuses the cached layer instead of rebuilding it. This is the single biggest lever for fast,
cheap CI builds, and it's why **instruction order in a Dockerfile matters**:

```dockerfile
# Bad: any source change invalidates the pip install layer too
COPY . .
RUN pip install -r requirements.txt

# Good: dependency install only re-runs when requirements.txt changes
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

In the "good" version, editing `app.py` and rebuilding only re-executes the `COPY . .` layer and
anything after it — the (often slow) `pip install` layer stays cached because `requirements.txt`
didn't change. On a CI pipeline that rebuilds on every commit, this is the difference between a
20-second build and a 3-minute build.

## Multi-stage builds

A **multi-stage build** uses more than one `FROM` in a single Dockerfile, where later stages can
selectively copy artifacts from earlier stages while discarding everything else — build tools,
compilers, intermediate files, dev dependencies. The final image contains only what's needed to run
the app, not what was needed to build it.

```dockerfile
# ---- Stage 1: build dependencies (may need compilers, build tools) ----
FROM python:3.11-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ---- Stage 2: slim runtime image ----
FROM python:3.11-slim
WORKDIR /app

# Copy only the installed packages from the builder stage, not build tools
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

COPY . .

# Run as a non-root user — a basic security practice, and something
# App Service / container security scanners will flag if skipped
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000

# uvicorn (an ASGI server) running the FastAPI app object, not the dev-only `fastapi dev` command
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Why this matters for a document-ingest service specifically: the final runtime image doesn't need
`pip`'s build cache, compilers for any C-extension dependencies, or test fixtures — it needs a Python
runtime, the installed packages, and the app code. A smaller image means faster deploys to Azure App
Service (less to pull), a smaller attack surface, and lower cost if the registry charges by storage.

A note on the last `CMD` line: **never ship `fastapi dev` or `uvicorn --reload` (the auto-reloading
development mode) in production.** FastAPI is an ASGI application, so it needs an ASGI server —
`uvicorn` is the standard choice — rather than a WSGI server like plain `gunicorn`. Two production
shapes are both legitimate, and which one to pick is itself a reasonable interview question:

- **Plain `uvicorn` with multiple workers** — `uvicorn app.main:app --host 0.0.0.0 --port 8000
  --workers 4`. Simple, fewer moving parts, and enough for most single-container deployments where
  Azure App Service itself handles restarts and scaling out at the instance level.
- **`gunicorn` as a process manager, with `uvicorn.workers.UvicornWorker` as the worker class** —
  `gunicorn app.main:app -k uvicorn.workers.UvicornWorker --workers 4 --bind 0.0.0.0:8000`. `gunicorn`
  adds mature process-management features `uvicorn` alone doesn't provide as robustly — worker health
  checks, graceful worker restarts on memory growth, and battle-tested signal handling — while
  `uvicorn.workers.UvicornWorker` still gives each worker process full ASGI/async support. This is the
  more defensible choice for a production deployment with several workers per container, and matters
  directly for this service — a file upload endpoint that's blocked handling one slow multipart request
  while three other uploads sit in a queue is exactly the kind of problem multiple workers address,
  on top of the concurrency a single async worker already gets from `await`ing I/O.

## `requirements.txt` for this service

The dependency set the multi-stage build above installs reflects the actual stack (chapters 02, 05, 06):

```
fastapi
uvicorn[standard]
sqlalchemy>=2.0
alembic
pydantic
pydantic-settings
msal
pyjwt[crypto]          # or python-jose[cryptography] — either validates Azure AD-issued JWTs
python-multipart        # required by FastAPI/Starlette to parse multipart/form-data uploads
azure-identity
azure-keyvault-secrets
gunicorn                # only if running gunicorn as the process manager, see above
```

Pinning exact versions (not shown above for brevity) in the committed `requirements.txt` — rather than
loose ranges — is what makes the Docker layer-caching discussion above actually reliable: an unpinned
`fastapi` could resolve to a different version on every rebuild even when the file's *contents* look
unchanged to a casual reader, silently invalidating the "requirements.txt didn't change" assumption
the caching argument depends on.

## .dockerignore

Just like `.gitignore`, a `.dockerignore` file prevents unwanted files from being sent to the Docker
build context (and therefore from potentially ending up in a layer):

```
.git
__pycache__
*.pyc
.venv
.env
tests/
*.md
```

Excluding `.env` here is not optional — a `.env` file with real Key Vault access secrets or connection
strings baked into an image layer is a genuine security incident, since anyone who pulls the image
(or an old cached layer) can extract it. Secrets belong in Key Vault and App Service configuration,
never in the image (see chapter 05).

## Container registries and getting the image to Azure

A built image needs to live somewhere Azure App Service can pull it from — a **container registry**.
Azure Container Registry (ACR) is the natural choice for an Azure-hosted service: private by default,
integrates with Azure AD for access control, and sits in the same region as the App Service instance
for fast pulls.

The typical flow, usually wired into a CI pipeline rather than run by hand:

```bash
# Authenticate to the registry
az acr login --name myregistry

# Tag the locally built image for that registry
docker tag doc-uploader:1.4.0 myregistry.azurecr.io/doc-uploader:1.4.0

# Push it
docker push myregistry.azurecr.io/doc-uploader:1.4.0

# Point App Service at the new image (or let a CD pipeline do this)
az webapp config container set \
  --name doc-uploader-svc \
  --resource-group my-rg \
  --container-image-name myregistry.azurecr.io/doc-uploader:1.4.0
```

Tagging with a specific version (`1.4.0`) rather than always pushing `latest` matters operationally:
it makes deployments traceable ("which image is running in prod right now?") and rollback trivial
("point App Service back at `1.3.9`") — both of which are hard to do cleanly if every build overwrites
the same `latest` tag.

## How this maps back to the resume bullet

"Built and Implemented an end to end document uploader-ingest Azure App service" implies the candidate
owned this whole chain: writing the Dockerfile, getting local builds fast via layer caching, keeping
the production image lean with a multi-stage build, pushing to a registry, and configuring App Service
to run it. In an interview, being able to walk through *why* each Dockerfile line exists — not just
that it exists — is what separates "I copied a Dockerfile template" from "I understand what I
containerized and why." The next chapter (04) picks up exactly where this one ends: what happens once
that image is running on Azure App Service, and when a piece of this workload is better expressed as
an Azure Function instead.
