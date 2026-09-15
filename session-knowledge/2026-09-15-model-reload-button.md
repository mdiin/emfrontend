# 2026-09-15 — model reload button

Repository: `emfrontend`. HEAD at the time of writing: `23b6636`
(`chore: bump version to 0.8.0`, committer date `2026-09-15`).

**Date note.** The convention in `README.md` is one dated file per session
named for the session's final commit (`git log -1 --format=%cs`, i.e. the
*commit* date). This session's work is **not yet committed**, so this file is
named for the current date from `date +%F` — `2026-09-15` — which today happens
to coincide with the HEAD commit date. If the eventual commit lands on a
different date, this file keeps the name it was written under; nothing below
depends on it.

Work covered by this session: add a manual **Reload model** action to the app
bar, specced as a fresh connection forced from a healthy stream, and record how
it reuses the existing reconnect/reseed path. Verification commands and their
observed output are recorded in section f.

Everything below is traceable to this repository — re-run the commands shown,
or read the cited `file:line` locations. Where the session brief's line
approximations did not match the working tree, the verified number is given
inline under *Brief vs repository* rather than repeated as fact.

---

## a. The reload capability

### Spec

`surface Viewport` gains one provided action, `em-frontend.allium` line 1133:

```allium
    -- Forces a fresh connection, and therefore a fresh full snapshot; always
    -- available, filtered view or not, and the current view state (zoom, pan,
    -- filter and field-flow highlight) is preserved across the reseed.
    ReloadModel(viewer, viz)
```

and a new rule, `em-frontend.allium` line 441 (comment block lines 433–440):

```allium
rule ReloadRestartsConnection {
  when: ReloadModel(viewer, viz)
  let connection = SSEConnection
  requires: connection.state = connected
  ensures: connection.state = reconnecting
}
```

The rule drives the connection along the **existing** `connected ->
reconnecting` edge already declared in `entity SSEConnection`
(`em-frontend.allium` line 325; entity at line 319). No new state, no new
transition and no new entity is introduced: the parallel rule
`RetryRestartsConnection` already uses the `error -> connecting` edge, and
`ReloadRestartsConnection` uses the healthy-stream one.

> **Correction — this claim was DISPROVED by the weed pass (added later; the
> original wording above is kept deliberately).** The *spec* introduced no new
> transition, but the *code* did. `reload-model!` started a fresh driver and
> `drive-stream!` announced `:connecting` on its first attempt (a hardcoded
> `(if (zero? attempt) :connecting :reconnecting)`), so the reload's observed
> sequence was `connected -> reconnecting -> connecting -> connected` — and
> `reconnecting -> connecting` is **not** a declared edge of `SSEConnection`'s
> transition graph (`em-frontend.allium` lines 321–330). So the reload wrote
> an undeclared state change after the rule's `:reconnecting`, i.e. the code
> violated the spec it implements.
>
> **Fixed code-side, spec unchanged** (`allium check` stays at baseline). The
> caller now supplies the state the driver announces for attempt 0
> (`initial-state`, threaded through `drive-stream!` → `connect!` →
> `start-stream!`), and each caller's driver announces exactly the state its
> rule ensures: `:connecting` for the cold start in `root` and for
> `retry-stream!` (RetryRestartsConnection), `:reconnecting` for
> `reload-model!` (ReloadRestartsConnection). A reload now yields
> `connected -> reconnecting -> connected`, all declared edges; later
> attempts are still backoff retries and still announce `:reconnecting`.

Because the reload only fires from `connected`, and the reloaded connection
leaves that state, `requires: connection.state = connected` is *also* the
single-in-flight guard: while the fresh attempt is starting a second reload
cannot fire. This mirrors the same trick in `RetryRestartsConnection`.

### Why reload is modelled as a reconnect

The contract the frontend demands, `contract ChangeStream`
(`em-frontend.allium` line 82), exposes only two operations:

```allium
  connect: (stream_url: String) -> SnapshotSeed
  next_delta: (after: Delta) -> Delta
```

There is no "re-read current state" call, and `@invariant NoReplay`
(line 96) states that on reconnect the consumer gets a *fresh snapshot of
current state*, then deltas from that point — prior delta-built state is
discarded and reseeded. So the only way to obtain a complete snapshot again is
to (re)connect. That is why reload is a reconnect: the fresh snapshot arrives
through the existing `SnapshotReceived` path, whose reconnect rule is
`ReconnectPreservesViewport` (`em-frontend.allium` line 378), which reseeds
the store wholesale. Reload therefore adds no new ingest path — it just forces
the one that already exists.

