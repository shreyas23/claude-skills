# Product Spec Grading Rubric

> Metric for evaluating whether a product specification is good enough to build from,
> test from, and defend decisions with. Designed to be machine-evaluable: each dimension
> has observable signals that can be checked against a codebase.
>
> **Audience:** Engineers implementing from the spec, reviewers auditing spec quality,
> and automated grading tools.
>
> **Scope:** Product specs for software applications — feature catalogs, workflow
> references, rebuild plans, target specs. Not API docs, not architecture decision records.
>
> **Multi-document specs:** Grade the set as a unit. The primary document (declared
> source of truth) carries most weight. Companion documents contribute where referenced
> but contradictions between them penalize Document Craft.
>
> **Target vs current-state specs:** Both use the same 10 core dimensions. Target specs
> add DIM-8 (Feasibility) — can the proposed design actually be built? Current-state
> specs skip DIM-8; their max score is 105 instead of 114.
>
> **Provenance:** The dimensions draw on IEEE 830 / ISO-IEC-IEEE 29148 requirements
> quality characteristics, RFC 2119 normative keywords, INVEST, Amazon Working-Backwards
> PR/FAQ, and Google design-doc review. The *grading method* (§ Scoring Discipline) draws
> on the LLM-as-judge literature — G-Eval, Prometheus, MT-Bench, PoLL — to keep an
> automated grader calibrated and hard to game.
>
> **Competitive benchmarking:** A **non-scoring** pass (a peer of Adversarial Grilling)
> benchmarks *target* specs against the **Competitor Reference Set** (bottom of this
> rubric) and emits `[COMPETITIVE]` adopt/adapt/decline proposals for human review. It
> moves no score directly and never edits the spec; where a consequential decision was
> made without benchmarking the field, that finding feeds DIM-5. See the skill's Step 6c.

---

## Scoring Model

### Per-Dimension Scoring (0–3)

| Score | Label | Meaning |
|-------|-------|---------|
| **3** | Excellent | Exhaustive coverage of this dimension. Passes adversarial scrutiny. |
| **2** | Good | Solid coverage with minor gaps that don't block work. |
| **1** | Weak | Major gaps or errors. Useful as a starting point only. |
| **0** | Absent | Not addressed, or content present is more wrong than right. |

**Wrong vs absent:** A score of 0 means absent or net-negative (more misleading than
helpful). Content that exists with a mix of correct and incorrect claims scores ≥1
based on the ratio. The Correctness dimension captures error risk; other dimensions
assess the content that IS present.

Each dimension below carries **anchor descriptions** at every score level. Grade by
selecting the anchor the spec best matches, then emit that anchor's number — do not
free-float an integer. Every dimension in the tables gives level-3 criteria explicitly;
levels 2/1/0 degrade from it.

### Grade Scale

| Grade | Range | Meaning |
|-------|-------|---------|
| **A+** | 97–100% | Exceptional. Build, test, and defend decisions — all from this spec. |
| **A** | 94–96% | Exhaustive and correct. Minor polish only. |
| **A−** | 90–93% | Near-exhaustive. A few gaps that won't block work. |
| **B+** | 87–89% | Strong foundation. Specific fixable gaps identified. |
| **B** | 83–86% | Good reference. Needs targeted additions before building. |
| **B−** | 80–82% | Useful but needs work in multiple dimensions. |
| **C+** | 77–79% | Covers the system but significant quality gaps. |
| **C** | 73–76% | Incomplete. Major sections need work. |
| **C−** | 70–72% | Barely useful as a build reference. |
| **D+** | 67–69% | Unreliable for building. Starting point for research only. |
| **D** | 63–66% | Severely deficient. More gaps than content. |
| **D−** | 60–62% | Nearly absent. Placeholder content only. |
| **F** | < 60% | Missing or mostly wrong. |

### Score Calculation

```
max_score = (5+4+4+5+4+3+2+3+3+2+3) × 3 = 114    [target spec, 11 dimensions]
max_score = (5+4+4+5+4+3+2  +3+2+3) × 3 = 105    [current-state spec, no Feasibility]
raw_score = Σ(dimension_score × dimension_weight)
percentage = raw_score / max_score × 100
```

Per-dimension scores may be **continuous** (the median/mean of repeated samples — see
Scoring Discipline). Keep them continuous through the weighted sum; round only at the
final letter-grade cut. This prevents integer ties from stalling the fix loop on a
plateau that is noise rather than signal.

### Hard-Fail Gates

A letter grade hides which dimension failed. Independent of the percentage:

- **Any critical-path dimension (DIM-1 Correctness, DIM-4 Precision) scored < 2 ⇒ the
  spec cannot grade above C+**, regardless of the weighted total.
- **DIM-1 < 1 (net-misleading) ⇒ automatic F.** A spec that lies is worse than no spec.

Report gate status alongside the letter.

