# 2026-09-15 — spec/code weed pass

Repository: `emfrontend`. HEAD at the time of writing: `23b6636`
(`chore: bump version to 0.8.0`, committer date `2026-09-15`).

**This was a read-only weed pass.** Two things must be said before anything
else, because they bound every claim below:

- **No file was modified by the pass itself.** The only command the pass ran
  against the artefacts was `allium check em-frontend.allium`. In particular
  the **ClojureDart test suite was NOT run by the pass**. So "tests pass" is
  *not* an observation of this pass: every test-related statement below is a
  statement about test *content*, read from the tree (`test/em_frontend/*.cljd`
  was read, not executed). The only verification any fix received this session
  is whatever the fix's own author recorded; see section 5.
- **State weeded.** The working tree as it stood when the pass ran — HEAD
  `23b6636` plus the **uncommitted reload change** (the `ReloadModel` /
  `ReloadRestartsConnection` work recorded in
  `2026-09-15-model-reload-button.md`). The pass ran against the working tree
  as well as the spec, since the divergences it found are mostly in the
  reload change itself.

**Verdict: DIRTY.** The spec and the code diverge. Five true divergences were
found (section 2, class A), of which **two were fixed during this same
session** (A1 and A2, both code-side, spec untouched) and three remain **open**
(A3, A4, A5). A further nine coverage gaps were recorded but not judged
divergences (section 3, class B), and six previously-known items were confirmed
as intentional (section 4, class C).

**`allium check` matched the recorded baseline exactly.** The run reproduced
the diagnostic set in `2026-09-15-spec-code-alignment.md` section b — the same
**3 warnings and 2 infos**, at the same locations: line 37, lines 40 and 40,
line 139, line 268 — with exit code `1`. **No regression.** Neither the
uncommitted reload change nor the two fixes applied in this session moved a
diagnostic, which is consistent with all three having been made code-side with
the spec untouched.

---

## 1. Reload change verified

The pass confirmed three things about the uncommitted reload change, and
recorded them here so a later session does not have to re-derive them:

1. **The store action matches its rule.** `store.cljd` `reload-model` is a
   faithful implementation of `rule ReloadRestartsConnection`
   (`em-frontend.allium:441`, comment block :433-440): the guard is
   `:connected` (the rule's `requires: connection.state = connected`), the
   target is `:reconnecting` (the rule's `ensures`), and it returns
   `:effects []` (the rule declares no effects). The `:connected` guard is
   also the single-in-flight guard, exactly as intended.
2. **The reseed genuinely preserves the view state.** The reload adds no ingest
   path: its fresh snapshot travels the existing `SnapshotReceived` /
   `ReconnectPreservesViewport` route (`em-frontend.allium:358`, :378), which
   leaves `zoom` / `pan_x` / `pan_y` and the view-local filter and trace state
   alone. So the product decision recorded in the reload note (section b of
   `2026-09-15-model-reload-button.md`) is a *code* fact, not just an intention.
3. **The app-level driver lifecycle has no vocabulary in the spec.** The
   reload's real mechanism — retire the live driver, then start a fresh one —
   is unmodelled: the spec says nothing about driver lifetimes. This is
   **precedent, not an omission**: `RetryStream`'s `start-stream!` is equally
   unmodelled. Being outside the spec's vocabulary, this lifecycle is
   **invisible to `allium check`** — the pass could not have found the A1/A2
   bugs by running `allium check` (it found them by reading the code), and a
   future session should not expect the checker to police that layer.

---

## 2. True divergences (class A)

Five. Two were **FIXED during this session** (A1, A2 — code-side, spec
untouched, `allium check` at baseline); three remain **OPEN**.

| | Subject | Status | Side that moves |
|---|---------|--------|-----------------|
| A1 | `reconnecting -> connecting` write on reload (undeclared transition) | **FIXED** | code |
| A2 | `retry-stream!` discards its driver handle → two live drivers | **FIXED** | code |
| A3 | wireframe node shape / root tag | OPEN | code (and thecli docs) |
| A4 | `Element.is_information_complete` derived-vs-stored provenance | OPEN | needs a human decision |
| A5 | `stream_test.cljd` fixtures drift from the wire examples they claim to copy | OPEN | code (test fixtures) |

### A1 (FIXED, code-side) — the reload wrote an undeclared transition

- **Spec:** the `SSEConnection` transition graph (`em-frontend.allium:321-330`)
  declares no `reconnecting -> connecting` edge.
- **Code:** the reload's `:reconnecting` (the rule's `ensures`) was immediately
  overwritten by the freshly started driver announcing `:connecting` on attempt
  0 — an undeclared `reconnecting -> connecting` write.
- **Resolution (code-side, spec untouched):** a caller-supplied `initial-state`
  is threaded through `drive-stream!` / `connect!` / `start-stream!`, so each
  caller's driver announces exactly the state its own rule ensures —
  `:connecting` for the cold start and for `retry-stream!`, `:reconnecting` for
  `reload-model!`. A reload is now `connected -> reconnecting -> connected`,
  all declared edges. `em-frontend.allium` was not touched.

### A2 (FIXED, code-side) — a lost driver handle left two live drivers

- **Spec:** the reload's single-in-flight intent (and, generally, the assumption
  that one connection is delivering).
