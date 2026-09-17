# 2026-09-17 — connection state readout

Work covered by this session: make the SSE connection state visible to the
viewer, in answer to three asks — a label ("Connected" / "Disconnected"), a
loading spinner while an attempt is in progress, and an icon showing whether a
stream is live — with the agreed polish (a fixed-width label box so the readout
cannot shuffle the buttons beside it, and a red tint on the failed state; no
tooltip showing the raw state keyword).

This is the missing half of `2026-09-15-model-reload-button.md`: `ReloadModel`
and `RetryStream` both move `SSEConnection.state`, and before this change
**nothing in the UI read that state at all**, so a reload of an unchanged model
looked exactly like a dead button (see section d).

Files: `em-frontend.allium`, `src/em_frontend/app.cljd`,
`test/em_frontend/app_test.cljd`, this note, `session-knowledge/README.md`.

## a. The spec: a viewer-facing surface for the connection state

Appended at the end of the Surfaces section (`em-frontend.allium:1191-1212`, at
EOF, which is why no baseline diagnostic line moved):

```allium
surface ConnectionStatus {
  facing viewer: Viewer
  context connection: SSEConnection

  exposes:
    connection.state

  @guarantee ConnectionStateVisible
    -- The states the readout shows are distinguishable to the viewer:
    -- connected reads as a live connection, connecting and reconnecting as
    -- a connection attempt in progress, and error as a connection that is
    -- down until the viewer retries.
}
```

A new surface rather than an extension of an existing one: `surface Viewport`
exposes only `viz.*` (the view state), and `surface StreamErrorOverlay` exposes
`viz.state` and is conditioned on `state = error`, so neither could carry a
readout that must be visible in *every* connection state, including the window
before the first snapshot exists. `ConnectionStatus` follows the same shape as
`StreamErrorOverlay` (`facing` → `context` → `exposes` → `@guarantee`), and its
prose comment states that label wording, icon, progress indicator and alarm
styling are renderer constants — the spec models the three-way distinction,
not the strings.

No rule was added or changed: the state is already produced by `SSEConnection`'s
transitions block plus `ReloadRestartsConnection` / `RetryRestartsConnection`;
the surface only makes it observable.

## b. The mapping (renderer constants, pinned by `cljd.test`)

`SSEConnection.state` (`enum SSEState`) → what the readout shows:

| state | label | icon | spinner | alarmed |
|---|---|---|---|---|
| `connected` | "Connected" | `cloud_done` | no | no |
| `connecting` | "Connecting" | `cloud_queue` | **yes** | no |
| `reconnecting` | "Reconnecting" | `cloud_queue` | **yes** | no |
| `error` | "Disconnected" | `cloud_off` | no | **yes** (red) |

Decisions worth remembering, all deliberate:

- **`connecting` reads as an attempt in progress, not as "Disconnected"** — it
  is the same "not live yet" situation as `reconnecting`, and on the cold start
  it is the only thing on screen besides the body's spinner.
- **The wording lives in the renderer, not the spec** (the spec names the
  distinction). `connection-presentation-test` is therefore the drift guard for
  the strings, not the spec.
- **The readout is always visible**, including while `error` (where the overlay
  also shows) and before the first snapshot.
- **The fallback fails closed**: a `nil`/unknown state reads as a down
  connection, alarm included — the same reading as `error`.

## c. The code seams (`src/em_frontend/app.cljd`)

- **`connection-presentation` (line 449)** — public and pure, takes the *store*
  and returns the whole four-key readout map (`:label :icon :in-progress?
  :alarm?`) per state. Taking the store is what pins the read site
  (`[:sse-connection :state]`) together with the presentation.
- **`connection-status` (line 476)** — private widget; owns the *drawing*
  (spinner in place of the icon when `:in-progress?`, the glyph from `:icon`,
  the tint from `:alarm?`) and the min-width label box.
- **The mount (line 539)** — last element of `app-bar`'s `.actions`, so it reads
  as a status rather than a third button; the width it holds is what keeps the
  Snap-to-content and Reload buttons from moving.

No new store state, atom, effect or callback: the readout reads what
`root`'s `:watch [st store-atom]` already re-renders on, and touches neither
the reload/retry guards nor the error overlay.

## d. What it changes for the earlier reload session

That session's note records the reload as the shipped remedy for the grey-box
delta bug, and the earlier investigation of this repository recorded that the
reload gave the viewer **no visible feedback** and was a silent no-op whenever
the connection was not `:connected`. Concretely, this readout:

- makes `connected -> reconnecting -> connected` visible, so a reload looks like
  something happened even when the model it reseeds is identical;
- shows "Disconnected" when the first attempts fail **before any snapshot has
  arrived** — the case where `ErrorStateShowsMessage` cannot fire (it requires
  an existing Visualization) and the screen previously showed only a bare
  spinner;
- explains the reload's `:connected` guard: with the readout on screen, "the
  reload does nothing" is legible as "the stream is not live", not as a bug.

## e. Verification

Run in this order, all green:

1. `/home/mvi/.local/bin/allium check em-frontend.allium` — exit 1 with exactly
   the baseline diagnostics (warnings at 37/40/40, infos at 139/258). The list
   was compared, not just the exit code; `allium analyse` adds no findings.
