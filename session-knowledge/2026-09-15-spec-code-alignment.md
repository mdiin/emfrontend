# 2026-09-15 — spec/code alignment session

Repository: `emfrontend`. Baseline HEAD at the time of writing: `1e5e0e3`
(committer date `2026-09-15`, from `git log -1 --format=%cs`).

Work covered by this session: weed `em-frontend.allium` (the spec) against the
implementation, walk each divergence to a decision, apply fixes, weed again,
then triage what remained and deliberately leave it in place.

Everything below is traceable either to this repository (re-run the commands
shown) or to the session brief that commissioned this file. Where a claim in
that brief did **not** reproduce when re-checked, it is flagged inline under
*Brief vs repository* rather than repeated as fact.

---

## a. What this session did

1. Weeded the spec (`em-frontend.allium`) against the implementation.
2. Walked six divergences to decisions and applied six fixes plus a
   stale-comment fix (see the commit log in section g).
3. Weeded again, which produced a further round of spec fixes.
4. Triaged six pre-existing gaps and two model fields, and kept them
   deliberately (section e).

The resulting commit series is recorded verbatim in section g, which is the
authoritative account of what actually changed.

---

## b. Before you run a check

- The spec is `em-frontend.allium` at the repository root. Its first line is
  `-- allium: 3` (language version 3).
- The CLI is `/home/mvi/.local/bin/allium`, version **3.6.0**:

  ```
  $ /home/mvi/.local/bin/allium --version
  allium 3.6.0 (language versions: 1, 2, 3)
  ```

- Run it as:

  ```
  /home/mvi/.local/bin/allium check em-frontend.allium
  ```

### Expected baseline

Measured by running the command at `1e5e0e3`. The exit code is **1** and the
output is exactly these five diagnostics — **3 warnings and 2 infos**:

| # | Severity | Code | Location | Message |
|---|----------|------|----------|---------|
| 1 | warning | `allium.externalEntity.missingSourceHint` | line 37, col 17 | External entity `Delta` has no obvious governing specification import in this module. |
| 2 | warning | `allium.entity.unused` | line 40, col 17 | Entity `SnapshotSeed` is declared but not referenced elsewhere in this specification. |
| 3 | warning | `allium.externalEntity.missingSourceHint` | line 40, col 17 | External entity `SnapshotSeed` has no obvious governing specification import in this module. |
| 4 | info | `allium.field.unused` | line 139, col 3 | Field `EventModel.elements` is declared but not referenced elsewhere. |
| 5 | info | `allium.field.unused` | line 268, col 3 | Field `SpecStep.spec` is declared but not referenced elsewhere. |

Raw output recorded verbatim:

```json
{
  "command": "check",
  "diagnostics": [
    {
      "code": "allium.externalEntity.missingSourceHint",
      "location": {
        "col": 17,
        "file": "em-frontend.allium",
        "line": 37
      },
      "message": "External entity 'Delta' has no obvious governing specification import in this module.",
      "severity": "warning"
    },
    {
      "code": "allium.entity.unused",
      "location": {
        "col": 17,
        "file": "em-frontend.allium",
        "line": 40
      },
      "message": "Entity 'SnapshotSeed' is declared but not referenced elsewhere in this specification.",
      "severity": "warning"
    },
    {
      "code": "allium.externalEntity.missingSourceHint",
      "location": {
        "col": 17,
        "file": "em-frontend.allium",
        "line": 40
      },
      "message": "External entity 'SnapshotSeed' has no obvious governing specification import in this module.",
      "severity": "warning"
    },
    {
      "code": "allium.field.unused",
      "location": {
        "col": 3,
        "file": "em-frontend.allium",
        "line": 139
      },
      "message": "Field 'EventModel.elements' is declared but not referenced elsewhere.",
      "severity": "info"
    },
    {
      "code": "allium.field.unused",
      "location": {
        "col": 3,
        "file": "em-frontend.allium",
        "line": 268
      },
      "message": "Field 'SpecStep.spec' is declared but not referenced elsewhere.",
      "severity": "info"
    }
  ],
  "findings": [],
  "spec_file": "em-frontend.allium"
}
```

