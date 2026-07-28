# 03 — Multi-Department Architecture and Business-Line Routing

## The real department inventory (memorize this table, not "Credit Ops")

The service handles exactly four business lines with dedicated roles, plus a generic bucket. This is
read directly from `app.py`'s `/role-check` endpoint and `FULL_ARCHITECTURE.md` §3 — not reconstructed:

| Business line | AAD role | Gated by feature flag | Data flow | Approval? | Email? |
|---|---|---|---|---|---|
| **IWPB** | `stitt.ingester` | — (default; always visible once the user has this role) | Staged in Blob → DB row → approver notified → ingested **only on approval** | **Yes** | Yes |
| **FEMA** | `stitt.ingester.pilot` | `FEATURE_LOGICAL_SEPERATION` | Direct to `INGEST_API` | No | No |
| **TPMB** | `stitt.ingester.tpmb` | `FEATURE_THIRD_PARTY_BOT`, **and** the user must hold all three of `stitt.ingester` + `.pilot` + `.tpmb` | Direct to `INGEST_API` | No | No |
| **GTRM** | `stitt.ingester.gtrm` | `FEATURE_GREEN_TIME`, **and** the user must hold all four roles | Direct to `INGEST_API` | No | No |
| RESEARCH / WCS / GPS / WCL / OTHER / GENERAL | *(no dedicated role — generic Business Line field)* | — | Direct to `INGEST_API`, the original "usual flow" | No | No |

**There is no "Credit Ops" business line anywhere in this codebase.** If that name comes up again in
prep or in an interview, correct it in the moment to IWPB/FEMA/TPMB/GTRM.

## One shared process, not four microservices

`FULL_ARCHITECTURE.md` §1 states it plainly: *"No new microservices were introduced for any
department-specific logic; every department's behavior lives in this one process, differentiated by
`use_case` string branching and a feature-flag-driven role model."* Concretely, in `app.py`'s
`upload_files()`:

```python
if use_case.upper() == iwpb_workflow.IWPB_USE_CASE:   # "IWPB"
    # staged-in-blob, approver-gated path — the only fork in the whole function
    ...
    return {...}

# every other business line — including FEMA, TPMB, GTRM — falls through to here:
# batch-initialize, then per-file ingest, direct to INGEST_API, no approval, no email
```

One `if` statement is the entire architectural split between "the approval-gated department" and
"every other department." `/search-ingested-documents/{use_case}` and `/remove-ingested-document/{use_case}`
are equally uniform — they don't even branch on IWPB, because approved IWPB documents become
indistinguishable from any other department's ingested documents once they land in HEXA.

## Why one shared service instead of one microservice per department — a real interview question with a defensible answer

An interviewer who understands the domain will likely ask: *"why didn't you build IWPB, FEMA, TPMB,
and GTRM as four separate services?"* The honest, defensible answer has two sides:

**The case for splitting them (blast radius / isolation):**

- A bug or an outage in one department's code can't take down another's — IWPB's approver workflow has
  meaningfully more moving parts (Blob staging, SMTP, a background loop, a SQL table) than the other
  three departments combined, so it's the most likely place for a bug to originate.
- Independent deployability — shipping an IWPB fix wouldn't require redeploying (and re-testing) the
  FEMA/TPMB/GTRM code paths at all, even though today they're all in the same process and the same
  deploy.
- Independent scaling — if IWPB's SMTP/Blob/DB workload ever became meaningfully heavier than the
  other departments' simple proxy calls, a shared App Service Plan means they scale together whether
  they need to or not.

**The case for what was actually built (one shared service):**

- **Operational simplicity dominates at this scale.** Four departments, one of which (IWPB) has real
  extra logic and three of which are near-identical thin proxies, is not enough surface area to justify
  four deployment pipelines, four sets of Application Settings, four health checks, and four on-call
  rotations. The generic departments' entire logic is ~150 lines shared with IWPB's upload endpoint —
  splitting that into its own service would mean duplicating (or awkwardly sharing) the `INGEST_API`
  client, the session/auth plumbing, and the `/role-check` logic across service boundaries for very
  little isolation benefit.
- **The actual blast-radius risk is contained without a service split.** Each of the three sweep
  responsibilities in the background loop (chapter 06) is independently wrapped in `try`/`except`, and
  IWPB's own logic lives in a separate module (`iwpb_workflow.py`) that only ever touches
  `IWPBDocumentWorkflow` — a bug in IWPB's approve/decline logic can't corrupt a FEMA/TPMB/GTRM upload,
  even inside the same process, because they never share mutable state or a code path.
- **The role/feature-flag model already expresses the real boundary.** Access control (who can even see
  a department in their dropdown) is enforced once, centrally, in `/role-check` — splitting into four
  services would mean re-implementing (or centralizing via a gateway) that same role logic four times,
  for departments where three of the four have functionally identical needs.

The framing that reads best in an interview: *"the departments differ enormously in how much state and
process they need — IWPB genuinely warrants its own module, which is exactly what it got, just not its
own deployable service — while FEMA/TPMB/GTRM are thin enough that splitting them into their own
services would have added four times the operational surface for close to zero isolation benefit,
since they don't hold any state of their own to protect from each other."* That's a real trade-off,
argued from the actual shape of the code, not a generic "microservices are complex" hand-wave.