### Double-Counting Rule

A single defect counts against **one** dimension only:

- **Claim wrong when written** → Correctness only
- **Claim correct when written, code changed since** → Freshness only
- **Same claim in two documents, they disagree** → Document Craft only
- **Decision stated without rationale** → Decision Quality only (not also Precision)
- **Undocumented behavior inside a documented section** → Correctness (silent omission),
  not Traceability
- **Missing trace link / matrix (requirement has no code or test, or code has no
  requirement)** → Traceability only
- **Requirement lacks an importance/obligation marker** → Prioritization only
- **Missing problem statement / user value / success metric** → Justification & Value only
- **Consequential decision made without benchmarking the competitor set** → Decision
  Quality (DIM-5) only. The competitive-benchmarking pass itself emits no score; a
  competitor feature the product simply lacks is a scope proposal, not a defect, and
  penalizes nothing.

---

## Scoring Discipline (judge protocol)

These rules govern *how* an automated or human grader assigns the 0–3 scores. They exist
because an LLM judge left unconstrained is inconsistent (integer ties, drift) and gameable
(rewards length, formatting, citation-shaped text, and its own prior edits). Every grading
pass MUST follow them.

1. **Reference-guided grading for verifiable dimensions.** For DIM-1 (Correctness),
   DIM-3 (Freshness), and DIM-11 (Traceability), first derive from the *code* what the
   correct behavior/API/link is, then grade the spec against that reference. Never grade a
   claim on its own fluency or plausibility. (MT-Bench: reference-guided grading cut
   math-grading failures ~70%→15%.)

2. **Reason before scoring; quote evidence.** For each dimension, write the rationale
   first, then the integer. Every score must cite a **verbatim quoted passage** from the
   spec (or, for an absence, name the section where it should appear). A secondary check
   confirms the quote actually supports the score — this blocks earning points by *adding*
   citation-shaped or template-shaped text ("citation theater").

3. **Sample and reduce.** Score each dimension **3–5 times at temperature 0** and take the
   **median**. If the inter-sample spread is ≥1 point, raise a **confidence flag** on that
   dimension and escalate it (a stronger re-grade, a diverse-model check, or human review).

4. **Neutrality clauses.** Every scoring prompt is explicitly **length-neutral,
   format-neutral (markdown ≠ quality), and citation-neutral (a citation is not evidence
   unless verified)**, and must **never be told a target grade**. Judges reward longer,
   prettier, more confidently-worded text absent these clauses.

5. **Blind, stateless, cross-checked re-grading in any fix loop.** When a grader re-grades
   a revised spec, it sees **only the final spec text** — not the diff, not the prior
   grade, not the word "revised." Self-preference and refinement-aware bias otherwise
   inflate the score for the act of revising. Where possible the re-grade and the
   acceptance gate use a **different model family** than the one that wrote the fixes.

6. **Structured output.** Emit one uniform record per dimension:
   `{dimension, score, confidence, rationale, evidence_quote}`. Parse the number from the
   structure, never from prose.

---

## Dimensions

### DIM-1: Correctness (weight: 5)

> Are factual claims about the system accurate?

The highest-weighted dimension. A wrong spec is worse than no spec — engineers build
on false assumptions, and the resulting bugs are the hardest to diagnose because
"the spec said so." Grade **reference-guided**: derive the truth from code first.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | 0 errors in an **exhaustive** verification: ≥50 claims checked covering every section, ≥20 behavioral, AND an adversarial pass finds no undocumented behavior in documented sections. |
| 2 | 0 critical-path errors. ≤2 minor errors across ≥50 claims. |
| 1 | 1 critical-path error, or ≥3 minor errors, or adversarial pass finds significant undocumented behavior. |
| 0 | Multiple critical-path errors, or no verifiable claims present. |

**Severity tiers:**
- **Critical-path:** wrong claim about persistence, security, state management, coordinate
  math, or auth. Would cause data loss or corruption if built as specified.
- **Minor:** wrong claim about UI details, non-critical edge cases, cosmetic behavior.

**Sampling protocol:** ≥50 claims, stratified so every §-level section has ≥2 checked.
≥20 must be behavioral ("X happens when Y"), not just name checks. Prioritize
persistence, error handling, state transitions.

**Adversarial pass (required for score 3):** pick 5 random code paths from the codebase
that fall within a documented section's scope. Check whether the spec describes them.
This catches "correct about what it says, silent about what it should say." (The
*systematic* version of this check — is there a trace link for every code path? — is
scored under DIM-11 Traceability; here it feeds Correctness as silent omission.)

**Common failure patterns:**
- Describing planned behavior as current behavior
- Conflating two similar but distinct enums, types, or code paths
- Documenting error handling that doesn't exist ("throws" when code silently returns)
- Mixing items from different layers of the system in a single count

---

### DIM-2: Completeness & Honesty (weight: 4)

