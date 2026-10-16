# Changelog

## 1.0.1 - 2026-10-05

Fixes from review.

- Spend cap: a model call that cannot be priced afterwards (model missing from the price table, no price table and no `cost_usd`, unusable `cost_usd`, unreadable token counts) is now always logged before the error is raised, at a deliberately high placeholder cost. Before, the money was spent and nothing was logged, so the cap could be bypassed without limit.
- Price lookup ignores case and surrounding spaces in model names and table keys.
- A single non-UTF-8 byte in `actions.jsonl` no longer makes the whole log unreadable; that line is skipped.
- `text_file_action` keeps line endings (CRLF, lone CR) exactly, so undo restores the original bytes and no longer refuses a file whose new text has CRLF.
- Atomic writes keep the target's permission bits (new files get the umask mode) instead of 0600.
- The file lock is keyed on the resolved path, so the same folder spelled two ways cannot deadlock a thread.
- `undo` reports `undone_but_not_recorded` instead of raising when the rollback row cannot be written.

## 1.0.0 - 2026-10-05

First release.

- Envelope contract: one strict JSON shape for a model reply (findings and proposed actions); a malformed reply is rejected whole.
- Registry of action kinds, empty by default: a model can only name a kind you registered, each with an apply function, an optional inverse and an optional shape validator.
- Dock: `propose` (nothing changes), `plan` (dry run), `apply` (re-checks everything, idempotency keys), `undo` (calls your inverse with the recorded diff).
- Append-only action log (`actions.jsonl`): undo appends a row instead of rewriting; torn lines are skipped; cross-process safe on POSIX.
- Controls: master switch, per-verb and per-panel switches, risk ceiling; the *do* verb is off by default; unknown keys and wrong types are errors.
- Daily spend cap in the operator's time zone, per-call cap, and a pluggable outside cap that can only lower the cap.
- Model fallback by name from the controls, and cost from a price table you supply (no prices ship; an unknown model is an error).
- `text_file_action`: ready-made handlers for replacing a text file inside one folder, with path-escape protection and a refusal to undo over later edits.
- Command line: `demo`, `log`, `spend`, `controls`, `undo`.