### The `allium check` diagnostic set is unchanged

Measured after the change (see section f for the exact command). The set is
**identical to the baseline recorded in
`2026-09-15-spec-code-alignment.md` section b**: the same **3 warnings and 2
infos**, at the same locations — `Delta` line 37, `SnapshotSeed` lines 40 and
40, `EventModel.elements` line 139, `SpecStep.spec` line 268 — exit code `1`.
The reload change introduced **no new diagnostic**. As that baseline file
warns, compare the diagnostic *list*, not just the exit code: a new
info-only diagnostic would not change the exit status.

---

## b. Product decision: reload refreshes the model, not the view

Reload is deliberately a **data refresh that keeps the user's place**. Zoom,
pan, the timeline filter and the field-flow highlight are all preserved across
the reload. Nothing in the reload path touches the `Visualization` entity:

- `ReloadRestartsConnection` only moves `SSEConnection.state`;
- the reseed goes through `ReconnectPreservesViewport`, which explicitly
  leaves `zoom` / `pan_x` / `pan_y` (and the view-local filter and trace
  state) unchanged.

This was a **deliberate choice, not an oversight**. A full reset (re-created
`Visualization` at the config defaults) was the alternative; the reload was
offered and taken as the restart-equivalent *data* refresh that does not lose
the user's position. The rationale is exactly `ReconnectPreservesViewport`'s:
a snapshot arriving mid-session must not yank the viewport out from under the
viewer. Making reload a reset would have meant a second, contradictory
reseed rule.

---

## c. Implementation notes

### `src/em_frontend/store.cljd`

`reload-model` (line 125) is a pure store action mirroring `retry-stream`
(line 112) — guarded on `:connected`, one state move, no effects:

```clojure
(defn reload-model
  "rule ReloadRestartsConnection: ... Guarded on `connected` ..."
  [store]
  (if (= :connected (get-in store [:sse-connection :state]))
    {:store (assoc-in store [:sse-connection :state] :reconnecting) :effects []}
    {:store store :effects []}))
```

It returns the uniform `{:store ... :effects []}` shape every rule in this
library returns, so `apply-outcome!` needs no special case. The `:connected`
guard makes a reload that arrives while an attempt is starting, while
reconnecting, or after a failure a no-op (the failed case belongs to
`RetryRestartsConnection`).

### `src/em_frontend/stream.cljd`

The driver had to become retireable, otherwise a reload would leave **two live
drivers writing the same atoms** (the old connection would keep delivering its
own snapshot/deltas into the store alongside the new one). Three changes:

- The connection + backoff loop moved out of `connect!` into a private
  `^:async drive-stream!` (line 266), so `connect!` no longer `await`s and can
  return synchronously.
- `connect!` (line 338) now **returns a driver handle**:

  ```clojure
    (drive-stream! stream-url on-sse-state on-snapshot on-delta on-invalid stopped live!)
    {:cancel cancel!}))
  ```

  `cancel!` sets the `stopped` flag first (idempotent, and stops the loop from
  retrying), then stops the subscription, completes the loop's `Completer` and
  closes the `http/Client`.
- A new `disconnect!` (line 379) retires a handle, idempotent and safe on a
  handle whose driver already ended (e.g. on `:error`).

### `src/em_frontend/app.cljd`

`reload-model!` (line 125) is the effectful wiring: retire the live driver,
apply the pure outcome, start a fresh driver.

```clojure
  (when (= :connected (get-in @store-atom [:sse-connection :state]))
    (stream/disconnect! @stream-handle)
    (apply-outcome! store-atom ui-atom (store/reload-model @store-atom))
    (reset! stream-handle (start-stream! store-atom ui-atom)))
```

The current driver handle is retained in a `:managed` atom so the reload can
find it — `stream-handle` (declared line 493, seeded line 494 with
`(start-stream! store-atom ui-atom)`). The button lives in `app-bar`
(line 413), in `.actions`, next to Snap to content:

```clojure
     (m/IconButton
       .icon (m/Icon m/Icons.refresh)
       .tooltip "Reload model"
       .onPressed (fn [] (reload-model! store-atom ui-atom stream-handle) nil))
```

No view state is read or written anywhere in `reload-model!`, which is the
code-level expression of the decision in section b.

