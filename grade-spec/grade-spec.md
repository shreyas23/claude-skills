---
description: Grade a product spec against the 11-dimension rubric, then fix issues until the grade stabilizes or hits A-
---

Grade and iteratively improve a product specification using the rubric at `docs/plans/PRODUCT_SPEC_RUBRIC.md`.

**Input:** `$ARGUMENTS` is the path to the spec file (or directory of spec files). If empty, default to `docs/plans/TARGET_PRODUCT_SPEC.md`.

## Grading discipline (applies to every pass)

The rubric's **Scoring Discipline** section is binding. In short:

- **Reference-guided grading for verifiable dimensions** (DIM-1, DIM-3, DIM-11): derive the correct answer from *code* first, then grade the spec against it — never grade a claim on its own plausibility.
- **Reason before scoring; quote evidence.** Each dimension score cites a verbatim spec passage (or names the section where it should appear, for an absence). Emit one record per dimension: `{dimension, score, confidence, rationale, evidence_quote}`.
- **Sample and reduce.** Score each dimension 3–5× at temperature 0 and take the median. Spread ≥1 point → raise a confidence flag and escalate that dimension (stronger re-grade or human review).
- **Neutrality.** Every scoring prompt is length-neutral, format-neutral (markdown ≠ quality), and citation-neutral (a citation is not evidence unless verified). **Never state a target grade to a grading agent.**
- **Hard-fail gates.** DIM-1 or DIM-4 < 2 caps the grade at C+; DIM-1 < 1 is an automatic F.

## Process

Run the grading loop. Each iteration: grade → identify highest-impact fixes → apply fixes → re-grade **blind**. Stop when the grade is A- or better, or when two consecutive iterations produce the same grade, or the held-out grader (Step 8) rejects the fixes.

Once per run (not every iteration), a non-scoring **Competitive Benchmarking** pass (Step 6c) benchmarks a *target* spec against the rubric's **Competitor Reference Set** and surfaces `[COMPETITIVE]` adopt/adapt/decline proposals for human review. It never edits the spec or moves the grade — it produces strategic backlog.

### Step 1 — Read the rubric and the spec

Read `docs/plans/PRODUCT_SPEC_RUBRIC.md` for the 11 dimensions, anchor descriptions, scoring discipline, hard-fail gates, and double-counting rule.

Read the target spec file(s). Identify the primary document (look for a "source of truth" declaration). Determine whether this is a target spec or current-state spec (target specs include DIM-8 Feasibility; current-state specs skip it — max 105 instead of 114).

### Step 2 — Pass 1: Structural analysis + text lint (spec text only)

Use a fork agent. Check:
- TOC, stubs, source-of-truth, companion list, terminology
- Code reference density and cross-reference resolution
- Gap inventory (IDs, severity, audit methodology)
- **Non-goals / out-of-scope section present?** (DIM-2)
- **Security & privacy addressed** (authz/RBAC, data handling, PII)? (DIM-2)
- **Problem statement, target user/beneficiary, and success metrics present?** (DIM-9)
- **Prioritization:** RFC 2119 keyword usage (MUST/SHOULD/MAY, caps-only per RFC 8174), MoSCoW, or Essential/Conditional/Optional markers (DIM-10)
- **Weak-word / ambiguity scan** — flag `appropriate, adequate, sufficient, efficient, fast, robust, user-friendly, easy, flexible, seamless, minimize, maximize, optimize, as appropriate, as needed, if required, etc., and/or, support, handle, quickly, TBD` (DIM-4)
- **Compound-requirement scan** — flag requirements with multiple main verbs / "X and Y and Z" (non-singular) (DIM-4)
- **TBD/TBR/"???" scan** — any placeholder means "not complete" (DIM-2)
- Redundancy (same content in multiple sections)
- Density test (3 random 50-line blocks — count load-bearing lines)
- Decision rationale count (decisions with "because" vs bare statements)

