# 06 — Multi-Service Version Pinning and Environment Drift

## Why this chapter exists

This chapter exists for the same reason course 05's chapter 05 exists: because there's a sharp,
specific interview question hiding behind the pleasant-sounding phrase "we deployed the chatbot,
model-risk-monitoring, and document-uploader services independently." The question is: **"if you roll
back one of these three services, how do you make sure the environment doesn't end up in an
inconsistent combination of versions?"** — and the honest answer, worked through properly, is a much
stronger interview moment than pretending the problem doesn't exist.

**A note on how this chapter was built, unlike course 05's chapter 05:** course 05 was reconstructed
from a real proprietary source repository, so its versioning chapter could quote an actual function
name and describe an actual absence of logic. There is no equivalent source repository backing this
CI/CD course — everything below, including every YAML fragment, manifest shape, and bug story in this
and the next chapter, is a **plausible, technically detailed illustrative reconstruction** of how a
Consultant-II on this engagement would realistically have built and reasoned about this problem, not a
verified fact about a specific pipeline that existed. Treat it exactly the way you'd treat a strong
system-design answer: concrete, defensible, and honest about being a design rather than a citation.

The honest answer, like course 05's, has five parts: what a realistic setup does today (three
independently-versioned services, no shared release train), a manual workaround that closes most of
the gap without solving it structurally, why "is this service's own deploy healthy" is a different
question from "is this service's version compatible with what's currently live everywhere else," a
genuine proposed design for closing the gap properly, and a forward-looking tie to the
model-risk-monitoring platform (course 04) that shows systems-level thinking rather than
CI/CD-tunnel-vision.

## Part 1 — What a realistic setup does today: three independent release trains, no shared manifest

The chatbot (course 01), the model-risk-monitoring platform (course 04), and the document-uploader
service (course 05) each have their **own** Azure DevOps pipeline (chapter 02), their own Docker image
repository and tag scheme (`chatbot:<BuildId>`, `monitoring:<BuildId>`, `uploader:<BuildId>`), their
own Azure DevOps `Environment` resource per client per tier (`chatbot-hsbc-prod`,
`monitoring-hsbc-prod`, `uploader-hsbc-prod`, and the Bank of America equivalents), and — critically —
their own **independent release cadence**. The chatbot team ships prompt and retrieval tweaks multiple
times a week; the monitoring platform ships evaluation-metric changes on a slower, more deliberate
cadence because its output feeds model-risk sign-off; the document-uploader ships closer to "when a
department needs a new business-line route." Nothing about that is wrong — different services
genuinely do have different rates of change, and forcing them onto a single synchronized release train
would slow down the two faster-moving services for no benefit.

The gap is what's *missing*, not what's *present*: each service's pipeline (chapter 02) and rollback
mechanics (chapter 04) are genuinely solid *in isolation* — a slot swap back or a redeploy of the
previous image tag for the chatbot is a fast, well-understood, low-risk operation. But nothing at the
platform level records, at any point in time, **the full triple** — "HSBC's environment is currently
running chatbot vX, monitoring vY, and uploader vZ, and that specific combination is one that was ever
deployed together, tested together, or known to be compatible." Each service's own smoke test (chapter
02) only exercises that one service's own contract; nothing cross-checks it against what the other two
services currently expect. A release manager approving a chatbot rollback in the `chatbot-hsbc-prod`
Environment sees only that one approval request — there is no equivalent visibility into "monitoring
v41 currently assumes chatbot's response envelope includes a field that only exists from chatbot v56
onward."

This is the plain, first-sentence version of the answer, and it's worth being just as direct about the
gap here as course 05's chapter 05 is about the versioning gap: *"Today, each of the three services
rolls back independently and safely on its own terms — but nothing automatically checks whether the
resulting combination of versions across all three is one that's actually known to work together."*

## Part 2 — The manual workaround: a compatibility matrix and a cross-service version check

Nothing about the current setup *prevents* catching this by hand — it's just not automated or
enforced. Two pieces, realistically, would exist today to partially close the gap:

