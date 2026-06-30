# Daemon — triggers & publish path

Runtime layer. Vocabulary is `types.kdl`, table-setting `profile.kdl`, output
`checker-contract.md`. This is what watches, when it rechecks, and how the
warrant reaches the wire.

## Two clocks (read first)

- **`epoch`** — daemon-internal monotonic batch counter. Subscriber dedup,
  retained-message versioning, scheduler slots. *Never* compared to git.
- **timestamp / commit-ref** — external. `reviewed`, `fresh-for` (duration),
  subject `last_change`, vouch `expires`. All staleness math lives here.

> Correction to the previous pass: `reviewed` was typed `epoch`. It's a
> `timestamp` (date or git commit-ref). Absolute currency is
> `now - reviewed > fresh-for`, which needs a wall clock — not the batch
> counter. The two clocks were quietly conflated; they're now separate.

## Staleness has two independent axes

- **Absolute currency** (already built): `now - reviewed > fresh-for`.
  Clock-triggered. "Not looked at in too long, regardless of change." Floors
  readiness to `unavailable`.
- **Relative staleness** (your idea): `subject.last_change > reviewed`.
  Event-triggered. "Reviewed *before* the subject moved." A doc reviewed
  yesterday is relatively-stale if its subject changed today; a doc reviewed two
  years ago is *not*, if nothing it covers changed since.

They're orthogonal — a doc can pass one and fail the other. And relative
staleness is, by default, a **review advisory**, not a failure: a moved subject
makes a doc *suspect*, not *wrong*. That's the accuracy boundary again — the
checker sees that the ground moved, not that the words are now false. So it
raises a flag and does **not** touch `ok` or readiness unless you opt in
(`tracks … flags="provisional"`), which floors readiness to `provisional`.

## Subject relation — what a doc "covers" (`tracks`)

Per doc type:
- `tracks subjects="<edge>"` — the targets of that edge (`design-doc`→documents,
  `skill`→augments).
- `tracks scope="enclosing"` — the doc's enclosing anchor subtree (`readme`,
  `context-doc`).
- A node may add path globs in its own frontmatter: `tracks "src/checker/**"`.

`process-record` tracks nothing — timeless, same reason it has no `currency`.

## Two change signals per node

- **frontmatter-hash** drives **conformance** rechecks. Body-agnostic; prose
  edits never reconform. Unchanged from before.
- **`last_change`** = commit time of the subject's whole file / covered path
  (`git log -1 -- <path>`, or mtime). Drives **review**. Body *included* — a
  body or code edit is exactly what should flag a doc, even though the subject's
  type and frontmatter are still valid.

So a pure-body edit to `P`: no reconformance (correct — `P` still conforms), but
`P.last_change` advances → docs that track `P` get a review advisory. Precisely
the intent.

## Trigger taxonomy (all five)

| trigger | detect | recheck set | cost |
|---|---|---|---|
| content change | frontmatter-hash moved | the doc: local + outgoing edges | O(file) |
| identity change | id/type/create/delete | doc + `rdeps[old] ∪ rdeps[new]` | O(fan-in) |
| schema change | types/profile hash moved | re-elaborate; schema-invalid or all-dirty | O(repo) |
| clock expiry | wheel fires at `reviewed + fresh-for` | that doc's currency | O(doc) |
| **subject change** | `last_change` advanced on a tracked path/node | docs tracking it — already its `rdeps` via documents/augments | O(fan-in) |

The win: **subject-change reuses `rdeps`**. The `documents`/`augments` edges —
made referential-only and non-readiness-propagating earlier — were waiting for a
job; this is it. They already place a doc in its subject's `rdeps`, so a subject
change enqueues the doc for free; the doc's recheck only adds the
`last_change > reviewed` comparison. No new propagation, no new index — the one
addition is a *broader watch* (file/path commit time, not just frontmatter-hash)
feeding that comparison.

## Timing wheel (the only clock-driven trigger)

Absolute currency needs a clock, not a watcher — the first non-file trigger.
Keep a wheel: each doc with `fresh-for` schedules one entry at
`reviewed + fresh-for`; firing rechecks that doc's currency, floors readiness if
expired, publishes. One entry per doc-with-`fresh-for`; firing is O(due docs).
Relative staleness does **not** use the wheel — it's event-driven off subject
commits.

## Publish path — the warrant on the wire

Topic tree under `checker/<repo-id>/`:

| topic | retain | payload |
|---|---|---|
| `warrant` | **retained** | full certificate rollup + `epoch` — the trust gate for context-pull agents; a late subscriber gets current state at once |
| `shape/<anchor>` | **retained** | per-anchor completeness; an agent in one subtree subscribes to just its anchor |
| `diagnostics/<file>` | transient | per-file conformance deltas; react only to what changed |
| `review` | transient | relative-staleness advisories `{doc, subject, subject_commit, reviewed}` — your stream, on its own topic so it's consumable without the error feed |

Retention rule: anything a late subscriber must *know* → retained; anything that
only makes sense as an *event* → transient.

**LWT** — the daemon's Last Will publishes `level:"unwarranted", daemon:"down"`
to `…/warrant`. This is the read-path's safety: a context-pull agent must never
trust a stale retained `warranted` from a dead daemon. The instant the
connection drops, the broker flips the gate to untrusted.

**`epoch`** stamps every envelope; subscribers keep the highest and discard
older. Persisted across restart (resume monotonic) so post-restart messages
aren't mistaken for stale.

## Publish loop (per settled batch)

1. Coalesce the batch first — debounce, or treat a git `post-commit`/`post-merge`
   as one transactional event — so a checkout doesn't storm.
2. If types/profile hash moved → re-elaborate. Fail → publish `level:"unwarranted"`
   to `…/warrant`, stop.
3. Run the recheck set across all five triggers. Replace each touched file's
   diagnostics; update the `failing` set; emit review advisories where a tracked
   subject's `last_change > reviewed`.
4. Recompute affected anchors' shapes; fold `complete`.
5. Assemble the warrant; bump and persist `epoch`.
6. Publish: retained `warrant`, retained changed `shape/<anchor>`, transient
   `diagnostics/<file>` deltas, transient `review` advisories.

## Cold start

Same full pass as before; the cache turns "parse + check everything" into
"hash everything, parse the misses." Two additions: read each tracked subject's
`last_change` (batched `git log`) to seed relative staleness, and populate the
wheel from `reviewed + fresh-for`. Both derive purely from files, so the cache
stays reconstructible and the persistence invariant holds: delete the state dir,
re-`track`, everything rebuilds from the repo.
