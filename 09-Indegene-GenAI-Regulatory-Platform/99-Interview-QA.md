# 99 — Interview Q&A: GenAI Regulatory Platform (ICF / PLPS / SOC)

Read this last, after the chapters are fresh. Questions are grouped behavioral -> technical deep-dive
-> system design -> retrospective, roughly the order a real interview loop tends to move through them.

**Q2 is the question most worth over-preparing for.** This course has a module — SOC (Summary of
Changes) — whose entire purpose is tracking what changed between clinical trial Protocol versions,
which makes "how do you handle a revised version of a document" map onto this project more directly
than almost anywhere else in this curriculum. It's placed second, immediately after the project
walkthrough and ahead of the rest of the behavioral warm-up, along with three sharp technical
follow-ups (Q3-Q5) an interviewer is likely to chain onto it. As with the rest of this course, there
is no source repository behind these answers — they are a deliberately specific, technically
grounded *reconstruction*, not a verified account of a real implementation, and that honest framing
is itself part of a strong answer here given the patient-safety stakes.

---

## Behavioral

### Q1. Tell me about this project — walk me through it end to end.

Use the STAR summary from `00-README.md` as the 90-second skeleton, then be ready to expand on any
piece. Short version: "Indegene runs clinical trials, and every trial's Protocol document has to be
turned into several derivative documents — an Informed Consent Form for patients, a Plain Language
Protocol Synopsis, a Summary of Changes across amendments, and country-specific comparisons. I built
end-to-end pipelines that structured the Protocol document, used instruction-based generation with a
fine-tuned LLaMA 3 to draft each module's sections, and used a graph representation with Cypher
queries for the country comparison — with every generated section routed to a human reviewer before
use, because inaccurate patient-facing or regulatory content isn't just a quality issue, it's a
patient-safety and compliance risk."

### Q2. When a clinical trial protocol is amended, how do you make sure every dependent document reflects the new version and none of them silently still reflect the old one?

This is the question this course is best-built to answer, because one of the platform's own
modules — SOC — exists specifically to track version-to-version protocol change. The full worked
answer lives in chapter 06
(`06-protocol-amendment-versioning-and-document-revision-handling.md`); this is the compressed,
interview-ready version:

> "A realistic implementation of this bumps a `protocol_version` on every new Protocol upload,
> re-parses the amended document through the same structuring pipeline a first version goes
> through, regenerates the ICF and PLPS sections the amendment touches, and generates a SOC entry
> diffing the new section text against the immediately-prior version. The gap I'd expect, and would
> go looking for, is that SOC diffing is probably still textual — section by section — rather than
> wired into the same graph-based comparison built for Country Specific Protocol Comparison, so a
> downstream requirement that references an amended section, but wasn't itself edited, could be
> missed by both the SOC and the regeneration trigger.
>
> And there are genuinely two different things to get right here, not one: whether the SOC
> accurately lists what changed, and whether every dependent document — the ICF, the PLPS — has
> actually been regenerated from the new version, with none of them silently still reflecting the
> old one. Those can fail independently — you can have a perfectly accurate SOC while a participant
> is still being handed an ICF built from the old protocol. My proposed fix is a `protocol_version`
> field propagated through every generated document, a dependency check that flags any ICF/PLPS
> whose version trails the Protocol's current version as `needs_regeneration` on a reviewer-facing
> queue, and an explicit, code-enforced invariant that a SOC entry always diffs against the
> immediately-prior version — so if amendments land in quick succession, nothing skips a version
> and buries an intermediate change inside a bigger diff.
>
> And critically, none of that changes the review requirement. An amendment-triggered regeneration
> is not a fast path — an amendment is usually exactly when a section needs the *most* scrutiny, not
> the least, because it means something about the trial's design, eligibility, dosing, or safety
> monitoring actually changed. Every regenerated document still goes through the same grounding
> check and the same human regulatory-affairs reviewer a first draft would."

Say plainly, if asked directly, that this is a reasoned reconstruction rather than a description of
a verified codebase — in a patient-safety domain, being precise about what's confirmed versus
reasoned-through is itself part of the answer, not a hedge that weakens it.