- **Code:** `retry-stream!` called `start-stream!` and **discarded** the
  returned `{:cancel …}` handle. After a retry, the live driver (call it H2)
  had no recorded handle, so a subsequent reload cancelled the **DEAD** H1 in
  `stream-handle` and started H3 — leaving H2 and H3 both writing snapshots and
  deltas into the same atoms: **double reseed, every delta applied twice**.
- **Resolution (code-side, spec untouched):** `retry-stream!` now takes
  `stream-handle`, retires the current handle
  (`stream/disconnect!` — nil-safe and a no-op on a dead driver) and records
  its replacement. **Every** driver-start site now records its handle.

### A3 (OPEN) — wireframe node shape / root tag

- **Spec:** `WireframeFieldOverlaysTint` (`em-frontend.allium:822-836`) and the
  `wf_field_overlay_names` black box (:798-812).
- **Code:** `stream.cljd` `norm-wireframe-node` (:58-72) and `canvas.cljd`
  `draw-wf-node` (:246-247). Tests:
  `test/em_frontend/stream_test.cljd:143-152` and the snapshot fixture
  (:160-180).
- **The governing schema disagrees.** `../thecli/docs/change-stream.schema.json`
  (`$defs/WireframeNode`) and emcli
  (`../thecli/src/emcli/wireframe.clj:104-113, 289, 331-337`) both use:

  ```
  [tag, {-id}, {attrs}?, …children]
  ```

  with the root required to be `canvas` (`wireframe.clj:231-232` and
  `../thecli/doc/wireframe-dsl.md:81`). The frontend instead assumes
  `[tag, {attrs incl -id}, …children]` with a `screen` root.
- **Consequences, all three observable:**
  1. content attributes are **dropped**, so the field-name overlay can never
     fire;
  2. the attrs map is normalised into a **garbage node**;
  3. the `:canvas` root is **unrecognised**, so a real screen wireframe renders
     as a single box.
- **Also:** `../thecli/docs/change-stream.md:241`'s prose contradicts its own
  schema and emcli, so a fix touches thecli's docs as well as this repository's
  code. **The tests encode the wrong shape** — a green suite is not evidence
  here.

### A4 (OPEN, needs a human decision) — `Element.is_information_complete` provenance

- **Spec:** the field is declared **derived** (`em-frontend.allium:191-206`),
  with its helper `unsourced_fields` at :229-239.
- **Code:** the store keeps the **wire flag**
  (`stream.cljd:85`), and the derivation exists only as
  `model/unsourced-fields` (`model.cljd:103-119`), which feeds the tooltip body
  (`app.cljd:369-374`).
- **Consumers of the stored flag:** `canvas.cljd:285`, `app.cljd:369`,
  `panels/element_details.cljd:44-45`.
- **Nothing asserts the two agree.** There is no test, and no runtime check,
  tying the wire flag to the derived value.
- **Not currently observable** — emcli computes the flag and restates it
  (`../thecli/docs/change-stream.md:100,183-187`), so the two agree in
  practice. It **becomes observable for a payload that omits the flag** (see
  A5). This is the one divergence that needs a **human decision** (derived in
  code, or declared as wire-supplied in the spec) rather than a mechanical fix.

### A5 (OPEN) — the wire-drift guard no longer matches the wire