2. `clj -M:cljd:test test` — `110: All tests passed!` (the suite has grown since
   the reload session's recorded 98; this change adds the assertions below).
3. `clj -M:cljd clean && clj -M:cljd compile` — clean, **without** `:test` (the
   `AGENTS.md` gotcha: the `:test` alias leaves stray `*_test.dart`
   cross-imports in `lib/cljd-out`).
4. `flutter analyze` — the same 13 pre-existing issues as before this change
   (generated `lib/cljd-out/cljd/*.dart`, plus the `Matrix4.translate/scale`
   calls in `apply-wheel-zoom!` and `snap-to-content!`); none inside the new
   code, which compiles to `lib/cljd-out/em-frontend/app.dart:381-550`.
5. The compiled output was read, not assumed: `app_bar`'s actions is a
   3-element vector whose last entry is `connection_status(st)`, and
   `connection_presentation` reads `[:sse-connection :state]` and returns the
   `:alarm?` key.

`connection-presentation-test` (`test/em_frontend/app_test.cljd:34-66`) pins all
four states by exact four-key map, asserts the three readings pairwise
distinct, and asserts that a store carrying only a visualization state falls
back — so a mis-keyed read site (or a `[:visualization :state]` fallback) fails
the suite.

## f. The weed loop (weed → fix → weed)

Three passes, each read-only and each against the working tree:

1. **Pass 1 — CLEAN** (no class-A divergence; the read site, the four distinct
   readings and the non-effects were all corroborated). It recorded three
   class-B gaps: B1 the widget's use of the mapping untested, B2 the mount
   untested, B3 the read site unasserted.
2. **Fix 1 — B3 closed (and B1's decision half).** Fixed by making
   `connection-presentation` take the store and return the complete map, and by
   the mis-keyed-store assertion above. B1/B2 remain for the drawing and the
   mount, which no test in this repository can reach.
3. **Pass 2 — CLEAN**, but it found that fix 1 had introduced B4: the new
   docstring claimed the mapping "leaves the widget nothing to decide", while
   `connection-status` still decides spinner-over-icon, glyph and tint. Two
   nits: the spec prose enumerated three renderer constants where the readout
   carries four, and a test message said "the three readings are
   distinguishable" while asserting one pair.
4. **Fix 2 — B4 and both nits closed.** Docstrings now claim exactly what the
   tests establish and name the drawing as uncovered; the spec prose includes
   "any alarm styling"; the test asserts all three pairwise inequalities and
   probes the out-of-enum case with `:half-open`.
5. **Pass 3 — CLEAN, converged.** B4 and the nits confirmed closed. Its one new
   item was B5, a wording imprecision in fix 1's own justification: the
   docstrings said the drawing was "not reachable from `cljd.test` (there is no
   widget harness)", which misattributes this repository's missing harness to
   the framework.
6. **Fix 3 — B5 closed.** Both docstrings now scope the claim to this
   repository ("cljd.test itself supports widget runners: the gap is this repo's,
   not the framework's"). No fourth pass was run: the change after pass 3 is
   comment text only, and the suite, the compile and `allium check` were re-run
   green on it.

## g. Accepted gaps, parked questions and intentional non-fixes

- **G1 — the drawing and the mount are untested.** `connection-status` and
  `app-bar` are private and this repository has no widget harness, so nothing
  asserts that the four-key map becomes a spinner/glyph/tint or that the readout
  is in `.actions` at all. This is **not** a framework limit: `cljd.test`
  supports widget runners (`:runner (ft/testWidgets [tester])`, ClojureDart
  `doc/TESTING.md:15-37` at the pinned revision) and `flutter_test` is already a
  dev dependency, so a widget test is possible if someone wants it. Recorded,
  not papered over.
- **G2 — pre-snapshot `Viewer` identification (parked, human decision).** The
  surface faces `viewer: Viewer`, but `Viewer` is `identified_by` a
  `Visualization` in `{rendered, empty, error}` and no Visualization exists
  before the first snapshot, so in that window the readout is on screen while
  its actor has no identifying instance. The same looseness already holds for
  `surface Viewport` (its buttons are mounted unconditionally) and for
  `StreamSource` vs `StreamIngest`, so it was judged class C — but the prose
  "present whenever the app is running" is only true of the code, not of the
  actor's identity condition. Either narrow the prose or accept the convention.
- **G3 — the layout claim is by construction.** The min-width box keeps the
  readout's width constant for the four labels it ships; a longer label would
  still widen it. No test can pin that.
- **Intentional, recorded rather than "fixed":** the fallback arm duplicates the
  `:error` reading verbatim (deliberate, and both arms are test-pinned); the
  `:icon` values for `connecting`/`reconnecting` are carried but never drawn
  (the spinner wins — kept so every state's map is complete); the renderer
  numbers (16.0, 2.0, 6.0, 104.0, 14.0) are inline literals; and on the first
  frame, before any snapshot, the body's spinner and the readout's spinner show
  at the same time (both truthful).

## h. For the next session

- Re-run `allium check` before trusting anything above: the baseline is valid
  only while the spec is unchanged (it is now 1213 lines; the baseline
  diagnostics sit at 37/40/40/139/258, all far above the new surface).
- The docstrings in `app.cljd` are deliberate about what is and is not covered —
  do not "fix" G1 by weakening them, and do not read them as a coverage claim.
- If `SSEState` ever gains a value, `connection-presentation`'s `case` will
  send it to the fail-closed arm: the readout will claim a down connection
  rather than crash, so give any new state its own arm and a test row.
