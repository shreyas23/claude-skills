# Worked example — WEB-1330

Real ticket from this workspace, authored in-house, and the best-formed feature ticket in
it. Study the mechanics, not the subject.
<https://linear.app/looq-engineering/issue/WEB-1330>

Its headings predate this skill's structure. The mapping:

| WEB-1330 heading | Section |
|---|---|
| Purpose (¶1) | Problem |
| Purpose (¶2) | Context — why this approach, and what it costs to not take it |
| Format, Critical | Context |
| Required implementation | **Does not map.** See "What not to copy" below. |
| Acceptance criteria | Acceptance criteria |
| Notes (¶1) | Out of scope |
| Notes (¶2-3) | Context |

It carries no explicit **Desired outcome**. That is the one thing the structure adds:
the reader has to assemble the end state from the implementation list. One sentence
("Web can read a capture's GPS track at upload time, or determine that the capture has
none") would have paid for itself.

## The ticket

> ## Purpose
>
> LMR v31 carries the capture's GPS track **in plaintext in the header**, and web does not
> read it. It is the only source of capture position available at upload time — the
> alternative is decrypting `rig_000/gps/record_*` and parsing raw SBF, which web has no
> capability for and which would require the private key in the browser.
>
> `LMR_HEADER_CONFIGS[31]` (`gnssFileParsing.ts:50-54`) already reads this header; the
> field sits in the same 8192-byte slice. No decryption, no S3 round-trip, no extra fetch.
>
> ## Format
>
> Defined at `capture_service/src/trajectory/__init__.py:127-130`, field span
> `TRAJECTORY [2603, 6718]` per `lmr_file/lmr_file_pkg/def/31.json`:
>
> ```
> [0:2]   u16 count        (little-endian, max 512)
> [2:18]  4 x i32 bbox     (minlat, minlon, maxlat, maxlon)
> [18:]   count x (i32 lat, i32 lon)
>                          all fixed-point round(deg * 1e7)
> ```
>
> ## Critical: an empty trajectory writes a zeroed bbox, not an absent one
>
> ```python
> if q:
>     bbox = (min(lats), min(lons), max(lats), max(lons))
> else:
>     bbox = (0, 0, 0, 0)
> ```
>
> Reading the bbox without checking `count` first yields **null island** — a
> plausible-looking coordinate that is wrong. Note also that the Python getter returns 18
> zero bytes rather than `b''` when unpopulated, so truthiness is not a valid emptiness
> test. `count == 0` **is the only correct check.**
>
> ## Acceptance criteria
>
> * A populated v31 trajectory decodes to the expected bbox and point count against a
>   fixture with known values.
> * An empty trajectory is reported as absent, **not** as a bbox at `(0,0)`.
> * A pre-v31 header reports absent without error.
> * Out-of-range values after decoding are rejected.
> * Fewer than the minimum valid points is distinguishable from absent.
> * `FW_VERSION` is available in the output.
>
> ## Notes
>
> Enabling work only — no comparison logic here. [...]

## Why it works

**The problem is a problem.** "Web does not read it" is a capability gap, not a task. The
sentence that follows says why it matters and why the obvious alternative is off the
table, so nobody re-litigates SBF decryption in review.

**Six anchors, all specific.** Two file:line spans, a schema file, a byte range, a symbol
name, a code excerpt of the producer. The assignee opens the ticket and starts reading
code, not asking questions.

**The gotcha has its own heading.** An empty trajectory serializing as `(0,0,0,0)` is the
one thing that silently produces a plausible wrong answer. It is bolded, explained, shown
in producer source, and given the exact correct test (`count == 0`). This is the single
highest-value paragraph in the ticket.

**Scope is fenced from both sides.** Notes says what this ticket does *not* do (comparison
logic) and Context says what *not* to rebuild (producer-side quality gating, listed
defense by defense). Both prevent real, likely scope drift.

**Four of six criteria are negative cases.** Empty, pre-v31, out-of-range, below-minimum.
Each is falsifiable and each names its check. Compare against what a happy-path-only
version would have caught: nothing that actually breaks in production.

**Related work is linked, not summarized.** WEB-1325, DEV-384, DEV-385, each with one line
on why it is separate.

## What not to copy

WEB-1330 has a **Required implementation** section. Do not reproduce it. It is the one part
of an otherwise exemplary ticket that takes work away from the assignee, and every bullet
in it is either a requirement wearing an instruction's clothes or a fact that belongs in
Context.

> ## Required implementation
>
> * Decode count, bbox, and points from the header slice already being read.
> * `count == 0` → trajectory absent (expected for every pre-v31 capture, and for any v31
>   capture that never closed its session).
> * Post-decode range guard: `|lat| ≤ 90`, `|lon| ≤ 180`. Cheap, and catches any future
>   producer defect.
> * Below ~10 valid points, report the trajectory as present-but-insufficient rather than
>   usable.
> * Expose `FW_VERSION` (LMR header `[474, 574]`) alongside the trajectory, so a known-bad
>   firmware range can be quarantined retroactively without re-parsing.
>
> Keep consumer-side hardening narrow. The producer already gates on quality ≥ 2,
> `hAcc ≤ 25 m`, finite, in-range, null-island, 1 m stationary dedupe, and Douglas-Peucker
> simplification — do not rebuild those defenses.

Redistributed:

| Bullet | Where it goes |
|---|---|
| Decode count, bbox, points from the header slice already being read | Context, restated as a constraint with its reason: the field is inside the 8192-byte header slice `gnssFileParsing.ts:50-54` already fetches, so no additional read is required. |
| `count == 0` → absent | Context. It is a fact about the producer's serialization, already established under **Critical**. The requirement it implies is criterion 2. |
| Post-decode range guard | Criterion: decoded values outside `\|lat\| ≤ 90` / `\|lon\| ≤ 180` are rejected. Already present as criterion 4. |
| Below ~10 points, report present-but-insufficient | Criterion 5, once the threshold is fixed at 10 rather than "~10". |
| Expose `FW_VERSION` | Criterion 6. The byte range and the retroactive-quarantine rationale go to Context. |
| Keep consumer-side hardening narrow | Out of scope, with the producer's defense list verbatim. |

Nothing is lost in the move. The section was carrying six things, four of which the
acceptance criteria already stated, and the ticket gets shorter.

## The same ticket, badly written

> **Improve LMR trajectory handling**
>
> We need to parse the trajectory from the LMR header so we can get capture positions.
> The header has a trajectory field with lat/lon points and a bounding box. This will let
> us show where captures were taken and validate base stations. Should handle edge cases
> appropriately and be robust to older files.
>
> Acceptance criteria: trajectory parsing works correctly and handles empty trajectories.

Every defect this skill gates on, in six lines:

- Title names a direction, not a change.
- "We need to parse" is the solution standing in for the problem. Why does it matter that
  we cannot? Unanswered.
- Zero anchors. The assignee greps for two hours to find what the ticket knew.
- The null-island trap is gone. They will ship it and it will look fine.
- Scope drifts into base-station validation, which is a different ticket (WEB-1325).
- "Appropriately", "robust", "correctly", "works" — four weak words, nothing falsifiable.
- Two criteria, both happy-path, neither naming a verification.

## Acceptance criteria, before and after

| Weak | Falsifiable |
|---|---|
| Handles empty trajectories | An empty trajectory is reported as absent, not as a bbox at `(0,0)` |
| Robust to older files | A pre-v31 header reports absent without error |
| Parsing works correctly | A populated v31 trajectory decodes to the expected bbox and point count against a fixture with known values |
| Validates the data | Decoded values outside `\|lat\| ≤ 90` / `\|lon\| ≤ 180` are rejected |
| Fast enough | Header decode adds under 5 ms per capture at p95, measured over the 200-capture upload fixture |
| Good UX for missing data | Below 10 valid points, the trajectory is reported present-but-insufficient, distinct from absent |