> **Correction — `retry-stream!` used to discard the driver handle (added by
> the weed pass; FIX A2).** The handle thread described above was incomplete:
> `retry-stream!` — the *other* path that starts a driver — called
> `start-stream!` and threw the returned `{:cancel ...}` away, so only `root`
> and `reload-model!` wrote the `stream-handle` atom. Reachable sequence:
> the stream fails (`:error`, driver dead) → the viewer retries (fresh live
> driver H2, its handle lost) → the viewer clicks Reload, which cancelled the
> **dead** H1 recorded in `stream-handle` and started H3 — leaving H2 and H3
> both delivering snapshots and deltas into the same atoms (double reseed,
> every delta applied twice). That defeats exactly the single-in-flight
> guarantee the reload was built for.
>
> **Fixed**: `retry-stream!` now takes `stream-handle`, retires the current
> handle first (`stream/disconnect!`, nil-safe and a no-op on an already-dead
> driver) and records its replacement —
> `(reset! stream-handle (start-stream! store-atom ui-atom :connecting))` —
> mirroring `reload-model!`. The handle is threaded `root` → `overlay-stack`
> → the error banner's `:on-retry`, the same path the reload's handle already
> travels to `app-bar`. Every place that starts a driver (cold start, retry,
> reload) now records its handle, so `stream-handle` always names the live
> driver. The `:managed` `stream-handle` atom and its `:dispose nil` are
> unchanged — disposal semantics were explicitly out of scope.
>
> **Line numbers (annotated by the weed pass).** The line numbers cited in
> this section were correct when written; the FIX A1/A2 edits added docstring
> lines, so re-derive rather than trust them. Verified after the weed pass:
> `src/em_frontend/stream.cljd` has `drive-stream!` at line 266 (unchanged),
> `connect!` at line 344 and `disconnect!` at line 392;
> `src/em_frontend/app.cljd` has `start-stream!` at 99, `retry-stream!` at
> 121, `reload-model!` at 143, `overlay-stack` at 361, `app-bar` at 438, and
> `stream-handle` declared at line 520 / seeded at 521. `src/em_frontend/store.cljd`
> was not touched by the weed pass, so its citations (e.g. `reload-model` line
> 125) still hold.

---

## d. Tests

`test/em_frontend/store_test.cljd` gained `reload-restarts-connection-test`
(line 119), covering the three categories the store tests use:

- **rule_success** — from `:connected`, `reload-model` moves the connection to
  `:reconnecting`, leaves `Visualization.state` at `:rendered` (untouched) and
  produces no effects;
- **transition_edge** — the reload is the witnessing rule for the
  `connected -> reconnecting` edge and lands exactly in that target state;
- **rule_failure** — the `:connected` guard is the single-in-flight guard: for
  each of `:error`, `:connecting`, `:reconnecting` the state is left unchanged.

The suite reports `98: All tests passed!`.

---

## e. Mandatory verification workflow

Run in the order recorded in `2026-09-15-spec-code-alignment.md` section f
(and `AGENTS.md`):

1. `/home/mvi/.local/bin/allium check em-frontend.allium` — diagnostic set
   **unchanged** versus the baseline (section a above; exit `1`).
2. `clj -M:cljd:test test` — `98: All tests passed!`
3. `clj -M:cljd clean && clj -M:cljd compile` — **without** `:test`;
   `All clear! 👌`, exit `0`.

Step 3 is not optional and must follow step 2: the `:test` alias adds `test`
to `:paths` and leaves stray cross-imports of the `*_test.dart` files in the
shared `lib/cljd-out` production build output, after which `flutter build` /
`flutter analyze` fail with `uri_does_not_exist` errors pointing at
`test/cljd-out/...`. (The compile emits a pre-existing
`DYNAMIC WARNING: can't resolve member listen on target type dynamic ...
em_frontend/stream.cljd:319`; it is not an error and the build is `All clear`.)
> *(Annotated by the weed pass: FIX A1 added docstring lines above it, so this
> same pre-existing warning is now reported at `em_frontend/stream.cljd:325`.
> Same single warning, same message, still not an error; `clj -M:cljd clean &&
> clj -M:cljd compile` still exits `0`. The recorded success banner on the
> re-run was `Easy peasy! 😎` rather than `All clear! 👌` — the exit code is
> the contract, not the quip.)*

---

## f. DEFERRED KNOWLEDGE — the grey-box bug, deliberately not fixed

**Not fixed, and not to be "fixed" opportunistically.** Reload was chosen as
the remedy for a symptom described in session as a **grey/blank element
card**: an element renders as an empty white/grey box with no name and no
field lines. The reload button clears it because it forces a fresh snapshot,
which re-registers full elements — but that treats the symptom, not the
cause. Below are the three code-traceable causes found while investigating.
They are recorded here so a future session can pick them up; the root-cause
delta fix was **deferred in favour of the reload button**, which is what the
session decided to ship.

