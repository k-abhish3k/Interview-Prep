# 05 — GitHub vs. Azure Repos, and Branching Strategy

## Why this chapter matters

The resume lists **GitHub** alongside Azure DevOps, and a real consulting engagement is just as
likely to use Azure Repos (Azure DevOps's native Git hosting) as GitHub, depending on the client's
existing tooling. An interviewer may probe whether you understand these as *interchangeable Git
hosts with different ecosystem integrations*, not fundamentally different version-control systems —
and separately, whether you have an opinion on branching strategy and PR gating, since that's the
process discipline that keeps a fast-moving GenAI codebase (chatbot, document-uploader) from breaking
in production. This chapter covers both.

## Branching Strategies

**Trunk-based development** (introduced in chapter 01) is short-lived feature branches cut from and
merged back into a single `main` branch quickly, usually within a day or two, gated by a pull request
and CI checks. It optimizes for continuous integration: the longer a branch lives separately from
main, the more it drifts and the more painful the eventual merge. For a small team iterating fast on
a chatbot's prompts and retrieval logic, this is almost always the right default — long-lived
branches actively fight against the "merge frequently, catch integration bugs early" goal that CI/CD
depends on.

```
main ----o----o----o----o----o----o----o----> (always deployable, protected)
           \        \        \
            feature/  feature/ feature/
            retry-    prompt-  new-endpoint
            logic     tuning   (each: hours-days, PR-gated merge back to main)
```

**Feature branches** are the unit trunk-based development is built from — one branch per unit of
work, deleted after merge. The discipline that makes this work is keeping them *short-lived*; a
feature branch open for three weeks isn't really "trunk-based," it's GitFlow with extra steps.

**Release branches** are a longer-lived branch cut from main at a specific point, used when a
release needs to be stabilized (bug-fixed) independently while main keeps moving forward with new
feature work — common in shrink-wrapped software with versioned releases, less common for a
continuously-deployed internal service. On a consulting engagement, release branches show up more
often when a client needs a specific, frozen version for a compliance audit or a formal UAT sign-off
window — exactly the kind of ask a banking client like HSBC or Bank of America makes routinely; day-to-day
work still flows through trunk-based feature branches.

**GitFlow**, for contrast, formalizes `develop`, `release/*`, and `hotfix/*` branches on top of
`main` — more ceremony, more merge overhead, built for infrequent scheduled releases. Worth knowing
the shape of it (interviewers ask "GitFlow vs. trunk-based, why?") but not worth defending as the
right choice for a fast-iterating GenAI service.

## Pull Request Gates and Required Reviewers

A PR gate is a rule that blocks merging until conditions are met — this is where the "process"
discipline and the "pipeline" mechanics from earlier chapters connect. Typical gates:

- **Required status checks** — the PR pipeline (build, lint, test — the CI trigger from chapter 02)
  must report success. A red check blocks the merge button entirely.
- **Required reviewers** — at least one (often two, for infra/security-sensitive paths) human approval
  before merge is allowed. On Azure Repos this is a **branch policy** on `main`; on GitHub it's
  **branch protection rules**. Both support path-based rules — e.g., requiring a specific reviewer
  (or team) for changes under `terraform/` or `infra/`, since infrastructure changes carry different
  risk than an application-code change.
- **Linear history / no direct pushes** — blocks pushing straight to `main`, forcing every change
  through a reviewed PR, which is what makes "every deployed change has a reviewable diff and an
  approver" actually true rather than aspirational.
- **Merge strategy** (squash vs. merge commit) — squash-merging feature branches keeps `main`'s
  history as one commit per reviewed change, which pairs well with tagging each commit's build
  artifact by commit SHA (chapter 01) — one commit, one artifact, one clear audit trail.

For the chatbot and document-uploader services, a reasonable policy is: 1 required reviewer for
application code, 2 for changes touching `terraform/` (since a bad infra change has blast radius
beyond one service), required CI status checks, and no direct pushes to `main`.

## GitHub Actions vs. Azure DevOps Pipelines

These solve the same problem — YAML-defined, event-triggered CI/CD — with different ecosystem
integration, and the practical differences worth knowing:

| | Azure DevOps Pipelines | GitHub Actions |
|---|---|---|
| Config location | `azure-pipelines.yml` (or configured path), separate from repo hosting if using GitHub as source | `.github/workflows/*.yml`, lives in the same repo it builds |
| Native Azure integration | First-party — Environments with built-in approval checks, deep service-connection model, tight `AzureWebApp@1`/`AzureCLI@2` task support | Strong via the `azure/*` official actions, but one layer removed from being "the same product" |
| Reusable logic | Templates (`- template: path.yml`) | Reusable workflows (`workflow_call`) and composite actions |
| Secrets/config | Variable groups + Key Vault-linked variable groups, service connections | Repository/organization/environment secrets, OIDC federation to Azure |
| Marketplace/ecosystem | Azure DevOps Marketplace (smaller, more enterprise-focused) | GitHub Marketplace (much larger community action ecosystem) |
| Best fit | Client already standardized on Azure DevOps for repos + boards + pipelines as one suite | Client's code already lives on GitHub, or wants the larger community-action ecosystem |

**They interoperate** — a very common real-world setup on a consulting engagement is source code
hosted on **GitHub** (client preference, or an existing org-wide GitHub Enterprise setup) with
**Azure DevOps Pipelines** still handling build/release, connected via a GitHub service connection
that lets an Azure Pipeline trigger on GitHub pushes/PRs and check out the GitHub repo directly. This
is worth naming explicitly if asked "can you use GitHub and Azure DevOps together" — yes, and it's a
normal pattern, not a workaround.

The practical decision criterion isn't "which is technically better" (they're close to feature-parity
for straightforward container-build-and-deploy pipelines) — it's **where does the client's team
already live**. A client with existing Azure DevOps Boards for work-item tracking and existing branch
policies on Azure Repos gets less value from introducing GitHub Actions alongside it than from just
using Azure Pipelines against the same repo. A client whose engineering org has standardized on GitHub
for source hosting, code review, and possibly GitHub Actions already gets less value from forcing
Azure DevOps into the loop just for pipelines. Being able to articulate that tradeoff — rather than
asserting one tool is categorically better — is the senior-sounding answer.

## Tying It Back

For the chatbot and document-uploader services, the realistic setup this course assumes is: source in
Azure Repos or GitHub (client-dependent), trunk-based branching with short feature branches, PR gates
requiring green CI checks and at least one reviewer (two for `terraform/` changes), and Azure DevOps
Pipelines as the build/release engine regardless of which Git host is in front of it — because the
deep Environment/approval-gate and service-connection integration from chapters 02 and 04 is most
mature there.