**Scores:** DIM-2 (partial), DIM-4 (lint portion), DIM-5 (Decision Quality), DIM-7 (Document Craft), DIM-9 (Justification & Value), DIM-10 (Prioritization)

### Step 3 — Pass 2: Code verification, reference-guided (exhaustive)

Launch parallel fork agents, one per domain. For each domain, **first derive from code what the correct behavior/API is**, then grade the spec's claims against that reference. **Verify ≥50 claims** covering every §-level section (≥2 per section), with ≥20 behavioral claims.

**Adversarial / backward-traceability pass (required):** pick 5 random code paths from the codebase that SHOULD fall within a documented section's scope. Check whether the spec describes them accurately. Undocumented behavior inside a documented section → DIM-1 (silent omission). Whether each path is traceable to a stated requirement at all → DIM-11 (backward traceability / gold-plating).

For each claim: classify as CORRECT, WRONG (never true), or STALE (was true, changed). If wrong/stale: severity (critical-path or minor).

Apply double-counting rule: stale → Freshness only, authoring-time error → Correctness only, missing trace link → Traceability only.

**Scores:** DIM-1 (Correctness), DIM-3 (Freshness), DIM-11 (backward traceability)

### Step 4 — Pass 3: Completeness audit

Fork agent establishes the completeness denominator:
- Route/page audit against spec sections
- Zustand store audit (primary actions documented?)
- API endpoint audit
- Check each domain covers all 5 section types (workflow, data model, persistence, I/O, error handling)
- Check gap inventory for IDs, severity, audit methodology
- **Forward traceability:** does each requirement resolve to a file-path code reference AND a derivable test? (DIM-11)

**Scores:** DIM-2 (Completeness & Honesty, complete), DIM-11 (forward traceability)

### Step 5 — Pass 4: Precision + Coherence

Fork agent runs both tests:

**Precision test:** Pick **5 workflows from different domains**. For each:
- Can a new engineer implement from the spec alone — including error handling and edge cases?
- Can you derive ≥5 test cases (including ≥2 negative tests) from the spec alone?
- **Does the workflow carry ≥1 concrete worked example (Given-When-Then or real values)?**
- A workflow that specifies only the happy path, or carries no worked example, does NOT pass either test.

**Coherence checks:**
- Pick 5 workflow-to-data-model references — do workflows use fields that exist in the model?
- Walk each user journey end-to-end — do all step references resolve?
- Check state names in workflows match data model lifecycle fields
- Verify keyboard shortcuts table matches shortcuts in workflow sections
- Units/measurement systems homogeneous?

**Scores:** DIM-4 (Precision), DIM-6 (Coherence)

### Step 6 — Pass 5: Feasibility (target specs only)

Skip for current-state specs. Fork agent checks:
- Migration notes section exists with per-feature-area migration path
- Dependencies between features identified (build ordering)
- Technical constraints acknowledged (viewer backend, persistence format, auth)
- Breaking changes called out with migration strategy
- Carry-over features explicitly marked as unchanged

**Scores:** DIM-8 (Feasibility)

### Step 6b — Adversarial Grilling

This pass exists to find what the other passes missed. Launch a fork agent whose
sole job is to attack the spec. The agent should assume the spec is wrong and try
to prove it. This is NOT verification (Pass 2 does that). This is an adversary
looking for the kinds of problems that survive verification. **Run it every fix-loop
iteration.**

**The grilling agent should:**

1. **Challenge plausible-sounding claims.** Pick 5 claims that SOUND right but are
   the kind of thing that's often wrong — counts ("20 types"), behavioral chains
   ("X triggers Y which causes Z"), and implicit assumptions ("this is persisted"
   without saying where or how).

2. **Find what's conspicuously absent.** For each major section, ask: "What question
   would a new engineer ask after reading this that the spec doesn't answer?" List
   concrete unanswered questions — not vague "could be more detailed" but specific
   "§3.3.4 says snapping is within 2m but doesn't say what coordinate space the
   distance is measured in."