Exit code: `1`.

> **2026-09-15 — baseline after the A4 spec edit.** The spec has since moved
> (A4: `Element.is_information_complete` re-declared as wire-supplied; see
> `2026-09-15-spec-code-weed.md`). Re-running
> `/home/mvi/.local/bin/allium check em-frontend.allium` still gives the **same
> five diagnostics, same codes, same order, same severities**, exit code still
> `1`. The only change is location: the `allium.field.unused` for `SpecStep.spec`
> moved from line 268 to **line 258** (the edit removed a net 10 lines above it;
> the file went from 1171 to 1161 lines). The other four (lines 37, 40, 40, 139)
> are unchanged. A fresh session should expect **258, not 268**, and should
> still gate on the diagnostic list rather than the exit status.

### Any new diagnostic is a regression

Compare the **diagnostic list**, not just the exit code (see the exit-code
correction below — infos alone do not change the exit code, so the exit code
alone is not a sufficient check).

### The three warnings cannot be resolved from this repository

`Delta` (line 37) and `SnapshotSeed` (line 40) are declared `external entity`,
and `SnapshotSeed` is additionally unreferenced. `allium`'s language reference
explains the warning: an external entity is governed by another spec or system,
so the checker reminds you of that. Resolving it needs a governing-specification
import in this module.

That governing spec is **not** in this repository. The change stream is
emcli's, and its specification lives in the sibling repository `thecli`, e.g.
`../thecli/event-model.allium`, whose `contract ChangeStream` declares
`snapshot: (model: EventModel) -> Any` and `delta: (change: Any) -> Any`. Note
that that spec types the stream payloads as `Any` and does **not** declare
entities named `Delta` or `SnapshotSeed`, so even a cross-repository import
would not obviously satisfy the hint for these two names. Both are expected
baseline noise, not gaps to fix here.

> *Brief vs repository.* The session brief stated that
> `../thecli/docs/change-stream.md` is absent. It is **present**: 16,227 bytes,
> 285 lines, tracked in `thecli` as `docs/change-stream.md`
> (`git -C ../thecli ls-files --error-unmatch docs/change-stream.md` succeeds).
> `../thecli/event-model.allium` is present too. The substance of the point
> survives — the governing material is outside this repository, so the warnings
> cannot be cleared from here — but "absent" is wrong and is recorded here as
> wrong. The document in question is also prose (a consumer guide for the SSE
> wire format), not an Allium module.

### Both infos are explained artefacts, not gaps

Neither is a real modelling hole; both are consequences of the read-only
unused-field analysis described in section c. Left in place deliberately.

- **`EventModel.elements`** — `EventModel` has four sibling relationship
  declarations (lines 137–140): `timelines`, `swimlanes`, `elements`,
  `connections`. Three of them are read by an expression somewhere in the spec
  and one is not:
  - `timelines` — read by `EmptyModelRendered` / `ModelRendered`
    (`model.timelines.count = 0` / `> 0`, lines 396 and 405);
  - `swimlanes` — read by the Canvas band clause
    (`timeline.model.swimlanes`, line 1135);
  - `connections` — read by `FieldFlowHighlights` (`model.connections`,
    line 777);
  - `elements` — read by nothing, hence the info.

  `elements` being reachable in practice (the model plainly has elements) does
  not matter: only expressions that read a field count.

- **`SpecStep.spec`** — flagged, while its owner `Specification`'s
  `steps: SpecStep with spec = this` (line 262) is **not** flagged, because
  `spec.steps` is read at line 962 (`for step in spec.steps:`). `SpecStep.spec`
  is only ever named as the back-reference in the `with spec = this` clause,
  which does not count as a reference.

---

## c. Checker quirks on 3.6.0 (verified with probes)

This section is the most important content in this file. All probes below were
run against `/home/mvi/.local/bin/allium` 3.6.0 with `allium check`. They are
minimal standalone `-- allium: 3` specs, run from a scratch directory outside
the repository; nothing in these probes is part of the project. Each probe is
reproducible by pasting it into a scratch `.allium` file.

### 1. Unused-field analysis is read-only — a back-reference clause does not count