### Q3. What's actually different between "the SOC accurately lists what changed" and "every dependent document has been regenerated"? Can one be true while the other is false?

Yes, and demonstrating that they're genuinely independent is the strongest way to answer this
follow-up. SOC accuracy is a content question about *one document*: did it correctly identify every
substantive change, including anything affected transitively through a requirement reference, between
two Protocol versions? Dependent-document staleness is a *completeness/consistency* question across
the *whole set* of derivative documents: has the ICF a participant actually reads been rebuilt from
the current version, or is it still, silently, built from an older one? Chapter 07's second bug
narrative is the concrete version of exactly this failure: a reviewer approved a PLPS section
believing they were looking at the newly regenerated v2 draft, when a caching layer — keyed on
`section_id` alone, not on `protocol_version` — served them a stale rendering of the v1 content, even
though the database correctly held the new v2 text underneath. The SOC for that same amendment could
have been flawless, and the staleness bug would still have gone completely undetected by anyone
checking SOC accuracy alone, because they measure different things. The fix for that class of bug is
narrow and specific — cache keys must include `protocol_version`, not just the document identifier —
which is exactly why naming the two concerns separately matters: a fix for one doesn't automatically
cover the other.

### Q4. What happens if two amendments to the same protocol land close together — does the SOC diff ever skip a version?

It must not, and this is the detail worth being precise about rather than hand-waving past. If
Protocol v2 is approved and, before its dependent ICF/PLPS regeneration has even finished, v3 lands,
the SOC for v3 has to diff against v2 — not v1. Diffing v3 straight against v1 would silently fold two
separate amendments into a single combined SOC entry, which is exactly how an intermediate,
safety-relevant change could get buried inside noisier, aggregated diff noise instead of being called
out as its own event. Chapter 06's proposed design makes this an enforced invariant, not an
assumption: a SOC entry's `diffs_against_version` field must equal `protocol_version - 1`, checked
and rejected outright if violated, so the SOC chain is always v1→v2, v2→v3, v3→v4 — never a shortcut
that skips one. It's fine, and expected, for the *regeneration* of dependent ICF/PLPS sections to lag
behind a rapid sequence of amendments — that's what a `needs_regeneration` queue is for — but the
version-numbering and SOC-diffing bookkeeping itself must never lag or skip, because every reviewer
and every staleness check relies on that bookkeeping being unambiguous. Chapter 07 also describes the
concrete failure mode behind this: two amendment-processing jobs racing to assign the *same*
`protocol_version` number without a database-enforced uniqueness constraint, producing two documents
that both claim to be "version 3" — the fix there is the same kind of enforced-invariant thinking,
just at the version-assignment layer instead of the SOC-diffing layer.

### Q5. Walk me through a specific bug you'd expect in a pipeline like this, and what would have caught it earlier.