3. **Stress-test decisions.** For each design decision with rationale, try to refute
   the rationale. "The spec says instant commit is industry standard — but Blender
   uses instant commit for MODELING operations, not for operations with required
   metadata like annotations. Is this analogy actually valid?" The point isn't to
   win the argument — it's to find decisions where the reasoning doesn't hold up.

4. **Trace one feature end-to-end through every section.** Pick a single feature
   (e.g., "guy wire placement" or "ATC alignment") and follow it through: user
   journey → workflow → data model → persistence → export → error handling →
   migration notes. Document every gap, contradiction, or handwave you encounter.

5. **Find the lies of omission.** The spec describes what SHOULD happen. But what
   does it NOT describe that an implementer would need to know? Look for:
   - Concurrency scenarios not addressed (two tabs, two users)
   - Performance implications not acknowledged (what happens at 100 poles? 500?)
   - Permissions not specified (who can do this? Is it RBAC-gated?)
   - Undo boundaries not defined (can you undo across a save? across a session?)
   - State cleanup not specified (what happens to pending state on navigation away?)

6. **Hunt reward-hacking surface features.** Because the fix loop edits the spec to
   raise its own grade, check specifically for text added to game the grader rather
   than to improve the spec: **lexical** (rubric keywords sprinkled in), **tone**
   (confident closings), **self-praise**, **format** (rigid templates or added
   sections that raise the score without raising substance). Flag any grade gain that
   correlates with length growth rather than new load-bearing content.

**Output:** a ranked list of findings, most severe first. Each finding:
- What's wrong or missing (specific, not vague)
- Why it matters (what would go wrong if an engineer hit this gap)
- Suggested fix (if fixable from code/companion docs) or flag for human review

**This pass does NOT produce a dimension score.** Its findings feed into the scoring
of ALL dimensions — a grilling finding about a wrong claim adjusts DIM-1, a finding
about missing error handling adjusts DIM-4, etc. The grilling pass is the
calibration step that prevents inflated scores from the other passes.

### Step 6c — Competitive Benchmarking (target specs only, non-scoring)

Run **once per grading run** (not every fix-loop iteration — the competitor set is
stable between iterations, and its proposals are human-gated, not auto-applied).
**Skip for current-state specs:** proposing features to adopt makes no sense for a
spec that documents what already exists.

Launch a fork agent that benchmarks the spec against the **Competitor Reference Set**
(bottom of `PRODUCT_SPEC_RUBRIC.md`). Its job is to mine the best-in-class patterns
from comparable production platforms and surface which ones this spec should adopt,
adapt, or deliberately decline. This is the inverse of the grilling pass: grilling
hunts for defects in what the spec *says*; benchmarking hunts for opportunities in
what the spec *omits relative to the field*.

**The benchmarking agent should:**

1. **Map the overlap.** For each domain the spec covers (PLA / pole modeling, survey,
   alignment, point-cloud viewer, clearance, export, platform), name the comparable
   competitors from the reference set and what each does well.
2. **Find adopt-worthy patterns.** For each competitor strength, ask: does our spec
   already cover this? If not, is it a genuine improvement for *our* users, or
   competitor-specific noise? Keep only patterns that would make our product better.
3. **Find deliberate divergences.** Where the spec does something differently from the
   field, is that difference *justified* (a decision with rationale — feeds DIM-5) or
   *accidental* (nobody considered how the incumbents solve it)?
4. **Respect scope.** A competitive proposal expands product scope. It is NOT a spec
   defect that the product lacks a competitor's feature — **do not lower any score for a
   missing feature the product never claimed.**

**Specificity bar (same as grilling):** every finding must name the specific competitor
and capability. "Adopt an industry-standard alignment flow" is not a finding.
"PLS-CADD lets you save a sag-tension load case as a reusable template; §4 export has no
equivalent, so every pole re-enters load cases by hand" IS a finding.