A field is reported `allium.field.unused` unless some **expression** actually
reads it. Naming the field as the back-reference in another entity's
`with <field> = this` clause does **not** count as a reference.

Two otherwise identical probes, differing only in whether the rule expression
reads the back-reference field:

```allium
-- read.allium
-- allium: 3
entity Parent {
  name: String
  children: Child with parent = this
}

entity Child {
  parent: Parent
  label: String
}

rule ReadBackref {
  when: ChildTouched(child)
  requires: child.parent.children.count > 0
  ensures: BackrefSeen(parent: child.parent)
}
```

```allium
-- unread.allium  (identical except the one expression)
-- allium: 3
entity Parent {
  name: String
  children: Child with parent = this
}

entity Child {
  parent: Parent
  label: String
}

rule ReadBackref {
  when: ChildTouched(child)
  requires: child.parent.name.count > 0
  ensures: BackrefSeen(parent: child.parent)
}
```

Observed:

```
read.allium   : Field 'Parent.name'      unused (line 3)
                Field 'Child.label'      unused (line 9)
                -- Parent.children is NOT reported: child.parent.children reads it
unread.allium : Field 'Parent.children'  unused (line 4)
                Field 'Child.label'      unused (line 9)
                -- reading child.parent.name instead leaves the back-reference unread
```

`Parent.children` is declared identically in both, including the
`with parent = this` clause; only the expression that reads it differs. Hence:

- the diagnostic is present in one probe and absent in the other;
- the `with parent = this` back-reference clause alone does not clear it.

This single rule explains both baseline infos:

- `SpecStep.spec` is flagged although `Specification.steps` (its back-reference
  owner) is not, because the surface reads `spec.steps`;
- exactly one of `EventModel`'s four sibling relationships is flagged —
  `elements`, read by nothing, while `timelines`, `swimlanes` and `connections`
  are each read by an expression.

### 2. Unused-field analysis is name-based, not per-owner

A field named `kind` is masked whenever any **other** field named `kind` is
referenced anywhere in the spec. `Slice.kind` therefore reads as used while
being consumerless: `Element.kind` masks it (both are named `kind`).

Three probes isolating the effect. In all three, `Alpha.name` is read so it
stays quiet, and `Alpha.kind` is never read:

```allium
-- k1.allium
-- allium: 3
entity Alpha {
  name: String
  kind: String
}

rule UsesAlphaName {
  when: AlphaTouched(alpha)
  requires: alpha.name.count > 0
  ensures: AlphaSeen(alpha: alpha)
}
```

```allium
-- k2.allium  (adds a *different* entity whose field is also called `kind`, and reads it)
-- allium: 3
entity Alpha {
  name: String
  kind: String
}

entity Beta {
  kind: String
}

rule UsesBetaKind {
  when: BetaTouched(beta)
  requires: beta.kind.count > 0
  ensures: BetaSeen(beta: beta)
}

rule UsesAlphaName {
  when: AlphaTouched(alpha)
  requires: alpha.name.count > 0
  ensures: AlphaSeen(alpha: alpha)
}
```

```allium
-- k3.allium  (k2 with Beta's field renamed to `sort_order`, read updated)
-- allium: 3
entity Alpha {
  name: String
  kind: String
}

entity Beta {
  sort_order: String
}

rule UsesBetaKind {
  when: BetaTouched(beta)
  requires: beta.sort_order.count > 0
  ensures: BetaSeen(beta: beta)
}

rule UsesAlphaName {
  when: AlphaTouched(alpha)
  requires: alpha.name.count > 0
  ensures: AlphaSeen(alpha: alpha)
}
```

Observed:

| Probe | `allium.field.unused` for `Alpha.kind` |
|-------|----------------------------------------|
| `k1.allium` — no other `kind` anywhere | **reported** (line 4) |
| `k2.allium` — another entity's `kind` is read | **silent** (masked) |
| `k3.allium` — that field renamed to `sort_order` | **reported** again (line 4) |

So: an unreferenced field named `kind` reports; a referenced sibling elsewhere
named `kind` silences it; renaming that sibling makes it report again.

### 3. `open question` produces no diagnostic (contrary to the language reference)

