# 06 — Protocol Amendment Versioning and Document Revision Handling

## Why this chapter exists

This chapter exists because this course has the single strongest built-in analog to the "how do
you handle revised versions of the same document" question anywhere in the curriculum — and it
isn't a stretch to find it. One of this platform's own modules is **SOC (Summary of Changes)**,
and SOC exists *specifically* to track what changed between one clinical trial Protocol version
(amendment) and the previous one. A clinical trial Protocol is amended routinely — a new
eligibility criterion added after a safety signal, a dosing adjustment, a new site added that
triggers a country-specific addendum — and every amendment is exactly the "revised version of the
same document" scenario, except here it isn't an administrative document with low stakes: an
amendment that isn't correctly propagated to the ICF a participant reads before consenting is a
patient-safety issue, not a bookkeeping inconvenience.

**An honesty note before anything else, because this domain doesn't tolerate ambiguity about what's
verified and what isn't**: unlike course 3 (rebuilt from an actual source repository with function
names and confirmed bug reports), there is no source repository backing this course. Everything
below — what a realistic implementation does today, the gap, the proposed design — is a plausible,
technically detailed **reconstruction** built from the resume bullet, the general shape of ICF/
PLPS/SOC/Protocol-amendment workflows in clinical research, and standard practice for versioned
document-generation systems. It is written with the same confidence and specificity as course 3's
chapter so it reads as a strong interview answer, but every claim about "what the platform does" is
illustrative, not a verified fact about a real codebase. Say so plainly if an interviewer asks
whether this is confirmed implementation detail — the honest framing is itself part of a strong
answer in a patient-safety domain.

## Part 1 — What a realistic implementation does today

A protocol amendment is not a rare event — most clinical trials see several over their lifetime.
A realistic version of this pipeline treats an amendment as a first-class trigger, not a manual
re-run someone has to remember to kick off:

1. **A new Protocol document version lands** (uploaded to the per-client S3 bucket, chapter 00's
   architecture) and is assigned the next `protocol_version` — a monotonically increasing integer
   per protocol (`v1`, `v2`, `v3`, ...), not a free-text label, so "which version is later" is
   never ambiguous.
2. **It goes through structuring and parsing (chapter 01) exactly like a first version does** —
   section/heading detection, table extraction, boundary-preserving chunking — the amendment isn't
   treated as a diff-only operation at the structuring stage; the whole amended document is
   re-parsed into hierarchical section objects, because an amendment can shift section numbering
   or restructure content around the changed clause, and a plausible implementation shouldn't
   assume the amendment is isolated to exactly the paragraph that changed.
3. **ICF and PLPS sections affected by the amendment are regenerated** from the new version's
   structured sections, using the same per-section-type instruction-based generation pipeline
   (chapter 05) as a first-draft generation would use — same prompt templates, same grounding
   checks, same routing to a human reviewer. A realistic implementation narrows *which* sections
   need regeneration by comparing section-level content hashes or the heading/requirement IDs
   touched by the amendment, rather than blindly regenerating the entire ICF/PLPS from scratch
   every time — but that's an optimization, not a correctness requirement; regenerating everything
   is always a safe (if wasteful) fallback, and a plausible system would fail toward "regenerate
   too much" rather than "regenerate too little" when the narrowing logic is uncertain.
4. **A SOC entry is generated per changed section**, using exactly the `SOC_ENTRY_TEMPLATE` from
   chapter 05 — `previous_text` populated from the section as it read in the immediately-prior
   Protocol version, `current_text` from the new version, output as the structured
   `{"section": ..., "change_type": "added|removed|modified", "summary": ..., "rationale_if_stated": ...}`
   object. The word "immediately-prior" is doing real work here and is the subject of Part 4 below.

That's the honest, confident first sentence of this chapter's interview answer: *"A protocol
amendment bumps a version number, triggers re-parsing of the amended document, regenerates the
ICF and PLPS sections the amendment touches, and generates a SOC entry diffing the new section text
against the immediately-prior version — all before any of it reaches a reviewer."*

## Part 2 — The gap: SOC diffing is textual, not graph-based, for revisions

Chapter 04 built a genuinely good argument for why Country Specific Protocol Comparison uses a
graph model instead of a flat text diff: a text diff can't tell you that a changed lab-reference
requirement also affects a downstream eligibility criterion that references it. That same
blind spot exists — and, in a plausible implementation, likely goes unaddressed — for
**version-over-version SOC generation**, because the SOC entry template in chapter 05 takes
`previous_text`/`current_text` as plain strings, section by section. That's a textual, single-hop
comparison: it tells you section 5.2 changed, but if section 3.2 (an eligibility criterion)
`REFERENCES` section 5.2 (a lab requirement) the way `REQ-ELIG-02` references `REQ-LAB-01` in
chapter 04's own example graph, a text-only SOC pipeline has no mechanism to flag that section 3.2
is *also* affected, even though nothing in its own text literally changed.

