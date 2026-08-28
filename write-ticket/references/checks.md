# Checks and lookups

## Weak-word list

Flag every occurrence in a requirement or acceptance criterion. Each has no single
testable meaning, so each is a precision defect.

```
appropriate, adequate, sufficient, efficient, fast, robust, user-friendly, easy,
flexible, seamless, minimize, maximize, optimize, as appropriate, as needed,
if required, etc., and/or, support, handle, robustly, quickly, improve, better,
properly, correctly, works, TBD
```

Consolidated NASA / INCOSE / QuARS "words to avoid" list. It is the same list
`grade-spec` uses (`claude-skills/grade-spec/PRODUCT_SPEC_RUBRIC.md`, Unambiguity scan) plus
four verbs that show up specifically in tickets. Keep the two in sync when either changes.

**Compound-requirement scan.** Flag any criterion with multiple main verbs or "X and Y and
Z". A compound criterion gets marked pass while half of it is unbuilt.

**Placeholder scan.** Any `TBD`, `TBR`, `???`, or bare "figure out" means the ticket is not
ready to file. Either resolve it or make resolving it the ticket.

**Implementation scan.** Flag every imperative aimed at the assignee, anywhere in the
ticket: `add, create, implement, build, refactor, use, call, store, extend, wire, hook up,
modify, replace, split, cache, expose, rename`. Each one is a directive. Route it: an
observable requirement becomes an acceptance criterion, a fact about existing code or data
becomes Context, a fence becomes Out of scope, and the rest is deleted. See `SKILL.md` §3.

The exception is a mandatory constraint stated *with its reason and anchor* ("must reuse
the header read at `gnssFileParsing.ts:50-54`, because a second fetch doubles upload-path
S3 calls"). A bare directive with no reason is never the exception.

## Acceptance-criteria audit

Run over the whole set, not one criterion at a time. Details and rationale in `SKILL.md` §4.

| Question | Fails when | Fix |
|---|---|---|
| **Meaningful** — can it fail? | An either/or that covers itself ("closed, or documented as knowingly unchecked"). | Make the decision, or move it to its own ticket. |
| | A count or list asserted but never checked against the code ("each of the five entry points"). | Verify the enumeration like any other anchor. |
| | An unfalsifiable adjective ("an actionable message", "tests extend these files"). | State the observable, or delete. |
| **Reasonable** — proportionate to the structure? | Per-surface enumeration where the code has one funnel. | Enumerate call sites or branches instead. |
| | A criterion whose verification costs more than the defect it guards. | Fold into an existing fixture set. |
| **Necessary** — does deleting it lose coverage? | Another criterion's fixtures already fail whenever this one would. | Delete. |
| | A regression floor ("a valid input still succeeds"). | Delete; the rest of the set implies it. |
| | Depends on none of the ticket's blockers. | Not redundant, misplaced. Move to the split-out ticket. |

**Every count, list, and "all N surfaces" inside a criterion is an anchor.** Verify it
against the code before filing, exactly as you would a Context claim. A criterion built on
an unverified enumeration can demand a check that cannot exist.

**A stated exception beats a mechanical split.** Two clauses may share a criterion when
they are not separately shippable. Say why, once.

## Self-check card

Run before showing any draft. All six must pass.

1. **Problem is a problem, and nothing prescribes an implementation.** Delete every
   sentence naming the thing to be built. Does a problem remain? If not, rewrite from the
   pain. Then run the implementation scan over all five sections.
2. **Anchored.** Every factual claim carries a path, symbol, issue ID, URL, or pasted
   output — or an explicit `[ASSUMPTION]` marker with what would confirm it. Paths verified
   to resolve.
3. **Criteria hold up.** Each is singular, falsifiable by a reviewer without judgment, and
   names its verification. At least one covers a negative or boundary case. The set passes
   the meaningful / reasonable / necessary audit above.
4. **Weak-word scan clean**, compound scan clean, no placeholders.
5. **Out of scope names a real exclusion**, not "future work in general". **Context is the
   user's own product or workflow knowledge, or the section is absent** — no ticket
   cross-references (Linear relations carry those), no code archaeology (that is evidence,
   and belongs in Problem or beside its criterion).
6. **Cold reader has zero blocking questions.** Every referent resolves. No session
   artifacts ("the file I just wrote", "as we discussed").

## Repo → team routing

Verified against the workspace 2026-08-27. Confirm with `list_teams` before filing; teams
change.

| Team | Prefix | Repos |
|---|---|---|
| Web | `WEB` | `web/` (web-client, web-admin, web-backend, packages), `webapp/`, `webservice/`, `looq-website/`, all `web-*` worktrees, and web-owned terraform under `web/terraform` |
| Computer Vision | `CV` | `cloud_image_processing/`, `pipeline/`, `dev-pipeline-executor/`, `ariadne*/`, `gcp_transform/`, `panorama_generation/`, `semantic_segmentation/`, `looq-sfm/`, `YACL/`, `qinertia_service/`, `gaussian_splats/`, `depth-mapping/` |
| Devices | `DEV` | `capture_service/`, `capture-v2/`, `device_api/`, `lmr_file/`, firmware, on-device calibration |
| AI | `AI` | `pole/`, `cvat/`, `X-AnyLabeling-Server/`, labeling workflows, model training and evaluation |
| Cloud & Infra | `INF` | `tf-aws-prod/`, AWS account and org setup, shared infra provisioning |
| Engineering | `ENG` | Cross-cutting work owned by no single repo. A fallback, not a default — prefer the specific team. |

Ambiguous cases: infra that lives inside a product repo goes to that product's team
(`WEB-955` is web-repo terraform, filed to Web), not to Cloud & Infra. Pipeline work
triggered by a device defect goes to the team that owns the code being changed, with the
other ticket linked.

Remember the split rule: a change spanning two of these teams is two tickets and a parent.

## Linear fields

`save_issue` — `team` and `title` required on create, `id` only on update.

- **labels** — verified taxonomy: type labels `Feature`, `Improvement`, `Bug`; workstream
  labels (parent `Workstream`) `infrastructure`, `web-platform`, `pipeline-integration`,
  `evaluation`, `ml-model`, `data-pipeline`; migration labels `Migration: Gap`,
  `Migration: Retirement Blocker`. Team-local labels exist too (`client` on Web). Always
  `list_issue_labels` for the target team before setting. **`labels` replaces the full
  set** — when updating, include the labels already on the issue or they are removed.
- **priority** — `0` none, `1` urgent, `2` high, `3` medium, `4` low. Leave unset unless
  the user states one.
- **estimate** — leave unset. Sizing is the team's call, not the ticket writer's.
- **project / parentId** — set both when the work belongs to a project or a split parent.
  `list_projects` first; project names are long and easy to get wrong.
- **description** — markdown with literal newlines. No escape sequences.
- **title** — match the target team's recent convention (`list_issues` on that team). Web
  and CV both use a `FEAT:` prefix on many issues and none on others; the in-house
  best-formed tickets use no prefix and an imperative statement of the change
  ("Parse the v31 TRAJECTORY field from the LMR header"). Prefer that. Keep under ~80
  characters; the title is a scannable index entry, not a summary.
- **links** — attach the PR, dashboard, or doc rather than pasting a URL into prose.

Report the returned issue URL after creating.

Updating an existing ticket: `get_issue` first, apply the same gates to what is there, then
`patch` the specific passages. Do not rewrite a description wholesale — the existing text
carries decisions and comment threads reference it.
