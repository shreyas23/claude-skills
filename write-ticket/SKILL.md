---
name: write-ticket
description: Use this skill when writing, drafting, or improving a feature ticket for the Looq.ai Linear workspace. It grounds every claim in verified code anchors, enforces the Problem / Desired outcome / Context / Out of scope / Acceptance criteria structure, makes each acceptance criterion singular and falsifiable, runs a cold-reader gate, and creates the issue in Linear only after the draft is approved. Trigger phrases — "write a ticket for...", "draft a Linear issue", "turn this into a ticket", "file this as a feature", "make this ticket better".
---

# Write Ticket — grounded feature tickets

You are writing a **feature ticket** for the Looq.ai Linear workspace. The bar: an
engineer who was not in this conversation picks it up and finishes it without asking
you a question.

Output structure, in this order:

**Problem → Desired outcome → Context → Out of scope → Acceptance criteria**

Two failure modes dominate, and both produce a ticket that reads well and is worthless.
Everything below exists to kill them.

1. **Invented fact.** A claim about how the code behaves that nobody verified. The
   assignee builds against it and finds out at review time.
2. **Conversation-dependent scope.** The ticket only parses for someone who was in this
   session. The assignee guesses, and builds the wrong thing.

## 1. Ground before writing (always first)

Read the code before writing a word of the ticket. Derive what the system actually does,
then write. Never write a claim from plausibility.

**Every factual claim in Problem and Context carries an anchor:**

- `repo/path/file.ts:50-54` — a code span, repo-relative from `~/projects/dev/`
- a symbol name (`LMR_HEADER_CONFIGS[31]`, `parseTrajectory`) — survives line drift
- a Linear identifier (`WEB-1325`), a PR or commit URL
- pasted command output, error text, or a log line
- a concrete data reference (scan `24483`, site `261`, a fixture path)

Prefer path + symbol; add `:line` when the exact span matters. **Verify every path
resolves before emitting** (`ls`, `grep`). A broken anchor is worse than no anchor: it
burns the assignee's time and discredits everything else in the ticket.

**Anything you did not verify is marked inline as `[ASSUMPTION]`** with what would
confirm it. Do not launder an assumption into prose.

**Separate the two uncertainties** — they have different resolutions:

- **Fact uncertainty** (does this function behave this way?) → verify in-repo. Never ask.
- **Specification uncertainty** (which behavior does the user want?) → ask, but only when
  the answer changes an acceptance criterion. Everything else gets an `[ASSUMPTION]`
  marker and moves on. Over-asking is a failure mode, not diligence.

## 2. The five sections

The boundary between Problem, Desired outcome, and Acceptance criteria is where tickets
rot. Each says one thing, once.

**Problem** — the current broken or missing state, who it hurts, and the evidence. No
solution, no implementation, no "we need to add". Test: if the paragraph only makes sense
when it names the thing to be built, it is a solution in disguise. Rewrite from the pain.

> Bad: "Problem: the client has no compass widget."
> Good: "Problem: in the 3D view a user cannot tell which way is north. On a corridor
> capture every direction looks alike, so users re-orient by toggling to the map, which
> reloads the point cloud (`SiteView.tsx:210`). V1 solved this with a compass; V2 shipped
> without one."

**Desired outcome** — the end state in one to three sentences, observable from outside,
implementation-free. Not a restatement of the acceptance criteria, and not a task list.

**Context** — what the implementer needs that they would not get from reading the obvious
file. Anchors, data formats, the gotcha that will silently produce wrong output, prior
decisions and why the obvious alternative was rejected, hard constraints. This is the
section that earns the ticket its keep. If a subtlety can produce a plausible-looking
wrong result, give it its own bolded line (see `references/worked-example.md`).

**Out of scope** — explicit non-goals: adjacent work that will get swept in otherwise,
defenses another layer already provides, the follow-up this deliberately defers (link its
ticket). If nothing can be listed here, the scope is not yet bounded. Go back.

**Acceptance criteria** — see §4.

There is no implementation section. See §3.

## 3. No implementation

**The ticket never says how to build it.** No step list, no "add a method to X", no file
the assignee is told to edit, no algorithm, no schema they are told to write. That is the
assignee's work, and a ticket that pre-empts it either gets followed into a worse design
than they would have found, or gets ignored along with everything around it.

The line is *facts versus directives*, not *detail versus brevity*. A ticket can be dense
with specifics and still prescribe nothing:

- **Context** — how the existing code and data actually behave, verified. "The producer
  serializes an empty trajectory as a zeroed bbox, so absence is not distinguishable by
  inspecting the bbox." That is a property of the world; the assignee needs it and it
  constrains them without instructing them.
- **Constraint** — something externally fixed that the solution must satisfy: a wire
  format, an API contract another service already calls, a schema that cannot change, a
  budget. State the constraint and its source. Do not state the code that satisfies it.
- **Acceptance criterion** — an observable requirement. Anything you want *guaranteed*
  belongs here, phrased as a check, never as an instruction.