This is worth stating plainly as the honest gap, in the same register course 3's chapter 05 states
its gaps: *"the graph-based comparison machinery already exists in this platform for Country
Specific Comparison — modeling requirements as nodes and `REFERENCES` edges as first-class — but a
realistic implementation likely hasn't wired that same graph into version-over-version SOC
generation yet. SOC today is probably still a per-section text diff, which means a downstream
requirement that depends on an amended section, but wasn't itself edited, could be silently
missing from the SOC — and if it's missing from the SOC, nobody flags it for ICF/PLPS regeneration
either, because the trigger in Part 1 is 'this section's text changed,' not 'this section's meaning
was affected by something upstream.'"* The manual workaround available today, until the graph is
wired in for revisions specifically: a reviewer (or the medical writer preparing the SOC) manually
re-reads the sections that `REFERENCES` an amended section — using the same cross-reference
knowledge chapter 04's graph already encodes structurally — and adds any affected-but-unedited
section to the SOC and the regeneration queue by hand. That's exactly the same shape of gap as
course 3's "the pieces exist, they're just not wired together automatically" framing — the fix
isn't a new capability, it's reusing an existing one across a dimension (time/versions) it wasn't
originally built for (space/countries).

## Part 3 — Two concerns that are easy to conflate, and must not be

This is the crux of a strong answer here, and it's worth stating as two separate sentences an
interviewer can hold in their head independently:

- **Concern A: "the SOC accurately lists what changed."** This is a content-accuracy question
  about one document — did the SOC correctly identify every substantive change between Protocol
  version N-1 and version N, including downstream/referenced changes (Part 2's gap)? A SOC can be
  wrong in the same way any generated document can be wrong: it can miss a change, misclassify a
  change's type, or (per chapter 05's grounding defenses) state something not actually supported by
  either version's text.
- **Concern B: "every dependent document has actually been regenerated from the new version, and
  none of them are silently still reflecting the old one."** This is a *completeness and
  consistency* question across the whole set of derivative documents, not an accuracy question
  about any single one. A platform could produce a flawless, perfectly accurate SOC for amendment
  v2 — Concern A fully satisfied — while the ICF a participant is actually handed at the site was
  never regenerated and is still rendering v1's eligibility criteria. Those are two entirely
  different failure modes, and a system that only ever measures Concern A (SOC accuracy) has no
  way of detecting a Concern B failure (a stale ICF) at all, because the SOC and the ICF are
  different artifacts tracked, in this gap, by no shared mechanism.

The parallel worth drawing explicitly, because it's the same mistake in a different course: course
3's chapter 05 warns against conflating IWPB's approval/expiry state machine with actual document
versioning — "has this upload been approved" is a different axis from "is this the current
revision." The parallel here is exact: "does the SOC correctly describe the amendment" is a
different axis from "has every document that depends on the Protocol actually been rebuilt against
it." A platform can be excellent at one and silently broken on the other, and an interviewer asking
"how do you make sure nothing still reflects the old version" is asking about Concern B
specifically — a good answer names both concerns, states clearly which one SOC generation solves,
and doesn't let a good SOC stand in as evidence that Concern B is handled too.

## Part 4 — A proposed design: version propagation, dependency tracking, and the rapid-amendment chain

This is explicitly a **proposed design**, offered as the answer to "how would you actually close
this gap" — not a description of a built feature.

**1. Propagate a `protocol_version` field through every generated document, not just SOC.**

```python
# Proposed data model additions — illustrative, not implemented

class GeneratedDocument:
    document_id: str
    module: str              # "ICF" | "PLPS" | "SOC"
    section_id: str          # which structured Protocol section this was generated from/about
    protocol_version: int    # the Protocol version this document's CONTENT reflects
    generated_at: datetime
    review_status: str       # "pending_review" | "approved" | "rejected" | "needs_regeneration"

class SOCEntry(GeneratedDocument):
    diffs_against_version: int
    # MUST equal protocol_version - 1 for a valid SOC entry — see the rapid-amendment
    # handling below for why this is an explicit invariant, not an assumption.
```

Every ICF section, every PLPS section, and every SOC entry carries the `protocol_version` it was
actually generated from or about — not the version the *Protocol* is currently at, which may have
moved on since. This single field is what makes Concern B (Part 3) checkable at all: without it,
"is this ICF stale" is an unanswerable question, because nothing records what version it reflects.

**2. A dependency-tracking mechanism that flags stale dependents as `needs_regeneration`.**

```python
def flag_stale_dependents(protocol_id: str, current_protocol_version: int, documents: list[GeneratedDocument]):
    """Run whenever a Protocol's version advances. Any ICF/PLPS document whose
    protocol_version trails the Protocol's current version is not wrong — it's stale,
    and stale is a distinct, first-class state, not silently indistinguishable from current."""
    for doc in documents:
        if doc.module in ("ICF", "PLPS") and doc.protocol_version < current_protocol_version:
            doc.review_status = "needs_regeneration"
    return documents
```

The output of this check is not a log line nobody reads — it's a queue a regulatory-affairs
dashboard surfaces directly: "3 ICF sections and 1 PLPS section are flagged `needs_regeneration`
following Protocol v2 → v3." That queue is the concrete mechanism that answers "how do you make
sure nothing silently still reflects the old version" — the honest answer is never "the model
regenerates everything instantly and perfectly," it's "staleness is a tracked, visible state that
someone is accountable for clearing," which is a stronger and more honest claim than "it's always
current."