> Is every feature covered? Where coverage falls short, is the gap acknowledged? Is the
> boundary of scope declared, and are cross-cutting concerns addressed?

A spec that covers 60% of the system and pretends it covers 100% is more dangerous
than one that covers 60% and says so. Completeness measures breadth of coverage.
Honesty measures whether the spec owns its gaps — with IDs, severity, and audit
methodology — **declares what it deliberately does NOT do**, and addresses the
**cross-cutting concerns** (security, privacy) that mature review treats as mandatory.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | Every user-facing feature documented. Every domain covers all five section types (workflow, data model, persistence, I/O, error handling). Route/store/API audit shows ≥95% coverage. Known gaps cataloged with IDs, severity, and recommendations. Audit methodology described. **An explicit non-goals / out-of-scope section exists.** **Security & privacy are addressed** (authz/RBAC, data handling, PII). **No TBD/TBR/"???" tokens remain** (per IEEE 830/29148, any TBD ⇒ not complete). |
| 2 | All major features documented. ≤2 domains missing one section type. Coverage ≥85%. Major gaps cataloged with severity. Non-goals OR security addressed but not both; ≤2 TBD tokens. |
| 1 | Core workflows documented but significant features missing. Coverage ≥60%. Gaps mentioned but not systematically cataloged. No non-goals, no security treatment, or scattered TBDs. |
| 0 | More than half undocumented, or gaps not acknowledged. |

**Completeness per section type:**
- **Workflow:** trigger, steps, picks/clicks, state changes, commit path, branching, validation, undo
- **Data model:** all fields with types, hierarchy rules, constraints
- **Persistence:** save trigger, conflict handling, error recovery, data lifecycle
- **I/O:** format, parser file, validation rules, round-trip fidelity
- **Error handling:** behavior on invalid input, network failure, concurrent edits

**Honesty & boundary signals:**
- Gap inventory with IDs and severity levels
- Silent failure scenarios documented
- **Explicit "what's NOT supported" / non-goals section** (Google: non-goals are often
  more important than goals for bounding review — a non-goal is a plausible goal declined)
- **Security & privacy checklist addressed** (Google makes both mandatory pre-launch gates)
- Audit methodology description (how were gaps found?)
- **No TBD/TBR placeholders** left in the body

---

### DIM-3: Freshness (weight: 4)

> Does the spec match the current state of the codebase?

A stale spec describes a system that no longer exists. Only claims that were correct
when written but are now outdated count here. Claims that were never correct count
under Correctness. Grade **reference-guided** against current code.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | Every checked reference resolves to current code. Status/parity tables match implementation. No "planned" items that have shipped. Freshness dates present. |
| 2 | ≤3 stale claims, none in critical paths. Status tables may lag one development cycle. |
| 1 | Significant staleness. Status tables outdated. Features added/changed without spec updates. |
| 0 | Spec describes a fundamentally different system. |

**Signals:**
- Compare parity/status tables against current code
- Check "planned"/"future" items against shipped features
- Verify file paths and imports still exist
- Look for freshness markers (dates, version numbers)

---

### DIM-4: Precision (weight: 5)

> Can an engineer build from this spec AND a QA engineer test from it — without
> reading code?

This is the merged Actionability + Testability dimension. A spec is precise when it
provides enough detail to implement AND enough boundary conditions to verify. These
are not separate qualities — a workflow described without error handling is neither
fully implementable nor fully testable. Precision also requires that each requirement
be **unambiguous** and **singular**, and that workflows carry **concrete worked examples**.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | **Every** workflow with user interaction specifies: step sequence with click counts, state transitions, validation rules with exact conditions, error handling per failure mode, guard conditions, and at least one boundary condition with a concrete threshold. Precision test passes for ≥5 workflows. Derivation test produces ≥5 test cases (including negative tests) for each. **Weak-word scan is clean, requirements are singular, and each user-facing workflow carries ≥1 concrete worked example (Given-When-Then or real values).** |
| 2 | Most workflows pass precision + derivation tests (3–4 of 5). Critical-path workflows fully specified. Some workflows lack error handling or boundary conditions. A few weak words, compound requirements, or example-less workflows. |
| 1 | Happy paths described but error handling and boundaries are vague or absent. Precision test passes for ≤2. Pervasive weak words / compound requirements / no examples. |
| 0 | Reads as a feature list. No step sequences, no validation rules, no thresholds. |

**Precision test:** Pick 5 workflows from different domains. For each: could a new
engineer implement it from the spec alone — including error handling and edge cases —
producing behavior indistinguishable from the target? A workflow that specifies only
the happy path does NOT pass.

**Derivation test:** For the same 5 workflows, attempt to write ≥5 test cases per
workflow from the spec alone, including at least 2 negative tests (invalid input,
boundary violation, error scenario). If you can only derive happy-path tests, the
workflow does NOT pass.