## The feature-flag-gated visibility model

`/role-check` (`app.py`, ~lines 857-908) is the single place that decides which departments a signed-in
user even sees in the UI's Business Line dropdown. Read literally from the source:

```python
role_to_business_line = {
    "stitt.ingester": "IWPB",
    "stitt.ingester.pilot": "FEMA",
    "stitt.ingester.tpmb": "TPMB",
    "stitt.ingester.gtrm": "GTRM",
}

business_lines = [role_to_business_line[r] for r in user_roles if r in role_to_business_line]

if feature_business_line_tpmb:
    tpmb_roles = {"stitt.ingester", "stitt.ingester.pilot", "stitt.ingester.tpmb"}
    if tpmb_roles.issubset(user_roles):
        return ["IWPB", "FEMA", "TPMB"]

if feature_business_line_gtrm:
    gtrm_roles = {"stitt.ingester", "stitt.ingester.pilot", "stitt.ingester.tpmb", "stitt.ingester.gtrm"}
    if gtrm_roles.issubset(user_roles):
        return ["IWPB", "FEMA", "TPMB", "GTRM"]

return sorted(business_lines)
```

Two things worth calling out explicitly:

- **TPMB and GTRM are not simply "gated by a role."** A user needs the `stitt.ingester.tpmb` role *and*
  the two roles beneath it in the hierarchy (`stitt.ingester`, `stitt.ingester.pilot`) before TPMB shows
  up at all — and the feature flag `FEATURE_THIRD_PARTY_BOT` has to be on for that check to even run.
  GTRM stacks one level further: all four roles, plus `FEATURE_GREEN_TIME`. This reads as a
  progressive-rollout pattern — each new business line's visibility is deliberately built on top of
  having already been entitled to the ones before it, gated behind its own flag so it can be toggled
  independently of role assignment in Azure AD.
- **This is a full-department kill switch, not per-feature.** There's no partial-department flag (e.g.,
  "TPMB visible but read-only") — a department is either fully visible (and its whole upload/search/
  remove code path usable) or not visible at all, from this endpoint's point of view.

This result drives the UI's Business Line dropdown, which in turn drives which of the flows in the
department table above a given user can actually trigger — `/role-check`'s output isn't just cosmetic,
it's the front line of authorization for which upload/search/remove endpoints a user's session will
realistically ever call. (Chapter 04 covers RBAC end to end, including how `role_to_business_line` and
these flag checks are demonstrated as executable code in
`notebooks/04_role_to_business_line_rbac_demo.ipynb`.)

**One more precision worth having exactly right, confirmed by tracing every branch in
`notebooks/04_role_to_business_line_rbac_demo.ipynb`:** the unconditional `business_lines = [...]` line
at the top of the function builds its result purely from whichever `role_to_business_line` keys the
user's roles happen to match — it does **not** check any feature flag. The flags only gate the two
early-return *shortcuts* (`["IWPB","FEMA","TPMB"]` / `["IWPB","FEMA","TPMB","GTRM"]`). Read literally,
that means a user individually holding `stitt.ingester.tpmb` would see `TPMB` in the `sorted(business_lines)`
fallback even with `FEATURE_THIRD_PARTY_BOT` off — the flag's real job is narrower than "controls TPMB
visibility" in the general case; it controls the clean, all-roles-present shortcut specifically. In
practice this distinction likely never surfaces, since AAD roles are presumably provisioned
progressively (a user is unlikely to hold `.tpmb` without also holding the two roles beneath it) — but
it's the kind of precise, literal reading of the code that's worth having ready if an interviewer probes
deeper than the summary table above.

## The honest, bounded connection to downstream per-department search

The candidate's broader description of this project in conversation mentioned "4 different Azure AI
Search indexes per department" for retrieval, with RBAC-gated access to each. **That retrieval/indexing
layer is not visible anywhere in this repository.** This service's job stops the moment a file is
handed to `INGEST_API` — what HEXA does with it afterward (chunking, embedding, indexing, per-department
search-index ACLs) is entirely outside this codebase's scope; `INGEST_API`/HEXA is treated here as an
opaque external system of record.

What *is* true, and worth stating honestly rather than overclaiming: this service **fully implements**
the department/business-line RBAC model (`/role-check`'s role-to-business-line mapping and feature-flag
gating) that a downstream per-department search-index access-control model would naturally need to
consume — if HEXA does maintain separate indexes per department with RBAC-gated access, the entitlement
data this service already computes (which business lines a given AAD identity is allowed to touch) is
exactly the kind of signal that downstream system would need to enforce that access control correctly.
That's a defensible, honest way to connect the two systems in an interview — as a natural next
integration point this service's RBAC model would feed, not as a claim about code that isn't in this
repository.

## Tying It Back

"Built an end-to-end document uploader service" undersells what's actually interesting here: the real
engineering decision wasn't picking FastAPI or SQLAlchemy, it was correctly scoping **how much of the
system needed to be genuinely new** (IWPB's approver workflow — a real module, a real table, a real
background job) versus **how much could stay exactly as it was** (five other departments' proxy logic,
untouched). That's the concrete, source-grounded answer to "why does this look like one service instead
of a service-per-department platform" — and it's a stronger, more specific answer than a generic
microservices-vs-monolith essay, because it's argued from what each department actually needed rather
than from an architectural preference in the abstract.