- **Claim:** `test/em_frontend/stream_test.cljd:3-6` says its fixtures are
  byte-for-byte copies of `../thecli/docs/change-stream.md`'s worked examples,
  and presents itself as the wire-drift guard.
- **Reality:** `delta-create-element` (:38-39) and `delta-set-image-url`
  (:41-42) both **omit** `"field_origins":[]` and
  `"is_information_complete":true`, which the doc's real payloads carry
  (`change-stream.md:150,157`). The doc is explicit that the flag is **always**
  present (`change-stream.md:224`).
- **One fixture is accurate:** the `DeleteTimeline` cascade fixture (:55-56)
  **is** verbatim.
- **Consequence:** the guard cannot catch a change in exactly the fields that
  matter to A4 — it is anchored to a payload shape the wire does not use.

---

## 3. Coverage gaps (class B)

Not divergences — behaviour that is correct (or at least unrebutted) but
**unverified**, or spec-visible behaviour with no test at all. Recorded so a
later session does not mistake them for known-good.

- **B1 — `ViewportSnapsToContent` (`em-frontend.allium:704-719`) is untested.**
  It is implemented (`interaction.cljd:56-83`) and its sibling Viewport actions
  *are* covered, which makes the omission easy to overlook.
- **B2 — the reload's core mechanism is untested.** `connect!`'s handle,
  `drive-stream!`'s cancellation and `disconnect!` have no test. Stated plainly
  rather than dressed up: the suite has **no async/HTTP infrastructure**, so
  the UI wiring is **not reachable from `cljd.test`** as it stands. Say so
  rather than pretending coverage.
- **B3 — `@guarantee LargeModelScale` (`em-frontend.allium:1099-1101`) is
  unmeasured.** Nothing asserts the scale bound.
- **B4 — `@guarantee AlternatingBandShading` (`em-frontend.allium:1168-1170`)
  is untested.** Implemented (`canvas.cljd:341`), but the painter is not
  unit-tested.
- **B5 — the stored completeness flag vs `model/unsourced-fields`.** No test
  asserts the two agree (the testable face of A4).
- **B6 — surfaces' opening triggers are unmodelled, and the hit-test helpers are
  untested** (`canvas.cljd:392`, :403, :429).
- **B7 — `VisualizationState.loading` is never observable in the store.**
  `store.cljd:70-75` collapses it inside one pure call, so no store state ever
  reports it.
- **B8 — the reseed's side effects are unstated.** `store.cljd:68` drops the
  queue; `app.cljd:66-72` resets the animation maps and `:failed-connection`.
  Both are **consistent with `@invariant NoReplay`**, but the spec does not say
  them.
- **B9 — minor odds and ends.**
  - `config/animation-duration-ms` (`config.cljd:12`) has no spec counterpart.
  - `store.cljd:5-6` **overstates** which effects the canvas consumes.
  - The actors (`em-frontend.allium:912-921`) have no code counterpart.
  - `contract ChangeStream.connect: (stream_url: String) -> SnapshotSeed`
    (`em-frontend.allium:84`) does not match `connect!` returning a handle — a
    **pre-existing** abstraction mismatch that the new handle **aggravates**.

---

## 4. Known/intentional (class C) — confirmed, no action

Re-checked against the current tree; all still as previously recorded. **Do not
"fix" these.**

- **e.1 — the two `field.unused` infos (:139, :268).** Baseline noise from the
  read-only, name-based unused-field analysis (see
  `2026-09-15-spec-code-alignment.md` sections b and c.1). Still present.
- **e.2 — the model back-references are unbacked.** `PlacementWithinModel`,
  `ConnectionEndpointsWithinModel` and `SwimlaneWithinModel`
  (`invariants.cljd:13-49`) hold **vacuously**, as recorded.
- **e.3 — `IncompleteElementMarked` models the transition while the painter
  repaints.** `canvas.cljd:285-295` repaints on every draw. Left as is.
- **e.4 — the wireframe node tree is deliberately unmodelled.** Only
  `wf_field_overlay_names` is modelled. **Note A3 is *not* covered by e.4**: A3
  is about the *observer* of that tree — the shape the frontend reads — which
  e.4 does not excuse. e.4 excuses the *modelling* of the tree, not reading it
  wrong.
- **e.5 — `image_url` is rendered by the details panel**, not the card
  (`panels/element_details.cljd:41-42`).
