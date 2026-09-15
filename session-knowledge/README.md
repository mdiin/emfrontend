# Session knowledge

Durable notes from spec/code alignment work on this repository.

**Read this folder before you run an `allium check` or weed
`em-frontend.allium` against the code.** A fresh session otherwise has to
rediscover the expected diagnostic baseline, the checker quirks that make some
diagnostics misleading, and the conventions earlier sessions settled.

## Index

| File | Contents |
|------|----------|
| `2026-09-15-spec-code-weed.md` | Read-only weed pass over the uncommitted reload change: `DIRTY` verdict with `allium check` at baseline (3 warnings / 2 infos at lines 37/40/40/139/268, exit 1), the reload change verified (store action matches `ReloadRestartsConnection`; the reseed rides `SnapshotReceived` / `ReconnectPreservesViewport`; the driver lifecycle is outside the spec's vocabulary and invisible to the checker), five class-A divergences — A1 (undeclared `reconnecting -> connecting`) and A2 (`retry-stream!` discarding its driver handle, leaving two live drivers) **FIXED code-side**, A3 (wireframe node shape / `screen` vs `canvas` root), A4 (`Element.is_information_complete` derived-vs-stored provenance, needs a human decision) and A5 (test fixtures drifted from the wire examples they claim to copy) **OPEN** — nine class-B coverage gaps, six class-C items confirmed intentional with the three grey-box causes still deferred, and an ordered action list. |
| `2026-09-15-model-reload-button.md` | Manual model reload session: the viewer-invoked `ReloadModel` capability on `surface Viewport` and the `ReloadRestartsConnection` rule (a fresh connection forced from `connected -> reconnecting`, reusing the existing reseed path), the product decision that reload refreshes the model and not the view (zoom, pan, timeline filter and field-flow highlight preserved), the stream driver's new cancellation handle (`drive-stream!` / `connect!` returning `{:cancel ...}` / `disconnect!`), `98: All tests passed!` with the `allium check` diagnostic set unchanged, and the three grey-box delta-bug causes deliberately deferred in favour of the reload button. |
| `2026-09-15-spec-code-alignment.md` | Spec/code alignment session: the expected `allium check` baseline, two checker quirks verified with probes on Allium 3.6.0, conventions settled, gaps deliberately left in place, and the per-change verification workflow. |

## Conventions

- **One dated file per session**, named `<YYYY-MM-DD>-<topic>.md`, using the
  date of that session's final commit (`git log -1 --format=%cs`).
- **Append, never overwrite.** A new session adds its own file and a row above;
  it corrects an earlier file only to annotate a claim that has since been
  disproved, and says so in the text.

## Not part of the project

`.clj-kondo/`, `.lsp/` and `NOTES.md` at the repository root are untracked
local scratch entries. They are **not** part of this project and must not be
committed. Never use `git add -A` or `git add .` in this repository.
