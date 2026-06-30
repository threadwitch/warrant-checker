# Checker — Decisions, Issue Snapshots & Regime Provenance

**Register:** descriptive — extends the main spec; same by-example KDL v2 grammar.
**Touches:** §2 (axes), §3 (vocabulary), §5 (vouch), §7 (staleness), §9 (warrant),
§13 (decision log). **Purpose:** give the design-decision record and the
discovery→promotion regime tag a fully-specified home that stays *outside* the
certified path (INV-5) and *legible from a cold start* (INV-3 / DD-1).

Three constructs are added: a **`decision`** node type (corpus doc, git-versioned),
a **frozen issue snapshot** (a value, not a resolving edge — because crosslink is
volatile), and a **`regime`** tag (a `vouch`-sibling for provenance). One render-time
courtesy (**drift check**) is specified and explicitly fenced off from every verdict.

---

## A. Where a decision lives

A decision is an ordinary **corpus document** — `dd-0001.md`, markdown body + KDL
frontmatter — checked into git/jj like every other doc. It is *not* a separate
store and *not* a tape event. Two properties are kept distinct, since "immutable"
was overloaded:

- **append-only by convention** — authored once at decision-time, not rewritten.
  This is a discipline about *use*.
- **versioned by git/jj** — being checked in is what makes that discipline
  *visible and gateable*: edits show as diffs, the original stays in history,
  rewriting the past takes a loud operation a protected branch can refuse.

git is the versioning layer *under the whole repo*, not a second home for the file.
The decision **content** is a file-in-git; only the commit **binding** (§D) is
git-native text. Cold-start reader with nothing but the repo sees the decision and
why it was made.

---

## B. The `decision` type (extends §3)

```kdl
domain "DecisionLifecycle"  { oneof "proposed" "accepted" "rejected" }
domain "EntryMode"          { oneof "accepted" "proposed" }   // how it ENTERED — a birth fact
domain "ResolutionOutcome"  { oneof "accepted" "rejected" }

type "decision" {
    include "doc-common"                 // id, title, tags, status(=Activity), reviewed, fresh-for
    tracks nothing                       // a decision is a historical fact; it does not go
                                         // stale relative to a subject (cf. process-record, §3)

    field "choice"   type=string required=#true            // what was decided
    field "rejected" type=string required=#true            // the alternative it beat (DD-style, §13)
    field "reason"   type=string required=#true            // why

    // BIRTH FACT — authored once, never edited. Default "accepted" (solo path);
    // "--propose" sets "proposed" (PR / adversarial-review path). This is how the
    // decision ENTERED, not its current status — so it stays immutable even when the
    // current status later differs (born proposed, now accepted).
    field "submitted" type=ident required=#true domain="EntryMode" default="accepted"

    edge "supersedes" "decision" repeated=#true            // corpus-internal; see §C
}

// resolution of a PROPOSED decision — a SEPARATE append-only record, not an edit.
// Only meaningful for born-proposed decisions; carries accept OR reject.
type "resolution" {
    include "common"
    edge  "resolves" "decision"                            // later → earlier; existence + type (INV-4)
    field "outcome" type=ident   required=#true domain="ResolutionOutcome"   // accepted | rejected
    field "by"      type=string  required=#true            // who resolved
    field "at"      type=timestamp required=#true          // commit-time (INV-2 external clock)
    field "reason"  type=string                            // REQUIRED when outcome=rejected (loud, §5 idiom)
}

// current lifecycle is DERIVED — init from the birth fact, overridden by a resolution
// record if the decision was born proposed. Same derive-don't-author pattern as readiness (§2).
attribute "decision_lifecycle" domain="DecisionLifecycle" {
    init "submitted"                     // accepted (terminal) | proposed (awaits resolution)
    override from="resolves" field="outcome"   // proposed → accepted|rejected when a resolution lands
}
readiness from="decision_lifecycle" override="meet" {
    map "proposed" "provisional"         // under review — don't build on it yet
    map "accepted" "stable"
    map "rejected" "unavailable"         // a rejected decision must not be depended on
}
```

**The default needs no second file; born-accepted is one record.** The common path —
solo, unconditional — creates the decision already `accepted` via the `submitted`
birth field; there is *no* resolution record, and "accepted" is simply its resting
state. The doubling cost I flagged earlier was an artifact of an over-built draft, not
this design. One decision = one file on the default path.

