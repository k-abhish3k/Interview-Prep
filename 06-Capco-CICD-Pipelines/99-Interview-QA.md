# 99 — Interview Q&A: CI/CD Pipeline Implementation

Read this last, once chapters 01-07 and the notebooks are fresh. Questions are grouped behavioral →
technical deep-dive → system design → retrospective. Practice the behavioral ones out loud; for the
technical ones, focus on the *reasoning*, not memorizing the answer verbatim — follow-up questions
will probe whether you actually understand the tradeoff. **Q1 is the gotcha question worth having
fully internalized before anything else in this file** — it comes first, deliberately, the same way
course 05's document-versioning question leads that course's Q&A.

---

## Q1. If you roll back one of these three services, how do you make sure the environment doesn't end up in an inconsistent combination of versions?

This is the gotcha question. The full worked answer lives in chapter 06
(`06-multi-service-version-pinning-and-environment-drift.md`) — this is the compressed,
interview-ready version:

> "Each of the three services — chatbot, monitoring, uploader — has its own pipeline, its own image
> versioning, and its own rollback mechanics, and each of those is solid in isolation: a slot-swap
> rollback for any one of them is fast and low-risk on its own terms. What's missing is a shared record
> of which combination of all three versions is actually known to work together — today, nothing stops
> you from rolling back the chatbot to a version whose response schema the monitoring platform's
> evaluation pipeline no longer understands, because the chatbot's own smoke test only checks the
> chatbot's own contract, not what monitoring currently expects.
>
> Concretely: say the chatbot ships v58 with a formatting regression and gets rolled back to v55 — a
> fast, routine slot-swap-back that reports success because v55's own contract is intact. But
> `citations[]` was introduced in v56 specifically because monitoring v41's groundedness scoring reads
> that field. Monitoring is untouched by the rollback — still v41, still expecting `citations[]` — so
> every conversation the rolled-back chatbot produces from that point is missing a field the monitoring
> pipeline depends on, and depending on how defensively that parsing was written, the model-risk
> dashboard's hallucination-rate trend line can quietly spike for a window that's actually a
> version-compatibility artifact, not a real quality regression. The chatbot's own pipeline reported a
> clean, successful rollback the entire time — it had no way to know it had just broken something two
> services away.
>
> The manual stopgap today is a compatibility matrix and a `/version`-endpoint cross-check across all
> three services — the pieces exist, they're just advisory, not enforced, and they typically run after
> the fact rather than blocking anything. The design I'd actually build is a release manifest — a small,
> versioned artifact per client recording the current triple of service versions — validated against a
> compatibility matrix as the last stage of a platform-level release pipeline that runs after each
> service's own pipeline, and that gate applies to rollbacks exactly the same way it applies to forward
> deploys, so a bad combination gets caught before it serves traffic instead of after someone notices
> the dashboard looks wrong. And the fix doesn't stop at CI/CD — the monitoring platform itself needs
> every conversation event tagged with the chatbot version that produced it, so it can interpret a
> version boundary correctly instead of quietly averaging incompatible data into one trend line."

**The precise distinction worth drawing under follow-up pressure:** "this service's own rollback
succeeded" (its own build/test/deploy/smoke-test pipeline went green) is a completely different question
from "the resulting platform-wide combination of all three services' versions is one that's actually
safe" — the first is what chapter 04's rollback mechanics answer, the second is a combinatorial question
no single service's pipeline can answer on its own.

**Follow-ups to be ready for (drawing on chapter 07):**

- *"How would a bug like that actually get caught in your pipeline today, before this fix exists?"* —
  Only after the fact, and only if someone notices — the honest answer is that a false-positive-looking
  clean rollback is exactly the kind of failure chapter 07 calls out as the most dangerous class: it
  looks identical to success. The closest thing to an early-warning signal today is a manual
  `/version`-endpoint check, run on a schedule, not release-blocking.
- *"What's the actual failure mode if two of these services' rollout pipelines try to update the release
  manifest at the same time?"* — A real concurrency trap, covered in chapter 07: without an
  optimistic-concurrency check on the manifest write (version the manifest, write only if it hasn't
  changed since you read it), the second writer can silently overwrite the first writer's update with a
  manifest computed from stale data — the same class of problem Terraform's own state locking exists to
  prevent, just one layer up.
