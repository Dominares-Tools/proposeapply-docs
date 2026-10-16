# proposeapply — AI proposes, a person approves, one call undoes

**Paid tool: $149 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/proposeapply**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

proposeapply is the part between a model's suggestion and your data. A model proposes changes in one strict JSON shape, a person approves, and registered handlers apply them. Every change goes into an append-only log and can be undone with one call, under a daily spend cap and off-by-default switches. You supply the model call and the handlers.

## Get it

Buy proposeapply at https://dominares.org/tools/proposeapply: one payment of $149, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the proposeapply command (demo, log, spend, controls, undo), examples, and the full test suite.

Requirements: Python 3.10 or newer, standard library only. tzdata is optional, for named time zones on systems without a time zone database.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

A model suggests; your code does the work; proposeapply is the part in between that decides whether, records what happened, and can take it back.

1. **`propose`**: you pass a function that calls your model. proposeapply checks the switches and the spend cap *before* the call, validates the reply against a fixed JSON shape, and logs the call's cost. Nothing in your data changes.
2. **A person approves** (your UI, your chat, your review queue; proposeapply has none).
3. **`apply`**: each approved action is re-checked (switch on, risk under the ceiling, kind registered, shape valid), run by your handler, and logged with the before-state your handler returned.
4. **`undo(action_id)`**: calls your inverse with that recorded diff and appends a rollback row. Inside the window (default 7 days) it works even while the dock is switched off.

The model never picks what runs. It can only name a kind you registered, and it supplies values that your validator checks. Anything else is refused, and nothing is logged as applied.

```python
from proposeapply import Dock, Registry

reg = Registry()                      # starts empty: nothing is allowed
reg.register("crm", "add_tag",
             apply=add_tag,           # (predicted_diff) -> diff dict: do it, return what undo needs
             inverse=restore_tags,    # (diff) -> None: put it back (omit for one-way actions)
             validate=check_shape)    # (predicted_diff) -> raises ValueError on a bad shape; no IO

dock = Dock("state/", reg, tz="America/Los_Angeles")
dock.controls.save({"verbs": {"do": True}})       # applying is OFF until you turn it on

envelope = dock.propose("crm", "do", my_model_call)
result = dock.apply("crm", approved_actions, actor="sam")
dock.undo(result["results"][0]["action_id"])
```

`examples/quickstart.py` is this, complete and runnable offline.

#### The model call