1. **A manually-maintained compatibility matrix**, kept as a table in a shared wiki page or a
   `COMPATIBILITY.md` in a platform-level repo, updated by whoever ships a service change that has a
   cross-service dependency:

   | Chatbot version | Monitoring version(s) it's known to work with | Uploader version(s) it's known to work with | Notes |
   |---|---|---|---|
   | v54–v55 | v38–v40 | any | Pre-citations response envelope |
   | v56–v58 | v41+ | any | Response envelope adds `citations[]`; monitoring v41 requires it for groundedness scoring |
   | v59+ | v41+ | v20+ | Uploader v20 adds the `document_group_id` field monitoring's traceability view reads |

   This is genuinely useful and genuinely used in practice on multi-service platforms — but it's
   **advisory**, not enforced: nothing in the pipeline reads this table before a rollback runs, and
   nothing forces it to be updated the moment a compatibility-relevant change ships. It rots the same
   way any hand-maintained document rots: correct on the day it's written, silently stale six weeks
   later.

2. **A cross-service version-check step**, added as a manual or scheduled (not release-blocking) stage
   after any deploy: hit each service's own `/version` endpoint (a trivial route each service already
   needs for basic ops visibility), record the triple, and compare it against the compatibility table
   above — flagging a mismatch in a Slack/Teams notification rather than blocking anything. This is the
   direct analog of course 05 chapter 05's "search, remove, re-upload" manual workaround: **the pieces
   to check for this today already exist and already work** — three `/version` endpoints and a
   comparison against a table — the gap isn't that it's impossible to check, it's that nothing requires
   or automates the check before or after a rollback actually happens.

**Caveat worth stating alongside this, precisely because it mirrors course 05's chapter 05 caveat:**
a manual compatibility check that runs *after* a rollback has already gone out only tells you there's a
problem — it doesn't prevent the bad combination from serving real traffic in the meantime. That gap is
exactly what Part 4's proposed design is meant to close.

## Part 3 — Two concerns that are easy to conflate, and shouldn't be

It's tempting to treat "the rollback succeeded" as the end of the story, because each service's own
pipeline reports a clean, green result. Conflating these two questions is the exact mistake worth
actively avoiding in an interview answer:

- **"Is this service's own version healthy?"** — Did *this service's* build, test, deploy, and
  smoke-test pipeline succeed for the version now running? This is entirely what chapter 02's pipeline
  and chapter 04's rollback mechanics answer, and they answer it well: a chatbot rollback to v55, on its
  own terms, is healthy the moment its own smoke test (hit `/health`, hit a real `/chat` call, confirm a
  `200` and a well-formed response) comes back green.
- **"Is this service's version compatible with what the other two services currently expect?"** — This
  is a completely different, *combinatorial* question that no single service's pipeline can answer by
  itself, because each pipeline's smoke test only ever exercises that one service in isolation against
  its own contract. Chatbot v55's smoke test has no way of knowing that monitoring v41 — deployed
  independently, on its own schedule, with no dependency in either direction on the chatbot pipeline —
  now expects every conversation event to include a `citations[]` array that v55 doesn't produce.

Make this concrete with the exact failure mode: the chatbot ships v58, which turns out to have a
formatting regression, so it's rolled back to v55 — a fast, routine slot-swap-back, exactly as chapter
04 describes, and it reports success because v55's own contract (status code, response shape *for
v55's own schema*) is intact. But `citations[]` was introduced specifically in v56, because monitoring
v41's Ragas-based groundedness scoring (course 04) was changed to read that field directly. Monitoring
is untouched by the chatbot's rollback — it's still v41, still expecting `citations[]` — so every
conversation the rolled-back chatbot produces from that point forward is missing a field the monitoring
platform's evaluation pipeline depends on. Depending on how defensively that parsing code was written,
the result is either a hard error in the evaluation pipeline (best case — loud, immediately visible) or
a silent default to an empty citations list that gets scored as "ungrounded" (worst case — the
model-risk dashboard's hallucination-rate trend line quietly spikes for a window of otherwise-normal
conversations, and whoever reviews that dashboard for model-risk sign-off has no way to tell the spike
is a version-compatibility artifact rather than a real regression in chatbot quality). The chatbot's own
pipeline reported a clean, successful rollback the entire time — it had no way to know it had just
broken something two services away.

