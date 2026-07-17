# 04 — Azure App Service and Functions Deployment

## Two Azure compute models, one service

The resume bullet mentions "Azure App service" explicitly, and the broader skills list includes
"Azure Functions" alongside it — and that pairing isn't a coincidence. A realistic document-ingest
service almost always needs **both** compute models, because it has two fundamentally different kinds
of work to do:

1. **Always-on, request/response work** — accept an HTTP upload, validate it, return a response fast.
   This is what Azure App Service is built for.
2. **Event-driven, bursty background work** — react to "a new blob just appeared in storage" and do
   something (extract text, call Cognitive Services, update a status). This is what Azure Functions is
   built for.

Understanding *why* you'd split the workload this way — not just that both services exist — is the
core of this chapter, and it's a very likely system-design question in an interview.

## Azure App Service: the always-on REST API

Azure App Service is a fully managed **Platform-as-a-Service (PaaS)** for hosting web apps and APIs.
You don't manage VMs, OS patching, or load balancer configuration — you give it a deployment artifact
(code or, in this service's case, a container image) and a scaling plan, and Azure handles the rest.

**Why it fits the FastAPI layer specifically:**

- It's designed for **long-running, always-listening processes** — exactly the shape of a FastAPI/ASGI
  app serving HTTP requests continuously.
- **Web App for Containers** lets you deploy the Docker image built in chapter 03 directly, rather
  than requiring App Service's native runtime stack — important since the service was containerized
  specifically for portability and environment consistency.
- It integrates natively with **managed identity** (chapter 05), **deployment slots**, and
  **autoscaling** — all of which a production ingest API needs.

### Deployment slots

A **deployment slot** is a separate, live instance of the app (with its own hostname) that shares the
same App Service Plan as production. The standard pattern:

1. Deploy the new image to a `staging` slot. It gets real infrastructure and its own URL
   (`doc-uploader-svc-staging.azurewebsites.net`), completely isolated from production traffic.
2. Run smoke tests / manual verification against the staging slot.
3. **Swap** staging and production. Azure performs a warm swap — the staging slot's app is already
   "warmed up" (its worker process has already started, avoiding cold-start latency), and the swap
   itself is close to instantaneous with no downtime.
4. If something's wrong post-swap, swap back — instant rollback, since the previous production code
   is now sitting in the (former) staging slot.

For a document-ingest API where downtime means uploads start failing, slots are the difference between
a deploy being a non-event and a deploy being a risk that needs a maintenance window.

### Scaling

App Service supports two scaling axes:

- **Scale up** (vertical) — move to a larger App Service Plan tier (more CPU/RAM per instance). Useful
  if individual requests are resource-heavy (e.g., synchronously processing very large files, which
  argues against doing it synchronously at all — see the async pattern in chapter 01).
- **Scale out** (horizontal) — run more instances behind the built-in load balancer, either manually or
  via **autoscale rules** (e.g., "add an instance when average CPU > 70% for 5 minutes, remove one when
  it drops below 30%"). Because the service is stateless (chapter 01's REST statelessness principle in
  practice — no session data lives in the FastAPI process, it's all in SQL/blob storage), any instance
  can serve any request, so scaling out is safe with zero extra coordination logic.

### Container deployment specifics

```bash
az webapp create \
  --name doc-uploader-svc \
  --resource-group my-rg \
  --plan my-app-service-plan \
  --deployment-container-image-name myregistry.azurecr.io/doc-uploader:1.4.0

az webapp config appsettings set \
  --name doc-uploader-svc \
  --resource-group my-rg \
  --settings WEBSITES_PORT=8000
```

`WEBSITES_PORT` tells App Service which port the container listens on internally (matching the
`EXPOSE 8000` / `uvicorn --port 8000` bind from the Dockerfile in chapter 03) — a common source of
"deployment succeeded but the app returns 502" bugs when it's misconfigured.

### Startup command, if not using a container image

If this service were deployed via App Service's native Python runtime stack instead of Web App for
Containers (less relevant here, since chapter 03 containerizes it, but a common interview question in
its own right), App Service Linux needs an explicit **startup command** telling it how to launch an
ASGI app — there's no `flask run` equivalent it can infer automatically:

```bash
# Set directly via the CLI ...
az webapp config set \
  --name doc-uploader-svc \
  --resource-group my-rg \
  --startup-file "gunicorn --bind 0.0.0.0:8000 -k uvicorn.workers.UvicornWorker app.main:app"

# ... or via a startup.sh script referenced the same way, useful when the startup
# sequence needs more than one command (e.g., running Alembic migrations first):
# startup.sh:
#   alembic upgrade head
#   gunicorn --bind 0.0.0.0:8000 -k uvicorn.workers.UvicornWorker --workers 4 app.main:app
```

The container-image path (this service's actual deployment model) avoids this entirely — the `CMD` in
the Dockerfile *is* the startup command, baked into the image rather than configured separately in App
Service settings, which is one more reason Web App for Containers was the right choice here over the
native runtime stack.

## Azure Functions: event-driven post-upload processing

Azure Functions is a **serverless / Functions-as-a-Service (FaaS)** compute model: you write a
function that reacts to a **trigger**, Azure runs it on-demand, and you don't provision or manage any
standing infrastructure — it scales from zero to many concurrent executions and back automatically,
and (on a consumption plan) you pay per execution rather than for idle time.

**Why it fits the post-upload work specifically:** once the FastAPI service stores a file in blob storage
and returns `201 Created` (chapter 01's async pattern), something needs to react to that new blob —
extract text, maybe call Azure Cognitive Services' Document Intelligence for OCR/structure extraction,
update the SQL row's status, and eventually hand the content to the indexing pipeline that feeds
course 01's chatbot. That work is inherently **event-driven and bursty** (a flood of 200 uploads at
9am, then nothing for hours), which is exactly the load pattern Functions is priced and scaled for —
paying for an always-on App Service instance to sit idle waiting for occasional blob events would be
wasteful.

```python
import azure.functions as func
import logging

app = func.FunctionApp()

@app.blob_trigger(arg_name="myblob", path="uploads/{name}",
                   connection="AzureWebJobsStorage")
def process_uploaded_document(myblob: func.InputStream):
    logging.info(f"Processing blob: {myblob.name}, size: {myblob.length} bytes")

    # 1. Extract text / run OCR (e.g., Azure Cognitive Services Document Intelligence)
    # 2. Update the SQL metadata row's status: "uploaded" -> "processing" -> "ready"/"failed"
    # 3. Hand off to the downstream indexing pipeline (course 01's territory)
```

This blob-trigger pattern is the natural glue between this service and the chatbot's knowledge base:
the FastAPI service doesn't need to know anything about OCR or indexing — it just writes a blob and a status
row, and the Function reacts independently. That decoupling is valuable on its own merits (each piece
can be developed, deployed, and scaled independently) and is a good example to cite if asked about
separation of concerns in a system-design question.

## App Service vs. Functions: when to choose which

| Dimension | Azure App Service | Azure Functions |
|---|---|---|
| Trigger model | Always listening (HTTP) | Event-driven (blob, queue, timer, HTTP) |
| Scaling | Manual or autoscale rules, instances stay warm | Scales to zero, near-instant scale-out on burst |
| Cost model | Pay for allocated plan, even when idle | Pay per execution (consumption plan) |
| Best for | Steady, latency-sensitive request/response (the upload API itself) | Bursty background work (post-upload processing) |
| Cold start | Minimal (with slots) | Possible on consumption plan, mitigated by Premium plan |
| State | Long-running process, can hold connection pools | Stateless, short-lived execution (typically < 10 min) |

The rule of thumb worth stating plainly in an interview: **if a human or client is waiting
synchronously for the response, use App Service; if the work is triggered by an event and nobody is
blocked waiting on it, use Functions.** This document-ingest service needs both precisely because it
has both kinds of work — accepting the upload (App Service) and reacting to it afterward (Functions).

## Production hardening for a banking client (HSBC / Bank of America)

Everything above is the baseline mechanics of App Service and Functions. What actually shipped for
HSBC and Bank of America — two banking clients whose cloud governance teams review this kind of
architecture line by line — layered several banking-grade requirements on top:

**Deployment slots weren't optional.** With bank staff uploading real documents against this service
every business day, a deploy that took the API down for even a few minutes was a real incident, not a
minor inconvenience. Every release went through the staging-slot-then-swap pattern described above,
specifically *because* "zero-downtime" was a stated requirement, not a nice-to-have — a failed swap
had a defined instant-rollback path, which is exactly the kind of operational detail a banking client's
change-approval process wants to see before signing off on a release cadence.

**VNet integration and Private Endpoints were non-negotiable.** A banking client's security review
does not accept "App Service, SQL, and Key Vault are reachable from the public internet, but they're
password-protected" as an answer. The App Service (and the Function app) were VNet-integrated, and
their dependencies — Azure SQL and Key Vault — were exposed only via **Private Endpoints** inside that
VNet, so there was no network path to them that didn't go through Azure's private backbone. The only
public-facing surface was Azure Front Door / Application Gateway with a WAF in front of it, terminating
TLS and filtering malicious traffic before it ever reached the App Service. This is the kind of detail
worth stating plainly in an interview: "the database and the secrets store were never reachable from
the public internet, full stop" is a much stronger answer than "we used a firewall."

**Scaling rules were sized to real, not toy, document volume.** Two banks' worth of daily upload
traffic has a real shape — batch-like spikes around business-day start and end, not a smooth trickle —
so autoscale rules on the App Service Plan were tuned to that pattern (scale out on sustained CPU/queue
pressure, scale back down off-peak rather than staying provisioned for peak 24/7), and the Function
app's concurrency/batch settings were tuned so a burst of uploads didn't overwhelm downstream SQL
connections (see chapter 05's connection-pooling note in the 10x system-design answer, Q11 in
`99-Interview-QA.md`).

**Why Functions' consumption-plan cold starts mattered here.** The blob-triggered Function sits behind
a customer-facing SLA indirectly: a document that lands in blob storage but takes 30+ seconds for a
cold Function instance to even start processing directly delays the moment a bank employee sees
`status: "ready"`. On a **consumption plan**, Functions scale to zero when idle, which is cost-efficient
but means the *first* invocation after an idle period pays a cold-start penalty — unacceptable when the
client's expectation is "documents process promptly, every time," not "usually promptly, except right
after a quiet period." The realistic fix is a **Premium plan** for the Function app (pre-warmed
instances, no cold start, still elastic scale-out under burst) — trading the consumption plan's
pay-per-execution economy for predictable latency, which is the right trade for a production,
customer-facing ingest path. Framed as a decision: *"we accepted a higher baseline Functions cost to
eliminate cold-start latency, because the SLA was customer-facing, not best-effort."*

## How this maps back to the resume bullet

The resume bullet's "Azure App service" is the front-door REST API — the part a client directly calls.
The Functions piece is implied by the broader skills list and is the more defensible, realistic
architecture for anything beyond a toy CRUD demo: a production ingest service almost certainly needs
*some* asynchronous post-upload step, and Azure Functions with a blob trigger is the idiomatic Azure
way to build that without standing up and managing a message queue and worker pool by hand. Framing it
this way — "App Service for the synchronous API, Functions for the event-driven follow-up" — is a
strong, concrete answer to "why did you choose this architecture?"