`my_model_call(model_name)` is yours. It returns a dict with `text` (the reply, as JSON) or `envelope` (already parsed), plus `model`, `input_tokens`, `output_tokens` and, if you know it, `cost_usd`. Without `cost_usd` the dock needs `Dock(..., prices={"model-name": (usd_per_million_in, usd_per_million_out)})`; no prices ship, names are matched without regard to case or surrounding spaces, and a model name matches the longest table key that is a prefix of it (so `gpt-4o-mini-x` prices as `gpt-4o` unless `gpt-4o-mini` is also a key). A model missing from your table raises an error, but the call has already been paid for: it is logged against the daily cap at a deliberately high placeholder (the larger of your predicted cost and the verb's default, or the tokens at $15 / $60 per million, whichever is more), with `diff: {"rejected": "unpriced"}`. The same applies if the reply's token counts or `cost_usd` are unusable. An unpriced call is therefore never free for the cap. If the call raises and `provider.fallback_model` is set, the call is repeated once with that name.

The reply must be an object with `panel`, `verb` (`audit`, `decide` or `do`; they must match the request), `summary`, `model` (a string or `null`), and lists `findings` (`id`, `severity`, `title`, `detail`, `suggested_verb`) and `actions` (`id`, `kind`, `risk` as `low`/`medium`/`high`, `reasoning`, `predicted_diff` object). One bad field rejects the whole reply (`InvalidEnvelope`); its cost is still logged because the money was spent.

For the `do` verb, actions above the risk ceiling or failing their validator are dropped (listed under `dropped`). Every kept action is stamped `appliable` (is the kind registered) and `reversible` (does it have an inverse). A `decide` reply keeps every action so a person can choose, including kinds nobody registered, which are shown but never run.

#### Switches (`controls.json`)

| Setting | Default | Meaning |
|---|---|---|
| `master_switch` | `true` | off stops `propose` and `apply` everywhere |
| `verbs.audit`, `verbs.decide` | `true` | allow those calls |
| `verbs.do` | **`false`** | allow `apply`; the gate for changing anything |
| `verbs.do_max_risk` | `low` | highest action risk that may be applied |
| `panels.<name>` | `true` per registered panel | per-feature switch |
| `budget.daily_cap_usd` | `3.0` | spend cap for the operator's calendar day |
| `budget.per_action_cap_usd` | `0.5` | largest predicted cost of one call |
| `provider.<verb>_model`, `provider.fallback_model` | `null` | model names handed to your call |

Unknown keys, wrong types and an unrecognised risk word are errors, never ignored, and a controls file that cannot be read turns every switch off.

#### Spend cap

Before each call, today's logged spend plus the call's predicted cost must fit under `budget.daily_cap_usd`, and the prediction must fit under the per-call cap (predictions default to $0.10 audit, $0.50 decide and do; change them with `predicted_costs=`). "Today" is the day in `tz`, not UTC. Undone actions still count: the money is gone. `cap_source=` takes a function returning an outside ceiling (a plan allowance, a billing limit) or `None`; the cap in use is the lower of the two, so a setting here can never raise an allowance owned elsewhere. If that function raises, the call is not made.

#### The log

`state/actions.jsonl`, one JSON object per line, only ever appended to. An applied action is an `action` row (`id`, `ts`, `panel`, `verb`, `action_kind`, `risk`, `model`, tokens, `cost_usd`, `diff`, `applied`, `idempotency_key`, `reasoning`, `actor`); an undo is a separate `rollback` row. Reading folds them. A damaged line, including one that is not valid UTF-8, is skipped, and the next row is never glued onto it.

Give an action an `idempotency_key` (the helper `idempotency_key(kind, params, fields)` builds a stable one) and applying it twice runs your handler once and returns the first result. Undoing it frees the key.

#### `text_file_action`

For the common case of replacing a text file, `text_file_action(folder)` returns ready `(apply, inverse, validate)` functions. The model gives a relative `path` and the new text; absolute paths, `..` and symlinks that leave the folder are refused, writes are atomic, and undo refuses to overwrite a file that was edited after the change.

#### Command line

```bash
proposeapply log --dir state/ --limit 20          # newest first; --json for rows
proposeapply spend --dir state/ --tz America/Los_Angeles
proposeapply controls --dir state/ set verbs.do=true budget.daily_cap_usd=5
proposeapply undo ACTION_ID --dir state/ --registry mypackage.docks:build_registry
```

`--registry` is `module:attribute`, a `Registry` or a function that takes the state folder and returns one. To try an undo from the command line on the demo, run `proposeapply demo --leave-applied` and use the line it prints.

### Limits

- **It does not call a model, send anything, or ship prompts, panels, screens or an HTTP layer.** You supply the model call, the handlers and the approval step.
- **Undo is only as good as your inverse.** The dock hands it the diff your handler returned; it cannot know what else changed. Undoing an older change after a newer change to the same thing can restore stale data unless your inverse checks (the file helper does). A kind registered without an inverse cannot be undone (`not_reversible`).
- **The cap is a pre-call guard, not a hard stop.** It compares logged spend with a prediction, so calls running at the same moment can overshoot by about their predicted cost. A call that fails with an error has no known cost and is not logged. Spend you do not route through the dock is not counted; `cap_source` is the place to bring that in.
- **Handlers run while the log is locked**, so keep them short. The lock is re-entrant (the same state folder spelled relatively, absolutely or through a symlink is one lock), so they may call back into the dock.
- **If an undo's inverse succeeds but the rollback row cannot be written** (disk full, permissions), `undo` returns `{"ok": False, "reason": "undone_but_not_recorded: ..."}`: the change is reverted, the log still shows it live, and a retry would call your inverse again.
- **Local files only.** Locking is by file lock on one machine; it is exercised here on macOS with threads and separate processes. Windows locking is written but untested, and network filesystems are not supported.
- **The whole log is read for each spend check and lookup.** Archive a log that has grown to millions of lines.
- **Model output is validated for shape, not truth.** The ceiling and your validator limit what an action can do; they do not make a wrong suggestion right.

Python 3.10 or newer, standard library only. An optional extra, `pip install '.[tz]'`, adds a time zone database for systems that ship none (Windows, slim containers). Run the tests with `pip install '.[dev]' && python -m pytest`.

---

Docs mirror of the private repository Dominares-Tools/proposeapply; updated with each release.
