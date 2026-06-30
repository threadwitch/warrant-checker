# Checker — Design Spec
 
**Register:** descriptive — documents what the implementation will do, not what
any conformant implementation must do. **Grammar:** by-example KDL v2; no formal
production set this round. **Purpose:** design review / sanity check of the whole
design before the Rust pass.
 
Canonical machine-readable sources: `types.kdl` (vocabulary), `profile.kdl`
(per-repo). This document consolidates those plus runtime and output, and states
the cross-cutting invariants once so each construct can be checked against them.
 
---
 
## 0. What this is
 
A standalone, repo-agnostic checker for repositories of Markdown files with KDL
v2 frontmatter. The files are the single source of truth; the checker validates
each node's frontmatter against a declarative schema, resolves typed references,
and issues a **warrant** — an auditable statement of what was checked and on what
grounds it is believed. It is pointed at a range of repos (a hardware/software
inventory, project docs, agent context, monorepos, other people's repos), so the
vocabulary is shared and vendorable while the table-setting is per-repo.
 
Relative to agent-memory systems (Letta, Hermes, Egregore, crosslink): those
hold *derived, volatile, private* state. This holds *authored, validated, stable,
shared* context, and the warrant is the validation. The checker is the **trust
gate on the read path** of the stable layer — complementary to memory, not a
replacement.
 
| layer | file | content |
|---|---|---|
| vocabulary | `types.kdl` | domains, lattices, rows, node types, derived attribute, intrinsic constraint |
| table-setting | `profile.kdl` | repo plumbing + `shape` (which docs each anchor must have) |
| output | the warrant | certificate published per batch; the trust gate |
| runtime | the daemon | watches, rechecks incrementally, publishes |
 
Access surfaces, in priority order: a library/CLI (exit code + structured
diagnostics) and an MCP server are **authoritative** and synchronous; MQTT is an
**optional, additive** eventing layer and never sits on the correctness path.
 
---
 
## 1. Core invariants
 
A reviewer should be able to check every construct in §3–§9 against these five.
 
**INV-1 — Meet / confluence.** Every composition operator in the system is a
**meet** (greatest lower bound) on a **finite-height semilattice**: row `include`
merge, `refine`, readiness projection + override + currency, the
`effective_readiness` fixpoint, and cross-repo profile merge. Consequences,
which therefore hold everywhere without per-site argument:
- **Narrowing-only.** Meet can only shrink a domain. Composition that would
  *widen* is an error (e.g. `refine.widen`), not a silent change.
- **Order-independent.** Associative, commutative, idempotent → declaration order
  and include order never affect the result; diamonds resolve by meet with no
  tie-break.
- **Terminating.** Finite height → the fixpoint converges; a dependency cycle
  collapses to the meet over its strongly-connected component.
- **Bottom is an error.** A meet that reaches ∅ (`domain.empty`) is an
  unsatisfiable field, caught at load.
**INV-2 — Two clocks.** `epoch` is internal and ordinal (a monotonic batch
counter: dedup, retained-message versioning, scheduler slots). Everything in
staleness math is external wall-clock / git commit-time: `reviewed`, `fresh-for`,
subject `last_change`, vouch `expires`. The two are never compared. (A grep over
every time-typed field should confirm which clock it lives on.)
 
**INV-3 — Files as truth.** Persisted daemon state is a cache, reconstructible
purely from the files. A corrupt, stale, or absent cache can never change a
verdict — only speed. If in doubt, delete it and cold-start.
 
**INV-4 — Bounded propagation.** Referential edges check existence + type only
and do not propagate. Readiness propagates solely through `depends_on` via the
one fixpoint (INV-1 makes it terminate). No other construct traverses. The
value-constraint extension (§10) is the only thing that would deepen this, and it
is not built.
 
**INV-5 — Warrant boundary.** The warrant certifies *form, resolving references,
and currency*. It never certifies *truth*. `warranted` means well-formed and
current-signalled, not correct. Anything consuming a doc as context must not
overtrust the warrant past this line.
 
---
 
## 2. The model — three axes
 
`status` was originally one field doing three jobs; it is decomposed into three
orthogonal axes. The original "shared name vs. per-type distinct" tension
dissolved once the field was split: the job that wanted sharing was readiness,
the job that wanted per-type distinctness was lifecycle.
 
| axis | scope | kind | tier | meaning |
|---|---|---|---|---|
| **activity** | shared, in `common` as `status` | flat `{active, dormant}` | 0 (local) | are you acting on it now |
| **lifecycle** | per-type | categorical arc | 0 (local) | the entity's own arc |
| **readiness** | shared lattice | `unavailable < provisional < stable` | 2 (fixpoint) | fitness to depend on / currency |
 
Readiness is **derived**, never authored directly: projected from `lifecycle`,
optionally floored by `currency` and a down-only document override, then met over
`depends_on` into `effective_readiness`. Activity is the one purely exogenous
source — set by hand, read by nothing across edges. "Settled infrastructure off
my plate" = `readiness:stable ∧ activity:dormant`.
 
---
 
## 3. Vocabulary constructs (`types.kdl`)
 
**Lexical convention.** First positional arg names a thing; further positional
args are its value/target list; props (`key=value`) are modifiers; booleans are
`#true`/`#false`. Scalar types: `ident`, `string`, `int`, `timestamp` (date or
git commit-ref), `duration`.
 
**Domains and lattices.**
```kdl
domain "Activity" { oneof "active" "dormant" }        // flat set
lattice "Readiness" { rank "unavailable" "provisional" "stable" }   // ordered; meet = glb
```
`oneof` = unordered (tier-0 local only). `ordered`/`rank` = a lattice that can
feed a constraint or fixpoint. The syntax announces the tier.
 
**Rows and `include`.** A row is a labeled field set; `include` splices it in
(INV-1 meet). `common` carries `id, title, tags, status(=Activity)`. `doc-common`
includes `common` and adds `reviewed:timestamp` + `fresh-for:duration`.
 
**Fields.** `field "name" type=… required=#true unique=#true repeated=#true
min=… max=… domain="…"`.
 
**Types.** A node type includes a row, declares its own `lifecycle` and a
**total** `readiness` projection over that lifecycle, plus fields, edges, and
(for docs) `tracks`:
```kdl
type "software" {
    include "common"
    lifecycle "planned" "installed" "running" "deprecated"
    readiness from="lifecycle" override="meet" {       // override is down-only (INV-1)
        map "planned"    "unavailable"
        map "installed"  "provisional"
        map "running"    "stable"
        map "deprecated" "provisional"
    }
    edge "runs_on" "hardware" repeated=#true
    edge "enables" "project"  repeated=#true
}
```
The projection map **must be total** over the lifecycle domain — load error
otherwise. That totality check forces a decision about what every stage means for
dependability. For doc types readiness reads as **currency**, and a `currency`
clause contributes a floor when a review has expired:
```kdl
readiness from="lifecycle" override="meet" {
    map "draft" "provisional"  map "current" "stable"  map "stale" "unavailable"
    currency field="reviewed" within="fresh-for" else="unavailable"
}
```
 
**`refine` (available, unused here).** `refine "field" <subset…>` narrows an
inherited field's domain — the canonical instance of INV-1's narrowing-only rule
(widening → `refine.widen`). This vocabulary doesn't use it because the three-axis
split removed the shared-status-with-refinement case; it remains for repos whose
vocabularies genuinely share a domain that needs per-type narrowing. A no-op
`refine` is flagged by the vacuity battery (§6).
 
**`tracks`** (doc subject relation, for relative staleness, §7):
`tracks subjects="<edge>"` (subject = that edge's targets) or
`tracks scope="enclosing"` (subject = the doc's enclosing anchor subtree); a node
may add path globs in its own frontmatter. `process-record` tracks nothing.
 
**Derived attribute (the one fixpoint).**
```kdl
attribute "effective_readiness" lattice="Readiness" {
    init "readiness"
    meet over="depends_on"     // glb of self and deps' effective_readiness
}
```
Doc edges (`documents`, `augments`) are referential only and do **not** propagate
readiness — a stale README must not drag down the project it documents.
 
**Intrinsic constraint** (travels with the types; reads readiness, never
lifecycle or activity):
```kdl
constraint "deep-ready" {
    require "effective_readiness" is="stable" when="readiness" is="stable"
}
```
 
---
 
## 4. Table-setting (`profile.kdl`)
 
Per-repo, points at the vocabulary, never edits it. Plumbing: `id`, `use`,
`scan`, `frontmatter fence=…`, `vouch allow=… require-expiry=…`.
 
**`shape`** is a cardinality claim over the type-index, scoped to an **anchor** —
"does this anchor have the docs it should." Orthogonal to conformance: a repo can
be `warranted` (present nodes conform) yet `incomplete` (a required doc missing).
Not a fixpoint, not transitive; revalidates on identity changes only.
```kdl
shape "project" scope="subtree" {
    require "readme"         count=1
    require "design-doc"     count=1
    require "skill"          min=1
    require "process-record" min=0    // permitted, stated explicitly
}
```
**Scope** (the membership relation, the one new decision the anchor model forces):
- `subtree` (default) — anchor owns its directory subtree; membership by
  longest-prefix path match. Zero annotation; matches monorepo layout; works on a
  stranger's repo untouched. The filesystem coupling is opt-in and never reaches
  the id-based core.
- `repo` — one implicit anchor over the whole repo (the cheap single-project
  aggregate). Whole-repo is the anchor model with one implicit anchor.
- `edge via="<edge>"` — anchor owns nodes that link to it; for graph-native repos.
(If multiple repos want the same table-setting, the `shape` blocks lift into a
vendorable preset; INV-1 makes preset + local overrides compose order-independently.
Not split yet.)
 
---
 
## 5. Vouch — author grounds
 
A per-node frontmatter assertion: the author backs this node on their own credit;
the checker records the assertion without verifying. (This is the named concept
behind the retired `slag`/`EXEMPT` placeholder — closing a goal without proof.)
```kdl
vouch "reason required — why this is trusted unverified" { expires epoch=80 }
```
- Excludes the node from local + edge + constraint checks.
- **Reason required** (`vouch.no-reason` otherwise). Loud by construction.
- **Counted** in the warrant (`counts.vouched`, itemized in `vouchers[]`). Never silent.
- **Optional `expires`** — past it the vouch itself fails (`vouch.expired`), so
  trust-by-fiat can't rot into a permanent blind spot. Replaces a silent ignore-list.
- **Requirement-scoped form** waives one `shape` requirement for one anchor —
  "this project legitimately has no process-record" — counted at `scope:"requirement"`.
  (You cannot vouch a doc *present*; presence is fact, not trust.)
---
 
## 6. Vacuity battery
 
Load-time self-audit of the elaborated vocabulary (warning tier). A vacuous
construct is legal but inert; flagging it says the schema isn't doing what you
think. Finite domains make satisfiability **decidable**, so each is computed
exactly — the exact (not sampled) analog of mutation testing.
 
| code | fires when |
|---|---|
| `vacuous.refine` | a `refine` subset equals the inherited domain |
| `vacuous.guard` | a constraint `when` is unsatisfiable over declared domains |
| `vacuous.projection` | a readiness map sends every stage to one rank |
| `vacuous.constraint` | a `require` already entailed by structure |
| `vacuous.domain` | a `domain`/`lattice` with a single member |
| `unreachable.variant` | a domain variant no refinement permits anywhere |
| `vacuous.shape` | a `require min=0` with no `max` |
 
---
 
## 7. Runtime — triggers & staleness
 
**Two change signals per node.** `frontmatter-hash` drives **conformance**
rechecks (body-agnostic — prose edits never reconform). `last_change` = commit
time of the subject's whole file / covered path drives **review** (body included
— a code edit is exactly what should flag a doc whose frontmatter still validates).
 
**Staleness, two orthogonal axes.**
- **Absolute currency:** `now − reviewed > fresh-for`. Clock-triggered. Floors
  readiness to `unavailable`.
- **Relative staleness:** `subject.last_change > reviewed`. Event-triggered. A
  **review advisory**, not a failure — a moved subject makes a doc *suspect*, not
  *wrong* (INV-5). Does not touch `ok` or readiness unless opted in per type.
**Trigger taxonomy.**
| trigger | detect | recheck set | cost |
|---|---|---|---|
| content | frontmatter-hash moved | the doc: local + outgoing edges | O(file) |
| identity | id/type/create/delete | doc + `rdeps[old] ∪ rdeps[new]` | O(fan-in) |
| schema | types/profile hash moved | re-elaborate (or schema-invalid) | O(repo) |
| clock expiry | wheel fires at `reviewed+fresh-for` | that doc's currency | O(doc) |
| subject change | `last_change` advanced on a tracked path | docs tracking it (already its `rdeps`) | O(fan-in) |
 
Subject-change reuses the existing `rdeps` index — the referential-only doc edges
already place a doc in its subject's reverse-deps, so the only addition is the
`last_change > reviewed` comparison plus the broader (whole-file) watch.
 
**Timing wheel.** The one clock-driven trigger. One entry per doc with
`fresh-for`, scheduled at `reviewed + fresh-for`; firing rechecks currency.
Relative staleness does not use the wheel (it's event-driven).
 
**Incremental discipline.** `failing` is a set; each recheck replaces a file's
diagnostics wholesale and `ok = failing.empty` — never a maintained boolean.
Coalesce a burst (debounce, or treat a git `post-commit`/`post-merge` as one
transaction) before rechecking, so a checkout doesn't storm. Stamp every envelope
with `epoch`.
 
**Persistence (INV-3).** Content-addressable cache (immutable entries,
write-temp-fsync-rename) + one mutable manifest (atomic rename). Persist: parsed
AST keyed by frontmatter-hash; `local_diags` keyed by (frontmatter-hash,
flat-type-hash); `epoch`. Recompute every boot: id index, `rdeps`, edge
diagnostics, `effective_readiness`. Never persist derived attributes or indices.
 
**Cold start.** Full pass; the cache turns it into hash-everything-parse-the-
misses. Additions: seed each tracked subject's `last_change` (batched `git log`),
populate the wheel from `reviewed + fresh-for`. Both derive from files (INV-3).
 
**`init` vs `track`.** `init` makes a repo self-describing (scaffolds schema +
profile marker) and is daemon-agnostic — one-shot CLI validation works with no
daemon. `track <path>` registers the path on the daemon's watch-list (the only
authoritative daemon state), runs cold start, begins watching. `id` is logical
and repo-local, so a clone on a second machine carries the same identity;
disambiguate contending daemons by `host` in the topic.
 
---
 
## 8. Publish path
 
Topic tree under `checker/<repo-id>/`:
| topic | retain | payload |
|---|---|---|
| `warrant` | retained | full certificate + `epoch` — the trust gate |
| `shape/<anchor>` | retained | per-anchor completeness |
| `diagnostics/<file>` | transient | per-file conformance deltas |
| `review` | transient | relative-staleness advisories `{doc, subject, subject_commit, reviewed}` |
 
Retained = anything a late subscriber must know; transient = events. **LWT**
publishes `level:"unwarranted", daemon:"down"` to `…/warrant`, so no agent trusts
a stale retained warrant from a dead daemon. `epoch` dedups; persisted across
restart. MQTT is optional; the library/CLI/MCP path is authoritative.
 
---
 
## 9. The warrant (output)
 
`ok` = do present nodes conform; `complete` = are required nodes present
(orthogonal fold over `shapes[]`); `level` + `schema_vacuity` = how much to trust
the warrant itself. The second axis is the point — a skeptical reader sees the
verdict and the strength behind it.
```json
{
  "repo": "gear-inventory", "ref": "git:9f3a1c2", "epoch": 42,
  "types_hash": "sha256:…", "profile_hash": "sha256:…",
  "level": "warranted-with-vouchers",   // unwarranted | warranted | warranted-with-vouchers
  "schema_vacuity": "clean",
  "complete": false,
  "counts": { "nodes": 137, "edges_resolved": 204, "constraints_fired": 3,
              "vouched": 4, "reviews_pending": 2 },
  "ok": false,
  "diagnostics": [ /* severity, code, file, path, got/want, message */ ],
  "vouchers": [ /* {file|anchor, scope, reason, expires} */ ],
  "shapes": [ { "anchor": "project:tritavox", "warranted": true,
                "shape": "incomplete", "missing": [ {"require":"design-doc","count":1,"found":0} ] } ]
}
```
For a context-pull agent: `warranted` + `complete` = safe to pull as stable
context at `epoch`; `incomplete`/`unwarranted` = pull only the warranted subset
or decline. Subject to INV-5.
 
---
 
## 10. Boundaries & open threads
 
- **Accuracy is a non-goal** (INV-5). The checker never judges whether a doc is true.
- **Value-constraint extension (not built).** Two tiers if added: tier-1 a
  relational predicate over *declared* fields along an edge (e.g. `monotone`) —
  still depth-one, triggered by constrained-field change, bounded by fan-in;
  tier-2 a *derived* attribute over the graph (a fixpoint, transitive). Only
  `effective_readiness` is tier-2 today. Each new value constraint should be gated
  per construct so plain referential edges stay depth-one (INV-4).
- **Incremental retraction deliberately unbuilt.** Raising an input is monotone
  and cheap; lowering one is recomputed over the reachable affected component
  rather than maintained. True retraction (DRed-style) is not worth it at this scale.
- **Modifier-refinement** (strengthening optional→required as its own meet under a
  `require`/`pin` keyword) — deferred.
- **Preset split** of `profile.kdl` into vendorable table-setting + local
  plumbing — deferred until a second repo shares a table-setting.
---
 
## 11. Glossary
 
- **warrant** — the checker's auditable verdict for a repo at an epoch; both the
  grounds-for-belief and the certifying record.
- **vouched / vouch** — a node the author backs unverified; recorded, counted, loud.
- **activity / lifecycle / readiness** — the three axes `status` decomposed into.
- **anchor** — a node a `shape` is evaluated relative to.
- **shape** — a per-anchor cardinality requirement over doc types (completeness).
- **tracks** — a doc type's subject relation, for relative staleness.
- **currency** — absolute staleness: review age vs `fresh-for`.
- **epoch** — internal monotonic batch counter (INV-2); not a wall clock.
- **`rdeps`** — reverse-dependency index: `target_id → {(file, edge_path)}`,
  including unresolved edges keyed by the id they await.
---
 
## 12. Review checklist — where to look hardest
 
Ordered by load-bearing-ness; each maps to an invariant or a place the design is
most likely wrong.
 
1. **Meet everywhere (INV-1).** Confirm include-merge, `refine`, readiness
   projection+override+currency, `effective_readiness`, and profile merge are all
   genuinely narrowing-only and order-independent. Any operator that can widen, or
   whose result depends on declaration order, breaks the spine.
2. **Two clocks (INV-2).** Grep every time-typed field; confirm none compares
   `epoch` to a timestamp/commit-ref. This is the class of error already caught
   once (`reviewed` was mistyped `epoch`).
3. **Accuracy boundary (INV-5).** Confirm nothing in conformance, currency, or
   review claims a doc is *true* — only well-formed / current. Relative staleness
   in particular must stay an advisory, not a failure.
4. **Depth-one (INV-4).** Confirm no referential edge propagates, and that
   `effective_readiness` is the *only* fixpoint. Doc edges especially must not
   propagate readiness.
5. **Projection totality.** Every type's readiness map must be total over its
   lifecycle; an unmapped stage is a load error, not a silent default.
6. **`status` can't re-accrete.** Its domain is pinned to `{active, dormant}`;
   confirm nothing reintroduces lifecycle/readiness senses onto it.
7. **Cache reconstructibility (INV-3).** Confirm nothing persisted is load-bearing
   for a verdict — deleting the cache must change only speed.
8. **Vacuity coverage.** Are there inert constructs the §6 battery would miss?
9. **Shape scope coupling.** Confirm `subtree` scope's filesystem dependence is
   opt-in and never touches the id-based core.
10. **The currency time-trigger.** The only non-file trigger; confirm the wheel's
    cost is bounded and that a doc going stale by the clock alone is handled.
---
 
## 13. Decision log
 
Each row is a choice the design commits to, the reasoning, and the alternative it
beat. For review: challenge the decision against its rejected alternative, not
just the design's internal consistency.
 
- **DD-1 — Files-first, not DB-primary.** Markdown + KDL frontmatter is the source
  of truth (INV-3); indices and the warrant are derived. *Rejected:* a database as
  primary store — forfeits grep-ability, clean diffs, and harness-agnosticism, the
  three properties that make the corpus *manageable*. (Convergent with the
  agent-memory tooling surveyed — most keep markdown authoritative and derive a
  retrieval index.)
- **DD-2 — Standalone checker, not embedded in the notes app.** The checker is a
  separate binary/library the notes app depends on, never the reverse. *Rejected:*
  baking validation into one consumer — forfeits reuse across repos and the
  "drop-in pointed at a range of repos" goal. The field's prior art validates the
  *store*; almost none of it does schema enforcement, so this is the missing layer
  on top, not a competitor.
- **DD-3 — Three axes, not one `status`.** activity / lifecycle / readiness.
  *Rejected:* a single ordered `status` — every ordering mis-handles `retired`
  (lifecycle-maximal, readiness-minimal). The original shared-vs-distinct tension
  dissolved on splitting: readiness wanted sharing, lifecycle wanted per-type
  distinctness, activity wanted to be exogenous.
- **DD-4 — Subset-only refinement, enforced.** `refine` is a meet; widening is a
  load error. *Rejected:* allowing per-type widening — breaks the subtype relation
  and lets a subtype admit values its base rejects. The enforced direction also
  *instruments* the modeling decision: a widening attempt is empirical evidence the
  field isn't actually shared.
- **DD-5 — Warrant frame, not `slag`/`proof`.** "Warrant" carries both *epistemic
  grounds* and *certifying document* in one term; checker-derived = warranted,
  author-asserted = vouched. *Rejected:* `slag` (imported from Thermite's
  metallurgy, no conceptual fit) and `proof`/`sworn`/`attest` — `proof` overclaims
  (there is no solver), and `attest` root-collides with the checker's own output.
- **DD-6 — Shared types + per-repo profile.** Vocabulary is vendorable; anchors,
  scope, and cardinalities are local. Cross-repo composition is sound because
  profile merge is the same meet (INV-1). *Rejected:* a monolithic per-repo schema
  — no cross-repo vocabulary reuse, which the generalizable-tool goal requires.
- **DD-7 — Anchor model, subtree default.** Generalizes whole-repo (one implicit
  anchor) and covers monorepos and foreign repos with zero annotation via
  longest-prefix membership. *Rejected:* whole-repo-only (no multi-project) and
  edge-only (needs membership annotation a stranger's repo won't have). Subtree's
  filesystem coupling is opt-in and never reaches the id-based core.
- **DD-8 — MQTT as eventing, not RPC.** The library/CLI and MCP request paths are
  authoritative and synchronous; MQTT is an optional fan-out skin with retained
  status + LWT, never on the correctness path. *Rejected:* MQTT as primary RPC —
  reimplements request/response on a pub/sub bus and makes a local correctness
  check depend on an always-on broker.
- **DD-9 — CAS + manifest, not SQLite.** Content-addressed immutable entries plus
  one atomically-renamed manifest: transparent, grep-able, crash-safe by
  construction. *Rejected:* SQLite as the cache — a binary dependency and an opaque
  store; the persist-state-is-derived invariant (INV-3) means the cache needs no
  transactional guarantees a rename can't give. (SQLite-registry + CAS-docs hybrid
  noted as viable, not chosen.)
- **DD-10 — Decidable checks only, no solver.** Finite lattices, meet, totality;
  vacuity computed exactly. *Rejected:* SMT / bounded-model-checking machinery —
  unnecessary for structural conformance (an undecidable-problem tool aimed at a
  decidable one), and it would forfeit the speed and transparency that justify the
  whole files-first design. The dividing line from program verifiers is exactly
  decidability.
