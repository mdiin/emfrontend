# Session knowledge

Durable notes from spec/code alignment work on this repository.

**Read this folder before you run an `allium check` or weed
`em-frontend.allium` against the code.** A fresh session otherwise has to
rediscover the expected diagnostic baseline, the checker quirks that make some
diagnostics misleading, and the conventions earlier sessions settled.

## Index

| File | Contents |
|------|----------|
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