- **e.6 — `Delta` / `SnapshotSeed` stay external placeholders.** The two
  `missingSourceHint` warnings cannot be cleared from this repository.
- **The deferred grey-box causes — all three still present, still deliberately
  deferred** (see section f of `2026-09-15-model-reload-button.md`):
  1. `canvas.cljd:286` — unknown `:kind` falls back to grey;
  2. `store.cljd:163` replace vs `:221` merge — `element-updated` replaces
     rather than merges;
  3. `stream.cljd:156-157` + `store.cljd:254-270` — a partial element can reach
     the store in the first place.

---

## 5. Recommended actions, ordered

1. **change-code — A2.** *(done)*
2. **change-spec-or-code — A1.** *(done code-side; spec untouched)*
3. **change-code — A3**, plus **flag** the `../thecli` doc/schema contradiction
   (`change-stream.md:241` against `change-stream.schema.json` and
   `wireframe.clj`); the frontend tests encode the wrong shape and must move
   with it.
4. **human decision — A4.** *(done — spec side)* Decide whether
   `is_information_complete` is derived in code or wire-supplied, then make spec
   and code say the same thing. *(Resolved by moving the SPEC side:
   `em-frontend.allium` now re-declares `Element.is_information_complete` as the
   wire-supplied `is_information_complete: Boolean` (previously a derived clause
   at old lines 185-203); the sourcing predicate now lives only in
   `unsourced_fields`, whose comment gained a cross-reference, and the
   `FieldFlowHighlights` prose reference was retargeted from
   `Element.is_information_complete` to `Element.unsourced_fields`. No
   production code changed: the store still keeps the wire flag
   (`stream.cljd:85`), and `canvas.cljd`, `app.cljd`,
   `panels/element_details.cljd` and `store.cljd` still read it.)*
5. **change-code — A5.** *(done)* Restore the fixtures to verbatim copies of the
   doc's payloads so the wire-drift guard can actually guard. *(Done: the two
   fixtures in `test/em_frontend/stream_test.cljd` (`delta-create-element`,
   `delta-set-image-url`) are now byte-for-byte copies of `change-stream.md:150`
   and `:157` — each gained `"field_origins":[]` and
   `"is_information_complete":true` — and the expected map in
   `normalize-create-element-delta-test` moved from `:is_information_complete
   nil` to `true`. The ns docstring was narrowed to name which fixtures are
   verbatim and to state that `delta-add-wireframe` is hand-built to the
   wireframe wire shape; `delta-cascading-delete-timeline` was already verbatim.
   A3 remains open and its wireframe fixture was deliberately left untouched.)*
6. **change-code — B1.** *(done)* Add a snap-to-content test (its siblings are
   covered, so the omission is conspicuous). *(Done: a new
   `viewport-snaps-to-content-test` in `test/em_frontend/interaction_test.cljd`
   covers fit+centre, the filter-cleared-and-timelines-reappear case, the clamp
   binding against `config/max-zoom`, and the empty-content reset to `config`'s
   defaults. No production code changed.)*
7. **change-docs.** The two corrections this pass made were **annotated into
   `2026-09-15-model-reload-button.md`** (the disproved "no new transition"
   claim, and the discarded `retry-stream!` handle), per the append/annotate
   convention — nothing there was overwritten.
8. **optional spec prose** *(done)* acknowledging that `ReloadModel` is realised by
   retiring the live driver and starting a fresh attempt (section 1, point 3).
   *(Done: the comment on `ReloadRestartsConnection` now states that the app
   realises the reload as a driver swap -- the live driver retired and a fresh
   attempt started -- with `RetryStream`'s `start-stream!` cited as the same
   deliberately-unmodelled layer.)*
9. **no-action** for all of class C (section 4).
10. **triage** for the remaining B items (B2–B9; B2 in particular needs an
    async/HTTP harness before it can even be attempted). *(partly done — B3–B8
    triaged; B2 and B9 left unhandled)*