**3. Correct handling when amendments happen in quick succession.**

This is the detail worth being precise about, because it's the exact spot a sloppy implementation
gets wrong: if Protocol v2 is approved and, before its dependent ICF/PLPS regeneration and SOC
generation cycle has even finished, v3 lands (a second amendment shortly after the first — not a
hypothetical edge case in a live trial), the SOC for v3 **must diff against v2, not v1**. Skipping
v2 and diffing v3 straight against v1 would silently fold two separate amendments into one SOC
entry, which is exactly how a safety-relevant intermediate change could get buried inside a larger,
noisier combined diff instead of being called out on its own. The invariant worth enforcing in code,
not just in a design doc — `SOCEntry.diffs_against_version == SOCEntry.protocol_version - 1`,
checked and rejected if violated — is what guarantees the SOC chain is v1→v2, v2→v3, v3→v4, ...,
never v1→v3 with v2 skipped, regardless of how quickly amendments arrive relative to how fast the
regeneration pipeline can keep up. It is completely acceptable for the *regeneration* of dependent
ICF/PLPS sections to lag behind — that's what the `needs_regeneration` queue in step 2 is for — but
the *version-numbering and SOC-diffing bookkeeping* must never lag or skip, because that bookkeeping
is the ground truth every dependent regeneration and every reviewer relies on to know what "current"
even means.

## Part 5 — The tie back to the human-review gate: a version bump is not a fast path

Chapter 05 established that every generated section — clean grounding check or not — goes through
human regulatory-affairs review before use, as a hard gate, not a configurable setting. It would be
tempting, once `protocol_version` propagation and automated staleness-flagging exist, to treat an
amendment-triggered regeneration as lower-risk than a first-draft generation — "it's just a version
bump, the reviewer already approved this section once" — and fast-track it past review. That
reasoning is exactly backwards, and worth naming explicitly as a design decision this proposal
does **not** make: an amendment is very often the *reason* a section needs the most scrutiny, not
the least — a Protocol is amended precisely when something about the trial's design, eligibility,
dosing, or safety monitoring changed, which means the regenerated ICF/PLPS content is describing a
materially different thing than what the reviewer approved last time, even if the surrounding
prose structure looks similar. Every regenerated document, automated trigger or not, goes through
the identical grounding-check-then-human-review pipeline as a first-time generation — the only
thing the version-propagation design in this chapter changes is *whether staleness is visible and
tracked*, never *whether review is required*.

## The full interview-ready answer, assembled

> "This platform already has a module built specifically for this — SOC, the Summary of Changes,
> exists to track exactly what changed between Protocol amendments. A realistic implementation
> bumps a protocol_version on a new Protocol upload, re-parses the amended document, regenerates
> the ICF and PLPS sections the amendment touches, and generates a SOC entry diffing the new
> section text against the immediately-prior version. The gap I'd expect, and would go looking for,
> is that SOC diffing is likely still textual, section-by-section — not wired into the same
> graph-based comparison chapter 04 built for country comparisons, so a downstream requirement that
> references an amended section but wasn't itself edited could be missed by both the SOC and the
> regeneration trigger. And there are genuinely two different things to get right here, not one:
> whether the SOC accurately lists what changed, and whether every dependent document — the ICF, the
> PLPS — has actually been regenerated from the new version, with none of them silently still
> reflecting the old one. Those can fail independently. My proposed fix is a protocol_version field
> propagated through every generated document, a dependency check that flags any ICF/PLPS whose
> version trails the Protocol's current version as needs_regeneration on a reviewer-facing queue,
> and an explicit invariant — enforced in code, not just assumed — that a SOC entry always diffs
> against the immediately-prior version, so if amendments land in quick succession, nothing skips a
> version and buries an intermediate change inside a bigger diff. And critically, none of that
> changes the review requirement — an amendment-triggered regeneration is not a fast path, because
> an amendment is usually exactly when a section needs the most scrutiny, not the least. Every
> regenerated document still goes through the same grounding check and the same human
> regulatory-affairs reviewer as a first draft would."

## Tying It Back

The strongest thing about this chapter isn't the proposed `protocol_version`/dependency-tracking
design in Part 4 — it's Part 3's clean separation of two concerns that sound like one question but
aren't: SOC accuracy and dependent-document staleness are different failure modes, checked by
different mechanisms, and a candidate who can hold that distinction under follow-up questions is
demonstrating exactly the kind of systems thinking this domain rewards. As with the rest of this
course, every implementation claim above is an illustrative, technically-grounded reconstruction —
there is no source repository behind this platform the way there is for course 3 — so the honest
framing to use in an interview is "here's how I'd expect this to work, here's the gap I'd expect to
find, and here's how I'd close it," not "here's exactly what the code does." That framing is not a
weaker answer than a source-confirmed one; in a patient-safety domain, precision about what's
verified versus reasoned-through is itself a signal of the judgment this kind of system demands.
