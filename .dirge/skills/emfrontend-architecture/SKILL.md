---
title: emfrontend Architecture & Development Guide
description: Key patterns, conventions, and gotchas for working on the emfrontend ClojureDart/Flutter codebase
tags: [clojuredart, flutter, architecture]
---

# emfrontend Architecture & Development Guide

## Verification

```bash
bb test
```

All 67 tests should pass and the final compile should succeed cleanly.

---

## Core Pattern: Pure Store Rules

Every piece of business logic is a pure function `store -> {:store ... :effects [...]}`.

```clojure
;; All rules follow this exact shape
(defn some-rule [store & args]
  {:store (assoc-in store [:some :key] new-value)
   :effects [{:type :timeline-laid-out :timeline-id tid}]})
```

**Never** mutate atoms inside rule functions. Atoms live only in `app.cljd`; rules are pure.

## Effects System

Effects are declarative render/animation triggers returned from rules — never stored. The Flutter layer (`app.cljd`'s `apply-outcome!`) consumes them immediately:

- `:element-appears` — animate element onto canvas
- `:timeline-laid-out` — re-render a timeline's layout
- `:canvas-relaid-out` — full canvas reflow
- `:connection-highlighted` / `:connection-dimmed` — edge styling
- `:queued-deltas-flushed` — interaction ended, queue processed

## State Transition Detection

Use `becomes?` from `store.cljd` to detect transitions:

```clojure
;; Fires only when new-val = target AND prior-val ≠ target (including creation where prior is absent)
(becomes? prior-val new-val :connected)
```

## Delta Deferral During Interaction

While `(get-in store [:visualization :interacting])` is `true`, incoming SSE delta batches go to `:delta-queue`. `end-interaction` in `interaction.cljd` flushes them in commit order through the same `apply-delta-batch` they would have used live.

## Entity Conventions

- Entities are plain maps with keyword keys — no `defrecord`/`deftype`
- Enum values are keywords matching spec enum members: `:command`, `:state_change`, `:connected`, etc.
- Foreign keys are bare ids (e.g. `:slice :s1`), not embedded objects (except in snapshot wire format)

## Layout Constants (`layout.cljd`)

```
slice-width          220.0
slice-header-height   56.0
swimlane-row-height  120.0
swimlane-label-width  96.0
card-margin            8.0
```

Swimlane row 0 = default unnamed band; named swimlanes occupy rows 1+ in `swimlane-order`. **Unused swimlane rows are compacted out** — only rows that hold at least one placement reserve space.

## Edge Rendering (`edges.cljd`)

Connections render **right-to-left**: `to.index >= from.index` is an invariant. Edges are drawn from the `from`-card's right edge to the `to`-card's left edge.

The `drawn-edges` nearest-placement predicate: a `(from, to)` pair draws only when no closer `from` placement exists between them and no closer `to` placement exists between them (mutually nearest). This predicate deliberately does NOT enforce `to.index >= from.index` — that's kept in invariants.cljd separately.

`timelines-to-relayout` is called on connection create/delete to avoid full-canvas relayout — only affected timelines are re-laid-out (`ConnectionChangeRelaysOut` optimization in the spec).

## SSE Wire Format (`stream.cljd`)

- First event: `{"op":"snapshot","model":{...}}` — **nested** projection (timelines nest slices nest placements with embedded elements)
- Subsequent events: `{"op":"<OpName>","changes":[{"action","type","id","entity"?},...]}` — **flat** canonical records with bare ids
- `normalize-snapshot` and delta normalizers deliberately differ on `from`/`to`/`element` shape
- `SetConnectionDerivations` is the only op that updates a connection; `type=connection, action=updated` unambiguously means derivation update

Reconnect: `initial-backoff-ms=500`, `max-backoff-ms=30000`, max 8 attempts. Snapshot reseeds the store wholesale (prior delta state discarded), but viewport (zoom/pan) is preserved across reconnects.

## Store Shape

```clojure
{:event-model nil
 :timelines {} :swimlanes {} :slices {} :elements {}
 :placements {} :connections {} :specifications {} :spec-steps {}
 :visualization nil
 :sse-connection {:state :connecting}
 :delta-queue []}
```

## Config Constants (`config.cljd`)

```
default-zoom  1.0    min-zoom  0.1
default-pan-x 0.0    max-zoom  5.0
default-pan-y 0.0
```

## `canvas_fields` — Which Kinds Show Fields

```clojure
(def canvas-field-kinds #{:command :event :read_model})
;; :screen shows image thumbnail; :automation shows nothing
```

## Testing Conventions

- All tests are pure function tests in `test/em_frontend/`; no Flutter widget tests (the Flutter layer is intentionally thin)
- Test namespaces: `store-test`, `layout-test`, `edges-test`, `stream-test`, `model-test`, `interaction-test`, `config-test`
- Import: `[cljd.test :refer [deftest is testing]]`
- Fixtures use plain maps matching the entity shapes; no mock frameworks
- `stream-test` uses byte-for-byte real SSE payloads captured from a live `emcli serve` run — the guard against wire-schema drift

## Build Commands

```bash
bb test          # run all tests + clean + recompile
bb compile       # production compile only
bb clean         # clean cljd output
bb app           # compile + flutter run -d linux
bb dev           # hot-reload REPL dev (loads dev.cljd alongside main)
bb analyze       # flutter analyze
bb build-linux   # release linux build
bb build-macos   # release macos build
```

## CRITICAL: Test Build Gotcha

`clj -M:cljd:test test` leaves stray `*_test.dart` cross-imports in `lib/cljd-out`. **Always use `bb test`** (which does `clj -M:cljd:test test` → `clean` → `compile`) rather than running the test command directly. If you ran tests manually, run `bb clean && bb compile` before `flutter analyze` or `flutter build`.

## REPL-Driven Dev (`dev.cljd`)

`dev.cljd` is NOT required by `em-frontend.main`. Run `bb dev` to load it alongside main via `clojure -M:cljd flutter em-frontend.main em-frontend.dev`.

Key REPL helpers (after `(in-ns 'em-frontend.dev)`):
- `(pick!)` — arm the widget picker; click a widget to capture its scope
- `(current-store)` / `(current-ui)` — inspect live state
- `(swap! (store-atom) ...)` — poke live state
- `(mount! widget)` / `(mount! nil)` — swap in / revert a replacement widget
- `(ancestors)` — debugGetDiagnosticChain from picked widget

`*env` holds the full lexical scope at the picked call site; `:store-atom`/`:ui-atom` are always present.

## Spec Authority

`em-frontend.allium` (1024 lines, Allium DSL) is the **canonical authority** for every rule, surface, invariant, and entity. Every source namespace has a `ns` docstring citing which section(s) of the spec it implements.