## Part 4 — A genuine proposed design: a versioned, atomically-deployed release manifest

This is explicitly a **proposed design, not something implemented** — labeled as such throughout. A
real fix needs a durable artifact that represents "the full state of the platform," not just three
independent green pipeline runs.

**A release-manifest artifact, versioned in its own small repo (or a `platform/` folder shared across
the three service repos), recording the currently-intended combination per client:**

```json
{
  "manifest_version": 17,
  "client": "hsbc",
  "generated_at": "2026-07-20T14:03:00Z",
  "services": {
    "chatbot":    { "version": "v58", "image_tag": "58", "schema_version": 3 },
    "monitoring": { "version": "v41", "requires_chatbot_schema_version_gte": 3 },
    "uploader":   { "version": "v22", "requires_chatbot_schema_version": null }
  }
}
```

**A compatibility matrix, versioned alongside the manifest** (the machine-readable form of Part 2's
wiki table) — a small, structured file mapping schema/version ranges to what's known-compatible, so a
proposed manifest can be validated programmatically rather than by a human remembering a wiki page.
Notebook `04_version_compatibility_matrix_validator.ipynb` in this course's `notebooks/` folder
implements exactly this check, standalone.

**Deployed atomically, as the last stage of a "platform release" pipeline** that runs *after* all three
services' individual pipelines have each independently passed their own build/test/deploy/smoke-test
sequence (chapters 02 and 04) — the manifest is not written by any one service's pipeline on its own,
precisely because no single service's pipeline has visibility into the other two. This platform-release
stage:

1. Reads the currently-recorded manifest for the target client environment (from an Azure Storage blob
   or a small Azure App Configuration store — either works; the requirement is just that it's a single,
   shared, read-consistent location per client, not three services independently guessing at each
   other's state).
2. Computes the *proposed* new manifest — whichever service just deployed or is being rolled back,
   plus the other two services' versions **unchanged** from the current manifest.
3. Validates the proposed manifest against the compatibility matrix. If the resulting triple is a
   known-good combination, it commits the new manifest and the release is considered platform-complete.
   If it isn't a known-good combination, the stage **fails the platform release** — even though the
   individual service's own deploy and smoke test both went green — and requires an explicit,
   second-approver override to proceed anyway (for the rare, legitimate case of "we know this combo is
   fine, the matrix just hasn't been updated yet").
4. Crucially, this applies to **rollbacks** exactly the same way it applies to forward deploys — a
   rollback is not a special case that skips the check just because it's "going backward." The chatbot
   v55 rollback in Part 3's example would have been caught right here: the proposed manifest
   (`chatbot: v55, monitoring: v41, uploader: v22`) fails validation against the compatibility matrix
   the moment it's computed, *before* v55 fully takes over production traffic, not six hours later when
   someone notices the model-risk dashboard looks wrong.

The honest limitation worth naming: this only works if every service's pipeline is disciplined about
calling into the shared platform-release stage rather than treating its own rollback as complete once
its own smoke test passes — which means the platform-release stage has to be the *only* path to
production for any of the three services, not an optional extra step a hurried on-call engineer can
skip during an incident. That's a process/governance requirement as much as a technical one, and it's
worth naming candidly rather than presenting the manifest as a fix that enforces itself.

**A brief, honest note on two related but genuinely distinct forms of drift**, since they're easy to
lump into "versioning problems" but aren't the same failure mode as the cross-service compatibility gap
above: a Terraform state file going stale because someone made an out-of-band change directly in the
Azure Portal (the *infrastructure* Terraform believes exists no longer matches what's actually
deployed), and a variable-group value going stale after a secret is rotated outside the pipeline's own
rotation step (the *credential* a pipeline references no longer matches what the target resource
actually accepts). Both are real, both are worth being able to discuss fluently, and both are covered
with concrete bug narratives in chapter 07 — they're forms of environment drift, but they're drift in
infrastructure state and credentials, not drift in which *application version* is compatible with which,
so this chapter keeps them separate rather than conflating three different failure modes under one
label.

## Part 5 — The forward-looking tie to monitoring: version-tagging every unit of data, not just every deploy

This is the point worth volunteering to show systems-level thinking, the same way course 05's chapter 05
ties versioning forward into retrieval: **fixing this only at the CI/CD layer is necessary but not
sufficient.** The release-manifest design in Part 4 stops a known-bad combination from *staying* live —
but the model-risk-monitoring platform (course 04) is the system that has to *interpret* metrics
correctly across whatever version boundary already happened, including the brief window between a bad
deploy going out and the platform-release check catching it.

Concretely: every conversation event the chatbot emits should carry its own `schema_version` (or
`chatbot_version`) as a field on the event itself, not just as metadata the CI/CD pipeline tracks
separately. That lets monitoring's evaluation pipeline (the Ragas-based hallucination/robustness/
harmfulness scoring from course 04) do two things it currently has no way to do if the version is only
tracked at the deploy-manifest level: **(a)** select the correct parser/expected-schema for that specific
event, rather than assuming every event in a batch matches the *currently* deployed chatbot version, and
**(b)** explicitly exclude, flag, or re-score events that fall in a known bad-combination window instead
of silently averaging them into the same trend line as every other conversation. Without per-event
version tagging, a rollback boundary is invisible to the monitoring platform after the fact — it has no
way to distinguish "these five minutes of conversations were scored during a genuine known-incompatible
window" from "the chatbot actually got worse," which is precisely the ambiguity a model-risk dashboard
can least afford to have, since its entire purpose is telling a bank's model-risk function whether the
deployed model's behavior is still within approved bounds.

## The full interview-ready answer, assembled

> "Each of the three services — chatbot, monitoring, uploader — has its own pipeline, its own image
> versioning, and its own rollback mechanics, and each of those is solid in isolation: a slot-swap
> rollback for any one of them is fast and low-risk on its own terms. What's missing is a shared
> record of which combination of all three versions is actually known to work together — today,
> nothing stops you from rolling back the chatbot to a version whose response schema the monitoring
> platform's evaluation pipeline no longer understands, because the chatbot's own smoke test only
> checks the chatbot's own contract, not what monitoring currently expects. The manual stopgap is a
> compatibility matrix and a `/version`-endpoint cross-check — the pieces exist, they're just advisory,
> not enforced, and they run after the fact rather than blocking anything. The design I'd actually build
> is a release manifest — a small, versioned artifact per client recording the current triple of
> service versions — validated against a compatibility matrix as the very last stage of a
> platform-level release pipeline that runs after each service's own pipeline, and that gate applies to
> rollbacks exactly the same way it applies to forward deploys, so a bad combination gets caught before
> it serves traffic instead of after someone notices the monitoring dashboard looks wrong. And the fix
> doesn't stop at CI/CD — the monitoring platform itself needs every conversation event tagged with the
> chatbot version that produced it, so it can interpret a version boundary correctly instead of quietly
> averaging incompatible data into one trend line."

## Tying It Back

The strongest part of this answer isn't the manifest design in Part 4 — it's Parts 1 and 3: stating
plainly that today's three independent, well-built pipelines have no shared concept of cross-service
compatibility, and drawing a clean line between "this service's own rollback succeeded" and "the
resulting platform-wide combination is one that's actually safe" so the interviewer can see those two
questions are never being confused under pressure. As with course 05's chapter 05, confidence about
what's *not* built — and precisely why that gap matters — is just as strong a signal as the proposed
fix itself.