**Unambiguity scan (weak-word detection):** flag vague, untestable language — the
consolidated NASA / INCOSE / QuARS "words to avoid" list:
`appropriate, adequate, sufficient, efficient, fast, robust, user-friendly, easy,
flexible, seamless, minimize, maximize, optimize, as appropriate, as needed, if required,
etc., and/or, support, handle, robustly, quickly, TBD`.
Each occurrence in a requirement is a precision defect: it has no single testable meaning.

**Singular-requirement check:** flag compound requirements — multiple main verbs or
"X and Y and Z" in one requirement (QuARS "Multiplicity", 29148 "Singular"). A compound
requirement cannot be cleanly verified as pass/fail.

**Worked-example requirement (Specification-by-Example / BDD):** each user-facing
workflow should contain at least one concrete example with real values — Given-When-Then
or an equivalent worked scenario — not only an abstract description. The spec itself
should carry the example, not merely be *derivable into* one.

**What precision requires per workflow:**
- Concrete step count (not "user does X")
- Named state machine phases (if multi-step)
- Numeric thresholds (snap radius, debounce, max entries)
- Guard conditions with enable/disable criteria
- Error handling: what the user sees for each failure mode
- At least one boundary condition (what happens at exactly the threshold?)
- At least one concrete worked example

---

### DIM-5: Decision Quality (weight: 4)

> Are design decisions backed by reasoning that someone else can evaluate?

A spec without rationale is fragile — any stakeholder question requires going back to
the author. A spec with rationale lets the team evaluate decisions independently, push
back with evidence, and make informed tradeoffs when constraints change.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | ≥90% of design decisions include rationale with at least one of: industry precedent (named competitors or standards), quantitative data (click counts, timing, error rates), user research, or engineering constraint. Every decision that changes legacy behavior explains why. Rejected alternatives documented for the 5 most consequential decisions (Google: "alternatives considered" as a required section, with why each lost). |
| 2 | ≥60% of decisions have rationale. Major decisions (consolidation changes, architectural shifts) are justified. Some carry-over decisions lack justification. |
| 1 | ≤30% of decisions have rationale. Most are stated as requirements without reasoning. |
| 0 | No rationale. Pure feature list. |

