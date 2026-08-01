# 01 — CI/CD Fundamentals

## Why this chapter matters

Before touching Azure DevOps YAML or Terraform, you need to be able to explain — clearly, in
interview-answer form — what CI/CD actually *is* and *why each stage exists*. Interviewers ask this
because "I used Azure DevOps" is a tooling fact, not an engineering judgment. They want to know you
understand the problem the tooling solves. This chapter builds that foundation, then applies it to
the specific shape of problem the resume bullet describes: automating deployment for GenAI backend
services (the chatbot from course 01, the document-uploader from course 05) across multiple client
environments.

## Continuous Integration vs. Continuous Delivery vs. Continuous Deployment

These three terms get used interchangeably in casual conversation, but they mean different things,
and an interviewer who asks "what's the difference?" wants the precise answer.

**Continuous Integration (CI)** is the practice of merging code changes into a shared branch
frequently (multiple times a day, ideally) and automatically verifying each merge with a build and a
test suite. The goal is to catch integration problems — "my change breaks your change" — within
minutes of it happening, not weeks later during a painful merge. CI answers the question: *does this
code still work when combined with everyone else's code?*

**Continuous Delivery (CD)** extends CI one step further: every change that passes CI is
automatically packaged into a deployable artifact (a container image, a build output) and is *always
in a releasable state*. A human still decides when to actually push that artifact to production —
there's a deliberate approval step. CD answers: *is this artifact always ready to ship, on demand?*

**Continuous Deployment** removes the human approval step entirely — every change that passes all
automated gates goes to production automatically, no manual click required. This is common for
low-risk, high-confidence pipelines (e.g., a documentation site) but rare for client-facing
regulated-industry systems, which is why a consulting engagement like the ones in this curriculum
typically stops at continuous *delivery* with a manual production gate, not full continuous
deployment — and it's exactly why the HSBC and Bank of America production deployments in this course
(see `00-README.md`) always had a human approval step before prod, never fully automatic deployment.

For the chatbot and document-uploader services, the realistic setup is: CI on every pull request
(build, lint, test), CD to dev and staging automatically on merge to main, and a manual approval gate
before the artifact is promoted to the client's production slot. That distinction — CI everywhere,
CD to lower environments, a gate before prod — is worth stating explicitly in an interview because it
shows you understand *why* the gate exists (client risk tolerance), not just that "there's a pipeline."

## The Purpose of Each Pipeline Stage

A pipeline is a sequence of gates, and each gate exists to catch a specific class of problem as early
and cheaply as possible — the earlier a defect is caught, the cheaper it is to fix.

- **Build.** Compiles or assembles the code (for Python, this is mostly "install dependencies and
  confirm the app imports cleanly") and produces a build artifact. Catches syntax errors, missing
  dependencies, broken imports — the cheapest class of bug to catch, and the most embarrassing to
  ship.
- **Lint / static analysis.** Enforces code style and catches likely bugs (unused variables, type
  mismatches, obvious anti-patterns) before a human reviewer has to point them out. This keeps code
  review focused on logic and design, not formatting nitpicks.
- **Test (unit / integration).** Runs the automated test suite. Unit tests catch logic errors in
  isolated functions (e.g., a prompt-template formatter, a request validator). Integration tests
  catch problems in how components interact (e.g., does the FastAPI route actually call the ingestion
  function correctly, does the chatbot's LangChain pipeline actually call Azure OpenAI with the
  expected parameters using a mocked client).
- **Security / dependency scan.** Scans the dependency tree for known CVEs and scans the container
  image for vulnerable base-image layers. This matters more, not less, for client-facing consulting
  work — a vulnerability in a client's environment is a trust and contract problem, not just a
  technical one.
- **Artifact publish.** Packages the verified code into an immutable, versioned artifact — typically
  a Docker image tagged with a commit SHA or build number — and pushes it to a registry (Azure
  Container Registry). "Immutable and versioned" matters: it's what makes rollback possible later
  (redeploy the previous tag) and what makes "staging and prod are running the exact same bits"
  provable rather than assumed.
- **Deploy.** Takes the published artifact and applies it to a target environment — updates an Azure
  App Service or Function App to run the new container, applies any Terraform-managed infrastructure
  changes first if the environment's shape changed.
- **Smoke test.** A fast, shallow post-deploy check (hit `/health`, hit one real endpoint) that
  confirms the new deployment is actually serving traffic correctly before declaring the release
  successful. This is the pipeline's last line of defense before real users see a broken deployment.

## Trunk-Based Development vs. GitFlow

**Trunk-based development** means everyone works off a single long-lived branch (`main`/`trunk`),
using short-lived feature branches (hours to a couple of days) that merge back quickly behind
pull-request review, often with feature flags to hide incomplete work. It optimizes for fast,
frequent integration and is the natural fit for CI/CD — the pipeline only has to reason about one
branch as the source of truth for what gets deployed.

**GitFlow** uses long-lived `develop`, `release`, and `hotfix` branches alongside `main`, with a more
formal release-branching process. It made sense in an era of infrequent, scheduled releases (e.g.,
quarterly desktop software), but it adds merge overhead and delayed integration that fights against
CI/CD's goal of fast feedback.

For a consulting engagement building fast-iterating GenAI services, trunk-based development (short
feature branches, PR-gated merges to main, environment promotion via pipeline rather than via
long-lived Git branches) is the better fit and the one worth defending in an interview — see chapter
05 for the branching mechanics in detail.

## Why Pipelines Matter Specifically for AI/LLM Services

Traditional CI/CD assumes the artifact is deterministic: same code in, same behavior out. LLM-backed
services break that assumption in two specific ways that a pipeline has to account for:

- **Prompt/config drift.** A system prompt, a retrieval `top_k` value, a temperature setting, or a
  few-shot example set can change behavior just as much as a code change — but if those live in a
  loosely tracked config file or, worse, get edited directly in a portal, there's no audit trail and
  no way to correlate "the bot started answering worse" with a specific change. Treating prompt and
  config files as version-controlled artifacts that go through the same PR-and-pipeline path as code
  is what makes that class of regression debuggable.
- **Model version pinning.** Azure OpenAI deployments can be repointed to a new model version
  (e.g., a GPT-4 model upgrade) outside of your pipeline entirely, silently changing output quality,
  latency, or cost. A disciplined pipeline pins the model deployment name/version as an explicit,
  reviewed configuration value (ideally itself managed via Terraform, see chapter 03) rather than
  leaving it as portal-managed state, so a model version change is a deliberate, reviewed,
  rollback-able event — not a surprise.

This is the concrete answer to "why does CI/CD matter for a GenAI service, specifically, beyond any
other web service?" — because the artifact's behavior depends on more than its code, and the pipeline
is the mechanism that keeps that extra surface area (prompts, configs, model versions) under the same
discipline as the code itself.