When you catch yourself writing an instruction, convert it:

| Instruction | Becomes |
|---|---|
| "Decode count, then bbox, then points from the header slice" | Context: the field layout and where it is defined. |
| "Add a post-decode range guard for lat/lon" | Criterion: decoded values outside the valid range are rejected. |
| "Expose `FW_VERSION` alongside the trajectory" | Criterion: `FW_VERSION` is available in the output. |
| "Do not rebuild the producer's quality gating" | Out of scope, with the list of defenses the producer already applies. |
| "Use a Map keyed by scan id" | Nothing. Delete it. |

The last row is the common case. Most implementation text in a draft ticket carries no
requirement at all — it is the writer thinking out loud. It goes.

If a specific approach is genuinely mandatory, that is a constraint with a reason. Write
the reason, in Context, with its anchor. "Must reuse the existing header read at
`gnssFileParsing.ts:50-54`, because a second fetch doubles upload-path S3 calls" is a
constraint. "Reuse the header read" alone is a directive.

## 4. Acceptance criteria

Each criterion is **one assertion**, **falsifiable**, and **names how it is checked**.

- **Singular.** One main verb. No "X and Y". A compound criterion cannot be marked
  pass/fail cleanly, so it gets marked pass while half of it is missing.
- **Falsifiable.** A reviewer decides pass/fail without exercising judgment. If two honest
  reviewers could disagree, it is not a criterion yet.
- **Verification named.** A fixture and its expected value, a test path, a manual sequence,
  or a metric with a threshold and how it is measured. "Works correctly" names nothing.
- **Weak-word clean.** Scan against the list in `references/checks.md`.
- **At least one negative or boundary case.** Empty input, absent data, the pre-migration
  version, the limit. Happy-path-only criteria are the single most common defect.
- **Does not restate the desired outcome.** If a criterion is the outcome with "should"
  in front of it, delete it.

Given/When/Then when the behavior is interaction-shaped; a plain assertion when it is
state-shaped. Do not force the form.

## 5. Split rule

Propose multiple tickets under a parent when any of these hold:

- More than roughly six acceptance criteria.
- The criteria cannot all be verified at a **single merge point**. This is the real test.
- Scope crosses repo boundaries that deploy independently — `capture_service` (on-device),
  `cloud_image_processing` (pipeline), `web` (cloud), `tf-aws-prod` (infra). A change
  spanning two of these is two tickets plus a parent, near-always.
- The ticket contains an "and then" that could ship a week later without loss.

Say the split out loud with a one-line rationale; do not silently file three issues.

## 6. Cold-reader gate

Before showing the draft, read it as the assignee who has none of this conversation.

1. **Enumerate the questions they would have to ask.** Any question that blocks *starting*
   means the ticket is not done. Answer it in Context and re-run.
2. **Every referent resolves.** "This", "it", "the current behavior", "the new endpoint" —
   each must be unambiguous from the ticket text alone.
3. **No session artifacts.** No reference to a file you just wrote, a plot you just
   rendered, or "as we discussed". If it matters, restate it or link it.
4. **Every anchor resolves.** Re-check the paths.

## 7. Self-check before emitting

Run all six. Report the result. Do not show the draft until every one passes.

1. Problem states a problem, not a solution, and no section prescribes an implementation.
2. Every factual claim carries an anchor or an explicit `[ASSUMPTION]` marker.
3. Every acceptance criterion is singular, falsifiable, and names its verification.
4. Weak-word scan is clean (`references/checks.md`).
5. Out of scope names at least one real exclusion.
6. Cold-reader gate passes with zero blocking questions.

## 8. Linear handoff

**Draft in the conversation first. Create the issue only after the user approves the
text.** Never create and then ask.

On approval, resolve fields by **listing, never guessing** (`list_teams`,
`list_issue_labels`, `list_projects`, `list_issues`), then call `save_issue`:

- `team` — route by the repo the work lands in; table in `references/checks.md`.
- `labels` — always `Feature`, plus a workstream label when one applies. Labels replace the
  full set on update, so include existing labels when editing.
- `project`, `parentId` — set when the work belongs to a project or a split parent.
- `description` — the ticket markdown, literal newlines, no escape sequences.
- `title` — match the dominant convention of the target team's recent issues. Some teams
  prefix `FEAT:`, some do not. Check, do not invent.

Report the issue URL when done. To improve an existing ticket, read it with `get_issue`,
apply the same gates, and update with `patch` operations rather than rewriting the whole
description.

## 9. Writing rules

Technical register, prose over lists except where the structure is the list (acceptance
criteria, out of scope). No preamble, no restating the section heading in the first
sentence, no filler. Say the thing once. Bold only for a gotcha that will cost someone a
day.

## References

- `references/worked-example.md` — WEB-1330, annotated, plus a bad→good rewrite pair.
- `references/checks.md` — weak-word list, self-check card, repo→team routing, Linear fields.