**What counts as a decision:** any choice where a reasonable alternative exists.
"3-pick pole ID" is a decision (could be 2-pick). "Instant commit" is a decision
(could keep review card). Constraints with no reasonable alternative ("validate
required fields") are not decisions.

**Signals:**
- Count decisions with "because" / "why" / tradeoff language
- Check for "we considered X but rejected it because Y"
- Check for industry/competitor citations, standards body references
- Check for quantitative data (not just assertions)
- Check whether spec distinguishes intentional choices from carry-over legacy behavior

---

### DIM-6: Coherence (weight: 3)

> Do the pieces fit together as a system?

The other dimensions grade individual sections. Coherence grades the relationships
between them. A spec can have correct data models, precise workflows, and justified
decisions — but if the workflow references a data model field that doesn't exist,
or a user journey hits a dead end where no workflow covers the next step, the spec
is incoherent.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | Data models support every workflow that references them (field-level check). User journeys traverse the spec without dead ends. Cross-references between sections are bidirectional (if §3.3 references §7, §7 covers what §3.3 needs). State machines in workflows use the same states as the data model's lifecycle fields. Units and measurement systems are homogeneous (29148). No contradictions between sections. |
| 2 | Major cross-references are consistent. ≤2 dead ends in user journeys. Data models mostly support workflows. |
| 1 | Sections are siloed — written independently without checking cross-references. User journeys have multiple dead ends. Workflows reference data model fields that don't exist in §7. |
| 0 | Sections actively contradict each other. |

**Coherence checks:**
- Pick 5 workflow-to-data-model references. Does the workflow use fields that exist in the model?
- Walk each user journey end-to-end. Does every step reference a spec section that covers it?
- Check that state names in workflows match state names in data model lifecycle descriptions
- Check that export format descriptions match the data model (fields exported = fields in model?)
- Verify that keyboard shortcuts table matches the shortcuts described in workflow sections

---

### DIM-7: Document Craft (weight: 2)

> Is the spec well-made as a document — organized, consistent, efficient, navigable?

This is the merged Structure + Consistency + Efficiency dimension. A well-crafted
document is organized for searching (not linear reading), uses consistent terminology,
and says things once. (Bidirectional spec↔code linking was formerly graded here; it now
has its own dimension, DIM-11 Traceability.)

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | TOC with working anchors. Domain-first organization. Appendices for reference material. Source-of-truth declaration. Companion docs listed with roles. Uniform terminology (0 violations — each concept named by one unique term, per 830 "Unambiguous"). No redundancy (each fact once, cross-referenced not repeated). Code references include file paths (not just function names). Density test: ≥80% load-bearing lines in random 50-line blocks. |
| 2 | TOC present. Logical organization. ≤3 redundancies. ≤3 terminology inconsistencies. Most code references resolve. |
| 1 | No TOC. Code-structure organization. Significant redundancy. Inconsistent terminology. Few code references. |
| 0 | Unstructured. No code references. No consistency. |

**Density test:** Pick 3 random 50-line blocks. Count lines an implementer would need.
≥40/50 (80%) → good. 25–39 → acceptable. <25 → bloated.

---

### DIM-8: Feasibility (weight: 3) — target specs only

> Can the proposed design actually be built on the existing codebase?

A target spec that describes a beautiful system but ignores the constraints of the
current codebase, existing data, migration complexity, or technical debt is
aspirational, not actionable. Feasibility checks whether the spec acknowledges
reality. (29148 makes feasibility an explicit requirement quality, individual and set.)

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | Migration path from current to target is documented per feature area. Dependencies between features are identified (what must be built before what). Technical constraints acknowledged (e.g., "requires Potree API X which is jQuery-based"). Breaking changes called out with migration strategy. Carry-over features explicitly marked as unchanged. |
| 2 | Migration notes exist but are high-level. Major dependencies identified. Some constraints acknowledged. |
| 1 | Target state described with no migration path. "Build this" without addressing "from what." |
| 0 | Target state is incompatible with current architecture and doesn't acknowledge it. |

**Signals:**
- Check for a migration notes section or appendix
- Verify carry-over features are marked ("unchanged from current system")
- Check for dependency ordering between features
- Look for technical constraint acknowledgment (viewer backend, persistence format, auth flow)
- Check whether breaking changes are identified
- Verify that the spec accounts for data that already exists (e.g., existing qPole state format must be deserialized)

**Skip this dimension for current-state specs.** Score only for target specs, rebuild
plans, and redesign documents.

---

### DIM-9: Justification & Value (weight: 3)

> Does the spec say what problem it solves, for whom, and how success is measured?

A spec can be correct, precise, and coherent and still never state *why it should be
built*. DIM-5 grades the reasoning behind individual design decisions; this dimension
grades the reasoning behind the product itself. Amazon's PR/FAQ and INVEST's "Valuable"
both put customer value first; 29148 calls this being **able to be validated** (does
satisfying the spec actually meet the stakeholder need?), as distinct from verification.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | States the **user problem** explicitly, the **target user / beneficiary**, and **measurable success metrics** (adoption, task time, error rate, or equivalent). Each major feature ties back to a stated user value. Key assumptions and top risks named (PR/FAQ-style). |
| 2 | Problem and audience stated; success metrics vague or partial. Most features connect to a value statement. |
| 1 | Problem gestured at but no clear beneficiary or metrics. Features presented as self-evidently needed. |
| 0 | No problem statement, no audience, no metrics. Pure feature list with no "why build this." |

**Signals:**
- Is there a problem statement / motivation section?
- Is the target user or beneficiary named?
- Are there measurable success criteria (not "users will love it")?
- Do features reference the value they deliver, or just what they do?
- Are key assumptions and top risks acknowledged?

---

### DIM-10: Prioritization (weight: 2)

> Does the spec tell an implementer what is essential vs optional?

Under time pressure, an implementer must know what to cut. A spec where every
requirement reads as equally mandatory forces that judgment back onto the author or, worse,
onto the engineer guessing. IEEE 830 §4.3.5 requires requirements be **ranked for
importance and/or stability**; RFC 2119 provides the obligation vocabulary
(MUST / SHOULD / MAY).

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | Requirements carry consistent importance/obligation markers — RFC 2119 keywords (MUST/SHOULD/MAY, capitalized), MoSCoW, or Essential/Conditional/Optional — applied throughout, so binding and optional requirements are distinguishable at a glance. Release/build ordering reflects priority. |
| 2 | Major requirements marked; some sections leave obligation implicit. |
| 1 | Priority mentioned in a few places but mostly absent; most requirements read as uniformly mandatory. |
| 0 | No prioritization signal anywhere. |

**Signals:**
- Are RFC 2119 keywords used with their normative meaning (and only in caps, per RFC 8174)?
- Is there a MoSCoW / Essential-Conditional-Optional / P0-P1-P2 scheme?
- Can you tell, for any two requirements, which matters more?
- Does build ordering or a phased plan reflect the priority?

---

### DIM-11: Traceability (weight: 3)

> Can you link each requirement forward to code and tests, and each code path back to a
> requirement?

Traceability is what turns a spec into an auditable artifact. **Forward** traceability
(requirement → design → code → test) proves each requirement is realizable and testable;
a requirement with no derivable test is a coverage gap. **Backward** traceability
(code/test → requirement) catches **gold-plating** — implemented behavior that no
requirement asked for, which is exactly what the spec is silent about. 29148 §6.5.2.3
formalizes both directions with a Requirements Traceability Matrix.

**Scoring:**

| Score | Criteria |
|-------|----------|
| 3 | **Forward:** every requirement carries (or can be given) a concrete code reference with file path AND a derivable test — the derivation test (DIM-4) succeeds spec-wide. **Backward:** an adversarial sweep of random code paths finds each already covered by a documented requirement — no orphan behavior. Bidirectional links resolve. |
| 2 | Most requirements have code references; forward test-derivation works for critical paths. Backward sweep finds ≤2 orphan code paths. |
| 1 | Code references sparse or function-name-only (no file paths). Forward derivation works only for happy paths. Backward sweep finds significant undocumented behavior. |
| 0 | No code references. No way to link requirements to implementation in either direction. |

**Checks:**
- **Forward:** sample requirements; does each resolve to a file-path code reference and a
  test you can write from the spec alone?
- **Backward:** pick random code paths within documented scope; is each traceable to a
  stated requirement, or is it gold-plating? (Shares evidence with DIM-1's adversarial
  pass, but scores the *link coverage*, not the correctness of what's written — see the
  double-counting rule.)
- Are spec↔code references bidirectional (if the spec cites a file, the file's behavior
  is covered by the spec)?

---

## Evaluation Process

### For Manual Review

1. **Read the header.** Source-of-truth declaration, scope, companion list. (DIM-7)
2. **Scan structure.** TOC, section depth, stubs, appendices, density. Non-goals section?
   Problem statement? (DIM-2, DIM-7, DIM-9)
3. **Lint the text.** Weak-word / ambiguity scan, compound-requirement scan, TBD/TBR scan,
   RFC 2119 keyword usage. (DIM-4, DIM-2, DIM-10)
4. **Verify ≥50 claims against code, reference-guided.** Stratify across every section.
   ≥20 behavioral. Run adversarial pass (5 random code paths). Distinguish authoring-time
   errors from staleness. (DIM-1, DIM-3, DIM-11)
5. **Precision + derivation test on 5 workflows.** Can you implement AND write ≥5 tests
   (including negative tests) from the spec alone? Does each carry a worked example? (DIM-4)
6. **Assess decision quality and product justification.** Count decisions with rationale;
   check for industry precedent, rejected alternatives, quantitative data. Separately:
   problem statement, beneficiary, success metrics. (DIM-5, DIM-9)
7. **Walk user journeys.** Dead ends? Cross-references agree? Workflows use fields that
   exist? (DIM-6)
8. **Check completeness denominator.** Route, store, API audits. Gap inventory for IDs,
   severity, methodology. Security/privacy addressed? (DIM-2)
9. **Check traceability both directions.** Forward req→code→test; backward code→req
   (gold-plating). (DIM-11)
10. **Assess feasibility** (target specs only). Migration path, dependencies, constraints,
    breaking changes. (DIM-8)
11. **Adversarial grilling.** Assume the spec is wrong and try to prove it. Challenge
    plausible claims, find conspicuous absences, stress-test decision rationale, trace one
    feature end-to-end through every section, find lies of omission (concurrency,
    performance, permissions, undo boundaries, state cleanup). Grilling findings adjust
    scores downward. (All dimensions)
11b. **Competitive benchmarking** (target specs only, non-scoring). Benchmark the spec
    against the Competitor Reference Set (bottom of this rubric). Per domain, name the
    comparable platforms and their best-in-class patterns; mark each adopt/adapt/decline.
    Emit `[COMPETITIVE]` proposals for human review. Moves no score directly; a
    consequential decision made with no field benchmark feeds DIM-5. (Backlog, not a fix.)
12. **Score each dimension 0–3** per the Scoring Discipline (reference-guided, quoted
    evidence, median of repeated samples, neutrality). Apply double-counting rule and
    hard-fail gates. Compute weighted total.

### For Automated Grading (skill)

**Pass 1 — Structural analysis + text lint** (spec text only):
- TOC, stubs, source-of-truth, companion list, terminology consistency
- Code reference density, cross-reference resolution
- Gap inventory structure (IDs, severity, methodology); **non-goals section present?**
- **Security & privacy addressed?**
- **Problem statement / beneficiary / success metrics present?**
- **Weak-word / ambiguity scan, compound-requirement scan, TBD/TBR scan**
- **Prioritization: RFC 2119 keyword usage / MoSCoW / importance markers**
- Redundancy scan; density test (3 random 50-line blocks)
- Decision rationale count (decisions with "because" vs bare statements)
- **Scores:** DIM-2 (partial), DIM-4 (lint portion), DIM-5, DIM-6 (partial), DIM-7, DIM-9, DIM-10

**Pass 2 — Code verification, reference-guided** (requires codebase):
- Derive correct behavior from code first, then verify ≥50 claims, stratified, ≥20 behavioral
- Adversarial pass: 5 random code paths → is the spec accurate (silent omission → DIM-1)?
- Backward-traceability sweep: are those code paths covered by a requirement (→ DIM-11)?
- Check file paths resolve, status tables match
- **Scores:** DIM-1, DIM-3, DIM-11 (backward)

**Pass 3 — Completeness audit** (requires codebase):
- Route/page audit, store audit, API endpoint audit; compare against spec sections
- Forward traceability: does each requirement resolve to a code reference and a test?
- **Scores:** DIM-2 (complete), DIM-11 (forward)

**Pass 4 — Precision + Coherence** (spec + code):
- Precision test: 5 workflows, implement + test from spec alone
- Derivation test: ≥5 test cases per workflow including negatives; worked example present?
- Coherence: 5 workflow-to-data-model checks, user journey walk, keyboard shortcut consistency
- Boundary conditions and error handling per workflow
- **Scores:** DIM-4, DIM-6

**Pass 5 — Feasibility** (target specs only):
- Migration notes, dependency ordering, constraint acknowledgment, carry-over marking,
  breaking-change identification
- **Scores:** DIM-8

**Pass 6 — Adversarial Grilling** (mandatory, every iteration):

The grilling pass assumes the spec is wrong and tries to prove it. This is NOT
verification (Pass 2) — it's an adversary looking for problems that survive
verification. Findings from this pass adjust scores downward across all dimensions.
Run it **every fix-loop iteration**, and additionally hunt the four reward-hacking
surface features a fix loop tends to add: **lexical** (rubric keywords sprinkled in),
**tone** (confident closings), **self-praise**, and **format** (rigid templates / added
sections that raise the score without raising quality).

The grilling agent should:
1. **Challenge plausible claims.** Pick 5 claims that sound right but are the kind
   of thing that's often wrong — counts, behavioral chains, implicit assumptions.
2. **Find conspicuous absences.** For each major section, ask: "What question would
   a new engineer ask that the spec doesn't answer?" List specific unanswered
   questions, not vague "could be more detailed."
3. **Stress-test decisions.** For each decision with rationale, try to refute the
   rationale. Find decisions where the reasoning doesn't hold up.
4. **Trace one feature end-to-end.** Pick a single feature and follow it through
   every section: user journey → workflow → data model → persistence → export →
   error handling → migration. Document every gap or contradiction.
5. **Find lies of omission.** Concurrency scenarios, performance implications,
   RBAC/permissions, undo boundaries, state cleanup on navigation, multi-tab
   behavior, offline behavior, browser compatibility.

Output: ranked findings, most severe first. Each finding is specific (not vague)
with impact (what would go wrong) and suggested fix or flag.

**Scores:** adjusts ALL dimensions. A spec that passes verification but fails
grilling does not get a high grade.

**Pass 7 — Competitive Benchmarking** (target specs only, non-scoring, once per run):

Benchmark the spec against the **Competitor Reference Set** (bottom of this rubric).
For each domain the spec covers, name the comparable competitors and the best-in-class
patterns they exhibit; decide adopt / adapt / decline for each. This is the inverse of
grilling — it hunts opportunities the spec omits relative to the field, not defects in
what it states. Every finding must name a specific competitor and capability (same
specificity bar as grilling). A missing feature the product never claimed is a scope
proposal, **not** a spec defect — it lowers no score.

Output: ranked `[COMPETITIVE]` findings, each with `{pattern, competitor(s), gap
(quoted §), recommendation: adopt|adapt|decline, target section}`.

**Scores:** none. Findings feed (a) the fix step as `[COMPETITIVE]` flag-for-human-review
items (never auto-applied — they add unbuilt scope), and (b) DIM-5, only where a
consequential decision lacks any field benchmark. Because they never enter the spec
automatically, they cannot inflate the grade.

**Output:**
- Per-dimension record `{dimension, score, confidence, rationale, evidence_quote}` (0–3)
- Hard-fail gate status
- Verified claims list with results
- Adversarial findings (code paths not covered by spec) + backward-traceability result
- Precision test results per workflow
- Coherence check results
- **Grilling findings** (ranked by severity)
- Fix list sorted by `weight × (3 − score)` descending
- Overall grade with percentage and letter

---

## Dimension Weight Rationale

| Dimension | Weight | Rationale |
|-----------|--------|-----------|
| Correctness | 5 | Wrong content actively harms — engineers build on false assumptions. All other failure modes are passive. |
| Precision | 5 | Equal to Correctness. A spec too vague to build or test from is functionally absent for the sections that lack precision. Implementability and testability are the same requirement at different thresholds. |
| Completeness & Honesty | 4 | Missing coverage creates false confidence. Slightly below Correctness because hitting an undocumented feature is detectable; hitting a wrong claim is not. |
| Freshness | 4 | Equal to Completeness. Staleness is functionally equivalent to being wrong, but the double-counting rule prevents it from also penalizing Correctness. |
| Decision Quality | 4 | A spec without rationale is fragile — it can't survive a stakeholder review, a design challenge, or a new team member asking "why?" |
| Coherence | 3 | Catches the failure mode the other dimensions miss: sections that are individually good but don't fit together as a system. |
| Feasibility | 3 | Target specs only. A beautiful design that can't be built is worthless. |
| Justification & Value | 3 | A spec that never states the problem, the user, or success metrics may be built flawlessly and still solve nothing (validation vs verification). |
| Traceability | 3 | Turns the spec into an auditable artifact; the backward direction is the only systematic gold-plating detector. Elevated from a DIM-7 sub-bullet because forward-to-test and backward-to-requirement are load-bearing, not cosmetic. |
| Prioritization | 2 | Low weight — a spec can be built without it — but its absence forces cut/keep decisions onto the wrong people under pressure. |
| Document Craft | 2 | Lowest weight. Bad craft makes good content harder to use but doesn't change whether the content is right. |

**These weights should be calibrated** by grading 5–10 real specs against human judgment
and checking agreement (ordinal Krippendorff's α ≳ 0.7 before trusting the letter grades).
The rationale explains the relative ordering, not the exact numbers.

**Maximum possible score:**
- Target spec: 114 (11 dimensions)
- Current-state spec: 105 (10 dimensions, no Feasibility)

---

## Competitor Reference Set

> Curated list of comparable production platforms, consumed by the **Competitive
> Benchmarking** pass (skill Step 6c). It is **not a scoring input** — it drives the
> non-scoring pass that emits `[COMPETITIVE]` adopt/adapt/decline proposals for human
> review. **Target specs only.** Maintain this list as the field moves: add a platform
> when it ships a pattern worth benchmarking against, remove one that stops being
> comparable. Names already used as design grounding in `WORKFLOW_CONSOLIDATION.md`
> (Metashape, PLS-CADD, Trimble, NESC) MUST stay consistent with this table.

| Domain (spec §) | Comparable platforms | Best-in-class patterns to benchmark against |
|-----------------|----------------------|---------------------------------------------|
| Point-cloud viewer & rendering (§3.6) | Potree, CesiumJS, CloudCompare, Leica Cyclone 3DR, Trimble RealWorks | Eye-dome lighting, classification-driven visibility, section/measurement tools, clipping volumes, LOD streaming (COPC/Entwine/3D Tiles) |
| Photogrammetry / reconstruction (§1–§2) | Agisoft Metashape, Pix4Dmatic, Bentley ContextCapture / iTwin, RealityCapture | GCP/marker workflows, quality reports, tie-point editing, region-of-interest reprocessing, confidence/uncertainty surfaces |
| Pole modeling & structural analysis (§3.3 PLA, §4 export) | PLS-CADD / PLS-Pole (Power Line Systems), SPIDAcalc (Bentley), O-Calc Pro (Osmose), Bentley OpenUtilities / OpenTower | Component/framing catalogs, sag-tension & load-case templating, span/clearance templates, round-trip with analysis engines |
| Utility asset extraction from LiDAR (§3.3, §3.13) | Neara, Sharper Shape, Katapult Pro, Bentley OpenTower iQ | Automated pole/attachment detection, clash & clearance detection, network topology, encroachment analytics |
| Survey / field-to-finish (§3.4) | Trimble Business Center, Carlson Survey, Leica Infinity | Feature-code libraries, linework rules, layer/symbology profiles, point-to-CAD automation |
| Alignment / georeferencing (§3.5 ATC) | Trimble RealWorks registration, Leica Cyclone REGISTER, CloudCompare align | Target/GCP registration, per-point residual reporting, cloud-to-cloud refinement, revertable transforms |
| Clearance / vegetation / compliance (§3.13–3.15) | NESC (standard), GO 95 (CA), AiDash, Overstory, Neara | Rule-set libraries (NESC/GO 95), grow-in/fall-in risk, thermal-rating scenarios, encroachment reporting |
| Interaction & annotation UX (cross-cutting §3.12) | Blender (modal operators), AutoCAD / Civil 3D, Figma (multiplayer) | Modal confirm/cancel, command palette, non-destructive edit + undo history, keyboard-first workflows, presence/locking |
| Deliverables & interop (§4 export) | PLS-CADD, SPIDAcalc, O-Calc, LandXML / DXF ecosystems | Lossless round-trip, versioned exports, format validation, provenance metadata |

**Usage:** for each finding, cite the specific competitor **and** the capability — a
finding that can't name who does it and how does not clear the specificity bar. Prefer
patterns that improve *our* users' outcomes; discard competitor-specific noise. When the
recommendation is **decline**, record why (a justified divergence is itself DIM-5
rationale worth adding to the spec).