**Output — a ranked list of `[COMPETITIVE]` findings, highest-leverage first.** Each:
- **Pattern** — the capability/UX and which named competitor(s) do it well.
- **Gap** — what our spec says today (quote §X.Y) vs. the competitor pattern.
- **Recommendation** — adopt / adapt / decline, with a one-line why.
- **Where it would land** — the spec section that would host it, if adopted.

**Scoring:** none. This pass emits **no dimension score**, and its proposals are
**never auto-applied** — they add scope the code doesn't implement, which the "never
fabricate spec content" rule forbids. Its findings feed exactly two places: (a) Step 8
as `[COMPETITIVE]` **flag-for-human-review** items (the product owner decides whether to
take on the scope), and (b) DIM-5 Decision Quality — a *consequential* decision made
without benchmarking against the field is a rationale gap, fed in the same way a grilling
finding feeds a dimension. Because these proposals never enter the spec text
automatically, they cannot inflate the grade or trip the reward-hacking checks.

### Step 7 — Score and grade

Score all 11 dimensions (or 10 for current-state specs) per the **Grading discipline**:
reference-guided where applicable, reason-then-score with a quoted evidence passage,
**median of 3–5 samples at temperature 0**, confidence flag on spread ≥1. Apply the
double-counting rule and the **hard-fail gates**.

```
max_score = 114 (target) or 105 (current-state)
raw_score = Σ(dim_score × weight)          # dim_score may be continuous (median)
percentage = raw_score / max_score × 100
```

Map to letter grade: A+ (97-100) through F (<60). If a hard-fail gate trips, report the
capped grade (e.g. "C+ — capped by DIM-1 gate") not the raw percentage letter.

**Output the scorecard** with all dimensions (each as a `{score, confidence, rationale, evidence_quote}` record), then:
- Hard-fail gate status
- Verified claims table (≥20 claims shown)
- Adversarial findings (code paths not in spec) + backward-traceability result
- Precision test results per workflow (pass/fail with reasoning)
- Coherence check results (per check)
- **Grilling findings** (ranked by severity — the adversarial review)
- **Competitive benchmarking findings** (`[COMPETITIVE]`, ranked by leverage — adopt/adapt/decline, flagged for human review; target specs only)
- Any dimension with a confidence flag (escalate or mark for human review)
- Fix list sorted by `weight × (3 - score)` descending

### Step 8 — Fix (if grade < A-)

Apply fixes for the highest-impact issues:

**Auto-fixable:**
- Factual errors → correct with verified value from code
- Stale content → update to match current code
- Missing error handling → add from code (grep for error/throw/warn paths)
- Missing rationale → add from consolidation plan or companion docs
- Weak words → replace with concrete, testable language and thresholds
- Compound requirements → split into singular requirements
- Missing worked examples → add a Given-When-Then example from the workflow's code path
- Missing importance markers → add RFC 2119 keywords / MoSCoW from priority signals in code or plan
- Missing non-goals / problem statement / success metrics → add from product docs or companion plan
- Redundancy → replace duplicate content with cross-references
- Missing cross-references / trace links → add §X.Y links and file-path code references
- Terminology → standardize to declared convention
- Missing gap IDs/severity → add from code audit