1. **Unknown `:kind` falls back to grey, with no name and no fields.**
   `src/em_frontend/canvas.cljd` line 286, in `draw-element-card`:

   ```clojure
   kind-color (get element-kind-color (:kind element) m/Colors.grey)
   ```

   An element whose `:kind` is `nil` (or unrecognised) is drawn with the grey
   fallback border. Note the name is drawn from `(or (:name element) "")`
   (line 293) and the field lines from `model/canvas-fields` — so a *partial*
   element (see cause 3) is the thing that actually looks like an empty grey
   box: nothing to name, nothing to list.

2. **`element-updated` REPLACES the stored element rather than merging it.**
   `src/em_frontend/store.cljd` `element-updated` (line 159), body line 163:

   ```clojure
   store' (assoc-in store [:elements (:id element)] element)
   ```

   A delta entity that omits a key therefore erases the previously known
   `:kind` / `:name` / `:fields` for that element, leaving whatever the delta
   carried. Contrast the connection path, which merges rather than replaces —
   `connection-derivations-updated` (`store.cljd` line 221):

   ```clojure
   store' (update-in store [:connections (:id connection)] merge connection)
   ```

   > *Brief vs repository.* The session brief approximated this pair as
   > "`store.cljd` around line 148 ... around line 202". The verified working
   > tree has the replacing `assoc-in` at **line 163** (`element-updated` at
   > line 159) and the merging `update-in`/`merge` at **line 221**
   > (`connection-derivations-updated` at line 217; `connection-changed`'s
   > create branch also `assoc-in`s at line 213, which is fine on create —
   > there is nothing to merge into). The substance is unchanged: element
   > deltas replace, connection deltas merge.

3. **A partial element can be placed in the store in the first place.**
   Two registrations write an element with less than the full record:
   - **Connection endpoints.** `src/em_frontend/stream.cljd`
     `norm-connection-snapshot` (line 149) registers each endpoint as a
     partial `{id, name}` record into the elements index (the `swap!`/`merge`
     at lines 156–157). In snapshot normalization the placement walk runs
     first so the fuller element wins (see `norm-placement` line 120 and the
     ordering note in `normalize-snapshot` line 164), but the partial record is
     still a form that can reach the store.
   - **Placement deltas carry no element payload.** A delta `placement`'s
     `entity` is normalized as `{:id :slice :element}` where `:element` is the
     bare foreign-key id (`entity-normalizers`, `stream.cljd` line 198), and
     `placement-changed` (`store.cljd` line 254) only ever writes
     `:placements` — it never touches `:elements`. So a placement delta cannot
     supply the element's `:kind` at all, and any element known only via a
     placement delta (or via a connection endpoint) is partial.

So the natural root-cause fix is at cause 2: make `element-updated` merge the
delta into the stored element (like the connection path) instead of replacing
it, so a delta that omits `:kind`/`:name`/`:fields` cannot erase them. That
change was **deliberately deferred** in favour of the reload button.

---

## Follow-up pointers

- The reload button is the shipped remedy for the grey-box symptom; the
  delta-merge root-cause fix (section f, cause 2) is still open.
- Before trusting anything in section a, re-run `allium check` — the recorded
  diagnostic set is only valid while the spec is otherwise unchanged.
- This file was created as the single permitted new file for the task that
  produced it, so the `README.md` index has **not** been given its row yet.
  Whoever next edits `session-knowledge/README.md` should add a row for this
  file above the `2026-09-15-spec-code-alignment.md` row.
  > **Correction (added by the weed pass): this pointer is STALE.** The row
  > now exists — `session-knowledge/README.md` line 14 carries the
  > `2026-09-15-model-reload-button.md` row, above the
  > `2026-09-15-spec-code-alignment.md` row, as requested. Nothing further is
  > owed here.
- Two divergences found by the weed pass were fixed code-side with the spec
  untouched: the driver's attempt-0 announcement is now caller-supplied (FIX
  A1, see the correction under section a) so a reload is
  `connected -> reconnecting -> connected`, and `retry-stream!` now takes and
  records its driver handle (FIX A2, see the correction at the end of section
  c) so two drivers can never be left writing the same atoms. No test was
  added for either: the affected code is the async SSE driver and its UI
  wiring, which the current `cljd.test` harness cannot exercise (see section
  d — its coverage is the pure store actions, where nothing changed).