**`submitted` is a birth fact, not a mutable status — which is what preserves
immutability.** It records *how the decision entered* (accepted, or proposed), authored
once at creation and never edited. The *current* standing is the derived
`decision_lifecycle`, which may differ (born proposed, later accepted). So no field is
ever mutated: the birth fact is fixed, and the transition is a separate `resolution`
record. This is the distinction the "mutate or supersede?" question surfaced — entry is
authored-and-frozen, status is derived.

**`--propose` is current and rare, not future (no `unreachable.variant`, §6).** The
proposed path is reachable today — for PRs, or decisions slated for adversarial review —
so the stage isn't dead, just uncommon; the vacuity battery stays honestly quiet with no
standing exception. A born-`proposed` decision reads `proposed` until a `resolution`
record resolves it to `accepted` or `rejected`; state is computed in between.

**`resolution` carries accept *or* reject.** A proposed decision can be turned down, not
only ratified — so the record has an `outcome`, and a `rejected` outcome floors readiness
to `unavailable` (don't build on a rejected decision). `reason` is required on rejection
(loud, the §5 vouch idiom); optional on acceptance. A `resolution` pointing at a
born-`accepted` decision is `resolution.unsolicited` — you cannot resolve what was never
proposed (cf. "you cannot vouch a doc present," §5).

**Future: voting is the expansion of `resolution`, not a new mechanism.** Multiple signed
`resolution` records over one decision, with a profile-level threshold policy (how many
signatures, from whom), is the consensus direction — and it's INV-1-shaped: votes only
accumulate, acceptance is the point the count crosses the configured floor, a monotone
meet. The record type already admits it (multiple `resolves` edges); only the
threshold-in-profile and the count-derivation would be added. Not built now; the current
case is single-resolver.

**Cold-start cost, scoped to the proposed path only.** For a born-`accepted` decision,
standing is readable from the one file. For a born-`proposed` one, "is it accepted?" also
needs its `resolution` record — but both are durable git-versioned corpus artifacts
(unlike the crosslink case, §C), so it's an INV-3-clean *intra-corpus* read, not a reach
into volatile state. A stranger reads both files cold; files-as-truth holds.

**Activity is vestigial here.** `doc-common` carries `status(=Activity)`, but a
decision is *made*, not *worked on*; it sits `dormant` and nothing reads it across
an edge. Listed for row-conformance, not because it means anything.

**Supersession is advisory, not propagating (INV-4).** `supersedes` is a plain
referential edge — later decision → earlier decision, existence + type, **no
readiness propagation**. "Is `dd-0001` superseded?" is answered by `rdeps`
(reverse-deps over `supersedes`), surfaced as an *advisory* exactly like relative
staleness (§7), and does **not** lower the superseded decision's readiness. A
superseded-but-accepted decision still reads `stable` (it is well-formed and was
accepted); the consumer must check the advisory. *Rejected:* flooring readiness on
an incoming `supersedes` — it would make a second edge propagate into readiness and
deepen INV-4 for one construct, which the main spec reserves for the (unbuilt)
value-constraint extension only. If you ever want hard "don't depend on superseded
decisions," that is the place to revisit, with eyes open.

**Supersession ≠ ratification.** These are different edges with different meanings,
and conflating them is the trap. *Supersession* changes the **choice** — `dd-0009`
decides something different from `dd-0001`. *Ratification* changes only the
**status** — the *same* decision crossing `proposed → accepted`. Finalizing a
decision is never a superseding decision (that would record "I now decide what I
already decided," content-free log noise); it is a `ratification` record (§B). One
edge says *the answer changed*, the other says *the answer was ratified*.

---

## C. The frozen issue snapshot — a value, not an edge

A decision may reference the crosslink issue it answers. It must **not** do so as a
resolving edge: crosslink is derived/volatile/private and permits deletion, so a
checked edge from a durable artifact into it would dangle on cold-start and break
files-as-truth. Instead the decision **freezes what it cites at decision-time** —
read crosslink *once*, at capture; depend on it *never* again.

```kdl
// embedded in the decision's frontmatter; repeatable. A captured VALUE, not an edge.
addresses {
    handle        "iss-0042"            // crosslink id — for the reverse courtesy only
    cited         "<problem as posed at decision-time>"   // legible standalone, no crosslink needed
    identity-hash "sha256:…"            // over the identity projection ONLY (below)
    at            "git:9f3a1c2"         // capture moment (commit-time, INV-2 external clock)
}
```

This is deliberately **not** `edge "addresses_issue" "issue"`. The checker validates
the snapshot is **well-formed** (handle present, hash present, `cited` non-empty) and
never that crosslink still holds the issue — INV-5, now extended from *form-not-truth*
to *form-not-liveness*. The decision is self-contained: delete crosslink entirely and
the decision still reads.

**What the drift hash ranges over — the identity / state split.** The hash covers
the issue's *identity* (what makes it "the same problem") and excludes *workflow
state* (what churns without changing the problem). The test for each field: **does
changing this mean the decision was made against a different problem?**

```kdl
// the hash domain — reuses §3's row construct rather than a new keyword.
row "issue-identity" {
    field "title"     type=string       // identity
    field "statement" type=string       // identity — the problem as posed
    field "kind"      type=ident        // identity — build-step vs bug etc.; changing kind
                                        //            changes what was decided
}
// EXCLUDED from the hash (state — churns freely, never signals drift):
//   status, assignee, labels, board-position, timestamps, epoch
```

`status` is excluded **specifically**: an issue going `open → completed` is the same
problem, solved — hashing it would fire a false drift signal on every decision the
moment its issue closes.

---

## D. Reverse bindings (no mutation of the earlier artifact)

Every reference points **later → earlier**, so nothing is ever edited to record
something that happened after it was written. The forward views are derived via
`rdeps`, never stored on the earlier file.

| relation | carried by (the later artifact) | mechanism | checker view |
|---|---|---|---|
| issue *generated by* a decision | the **issue** (crosslink) | `from_decision: dd-0001` | crosslink's own; out of scope here — safe direction (volatile→stable) |
| decision *answers* an issue | the **decision** | `addresses` snapshot (§C) | well-formedness only (INV-5) |
| commit *implements* a decision | the **commit** | `Decision: dd-0001` trailer | referential edge — existence of `dd-0001` |
| decision *supersedes* a decision | the later **decision** | `supersedes` edge (§B) | existence + type; advisory `rdeps` |

A **trailer** is a machine-parseable `Key: value` line at the foot of a commit
description (same shape as `Co-authored-by:`). Under jj it rides the description
through every rewrite, and it cites the **stable handle** `dd-0001` — never a commit
hash — so jj's change-vs-commit-ID churn never rots it. "Which commits implement
`dd-0001`?" is `rdeps[dd-0001]` over the trailer edges; the decision file stays
forward-reference-free and immutable.

---

## E. The `regime` tag — provenance as a vouch-sibling (extends §5)

Provenance (was this built under discovery, or hardened against an independent
standard?) is **author-asserted** — the checker structurally cannot verify which
process produced an artifact (INV-5). So it takes the `vouch` idiom: reason
required, counted, loud, **expirable so it cannot rot**.

```kdl
regime "discovery" reason="exploring API shape; criteria emerge from use" {
    expires within="21d"        // promote-or-renew deadline, from this doc's commit-time
}
// values: discovery | specified | promoted
regime "promoted" reason="hardened against the acceptance suite" {
    against "spec:checker-conformance"      // the independent standard it was held to
}
```

- **Reason required** (`regime.no-reason` otherwise). Loud by construction, like vouch.
- **`promoted` requires `against`** — a pointer to *what independent standard*
  hardened it. A `promoted` with no `against` is `regime.unanchored` (you cannot
  promote against nothing; that is just `discovery` with a nicer label).
- **Clock — deliberate, per INV-2.** `vouch.expires` uses `epoch` (ordinal: "good
  for N more check-batches"). `regime.expires` uses **external commit-time**
  (`within="<duration>"` from the doc's own commit), because a promotion deadline
  means *real elapsed work*, not check-batches. This is the one place `regime`
  is **not** a clean vouch-twin: same idiom on every axis except the clock. Do not
  copy vouch's `epoch` here — that would put a provenance deadline on the internal
  ordinal clock and silently couple it to recheck frequency.

---

## F. Warrant integration (extends §9) — count, never certify

The warrant treats regime exactly as it treats vouchers: **counted and itemized,
never certified** (INV-5).

```json
"counts": { …, "by_regime": { "discovery": 3, "specified": 12, "promoted": 4 } },
"regime_expired": [ { "doc": "src/planner", "regime": "discovery",
                      "reason": "…", "since": "git:…" } ]
```

`regime_expired[]` **is** the unpromoted-discovery alarm — a `discovery` tag past
its deadline is the "promotion never happened" failure surfaced as a warrant line,
reusing vouch-expiry's machinery for free. It is a **warning-tier** signal (like the
vacuity battery, §6), never `ok=false`: aging out of discovery is a process smell,
not a conformance failure. The warrant reports the tag's *presence, form, and
expiry* — all form/currency questions it can answer — and asserts nothing about
whether the artifact truly was built that way.

---

## G. Drift check — render-time courtesy, fenced off from every verdict

When a decision is rendered **and crosslink is present**, the projection layer MAY
recompute the live issue's `issue-identity` hash and compare to the stored
`identity-hash`, surfacing *"issue `iss-0042` has changed since this decision"* —
advisory, the same relative-staleness-is-advisory move as §7 / INV-5.

Hard boundary:

- **On by default**, but lives in the **projection/display layer only**.
- **Never** enters the warrant, `ok`, `level`, readiness, or any verdict.
- It is the **only** place a decision artifact reads live crosslink, and only at
  *render*, never at integrity-time. The checked artifact stays crosslink-independent.
- **Cold-start (no crosslink):** show the frozen `cited` text; no drift signal,
  nothing broken — that is honestly all there is.

Drift is *correct*, not a bug: the decision really was made against the older framing.
The check is a courtesy to a reader who might assume the citation is current, not a
claim that anything is wrong.

---

## H. CLI view — decisions with computed state

Because active state is now **derived** (lifecycle from ratification, superseded-ness
from `rdeps`, regime from the tag, drift from the live snapshot compare), no single
file shows a decision's standing — the checker has to *fold* the scattered records.
That fold is exactly what a `decisions` subcommand should expose, so a human reads
computed state without doing the join by hand. It is a **projection over the warrant**,
not a new verdict — every column is already-derived data (§F), rendered.

```
$ checker decisions [--repo R] [--active] [--proposed] [--regime discovery] [--drifted] [--json]

HANDLE   STATE      READINESS    REGIME              ISSUE       FLAGS
dd-0001  accepted   stable       promoted            iss-0042
dd-0003  accepted   stable       discovery (14d)     iss-0051
dd-0007  proposed   provisional  specified           iss-0066    awaiting-resolution
dd-0009  accepted   stable       discovery (EXPIRED) iss-0071    regime-expired
dd-0011  rejected   unavailable  —                   iss-0073    resolved-reject: "scope creep"
dd-0001  accepted   stable       promoted            iss-0042    superseded-by dd-0012  drifted
```

- **STATE** — derived `decision_lifecycle`: `accepted` (born so, or resolved so),
  `proposed` (born proposed, unresolved), `rejected` (resolved against).
- **READINESS** — the §B projection off derived lifecycle (`rejected → unavailable`).
- **REGIME** — the §E tag; `(NNd)` shows remaining commit-time to deadline,
  `(EXPIRED)` past it (the `regime_expired[]` alarm, §F, rendered).
- **ISSUE** — the frozen snapshot handle (§C); `--json` includes `cited` + hash.
- **FLAGS** — derived advisories, none of them verdicts: `awaiting-resolution`
  (born proposed, no resolution yet), `resolved-reject` (with reason), `superseded-by`
  (`rdeps` over `supersedes`), `regime-expired`, `drifted` (live-vs-frozen identity
  hash, **only if crosslink present** — §G; absent on cold-start, never an error).

Discipline: this view **reads** derived state and renders it; it never *writes* and
never *decides*. `--json` emits the same rows structured, so an agent consuming the
read path (§9) gets the computed standing without re-deriving it. The drift column is
the one place it touches live crosslink, at render only (§G) — drop crosslink and the
column simply blanks, every other column intact from files alone.

---

## I. Decision log (§13 style) — choices this fragment commits to

- **DD-A — Decision is a corpus doc, not a tape event.** A cold-start human *needs*
  decisions to understand the code, so they are primary git/jj files, legible
  without tooling. *Rejected:* deriving the decision record from the CAS event tape
  — forfeits cold-start legibility (a stranger would see only a regeneration-
  dependent view). The high-volume tool tape stays in CAS; decisions do not.
- **DD-B — Frozen snapshot, not a resolving edge.** A decision freezes the issue's
  identity at decision-time and never reads crosslink again for integrity.
  *Rejected:* a checked `addresses_issue` edge into crosslink — a durable artifact
  referencing a deletable store dangles on cold-start and breaks files-as-truth.
  Read-now-depend-never inverts the failure.
- **DD-C — Hash identity, not state.** The drift hash ranges over title/statement/
  kind and excludes status/assignee/labels/timestamps. *Rejected:* hashing the whole
  issue — `status: open→completed` would fire false drift on every decision the
  instant its issue closes. "Same problem, solved" is not drift.
- **DD-D — `regime` is a vouch-sibling on the external clock.** Author-asserted,
  counted, expirable; provenance the checker cannot verify (INV-5). *Rejected:*
  putting regime in `readiness` — readiness is *derived dependability*, regime is
  *authored provenance*; conflating them is the §2 `status`-overload again. *Rejected:*
  vouch's `epoch` expiry — a promotion deadline is real elapsed work, not check-
  batches (INV-2).
- **DD-E — Supersession advisory, not propagating.** `supersedes` is referential;
  "superseded" is an `rdeps` advisory, not a readiness floor. *Rejected:* flooring
  readiness on supersession — deepens INV-4 for one construct outside the reserved
  value-constraint extension.
- **DD-F — Entry is an authored birth fact; resolution is an event.** A decision
  records *how it entered* (`submitted`: accepted default, proposed via flag) as a
  write-once field, and its *current* lifecycle is derived — born-accepted is terminal
  with no extra record, born-proposed is resolved by a separate `resolution` record
  (accept or reject). *Rejected:* a mutable `lifecycle` field — a mutable field on an
  immutable artifact is incoherent, the bug the born-accepted question surfaced.
  *Rejected:* making born-accepted emit a redundant self-resolution file — the default
  path needs no second artifact; accepted is simply its resting state. *Rejected:*
  recording finalization as a superseding decision — supersession changes the *choice*,
  resolution changes only the *status* (DD-E vs this).
- **DD-G — Resolution carries accept *or* reject, single-resolver now.** A proposed
  decision can be turned down; `resolution.outcome` is `accepted|rejected`, rejection
  floors readiness to `unavailable` and requires a reason (loud, §5). *Rejected:*
  ratification-only (accept) — a review that cannot reject is theater. *Deferred:*
  multi-signer voting with a profile threshold — the INV-1-shaped expansion of this
  same record, not built now (single-resolver is the current case).

---

## J. Review checklist additions (§12 style)

1. **Snapshot is a value, not an edge.** Confirm `addresses` never resolves against
   a live store and that decision integrity survives crosslink being absent/deleted.
2. **Hash domain excludes state.** Confirm `issue-identity` covers identity only;
   grep that `status` and friends are outside it (the false-drift class).
3. **Regime clock (INV-2).** Confirm `regime.expires` is external commit-time, never
   `epoch`; a provenance deadline must not ride the internal ordinal clock.
4. **`promoted` is anchored.** Confirm every `promoted` carries `against`; an
   unanchored promotion is `discovery` mislabelled.
5. **Drift stays out of verdicts.** Confirm the drift check touches the projection
   layer only and never `ok`/`level`/readiness/warrant.
6. **Reverse-only bindings.** Confirm no earlier artifact is mutated to record a
   later one; all of `from_decision`, `Decision:` trailer, `addresses`, `supersedes`,
   and `resolves` point later→earlier, forward views via `rdeps`.
7. **Entry frozen, status derived.** Confirm `submitted` is write-once (a birth fact)
   and that current lifecycle is the *derived* `decision_lifecycle`, never an authored
   editable status. A decision file edited after creation (other than a loud history
   rewrite) is the bug this guards.
8. **Resolution well-formedness.** Confirm a `resolution` targets a born-`proposed`
   decision (one against a born-`accepted` is `resolution.unsolicited`), and that
   `outcome=rejected` carries a `reason`.