**Flag for review (don't auto-fix):**
- Architectural decisions — changing these may misrepresent intent
- External system claims (Auth0, S3, Prefect) — can't verify locally
- Coherence issues between sections — may require design decisions
- Feasibility concerns — require architecture knowledge
- Success metrics / prioritization that require product-owner input
- `[COMPETITIVE]` adopt-proposals — a competitor pattern is a *scope* decision, not a spec fix. Present it for the product owner to accept, adapt, or decline. **Never auto-write a competitor's feature into the spec** — the code doesn't implement it, so it can't be verified.

After fixing, **re-grade blind, stateless, and cross-checked:**

1. **Blind re-grade.** Spawn a fresh fork that sees **only the final spec text** — not
   the diff, not the prior grade, not the target grade, not the word "revised." This
   defeats self-preference and refinement-aware bias (a judge otherwise rewards text
   merely for being a revision). Re-run ALL passes; don't carry forward prior scores —
   a fix in one section can break coherence with another.

2. **Held-out acceptance gate.** Additionally grade the revised spec with an
   **independent held-out grader — a different model family than the one that wrote the
   fixes** (e.g. if fixes were authored by Opus, grade with Sonnet/Haiku or a separate
   fork instructed with a rubric variant it hasn't optimized against). **Accept the fix
   only if the held-out grader also confirms the gain.** If the primary grade rose but
   the held-out grade did not, presume reward-hacking and **reject the fix**.

3. **Length/padding sanity check.** Log word-count delta alongside grade delta each
   iteration. A grade gain that tracks length growth rather than new load-bearing content
   is a red flag — reconcile against the Step 6b reward-hacking findings.

Report the delta (grade, held-out grade, word-count).

### Step 9 — Convergence

Stop when:
- Grade ≥ A- (90%) **and the held-out grader agrees**, OR
- Two consecutive iterations produce the same grade, OR
- The held-out grader rejects the fixes (primary rose, held-out did not — stop; the loop
  is gaming the primary grader), OR
- 3 iterations completed

Output:
- Final grade with per-dimension breakdown (and held-out grade)
- Grade progression (e.g., "C+ → B → A-")
- Total claims verified
- Fixes applied (with before/after)
- Word-count delta across iterations
- Any dimensions carrying confidence flags
- Remaining issues flagged for human review

## Critical rules

- **Never fabricate spec content.** All fixes must come from reading code or companion docs.
- **Grade reference-guided.** For verifiable dimensions, derive the truth from code first; never grade a claim on its own plausibility.
- **Never tell a grading agent the target grade,** and keep every scoring prompt length-, format-, and citation-neutral.
- **Re-grade blind and cross-checked.** The re-grade fork sees only the final spec text; the acceptance gate uses a different model family. Don't carry forward scores from prior iterations.
- **Reject grade gains the held-out grader can't confirm.** A self-improving loop that optimizes only the primary grader is reward-hacking, not improving.
- **Quote evidence for every score.** A score without a verbatim supporting passage (or a named absent section) doesn't count. Adding citation-shaped or template-shaped text is not an improvement.
- **Score 3 requires exhaustive work.** A spot-check that passes is a 2. A 3 means you checked thoroughly and found nothing — or found everything and fixed it.
- **The precision test must be hard.** A happy-path-only workflow, or one with no concrete worked example, does NOT pass. Error handling and edge cases are required for a pass.
- **Respect the hard-fail gates.** DIM-1 or DIM-4 < 2 caps the grade at C+ regardless of the weighted total.
- **Preserve the spec's voice.** Match existing style when fixing. Don't rewrite correct content.
- **The grilling pass is mandatory, every iteration.** Grilling findings adjust scores downward from what the verification passes would otherwise produce. A spec that passes verification but fails grilling does NOT get a high grade — the grilling is the reality check.
- **Competitive proposals are backlog, not fixes.** The competitive-benchmarking pass (Step 6c, target specs only) proposes patterns to adopt from *named* competitors; these are always `[COMPETITIVE]` flag-for-human-review items, never auto-applied, and they move no score. A spec must never gain scope the code doesn't implement just because a competitor has it. The only score they touch is DIM-5, and only when a consequential decision was made without benchmarking the field.
- **Be specific, not vague.** "Could be more detailed" is not a finding. "§3.3.4 says auto-snap within 2m but doesn't specify whether distance is measured in scene coordinates or world coordinates — these differ by the scan's similarity transform scale factor" IS a finding.