A probe containing `open question "Is this thing real?"` produced **no**
open-question diagnostic on 3.6.0. The only diagnostics were incidental
(`allium.entity.unused`, `allium.rule.unreachableTrigger`). The language
reference claims open questions "are surfaced by the specification checker as
warnings, indicating the spec is incomplete" — on 3.6.0 they are instead silent
prose markers. Do not rely on `allium check` to surface them.

### 4. `deferred` requires a `-- see:` location hint, and asserts Allium logic

`deferred Thing.work` with no hint reports
`allium.deferred.missingLocationHint`: *"Deferred specification 'Thing.work'
should include a location hint."* The language reference is explicit that
deferred declarations "represent Allium logic that is fully specified
elsewhere" — i.e. the detail exists as another Allium spec. It is therefore
**inapplicable** to behaviour that is implemented only in Dart/ClojureDart:
there is no Allium spec to point at, so `deferred` is the wrong tool for a
code-only implementation detail.

### 5. `List<A | B>` parses but is semantically inert

A field declared `things: List<A | B>` produces **no** diagnostic from
`allium check`, but the language has no union type. `allium parse` shows what
the checker actually built:

```
GenericType(List, args=[ Pipe(left=Ident(A), right=Ident(B)) ])
```

The `|` is parsed as a `Pipe` node in the type-argument position — the same
construct the language uses for an inline enum (`status: pending | confirmed`)
or a variant discriminator (`kind: Branch | Leaf`). The compound collection
types are `Set<T>`, `List<T>` and `Sequence<T>`, plus `T?` for optionality;
there is no union. So the pipe here is accepted and then ignored: it does not
express "a list of A or B", and nothing in `check`, `analyse` or `model` will
tell you so.

### 6. Recursion: expressible in value types, not in derived values

The value-type precedent is in this very spec: `value Field { … subfields:
List<Field> }` (line 52) — a value type referring to itself, used for nested
field trees. Any traversal over that nesting is therefore written as a
free-standing black-box call, which is the convention the spec already follows
(`connections_carrying_into`, `trace_path_elements`, `wf_field_overlay_names`,
each documented in the spec as `-- black box:`; lines 749, 766, 802).

> *Brief vs repository — the derived-value half did not reproduce.* The brief
> states recursion is "not expressible in derived values". A probe of a
> self-referential derived value—
> `depth: 1 + (parent?.depth ?? 0)` on an entity with `parent: Node`—was
> **accepted silently** by `allium check`, by `allium analyse` and by
> `allium model` (which lists `depth` under `derived_values` with no
> complaint). I could not reproduce a rejection, so I am not asserting this as
> a check-time behaviour. What I can state from this repository is the
> *convention*: the spec writes its traversals as black-box free-standing
> calls, and that is the only traversal form this spec uses. Treat "recursion
> is not expressible in derived values" as a semantic/execution limitation
> reported by the session, unverified by me, rather than as a diagnostic you
> will see.

### Note: exit codes — the brief is wrong here too

`allium help check` documents: exit `0` = "No errors or warnings", exit `1` =
"One or more errors or warnings were reported", exit `2` = no inputs.

Verified by probe:

- a spec whose only diagnostics are `info` severity (3 infos, 0 warnings,
  0 errors) exits **0**;
- a spec with a single warning exits **1**.

So `allium check` does **not** exit 1 for *any* diagnostic; it exits 1 when any
**error or warning** exists. The recorded baseline does exit 1 — because it
contains three warnings. This matters in practice: a newly introduced
info-only diagnostic (e.g. a new `allium.field.unused`) will **not** change the
exit code, so gate on the diagnostic list rather than the exit status.

---

## d. Conventions settled this session

- **No colour values in the spec, and no colour in effect names.** Effect names
  must not encode a colour: the incomplete-element effect is named
  `IncompleteBorderRendered` (line 716). No palette value or hex literal
  appears anywhere in the spec (checked: no `#rrggbb` / `0xrrggbb` in
  `em-frontend.allium`); the fact that the marker is red is stated in prose
  (lines 186, 711) while the actual colour constant lives in the renderer
  (`m/Colors.red`, `src/em_frontend/canvas.cljd` line 295).