Pick one and go deep rather than listing all of chapter 07's bugs shallowly — the chunking omission
is the strongest choice because it's the clearest patient-safety near miss: a Protocol section had an
unusually long paragraph where a dosing instruction was immediately followed, in the same paragraph,
by its safety-monitoring caveat ("...administer 100 mg every 4 weeks. Discontinue immediately if
injection-site reaction Grade 2 or higher occurs."). The chunker's fallback path split that paragraph
mid-sentence when it exceeded the token budget, so the generation call for the first chunk never saw
the caveat at all — and because the grounding check verifies that generated claims *are* supported by
the source, not whether the source's claims all *survived into* the output, an omission like this
sails through grounding checks cleanly and reaches a reviewer as a "clean" draft. That asymmetry is
the real lesson: grounding checks catch fabrication, not omission, and those need separate test
coverage. What would have caught it earlier is a golden-document regression test suite — a small set
of real or realistic Protocol sections with hand-verified expected chunk boundaries and a list of
sentences that must never be split apart, run on every change to the chunking logic, so a caveat
silently disappearing across a chunk boundary fails a test immediately rather than reaching a
reviewer's screen and depending on them catching it by re-reading the full source themselves.

---

## Technical deep-dive

### Q6. Why did human review stay in the loop instead of automating full publication? Wasn't that leaving efficiency on the table?

This is really a "do you understand responsible AI in a regulated domain" question. Good answer: the
efficiency gain was never meant to come from removing the human, it was meant to come from removing
the *blank page* — a medical writer reviewing and lightly editing an AI-drafted ICF section is much
faster than writing it from scratch, but they're still the accountable party who verifies it against
the Protocol before anything reaches a patient or regulator. Removing that step doesn't just add
risk, it changes who's accountable when something is wrong, which isn't a decision an engineering
team should make unilaterally in a domain like this.

### Q7. Describe a time a generated section needed correction. How did you find out, and what did you do?

Frame generically if you don't have a specific real incident to cite, but describe the *mechanism*
honestly: "The grounding check or the reviewer would flag a section — say, a generated PLPS paragraph
that paraphrased a lab value threshold slightly differently than the source stated it. The fix wasn't
just correcting that one instance; it was tracing whether the prompt template or an edge case in the
grounding check let it through, and feeding the reviewer's correction back into the next fine-tuning
round so the same class of error was less likely to recur." This demonstrates you think about
systemic fixes, not just one-off patches.

### Q8. How did you prioritize which module (ICF, PLPS, SOC, Country Comparison) to build first, or which to invest most in?

A reasonable, defensible answer: prioritize by a combination of (a) manual-effort volume — which
document type recurs most often per Protocol amendment — and (b) risk profile — patient-facing content
(ICF, PLPS) generally needs tighter grounding and review investment than internal regulatory-affairs
content (SOC, Country Comparison) even if the latter is technically simpler to generate. Be ready to
say you'd sequence the lower-risk, high-volume wins (e.g., SOC, which is more structurally templated)
early to build reviewer trust in the pipeline before tackling the highest-stakes patient-facing
content.

### Q9. How do you prevent an LLM from hallucinating a clinical claim not present in the source protocol?

Layered defense, not one silver bullet (full detail in Chapter 05): (1) scope every generation call to
the specific structured source section, never the whole document — narrows what the model can draw
from; (2) explicit prompt instructions to avoid unsupported claims and prefer omission over invention;
(3) an automated post-generation grounding check that extracts key terms/claims from the output and
verifies they're present in the source, flagging mismatches; (4) optionally, a second model call whose
sole job is to list unsupported statements in the generated text against the source — a narrower,
easier task than generation itself; (5) all of the above feed a human reviewer, who is the actual
final check — automated grounding checks reduce reviewer burden, they don't replace the reviewer. Worth
adding, per chapter 07: grounding checks catch fabrication, not omission — a caveat that got silently
dropped during chunking is a different failure mode that needs its own test coverage (Q5).

### Q10. LoRA fine-tuning vs. full fine-tuning vs. prompt engineering — how did you decide which to use for LLaMA 3 here?

Think of it as a ladder of increasing cost and increasing behavioral control (Chapter 02). Prompt
engineering alone is cheapest and was likely sufficient for tasks close to what an instruction-tuned
base model already handles well. Full fine-tuning is most expensive — you're updating every weight —
and generally overkill unless you need to change something fundamental about the model's underlying
knowledge or behavior, which wasn't the case here: the model already "knows" clinical language broadly
from pretraining. LoRA sits in the sweet spot: it adapts the model's *behavior* (reliably following
ICF/PLPS/SOC format, register, and terminology conventions) without touching the base weights, is
cheap enough to train a separate adapter per module/section-type, and preserves the base model's
general competence since the frozen weights never move. The deciding factor was that the required
change was narrow and behavioral (format/register discipline), not a change to the model's underlying
domain knowledge — exactly the case LoRA is designed for.

### Q11. Walk me through what LoRA is actually doing, mathematically.

