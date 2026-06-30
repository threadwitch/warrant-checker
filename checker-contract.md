# Checker output & policy contract

Companion to `types.kdl` (shared vocabulary) and `profile.kdl` (per-repo
table-setting). This file is checker **behavior** (vacuity), **output** (the
warrant), and **policy** (vouches). None of it is per-type schema.

The frame is **warrant**: every node makes claims, and the checker decides *on
what grounds each is believed* — derived by the checker (**warranted**) or
asserted by the author and recorded unverified (**vouched**). The certificate is
that record.

---

## 1. The warrant — verdict made auditable

`ok` answers *do present nodes conform?*; `complete` answers *are the nodes that
should be present, present?* — orthogonal. `level` + `schema_vacuity` answer
*how much should you trust the warrant itself?*

```json
{
  "schema_version": 1,
  "repo": "gear-inventory",
  "ref": "git:9f3a1c2",          // content/commit checked — the correlation tag
  "epoch": 42,                   // monotonic batch counter — subscriber dedup
  "types_hash": "sha256:…",      // shared vocabulary version
  "profile_hash": "sha256:…",    // table-setting version
  "level": "warranted-with-vouchers",
  "schema_vacuity": "clean",
  "complete": false,             // shape rollup — ORTHOGONAL to level/ok
  "counts": {
    "nodes": 137,
    "edges_resolved": 204,
    "constraints_fired": 3,
    "vouched": 4,                // author-asserted, excluded from checks — never hidden
    "reviews_pending": 2         // docs whose tracked subject moved since review (see daemon.md)
  },
  "ok": false,
  "diagnostics": [ /* severity, code, file, path, got/want, message */ ],
  "vouchers": [
    { "file": "drafts/new-rig.md", "scope": "node",
      "reason": "spec not finalized", "expires_epoch": 80 },
    { "anchor": "project:checker", "scope": "requirement", "require": "process-record",
      "reason": "no process to record yet", "expires_epoch": 90 }
  ],
  "shapes": [
    { "anchor": "project:tritavox", "warranted": true, "shape": "incomplete",
      "missing": [ { "require": "design-doc", "count": 1, "found": 0 } ] },
    { "anchor": "project:checker", "warranted": true, "shape": "complete" }
  ]
}
```

**`level` enum** (warrant grounds; the honest-degradation discipline, no solver):
- `unwarranted`             — elaboration failed; no document verdict possible (phase-one diagnostics only).
- `warranted`              — vocabulary elaborated, every node checked, nothing vouched.
- `warranted-with-vouchers`— checked, but `counts.vouched > 0`; the exception surface is in `vouchers[]`.

(`warranted-bounded` is reserved for if the value-constraint extension lands with bounded closure.)

**`complete`** — fold over `shapes[]`: `complete` iff every anchor is complete.

---

## 2. Vacuity battery — load-time, warning tier

Runs over the elaborated vocabulary before any document is checked. A vacuous
construct is legal but inert; flagging it says the schema isn't doing what you
think. Finite domains make satisfiability **decidable**, so each is computed
exactly — the analog of Thermite's mutation-kill battery, without sampling.

| code | fires when |
|---|---|
| `vacuous.refine`      | a `refine` subset equals the inherited domain (narrows nothing) |
| `vacuous.guard`       | a constraint `when` is unsatisfiable over declared domains (never fires) |
| `vacuous.projection`  | a `readiness from=lifecycle` map sends every stage to one rank |
| `vacuous.constraint`  | a `require` already entailed by structure (cannot fail) |
| `vacuous.domain`      | a `domain`/`lattice` with a single member |
| `unreachable.variant` | a domain variant no refinement permits anywhere |
| `vacuous.shape`       | a `require min=0` with no `max` (permits anything — documents intent, warns it constrains nothing) |

`schema_vacuity` = `clean` | `warnings:N`.

---

## 3. Vouch — author grounds, loud and counted

A per-node frontmatter assertion: *the author backs this on their own credit;
the checker takes it on trust without verifying.* Recognized regardless of
schema. (Was the `EXEMPT`/`slag` placeholder — now named under the warrant frame.)

```kdl
// in a note's frontmatter
vouch "reason required — why this is trusted unverified" {
    expires epoch=80      // optional: past this epoch the vouch itself errors
}
```

- Excludes the node from local + edge + constraint checks.
- **Reason required** — a bare vouch is an error (`vouch.no-reason`). Loud by construction.
- **Counted** in `counts.vouched`, itemized in `vouchers[]`. Never silent.
- **Optional `expires`** — after that epoch the vouch fails (`vouch.expired`), so
  trusted-by-fiat can't rot into a permanent blind spot. Replaces the old silent `ignore` glob.

**Shape-scoped vouch.** You can't vouch a doc *present* (presence is fact, not
trust), but you can vouch a *requirement waived* for one anchor — declared in the
anchor node's frontmatter, counted and itemized at `scope:"requirement"`:

```kdl
vouch require="process-record" "no process to record yet" { expires epoch=90 }
```

---

## 4. What a warrant means for a context document

The second purpose — stable context an agent pulls as ground truth, distinct
from agent memory. The warrant is the agent's **trust gate on the read path**:

- `warranted` + `complete` → safe to pull as stable context at `epoch`.
- `incomplete` / `unwarranted` → table-setting isn't ready; pull only the
  warranted subset, or decline.

**Boundary, state it wherever an agent consumes a doc:** a warrant certifies
*form, resolving references, and currency* — never *truth*. Accuracy is semantic
and outside the checker. `warranted` means well-formed and current-signalled, not
correct. Don't let a consumer overtrust it.

**Currency is the one new trigger.** `currency` makes a doc's readiness
time-dependent: a doc can go stale by the passage of time with no file edit —
the first revalidation trigger that isn't a filesystem event. The daemon
schedules a recheck at each doc's expiry epoch (a timing wheel keyed by expiry);
bounded and cheap, but it needs a clock, not just a watcher. Wire it with the
daemon publish path.