- **The incomplete-element marker is red.** The spec and the painter agree:
  `IncompleteElementMarked` renders "a persistent red border (not only on
  hover)", and `canvas.cljd` draws a red outer ring when
  `is_information_complete = false`.
- **Renderer geometry is deliberately out of scope** — per-tag heights, card
  sizing and content extent are not modelled. Where a bound is real but
  renderer-specific, the spec states the *behaviour* in prose and the number
  stays a named renderer constant. The worked example is the card field-line
  cap: the spec describes the cap, and `max-card-field-lines` (4) stays in
  `src/em_frontend/canvas.cljd` line 279.
- **`SelectFieldForTrace` traces transitively *into* the selected element and
  stops at a field introduction.** The sourcing predicate is defined **once**,
  in `model.cljd`'s `supplying-connections`, and shared with
  `is_information_complete` and `unsourced-fields` so the trace, completeness
  and the unsourced list cannot disagree about what counts as sourced
  (`src/em_frontend/model.cljd` lines 28 ff.; the docstring names this as "the
  ONE definition of that sourcing predicate").
- **Delivery obligations go in a contract `@invariant`; the consumer's
  counterpart goes in a surface `@guarantee`.** E.g. `contract ChangeStream`
  carries `@invariant SnapshotFirst` / `DeltaOrder` / `NoReplay` (lines 88–99)
  while `surface StreamIngest` carries `@guarantee ValidDeltasAppliedInOrder`
  (line 1077).
- **`SSEConnection` has `error -> connecting`** because a viewer retry is the
  way out of a failed stream: `reconnecting -> error` is terminal until the
  viewer retries, and the retry begins a fresh attempt (lines in the
  `transitions state` block of `entity SSEConnection`).
- **Conditioned surfaces use `context X: T where <expr>`.** E.g.
  `context element: Element where is_information_complete = false` (line 980),
  `context viz: Visualization where highlighted_path.count > 0` (line 1008).
- **A singleton entity is referenced in a rule with `let x = Entity`.**
  `ViewportSnapsToContent` does this with `let model = EventModel` (line 691).

---

## e. Gaps left in place, deliberately

1. **`SpecStep.spec` and `EventModel.elements` kept**, despite their
   `allium.field.unused` infos. Both are explained artefacts of the read-only,
   name-based analysis (section c.1), not modelling holes (section b).
2. **The spec's `model` back-references (`Element.model` and friends) are not
   backed by code.** The store holds a single `:event-model`
   (`src/em_frontend/store.cljd` line 20, `:event-model (:model snapshot)` at
   line 59) and no per-element model id, so nothing in the implementation can
   contradict — or confirm — an element's owning model. Consequently the
   invariants `PlacementWithinModel`, `ConnectionEndpointsWithinModel` and
   `SwimlaneWithinModel` (spec lines 835, 841, 847; mirrored in
   `src/em_frontend/invariants.cljd`) hold **vacuously**. This is domain
   modelling rather than enforced behaviour, and is kept as such.
3. **The marking rules describe the state transition, while the code also
   paints on first draw.** `IncompleteElementMarked` is triggered by
   `is_information_complete becomes false`, whereas the painter draws the red
   ring on every draw of an incomplete element (`canvas.cljd` line 295),
   including the first. The spec models the transition; the renderer is
   idempotent and repaints. Left as is.
4. **The wireframe node tree is deliberately unmodelled.** Only the field-name
   overlay predicate is modelled (via a rule plus a black-box tree walk,
   `wf_field_overlay_names`, spec line 802); the tree itself is not.
5. **`image_url` is rendered by the element details panel, not on the card.**
   `src/em_frontend/panels/element_details.cljd` renders it as
   `m/Image.network`, so the spec does not claim a card thumbnail.
6. **The two model fields** (`Delta` and `SnapshotSeed`) were triaged rather
   than modelled further: they stay `external entity` placeholders for emcli's
   stream payloads, which is why the two `missingSourceHint` warnings remain
   (section b).

---

## f. Per-change verification workflow

Every change in this session was verified with the same three steps, in this
order:

1. `allium check em-frontend.allium` — the diagnostic set must be **unchanged**
   versus the baseline in section b (compare the list, not just the exit code).
2. `clj -M:cljd:test test` — the ClojureDart test suite.
3. `clj -M:cljd clean && clj -M:cljd compile` — run **without** `:test`.

Step 3 is not optional, and is why step 2 must be followed by a clean rebuild:
the `:test` alias adds `test` to `:paths` and leaves stray cross-imports of the
`*_test.dart` files in the shared `lib/cljd-out` production build output, after
which `flutter build` / `flutter analyze` fail with `uri_does_not_exist` errors
pointing at `test/cljd-out/...`. This gotcha is documented in `AGENTS.md`.

---

## g. Commit log for provenance

Verbatim `git log --oneline 7ccb8e6..HEAD` (run at `1e5e0e3`):

```
1e5e0e3 feat(spec): model the wireframe field overlay and leave the node tree unmodelled
724c18c fix(spec): state the canvas card field-line limit the painter applies
ab779b4 fix(spec): model the swimlane order index that band rendering uses
03a2fb0 fix(spec): the screen image is shown in the details panel, not on the card
7158a30 fix(spec): expose the error step's error name on SpecificationExpanded
7d5569e fix(spec): the incomplete-element border is red, not amber
8d5b127 chore(test): drop stale resize reference from the viewport test header
e37d15a chore: drop the unreachable ViewportResizes rule and resize action
2e17191 fix(trace): highlight only the connections carrying the field into the element
8d55a25 feat(app): retry the stream from the error overlay
29ee4e2 fix(spec): scope DeltaOrder to delivery and add the consumer ordering guarantee
31ffa94 fix(spec): drop the false claim that slice kind drives column treatment
c1714a8 fix(canvas): shade swimlane bands alternately by row position
```

`7ccb8e6` is the last commit before this session (`chore: bump version to
0.7.2`). What each commit did, oldest first:

| Commit | What it did |
|--------|-------------|
| `c1714a8` | **Canvas fix.** Swapped the swimlane band shading from a fixed shade per swimlane to alternating by rendered row position, so adjacent bands contrast regardless of assignment. |
| `31ffa94` | **Spec fix.** Dropped the false claim that a slice's `kind` drives its column treatment. |
| `29ee4e2` | **Spec fix.** Scoped `DeltaOrder` to delivery (a contract `@invariant`) and added the consumer-side ordering obligation as a surface `@guarantee` (`ValidDeltasAppliedInOrder`). |
| `8d55a25` | **App feature.** Added a retry path from the error overlay, which is the viewer's way out of a failed stream (hence `error -> connecting`). |
| `2e17191` | **Trace fix.** `SelectFieldForTrace` now highlights only the connections that actually carry the field into the selected element, via the shared `supplying-connections` predicate. |
| `e37d15a` | **Cleanup.** Removed the unreachable `ViewportResizes` rule and its resize action (the rendering viewport is framework-sized and a resize does not reflow content-driven layout). |
| `8d5b127` | **Stale-comment fix.** Dropped a stale resize reference from the viewport test header, left over from `e37d15a`. |
| `7d5569e` | **Spec fix.** Corrected the incomplete-element border colour: red, not amber. |
| `7158a30` | **Spec fix.** Exposed the error step's error name on `SpecificationExpanded`. |
| `03a2fb0` | **Spec fix.** The screen image is shown in the element details panel, not on the card. |
| `ab779b4` | **Spec fix.** Modelled the swimlane order index that band rendering actually uses. |
| `724c18c` | **Spec fix.** Stated the canvas card field-line limit the painter applies (the number stays a named renderer constant). |
| `1e5e0e3` | **Spec feature.** Modelled the wireframe field-name overlay and explicitly left the wireframe node tree unmodelled. |

---

## Follow-up pointers

- Re-run `allium check` before trusting anything in section b; the baseline is
  only valid while the spec is unchanged.
- If a diagnostic appears that is not in the baseline table, it is a
  regression — but read section c first: a `field.unused` info may be an
  artefact of the read-only, name-based analysis rather than a real gap.
- Do not "fix" the three warnings in section b from inside this repository.