A pretrained weight matrix `W` (say `d × d`) is frozen. Instead of learning a full update `ΔW`
(`d²` parameters), LoRA learns two much smaller matrices `A` (`r × d`) and `B` (`d × r`) with rank
`r << d`, and the effective weight becomes `W + B·A`. Only `A` and `B` are trained. Because `r` is
small (often 4-64) relative to `d` (often in the thousands for a modern LLM's hidden dimension), the
trainable parameter count for that layer drops from `d²` to `2·d·r` — often under 1% of a full update.
The from-scratch numpy demo in `notebooks/02_instruction_finetuning_demo_lora.ipynb` makes this
concrete by directly comparing the parameter counts of a full update versus a rank-`r` decomposition
on the same matrix shape.

### Q12. Why model Country Specific Protocol Comparison as a graph instead of a simple diff between two documents?

A flat text diff operates at the line/token level and has no concept of *what kind* of thing changed
or how requirements relate to each other. A graph model represents Protocol requirements as nodes and
country-specific variations as `MODIFIES`/`ADDS` edges onto those nodes, and — critically — represents
cross-requirement dependencies (`REFERENCES` edges) explicitly. That means you can answer not just
"what changed for Germany" (a single-hop query) but "what else is affected by what changed for
Germany" (a multi-hop traversal following `REFERENCES` edges outward from the changed nodes) — a
question a flat text diff structurally cannot answer, because it has no model of which requirements
depend on which others. See Chapter 04 for the full Cypher examples, and chapter 07 for a candid bug
story about getting that traversal only partially right — a one-hop-only implementation that missed a
two-hop dependency (Q5's sibling story in the graph domain).

### Q13. Write (or explain) a Cypher query to find every requirement that a specific country's variant modifies.

```cypher
MATCH (c:CountryVariant {country: "Germany"})-[r:MODIFIES|ADDS]->(req:Requirement)
RETURN req.id, req.type, req.text, type(r) AS change_type
```
`MATCH` describes the pattern (a `CountryVariant` node with country property "Germany" connected by a
`MODIFIES` or `ADDS` relationship to a `Requirement` node), and `RETURN` pulls out the fields a
reviewer needs. Follow-up: extend it to also find downstream affected requirements by adding a second
`MATCH` clause that follows `REFERENCES` edges from `req` — this is Example 2 in Chapter 04. A sharper
follow-up still: what if the dependency is two hops away, not one? See notebook
`06_transitive_graph_diff_fix.ipynb` for the traversal that handles an arbitrary-depth chain rather
than only a single hop.

### Q14. How do you handle documents longer than the model's context window?

Two things, not one (Chapter 01). First, structure the document *before* generation — detect section/
heading boundaries so you know exactly where one section ends and the next begins, using a mix of
numbering-pattern regex, document-style cues (heading styles, font/size), and known-vocabulary
matching against standard Protocol section names. Second, chunk *within* that structure, never across
a section boundary — if a single section is still too long, split at the next-lowest structural
boundary (subsection, then paragraph) rather than at an arbitrary token count. The reason this matters
beyond just "fitting in the window": a naive fixed-size chunk can cut an eligibility criterion or
dosing rule in half, silently corrupting exactly the content whose completeness matters most in this
domain — chapter 07's chunking-bug story (Q5) is exactly this failure mode happening in practice
despite a boundary-aware design, which is why golden-document regression tests matter even when the
chunker is designed correctly in principle.

### Q15. What's the difference between a base LLM and an instruction-tuned model, and why does it matter here?

A base model is trained purely to predict the next token over raw text — it has no particular bias
toward "answering a request helpfully in a requested format," it just continues plausible text. An
instruction-tuned model has been further trained on `(instruction, input, output)` examples so it has
learned the *behavior* of following an instruction and producing output in the requested shape. For a
platform generating ICF/PLPS/SOC content at scale, across thousands of sections, you need that
reliability of format/behavior consistently, not "usually cooperative" — which is exactly why the
pipeline builds on an instruction-tuned base (LLaMA 3-Instruct or similar) and further specializes it
with LoRA fine-tuning per module, rather than prompting a raw base model.

### Q16. Why choose an open-weight model like LLaMA 3 over calling Azure OpenAI or AWS Bedrock directly?

Three converging reasons (Chapter 03): data residency/confidentiality — clinical Protocol content is
sponsor-sensitive and patient-safety-relevant, so keeping it inside a controlled environment rather
than sending it to a third-party API is a real requirement, not a preference; deep fine-tuning control
— LoRA/QLoRA fine-tuning needs direct access to model weights, which a closed API doesn't expose as
directly; and cost at sustained volume — a platform running continuous generation across many trials
and modules has enough steady inference volume that self-hosted infrastructure cost amortizes better
than linear per-token API pricing. That said, a managed API (Bedrock, Azure OpenAI) is still a
reasonable secondary backend for lower-sensitivity or exploratory work — it's a workload-by-workload
tradeoff, not an absolute rule.

### Q17. How would you evaluate whether the generated ICF/PLPS content is actually good, beyond "the reviewer approved it"?

A few complementary signals worth naming: automated grounding-check pass rate (what fraction of
generated sections have zero flagged unsupported claims before reaching a reviewer); reviewer edit
distance/edit rate (how much a reviewer had to change — a proxy for draft quality that's cheaper to
track than a full quality rubric); readability metrics for patient-facing text (e.g., a Flesch-Kincaid-
style reading-level score, since ICF/PLPS content has explicit plain-language requirements); and
format/schema compliance rate for structured outputs like SOC entries. None of these replace human
review, but tracking them over time tells you whether the pipeline (and its fine-tuning data) is
actually improving.

---

## System design

### Q18. Design a human-in-the-loop review workflow for AI-generated clinical documents that satisfies an auditability requirement.

Structure the answer around: **generation -> grounding check -> reviewer queue -> decision -> audit
log -> feedback loop.**

- **Generation** produces a draft section plus metadata: which source section(s) it was grounded in,
  which prompt template/model version/fine-tuned adapter version generated it, the `protocol_version`
  it reflects (chapter 06), and a timestamp.
- **Automated grounding check** runs immediately, attaching a flag (clean / needs-attention) and,
  where possible, which specific claims didn't match the source — this triages the reviewer queue so
  flagged items get priority attention rather than every section getting equal, undifferentiated
  scrutiny.
- **Reviewer queue** presents the draft alongside its source section, with matched/unmatched claims
  highlighted, so review time concentrates on what actually needs a human judgment call. The same
  queue also surfaces any document flagged `needs_regeneration` because a Protocol amendment moved
  past it (chapter 06) — staleness and grounding flags share one queue, not two separate blind spots.
- **Decision** is one of approve-as-is, approve-with-edits (capturing exactly what changed), or
  reject-and-regenerate — every decision is a discrete, recorded action, never an implicit "it's fine
  because no one complained."
- **Audit log** captures, immutably, per section: source document version, generation model/adapter
  version, prompt template version, reviewer identity, decision, timestamp, and (if edited) a diff of
  what the reviewer changed. This is what satisfies "auditability" concretely — given any published
  ICF/PLPS/SOC section, you can answer "who approved this, when, against what source version, and what
  did they change" without ambiguity.
- **Feedback loop**: reviewer edits become candidate training examples for the next LoRA fine-tuning
  round (Chapter 02), and aggregate flag rates by section type inform where prompt templates or
  grounding-check coverage need improvement.

Call out the tradeoff explicitly if asked: this workflow trades some speed (nothing is instant) for
traceability and accountability — which is the correct tradeoff in a domain where a wrong answer isn't
just a bad user experience, it's a patient-safety and regulatory-compliance failure.

### Q19. How would this system scale if Indegene tripled the number of trials it supports simultaneously?

Name the actual bottlenecks and how each is addressed: (1) generation throughput — self-hosted LLaMA 3
serving with continuous batching and multi-adapter serving (one frozen base, many LoRA adapters) scales
better than provisioning separate full models per module; (2) reviewer capacity — this is the real
constraint, since reviewers don't scale the way compute does; the grounding-check triage and edit-
distance tracking from Q17 become more important at scale, to make sure reviewer attention goes to the
sections that actually need it rather than being spread evenly; (3) graph size for Country Comparison —
a graph database handles growth in nodes/edges more gracefully than an equivalent relational join-chain
would, but query patterns should be checked against realistic scale (many trials × many countries ×
many requirements) rather than assumed to scale for free; (4) amendment concurrency — more simultaneous
trials means more simultaneous protocol amendments, which is exactly the concurrency caveat chapter 07
names (two amendments to the same protocol racing on which one "wins" as current) — at 3x trial volume,
a database-enforced version-assignment constraint stops being an edge-case nicety and becomes load-bearing.

---

## Retrospective

### Q20. What would you change if you rebuilt this today?

A strong answer names concrete, plausible improvements rather than "nothing" or vague platitudes:

- **Tighter automated grounding checks**, moving from term/entity-presence matching toward a
  dedicated, narrowly-scoped verifier model call (Chapter 05, defense #4) as a standard second pass for
  every generation, not just an occasional spot-check — catching subtler unsupported claims that
  simple string/entity matching misses, though note this still wouldn't catch an *omission* bug like
  chapter 07's chunking near miss — that needs a separate golden-document regression suite, not a
  better grounding check.
- **Richer graph schema for Country Comparison**, adding explicit provenance edges back to the exact
  source sentence/clause each requirement node was derived from, so a reviewer investigating a flagged
  difference can jump straight to the source text instead of re-locating it manually — and extending
  the same graph traversal to power version-over-version SOC diffing (chapter 06's named gap), so the
  transitive-reference blind spot gets closed once, for both country comparison and protocol-amendment
  diffing, instead of twice.
- **`protocol_version` propagation and dependency tracking from day one** (chapter 06), rather than
  retrofitting it — a `needs_regeneration` queue and an enforced SOC version-chain invariant are much
  cheaper to build into the data model at the start than to bolt onto a platform that's already treating
  "current version" as implicit.
- **Structured evaluation harness** — a held-out set of Protocol sections with human-approved reference
  ICF/PLPS/SOC outputs, scored automatically (grounding pass rate, schema compliance, readability) on
  every prompt-template or fine-tuning change, so pipeline changes are validated against a consistent
  benchmark instead of ad hoc spot-checking before rollout.
- **Multi-adapter serving from day one** (Chapter 03) rather than treating it as a later optimization —
  designing the serving layer around "one frozen base, swappable LoRA adapters per module" from the
  start avoids having to re-architect deployment once the number of specialized behaviors grows past
  what a single fine-tuned model can reasonably cover.

Close by reiterating the one thing you would *not* change: the human-in-the-loop review gate. Every
improvement above makes the reviewer's job faster and better-informed — none of them are framed as
"remove the reviewer," because that's the one design decision this project should never compromise on,
amendment-triggered regenerations included (chapter 06, Part 5).

---

## Client & production deployment

### Q21. This served two pharma clients, Eli Lilly and AstraZeneca, whose clinical trial protocols are highly confidential and competitively sensitive — how did your AWS architecture guarantee they could never be co-mingled or cross-exposed?

Lead with why this mattered more here than in a typical multi-tenant SaaS setup: Eli Lilly and
AstraZeneca compete directly in several therapeutic areas, so this isn't just "keep customer A's data
away from customer B" as a generic best practice — a leak would be a severe trust and legal incident
for Indegene specifically because the two clients are rivals. Then walk the isolation boundary at
every layer, not just one:

- **Storage**: separate S3 buckets per client (e.g. distinct buckets, not a shared bucket with
  per-client prefixes), each with its own bucket policy. Prefix-based separation inside one bucket
  relies on every IAM policy and every piece of application code getting the prefix scoping right,
  every time — a genuinely separate bucket makes an entire class of "forgot to scope the query"
  mistakes structurally impossible rather than just discouraged.
- **Identity/access**: separate IAM roles per client's pipeline, so the credentials the ECS Fargate
  structuring/orchestration service assumes for Eli Lilly's protocols have no policy path to
  AstraZeneca's bucket or vice versa — a compromised or misconfigured role for one client can't reach
  the other client's data even in the worst case.
- **Inference**: separate Sagemaker endpoints (or, at minimum, strict namespace/adapter isolation on a
  shared endpoint) for the fine-tuned LLaMA 3 model, so one client's in-flight request context and
  fine-tuning signal never mixes with the other's, even transiently in memory.
- **Comparison service**: the graph database backing Country Specific Protocol Comparison applies the
  same principle — no shared graph instance where a query written for one client's data could
  accidentally traverse into the other's.

Close with the meta-point: isolation here isn't a single control, it's defense in depth across
storage, identity, and inference — because for content this sensitive, you don't want the guarantee to
rest on any one layer being configured correctly.

### Q22. Why was self-hosting a fine-tuned LLaMA 3 on Sagemaker the right call here instead of using a closed third-party API, given the client base?

This is Q16's technical tradeoff answer, sharpened by the actual client base — use both together if
asked to go deep. The general tradeoff (Chapter 03) is data residency, fine-tuning control, and cost
at scale. What makes it concrete rather than theoretical here: Eli Lilly and AstraZeneca are direct
competitors, and both are sending genuinely unpublished, competitively sensitive clinical trial
protocol content into the platform. Neither client would accept — and no responsible vendor should
ask them to accept — that content being sent to a shared third-party inference API where you don't
control the infrastructure, logging, or retention. Hosting the fine-tuned model on a Sagemaker
real-time endpoint inside AWS infrastructure the platform's own IAM roles and VPC controls govern is
what lets you make a credible, specific claim to both clients: their protocol content stays inside a
boundary you control, full stop, rather than "we trust our API provider's security practices." That's
a materially stronger guarantee than a closed API can offer, independent of how good that provider's
own security is, simply because a closed API requires the data to leave your control boundary at all.
On top of that, deep fine-tuning control (LoRA per module) and cost at sustained daily volume across
two clients both point the same direction — but the residency argument is the one that would have been
disqualifying on its own for this client base, even if the other two didn't apply. Chapter 07's
concrete timeout/retry/auto-scaling values for that endpoint are worth having ready if this goes one
level deeper operationally.

### Q23. Walk through what happens in production if the human-in-the-loop reviewer rejects an AI-generated ICF draft — what's the actual workflow, and where does the system draw the line on autonomy?

Ground this in the review workflow from Q18, but narrate the rejection branch specifically, since
that's the case that actually tests whether "human-in-the-loop" is real or just a checkbox:

1. **The draft never reaches the client's submission workflow.** A rejection is a terminal state for
   that version of the draft — nothing downstream (document assembly, the client-facing regulatory
   submission pipeline) is triggered by a generation event alone; only an *approval* event triggers
   forward movement. This is the actual mechanism behind "nothing reaches a client unreviewed" — it's
   enforced by what's allowed to trigger the next pipeline stage, not just a policy statement.
2. **The rejection is logged with a reason**, same as an approval or an edit: reviewer identity,
   timestamp, source Protocol version, model/adapter version that generated it, and — critically — what
   specifically was wrong (an unsupported claim the grounding check missed, a tone/register issue, a
   missing eligibility condition). This is what makes the rejection useful beyond just "try again."
3. **Regeneration, not silent retry.** The section goes back through generation — potentially with
   reviewer feedback folded into the prompt for that regeneration attempt, or, if the rejection reveals
   a systematic issue (not a one-off), flagged for the prompt-template or fine-tuning-data owners to
   look at rather than just being individually retried.
4. **The reviewer's correction becomes training signal.** Consistent with Chapter 05's feedback loop,
   a reject-and-fix is exactly the kind of human-approved example that feeds the next LoRA fine-tuning
   round, so the same class of error is less likely to recur — the rejection isn't just a gate, it's a
   data point that improves the pipeline.
5. **Where the system draws the line on autonomy**: the model is trusted to draft and to flag its own
   uncertainty (via the grounding check), never to decide that a draft is good enough to skip review,
   and never to auto-correct a flagged claim on its own (auto-editing a flagged claim would just be the
   model guessing a second time, unsupervised). The only actor whose approval can move a section forward
   is a human reviewer — that boundary is fixed regardless of how good the grounding checks or the
   fine-tuned model get, because the risk being managed (patient-facing or regulatory-submission
   content) doesn't become lower-stakes just because the model's average quality improves. The same
   boundary applies, without exception, to an amendment-triggered regeneration (chapter 06, Part 5) —
   a version bump is never treated as grounds to skip this gate.