- *"Is this the same problem as Terraform state drift?"* — No, and it's worth drawing that line
  precisely: state drift (chapter 07's bug narrative 2) is infrastructure Terraform believes exists no
  longer matching what's actually deployed; the version-compatibility gap here is about *application*
  versions across independently-deployed services being mutually incompatible even though each one's own
  infrastructure and deploy are perfectly healthy. Related failure family — both are "environment
  drift" — but different mechanisms, different fixes, and conflating them under interview pressure is
  exactly the mistake worth avoiding.

---

## Behavioral (STAR-based)

### Q2. Tell me about a time you automated a manual deployment process.

**A.** *(STAR — same shape as the course's main summary; practice this version under 90 seconds.)*
**Situation:** At Capco, the chatbot assistant and document-uploader services I built were being
deployed manually — building a Docker image locally and updating Azure App Service settings by hand,
which was slow and inconsistent between environments. **Task:** Automate the build-and-release
process so changes moved from a merged pull request to a running environment reliably. **Action:** I
built Azure DevOps YAML pipelines with build/lint/test stages, a containerization stage pushing
versioned images to Azure Container Registry, and a Terraform-driven deploy stage with a smoke test
after every deployment, gating production behind manual approval. **Result:** *(Illustrative)*
Deployment time for a routine change dropped from about half a day to under 15 minutes, and failed
deployments dropped noticeably once tests and smoke checks were gating every release automatically
instead of relying on manual verification.

### Q3. Describe a deployment that went wrong. What did you do?

**A.** Frame a plausible, honest-sounding scenario rather than claiming a specific proprietary
incident: a deploy passed all automated checks but a downstream client-specific configuration value
(e.g., a Key Vault secret name that differed between environments) wasn't parameterized correctly,
causing the service to fail to start in staging. The response that matters here isn't "it never
happened" — it's the recovery mechanics: the deployment slot swap made rollback to the previous
version a one-command operation with no user-facing downtime, and the follow-up action was adding
that class of config to the smoke test (a `/health` check that also validates required config is
present) so the same failure mode is caught automatically next time, not just fixed once.

### Q4. How did you decide what to automate first when you started building the pipeline?

**A.** Start with the highest-frequency, highest-risk manual step — for these services, that was the
build-and-deploy cycle itself, since it happened on every change and every manual deploy was a chance
for environment drift. I prioritized: (1) automated build+test on every PR, since that gave the
fastest feedback loop for the least effort; (2) automated deploy to dev, to remove the most frequent
manual toil; (3) Terraform for the underlying infrastructure, once the App Service configuration
needed to be reproduced for a second environment and doing it by hand a second time was clearly not
sustainable; (4) the approval-gated prod stage last, since it needed the first three pieces trustworthy
before a client would be comfortable with prod deploys running through it at all.

---

## Technical Deep-Dive

### Q5. How do you handle secrets in a CI/CD pipeline?

**A.** Secrets never live in the YAML file or in application config committed to Git. In Azure
DevOps: secrets go into **variable groups** (optionally backed by Azure Key Vault, so the pipeline
pulls current values at run time rather than storing a copy) or into secret pipeline variables marked
`isSecret`, which Azure DevOps automatically masks from logs. Access to Azure resources happens via
**service connections** using a service principal or workload identity federation — the pipeline
never sees a raw credential, it references the connection by name and Azure DevOps handles the
authentication handshake. At the application layer, the deployed service itself authenticates to Key
Vault using a **managed identity** (see chapter 03's worked Terraform example) rather than an
application setting holding a secret string — so even the running app never has a Key Vault
credential to leak, just an Azure-issued identity token. The combined effect: no secret is ever
plaintext in Git, pipeline logs, or app configuration.

### Q6. Blue-green vs. canary deployment — when would you pick one over the other?

**A.** Blue-green (implemented via Azure App Service deployment slots) gives an instant, clean,
all-or-nothing cutover with a near-instant rollback (swap back) — I'd pick it for changes that are
well-tested and where I want a strong safety net but don't need gradual traffic exposure, which
covers most routine application-code deploys. Canary routes a small percentage of real traffic to the
new version first, watching metrics before increasing that share — I'd pick it specifically when the
change itself is uncertain enough that I want production signal before fully committing: for example,
a change to the chatbot's retrieval configuration or system prompt, where the failure mode (subtly
worse answers) wouldn't necessarily show up in a smoke test but would show up in a canary slice of
real usage. The tradeoff is operational complexity — canary needs traffic-splitting and real-time
monitoring infrastructure that blue-green via slots doesn't.

### Q7. How does Terraform state locking prevent conflicts?

**A.** Terraform state is the source of truth for what infrastructure it believes exists. If two
`terraform apply` runs execute concurrently against the same state — say, a developer running apply
locally at the same time the pipeline runs it — without locking, both could read the same starting
state, compute independent plans, and write conflicting results back, corrupting the state file or
producing infrastructure that doesn't match either plan. A remote backend (an Azure Storage Account
container, in this course's examples) supports **locking**: before any write, Terraform acquires a
lock on the state (via a lease mechanism on the blob); a second concurrent run attempting to acquire
the same lock fails immediately with a "state is locked" error rather than proceeding and racing the
first run. The practical implication for a pipeline: `terraform apply` should only ever run from one
place per environment (the pipeline, not also from developer laptops), and if you see a lock error,
the right response is to wait or investigate a stuck lock — never to force-unlock without confirming
no other apply is actually in progress.

### Q8. Walk me through what happens, stage by stage, when a pull request merges to main.

**A.** Merge to `main` fires the **CI trigger**. Build stage installs dependencies, runs lint, runs
the unit/integration test suite, and publishes results. If that's green, a security/dependency scan
runs against the code and, once built, the container image. The container-build stage produces a
Docker image tagged with the build ID/commit SHA and pushes it to Azure Container Registry — this
exact artifact is what will flow through every subsequent environment. A `terraform plan` runs against
the dev environment's state to preview any infra changes; if unattended dev deploys are allowed,
`terraform apply` follows automatically. The new image deploys to the dev App Service slot, followed
by an automated smoke test (`/health` plus one real endpoint). If that passes, the same
plan/apply/deploy/smoke-test sequence runs against staging. Promotion to prod repeats the pattern but
pauses at an **approval gate** configured on the prod Environment resource — a human reviews the
change (and the `terraform plan` diff, if infra changed) and approves before the final deploy and
smoke test execute against production.

### Q9. How would you pipeline-gate an LLM-backed app to catch a bad prompt or config change before it reaches production?

**A.** Two things have to be true first: prompts, system messages, and retrieval config (top_k,
temperature, model deployment name) need to be version-controlled files that go through the same PR
process as code — not values edited live in a portal — so a "prompt change" is a diffable,
reviewable Git change like any other. Given that, the gate itself has a few layers: (1) an automated
**eval/regression suite** — a fixed set of representative prompts with expected properties (contains
required disclaimers, doesn't leak system-prompt content, response passes a groundedness/relevance
check against a reference answer or a rubric-based LLM-judge) run as a pipeline stage after build,
similar in spirit to the RAG evaluation practices covered in course 04 (Ragas-based hallucination/
robustness checks) — a regression here should block merge or at least block auto-promotion past dev.
(2) A **canary rollout** (chapter 04) for prompt/config changes specifically, since their failure mode
is often subtle quality degradation rather than a hard error a smoke test would catch — route a small
percentage of real conversations to the new prompt and monitor a quality signal (thumbs-down rate,
fallback-triggered rate, latency/token-cost) before full rollout. (3) **Model version pinning** as an
explicit, reviewed Terraform/config value (chapter 01/03) so a model upgrade is a deliberate change
that goes through the same gate, not something that happens silently via portal-managed deployment
settings.

### Q10. What's the difference between a service connection and a variable group in Azure DevOps, and why do you need both?

**A.** A service connection is an *authentication* bridge — it lets the pipeline act as an identity
against an external system (an Azure subscription, a container registry, GitHub) without the pipeline
ever holding a raw credential. A variable group is a *configuration* store — named key-value pairs
(optionally sourced live from Key Vault) that parameterize a pipeline run, like an app name, a
resource group, or a secret connection string. You need both because they solve different problems:
the service connection answers "is this pipeline allowed to act on this Azure subscription at all,"
while the variable group answers "what values should this particular run use." In a multi-client
setup, each client typically gets its own service connection (scoped to only their subscription) and
its own variable group (holding only their environment's config), so a pipeline template parameterized
by both can safely deploy the same service to different clients without any risk of Client A's
pipeline touching Client B's resources.

### Q11. Why is trunk-based development generally preferred over GitFlow for a fast-iterating service like an LLM chatbot?

**A.** GitFlow's long-lived `develop`/`release` branches optimize for infrequent, scheduled releases
where stabilizing a release branch independently from ongoing feature work makes sense. A chatbot
service that iterates on prompts and retrieval logic multiple times a week doesn't have that shape —
long-lived branches would mean prompt/config changes sit unintegrated for days, increasing the chance
of conflicting changes and delaying the point where a regression is caught. Trunk-based development
(short feature branches, PR-gated merges to main, environment promotion handled by the pipeline
rather than by branch structure) keeps integration continuous, which is the entire premise CI/CD is
built on — the earlier a change is integrated and tested against everyone else's changes, the cheaper
problems are to find and fix.

### Q12. How do you validate infrastructure changes before they hit a shared environment?

**A.** `terraform plan` is the primary tool — it computes and displays the exact diff (adds, changes,
destroys) against the real environment's tracked state without applying anything, and that plan output
is what should be reviewed in the pull request, ideally posted as a PR comment by the pipeline so
reviewers see the infrastructure diff alongside the code diff. A plan showing an unexpected `destroy`
on a resource that holds state (a database, a Key Vault with live secrets) is the single most
important thing to catch before approving — Terraform will destroy and recreate a resource if a
change requires it and the resource doesn't support in-place update, which is sometimes silently
correct and sometimes catastrophic depending on what the resource holds. Beyond `plan` review, running
`plan` on a schedule against the *existing* environment (a drift-detection job) catches cases where
someone made a manual portal change that diverged from what Terraform believes exists — worth naming
if asked how you'd catch configuration drift over time, not just at deploy time.

---

## System Design

### Q13. Design a CI/CD pipeline for a multi-client consulting team where each client has an isolated Azure subscription.

**A.** Structure the design around **isolation as the primary constraint**, then reuse as the
secondary goal:

- **Repo structure:** one shared pipeline-template repo (or shared templates within each service
  repo) defining the common build → test → containerize → deploy shape, parameterized by
  environment. Each client's actual deployment config (subscription ID, resource names, SKU sizing)
  lives in per-client variable groups and Terraform variable files — never hardcoded into the shared
  template.
- **Per-client isolation:** one **service connection** per client subscription (scoped tightly, ideally
  via a service principal with least-privilege RBAC on just that subscription), one **variable group**
  per client per environment, and separate **Terraform state files** per client/environment
  (`key = "<client>/<service>/<env>.tfstate"` in the remote backend) so a state operation for Client A
  can never touch Client B's state.
- **Pipeline flow per client:** the shared template runs build/test once per service (client-agnostic
  — the code doesn't change per client), then a deploy stage matrix, one job per client environment,
  each parameterized with that client's service connection and variable group, each gated by that
  client's own Environment-resource approval check (so Client A's prod approval doesn't touch Client
  B's).
- **Guardrails:** a policy check (either an automated `terraform plan` review or a required-reviewer
  rule) on any change touching the shared template, since a bug there has blast radius across every
  client at once — the shared-template repo needs the *strictest* review gate in the whole system,
  precisely because it's shared.
- **Secrets:** each client's secrets live in that client's own Key Vault, referenced only by that
  client's variable group and service connection — no shared secret store across clients, so a
  compromised credential for one client can't be used to pivot into another's environment.

The overall shape: **one pipeline definition, many isolated invocations** — isolation lives in
*which credentials and state the pipeline is parameterized with per run*, not in duplicated pipeline
logic per client, which is what keeps the system maintainable as the client roster grows.

### Q14. How would you extend this pipeline to support rolling back an LLM model version specifically, as opposed to a code rollback?

**A.** Treat the model deployment name/version as a **pinned, versioned configuration value** managed
the same way as infrastructure (a Terraform variable or a version-controlled config file consumed at
deploy time — see chapter 01/03), not as portal-managed state. That makes a model version change a
normal, reviewed, artifact-versioned deployment like any other, which means the same rollback
mechanisms already covered apply directly: redeploy the previous config value through the pipeline
(fast, minutes, same mechanism as redeploying a previous Docker image tag), or — if the model swap
was rolled out via canary (Q6/Q9) — simply stop advancing the canary's traffic percentage and route
back to the previous model deployment, which is nearly instant since both versions are already live
side by side during a canary rollout. The key design point: a model version should never be a
"we changed it in the Azure OpenAI portal and hoped" event — it should be exactly as observable,
diffable, and rollback-able as a code deploy.

---

## Production Resilience and Pipeline Bug Narratives

### Q15. Tell me about a specific CI/CD bug you'd expect to find in a multi-client pipeline like this one.

**A.** *(Illustrative — chapter 07 has four, this is the strongest single one to lead with.)* A
copy-pasted template parameter: when a new client-specific setting is added by copying one client's
deploy-stage block (`DeployHSBC`) to create the other's (`DeployBofA`), it's easy to update the obvious
parameters — service connection, app name — and miss one, like `variableGroup`, still pointing at the
first client's secrets. The result deploys correctly to the second client's App Service and its own
service connection, but with the first client's Key Vault URI and connection strings injected as app
settings — a real cross-tenant secret leak, and one both deploys report as a clean success. The fix that
actually catches this isn't better code review of the YAML — it's an automated post-deploy assertion
that checks the *resolved identity* of what was injected (does the Key Vault URI in Bank of America's
app settings actually contain "bofa," not "hsbc") rather than only checking that the deploy succeeded.
That distinction — checking identity, not just outcome — is the one worth leading with.

### Q16. What's the most dangerous kind of pipeline failure, and why?

**A.** The ones that look identical to success. A build failure, a failed test, a failed `terraform
plan` — all of these stop the pipeline and announce themselves loudly, and recovery is cheap because
nothing was deployed yet. Far more dangerous: a `terraform apply` run with `-refresh=false` that
silently no-ops because it never re-queried a resource someone changed out-of-band in the Portal, or a
smoke test that keeps passing after a URL refactor because it's still hitting a stale route that happens
to return `200` for unrelated reasons. Both report a clean, green pipeline while the real, intended
change silently failed to take effect or the real regression went unchecked. The practical takeaway:
invest as much scrutiny in what a smoke test's assertions actually check (response body shape, not just
status code) and in never skipping a state refresh against shared infrastructure, as in the deploy
mechanics themselves — because the failure that costs the most is the one nobody notices until
something downstream depends on the thing that silently didn't happen.

### Q17. What's one hardening gap in this pipeline you'd name candidly rather than claim is solved?

**A.** The requirement that a `terraform plan` showing an unexpected `destroy` on a stateful resource
(a database, a Key Vault) gets manually reviewed before approval is enforced by reviewer diligence and
team convention today, not by an automated policy gate — nothing actually inspects the plan's JSON
output and blocks the approval step if it contains a destroy on a resource type known to hold live data.
The fix is well-understood and not exotic — an Open Policy Agent/Conftest-style check, or even a script
parsing `terraform show -json` (the same shape notebook `02_terraform_basics_demo.ipynb` and the new
plan-diff safety-gate notebook both work with), run as a required, blocking stage before the approval
gate. The honest framing: this wasn't automated because it felt like more upfront investment than the
number of actual close calls justified — but it's a known, well-scoped gap, and I'd prioritize it the
moment "a reviewer missed an unexpected destroy" stopped being hypothetical.

---

## Retrospective

### Q18. What would you change if you rebuilt this pipeline today?

**A.** A few concrete improvements, framed honestly as hindsight rather than as things that were
necessarily wrong the first time: (1) **Policy-as-code for infrastructure** — adding an automated
policy check (e.g., an Open Policy Agent/Conftest check on `terraform plan` output) that blocks
merges introducing non-compliant resources (a public-facing storage account, a missing tag) instead
of relying entirely on human review of the plan diff. (2) **A proper LLM eval gate in the pipeline
itself** (Q9) rather than relying only on manual testing before a prompt/config change ships — wiring
an automated Ragas-style evaluation suite (as covered in course 04) as a required pipeline stage, not
an optional pre-merge habit. (3) **Ephemeral PR environments** — spinning up a short-lived, fully
isolated environment per pull request (rather than only dev/staging/prod as fixed environments) so a
reviewer or the client can click a live preview link during code review, tearing it down automatically
on merge/close, which shortens the feedback loop even further than "deploy to shared dev on merge."
(4) **Tighter cost guardrails** — since LLM-backed services have a runtime cost profile
(token usage) that traditional web services don't, adding a cost/usage check to the smoke-test or
canary-monitoring step, not just uptime/latency. (5) **The release-manifest design from chapter 06** —
today, the chatbot, monitoring, and uploader services roll back independently and safely on their own
terms, but nothing checks whether the resulting combination of all three versions is one that's actually
known to work together (Q1). A versioned, atomically-deployed release manifest, validated against a
compatibility matrix as the last stage of a platform-level release pipeline, is the concrete next step —
and it's the one improvement on this list that isn't really optional once a second or third
cross-service compatibility incident actually happens, rather than staying hypothetical. None of these
are "the original pipeline was wrong" — they're the natural next layer of maturity once the foundational
build/test/deploy/gate discipline this course covers is already in place.

---

## Client & Production Context (HSBC / Bank of America)

### Q19. You deployed the same platform pattern for two banking clients, HSBC and Bank of America — how did the pipeline keep their environments, secrets, and deployments completely isolated from each other?

**A.** Isolation was enforced at every layer the pipeline touches, not just one: **(1) Compute** —
each bank got its own Azure App Service instance (and its own resource group), so there was never a
shared multi-tenant App Service serving both banks' traffic. **(2) Deployment mechanics** — the
pipeline definition itself stayed single (chapter 02's stage-matrix pattern), but each client's deploy
stage — `DeployHSBC` and `DeployBofA` — ran as an independent stage depending only on the shared
`Build` stage, never on each other, so a failure or rollback in one bank's stage had zero effect on
the other's run. **(3) Authentication** — each bank had its own Azure DevOps **service connection**,
scoped via RBAC to only that bank's Azure subscription/resource group, so the pipeline had no
credential capable of touching the other bank's infrastructure even if a template bug tried to. **(4)
Configuration and secrets** — each bank had its own **variable group**, backed by that bank's own Key
Vault, so HSBC's connection strings, API keys, and app settings never lived in a place Bank of
America's pipeline run could read, and vice versa. **(5) Approval** — each bank's production
`Environment` resource carried its own independently-configured approval check, so HSBC's sign-off
list and Bank of America's sign-off list were two separate gates, not a shared "approve for both"
click. The one thing that *was* shared on purpose was the pipeline **template** and the client-agnostic
build/test logic — since duplicating that per bank would mean two places to fix the same bug — but
everything that actually touches a bank's infrastructure, credentials, or data was parameterized and
scoped per client, never shared. That combination — same template, completely isolated invocation — is
the practical answer to "how do you serve two competitors from one platform without either knowing
the other exists."

### Q20. How would you design a rollback strategy for a customer-facing bank chatbot deploy that goes wrong during business hours?

**A.** The design has to assume the failure will be noticed by real users, not caught quietly in
staging, so speed and blast-radius containment matter more than root-causing the bug in the moment.
In priority order: **(1) Slot swap back first** (chapter 04) — since the previous version is still
warm in the swapped-out slot, this is a seconds-scale operation and should be the default first
response for any application-code regression; it buys time to investigate without users continuing to
hit the broken version. **(2) Scope the blast radius to one bank immediately** — because HSBC and
Bank of America run on separate App Service instances, confirm (or, if the deploy fanned out via the
stage matrix, ensure) the rollback targets only the affected bank's slot; the other bank's production
traffic should be completely unaffected and doesn't need any action. **(3) If the change was rolled
out via canary** rather than a full swap, the fastest option is simply routing traffic back to 0% on
the new version — no redeploy needed since the old version is already live side by side. **(4) If a
slot swap isn't available or the issue is deeper than application code** (e.g., a bad config value
baked into the image), redeploy the last known-good, immutable image tag through the same pipeline —
still a routine, minutes-scale operation, not a special "rollback pipeline." **(5) Communicate and
log** — for a bank client, a production incident during business hours typically has a severity
classification and notification process; treating the rollback as done only once it's both executed
*and* logged/communicated per the client's incident process is part of the actual job, not an
afterthought. The mechanism that makes all of this fast is preparation, not improvisation: slots kept
warm, immutable versioned artifacts, and per-client isolation are all decisions made *before* the
incident, not during it.

### Q21. What pipeline gates would you add specifically because the target system serves real bank customers, that you wouldn't necessarily need for an internal tool?

**A.** An internal tool's pipeline can usually get away with build, test, and a smoke test. A
system serving real HSBC or Bank of America customers through Azure Front Door/App Gateway, Azure AD,
and VNet-integrated App Service needs gates that validate the *production topology itself* is intact
on every release, not just that the application code works: **(1) A WAF/edge validation check** — confirm
the release is reachable only through Azure Front Door/Application Gateway and not directly against
the App Service's default hostname, since a banking-grade requirement is that the app tier is never
reachable except through the gateway. **(2) A network/connectivity check against VNet integration and
Private Endpoints** — verify the deployed app can still reach its Key Vault, database, and any other
dependency over the private network path, since a misconfigured Private Endpoint would break
connectivity silently rather than fail the app outright at build time. **(3) An Azure AD auth smoke
test** — an automated check that an unauthenticated request is correctly rejected and a valid token is
correctly accepted, catching an auth-config regression (a wrong tenant ID, a misconfigured app
registration) before real customers hit it. **(4) A per-client isolation check** — for a two-bank
platform specifically, an automated assertion that the HSBC deploy stage's service connection and
variable group actually resolved to HSBC's resources (and not, via a copy-paste template error,
Bank of America's) — cheap to check, and the one class of bug with the worst possible consequence if
it slips through. **(5) A monitoring/alerting readiness gate** — confirm Azure Monitor/Application
Insights is actually receiving telemetry from the newly deployed slot before promoting it to
production, since a "successful" deploy that's silently unobservable defeats the purpose of having
monitoring at all. None of these are exotic — they're the same build/test/gate discipline this course
teaches, just aimed at validating the production topology (edge, network, identity, isolation,
observability) instead of only the application code, because for a bank's customer-facing system,
all five of those are as capable of causing a real incident as a code bug is.

### Q22. Did switching the target service from Flask to FastAPI change anything about the pipeline itself?

**A.** Barely. The document-uploader service (course 05) is built with **FastAPI**, **SQLAlchemy** as
the ORM, and RBAC via **MSAL/OAuth 2.0** against Azure AD — and the honest answer is that almost none
of the pipeline described in this course cares which Python web framework sits inside the container.
The CI/CD stages — build/lint, unit/integration tests, security scan, Docker build-and-push to Azure
Container Registry, Terraform-provisioned App Service, slot deploy, approval gate, smoke test — are
identical regardless of framework, because the pipeline's unit of deployment is an **immutable
container image**, not source code it understands the internals of. The actual deltas are narrow and
all live inside the image, not the pipeline definition: (1) the **dependency-install list** —
`requirements.txt` now pulls in `fastapi`, `uvicorn[standard]`, `sqlalchemy`, and `msal` instead of
`flask`; (2) the **container startup command** — the Dockerfile's `CMD` runs the app via `uvicorn`
(an ASGI server, since FastAPI is async-native) instead of `gunicorn` fronting a Flask WSGI app; and
(3) the **health-check endpoint and expected response shape** the smoke test asserts against — a
FastAPI `/health` route (and, incidentally, FastAPI gives you a free `/docs` OpenAPI UI that's handy
for a manual sanity check post-deploy, which Flask doesn't provide out of the box). Everything else —
variable groups, service connections, the HSBC/BofA per-client stage matrix, Terraform-managed
infrastructure, approval gates, rollback via slot swap — stayed exactly the same. That's actually the
point worth making to an interviewer: it's a good illustration of *why* you containerize an
application in the first place — Docker is the abstraction boundary that decouples the pipeline from
framework internals, so a framework swap is a change to a handful of lines inside the image (deps,
start command, health route) rather than a pipeline redesign.
