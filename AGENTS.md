# emfrontend

A visualisation of the Event Model produced by the emcli.

## Tech stack

- ClojureDart
- InteractiveViewer for pan/zoom/scroll
- CustomPainter doing viewport-culled painting of your event model elements
- Standard Flutter text widgets/TextPainter for labels
- Custom-built grid-layout calculator (no library)
- Custom-built edge-routing algorithm (no library)

## Guidelines
- Use an idiomatic functional style

## End goal
- Package as a single binary

## Build/test workflow gotcha
Running `clj -M:cljd:test test` (the `:test` alias adds `test` to `:paths`)
leaves stray cross-imports of the `*_test.dart` files in the shared
`lib/cljd-out` production build output. `flutter build`/`flutter analyze`
will then fail with `uri_does_not_exist` errors pointing at
`test/cljd-out/...`. Run `clj -M:cljd clean && clj -M:cljd compile`
(without `:test`) before building or running the app again.

## Spec checks and session knowledge

`session-knowledge/` holds durable notes from past spec/code alignment sessions.
Read `session-knowledge/README.md` before running an `allium check` or weeding
the spec against the code: it records the expected `allium check` baseline, two
checker quirks that make some diagnostics misleading, and conventions settled in
earlier sessions.