> **2026-09-15 — A4/A5/B1 closed.** The three (items 4–6 above) are now
> resolved; **A3 remains open**, and its wireframe fixture was deliberately left
> untouched. **B5** (the testable face of A4) was closed as a side effect of
> A4: the new `wire-is-information-complete-matches-derived-unsourced-fields-test`
> in `test/em_frontend/stream_test.cljd` asserts, over the `real-snapshot`
> fixture, that each parsed element's `:is_information_complete` equals
> `(empty? (model/unsourced-fields element conns-with-from))` (element 8 `false`,
> element 9 `true`). Verification this session: `clj -M:cljd:test test` exit 0
> with `+100: All tests passed!` (both new tests present),
> `clj -M:cljd clean && clj -M:cljd compile` exit 0, and
> `/home/mvi/.local/bin/allium check em-frontend.allium` exit 1 with the same
> five diagnostics bar the `SpecStep.spec` line shift (268 → 258; see
> `2026-09-15-spec-code-alignment.md` section b). The preamble verdict above
> ("three remain open (A3, A4, A5)") is left verbatim: it records the state the
> pass itself measured.

> **2026-09-15 — B3–B8 triaged.**
> - **B3 — no action, by decision.** `@guarantee LargeModelScale` is kept as
>   wording: nothing in this harness can measure responsiveness at thousands of
>   elements (there is no benchmark/timing harness and the paint half needs a
>   real `m/Canvas`), so the guarantee stays aspirational rather than gaining a
>   test that asserts something adjacent.
> - **B4 — resolved.** The shade choice moved out of `paint` into the pure
>   `band-shade` helper (`canvas.cljd`, called by the painter), with
>   `band-shade-alternates-test`. It asserts the alternation shape (parity), not
>   the pixel colour — which is all "visually distinguished" requires.
> - **B6 — resolved, both halves.** (i) The three hit-test helpers now take
>   plain widget-local `x`/`y` instead of an `m/Offset`, which was the only
>   reason they were unreachable from `cljd.test`; all nine call sites are in
>   `app.cljd` and were adapted with no behaviour change. Six tests cover
>   placement hits/misses, the inclusive right/bottom edges, slice headers and
>   edge proximity. (ii) The missing opening triggers are now a stated decision
>   rather than an oversight: the spec's Surfaces section records that the
>   selection/hover/expansion state which opens the panels and tooltips is
>   transient renderer state held outside the model and is deliberately
>   unmodelled.
> - **B7 — resolved as documentation.** A spec note now states that `loading` is
>   resolved to `rendered`/`empty` within the same application and is never an
>   observable resting state (the pre-snapshot spinner is the absence of a
>   `Visualization`), and `app.cljd`'s dead `:loading` case comment was
>   corrected. No behaviour change.
> - **B8 — resolved.** The reseed's `:delta-queue` drop is asserted in
>   `reconnect-preserves-viewport-test`; the app's reseed-scoped ui resets were
>   extracted as the pure `ui-after-reseed` and are covered by the new
>   `test/em_frontend/app_test.cljd` — the app's first test namespace; and
>   `ReconnectPreservesViewport`'s comment now states the side effects. Caveat:
>   that a reseed *must* discard the deferred queue is still only implied by
>   `@invariant NoReplay`, not stated as an obligation of its own.
> - **B2 and B9 — left unhandled by decision.**
> Verification: `clj -M:cljd:test test` exit 0 with `+107: All tests passed!`
> (was 100), `clj -M:cljd clean && clj -M:cljd compile` exit 0, and
> `/home/mvi/.local/bin/allium check em-frontend.allium` exit 1 with the same
> five diagnostics at lines 37/40/40/139/258 — all three spec prose insertions
> sit below line 258, so the recorded baseline did not move (the spec grew 1161
> → 1173 lines).

---

## Follow-up pointers

- Re-run `/home/mvi/.local/bin/allium check em-frontend.allium` before trusting
  anything here; the baseline only holds while the spec is unchanged, and this
  pass left the spec untouched **by design** — the two fixes it applied were
  code-side.
- The two A fixes moved code, not the spec, and **no test was added for
  either** (see B2). If the reload or retry path is touched again, read section
  1 first: the layer those bugs live in is outside the spec's vocabulary and
  invisible to `allium check`.
- Treat A3 as the highest-value open item: it silently kills the field-name
  overlay and mis-renders every real screen wireframe, and the passing tests do
  not notice because they encode the same wrong shape as the code.
- Section 4's items are **confirmed intentional**. Do not "fix" them
  opportunistically; e.4 in particular does not cover A3.
