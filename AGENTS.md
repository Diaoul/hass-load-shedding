# Working in this repo

## What this is

One Home Assistant **blueprint** — `load_shedding.yaml` — plus its docs.
There is no application here, no build, no test suite, no CI.

A blueprint is a parameterised automation template. Users import it, then create
automations *from* it, filling in `input:` fields through the Home Assistant UI.
The file declares `blueprint.input` (what users pick), `variables` (computed
values), `triggers`, `conditions` and `actions`.

Consequences that are easy to miss:

- **The file in this repo is not what runs.** Importing copies it to
  `config/blueprints/automation/<user>/load_shedding.yaml` on the user's Home
  Assistant. Editing here changes nothing until they re-import. When debugging,
  confirm which version their install actually has.
- **A user's automation stores only the input values**, referencing the
  blueprint by path. So input keys are a public API: renaming one, or changing
  its selector type, breaks existing automations.
- **The import badge in `README.md` points at `main`**, not at a tag. Whatever
  is on `main` is what new users get, released or not.
- **It cannot be tested here.** Templates render only inside Home Assistant.
  Local checks catch YAML and Jinja mistakes; everything else is verified by the
  maintainer running it and exporting a trace.

## What it controls, and why that matters

Electrical load shedding: one automation for the whole house, keeping total
consumption under a capacity limit by disabling loads in priority order and
releasing them when there is room again. The decision tables in `README.md`
(Advanced Documentation) are the spec; keep the code and the tables in step.

- **The safety output is shedding.** It exists so the house does not draw more
  than the supply can give. A run that refuses to act is safe; a run that
  refuses to *shed* is not. When choosing what to do on bad input, stopping
  restoration is nearly always preferable to stopping shedding.
- **Overprovisioning is the premise, not a bug.** The sum of the managed loads
  is meant to exceed the supply. Anything that reserves capacity "just in case"
  — such as budgeting a load's full rated power against the restoration margin —
  quietly prevents loads from ever coming back. Restoration should use the
  headroom that exists, up to the point where shedding would intervene.
- **Shedding is preventive.** An idle load is shed too, because one left free to
  start would blow the budget the moment it does. It cannot count against the
  deficit, though: only power actually being drawn can be freed.
- **Commands only ever go to local helpers.** The blueprint writes to the
  `input_boolean` off overrides and stamps one `input_datetime`. It never
  commands a `climate` or `switch` entity. Whatever *honours* an override —
  typically another automation driving the actual load — is what talks to a
  cloud-backed integration, and that is where rate limits bite. Keep it that
  way: a change that commands the load directly puts this file on the wrong
  side of somebody's API quota.
- **A steady state must send nothing.** Both plans filter on the override's
  current state, so an override that is already correct is never re-commanded,
  and a run with an empty plan matches no branch and does not even stamp the
  tracker.

## How the automation actually works

Stateless reconcile. Every run recomputes, from current entity states, which
overrides should be on and which should be off, and commands only the ones that
differ. A missed run is harmless because the next one recomputes from scratch.

There is no list of shed loads anywhere. **The off override booleans are the
state**: override on means shed. The `input_datetime` helper holds one thing,
the timestamp of the last shed or restore, and that is what enforces the minimum
shed duration.

- `load_status` renders once per run and is what both plans are built from: per
  load, whether it is drawing power, whether it is already shed, its power, and
  whether its override is readable.
- `shed_plan` and `restore_plan` are each computed in a **single template**, then
  handed to a `repeat:`. They must stay that way — see the scoping note below.
- The two branches live in a `choose:` and are mutually exclusive by
  construction, since the restoration threshold is below the shedding one.
- Targets are computed from current state, not from what triggered the run. The
  trigger ids exist for readable traces and nothing reads them.
- Triggers are event-driven only: Home Assistant start, the meter, the capacity
  sensor. There is no periodic tick, so an override flipped by hand is picked up
  on the next meter update.
- The off overrides cannot be triggers: they live inside the `managed_loads`
  object list, and a state trigger needs entity ids.
- `mode: single`. An overlapping run is dropped and logs `Already running`.
  `max_exceeded: silent` is not set so that the drops stay visible.

## Syntax: track current Home Assistant

Use the current forms, not the legacy ones that still happen to work.

- `triggers:` / `conditions:` / `actions:`, not the singular keys
- `action:` for service calls, not `service:`
- `trigger: <platform>` inside a trigger, not `platform:`
- Template condition shorthand: `- "{{ ... }}"` in a `conditions:` list
- Purpose-built template functions over hand-rolled equivalents:
  `has_value(entity)` rather than comparing `states(entity)` against a list of
  `unavailable` / `unknown`
- Keep `blueprint.source_url`; it is what lets users update the blueprint in
  place after import

When a current form raises the minimum version, declare it rather than leaving
it implied:

```yaml
blueprint:
  homeassistant:
    min_version: "2025.7.0"
```

2025.7.0 is what the `object` selector's `fields:` and `label_field:` need —
both are absent from the 2025.6.0 tag and present in 2025.7.0. The plural
trigger keys only need 2024.10. If you use a selector option or template
function added later, raise `min_version` to match: an undeclared requirement
fails at import with a confusing error instead of a clear one.

## Selectors and inputs

- **Changing an input's selector type is breaking.** Home Assistant *merges* the
  stored value into the new shape rather than replacing it, so the automation
  can refuse to load. Document it, bump the major version, and quote the exact
  error users will see.
- **Selector `filter:` is a list of OR-ed filters.** `domain` and `device_class`
  meant to apply together go in the *same* list item.
- The off override is restricted to `input_boolean` on purpose. It is a flag
  another automation reads, not the load's own switch.
- Fields inside an `object` selector are all optional as far as Home Assistant
  is concerned. Read them with `.get()`, and let the guard decide whether a
  missing one is fatal.

## Template facts worth not rediscovering

- **A script action must be a dict.** `cv.script_action` raises
  `expected dictionary`, so a bare `- "{{ ... }}"` is *not* a valid action, even
  though it is valid inside a `conditions:` list. As an action, write
  `- condition: "{{ ... }}"`, which also takes an `alias:`.
- **`variables:` inside a `repeat:` are scoped to one iteration.** A running
  total reset on every pass, and a flag set in the loop never escaped it. Build
  the whole plan in one template before acting.
- A failing `condition` inside a `repeat` sequence ends the whole repeat, not
  the iteration. Use `if:`/`then:` to skip one item.
- **`continue_on_error: true` does not rescue the other entities of a grouped
  service call.** It only stops the error reaching the next step. One call per
  entity is what isolates a failure.
- **A state trigger with no `to`/`from`/`not_to`/`not_from` is `match_all`**, and
  fires on attribute-only changes too. Adding `not_to: [unavailable, unknown]`
  is what makes Home Assistant skip them.
- **`input_datetime`'s `timestamp` attribute is only an epoch when the helper
  has both date and time.** Date-only gives midnight's epoch, time-only gives
  seconds since midnight. `has_date` and `has_time` are readable state
  attributes, so a misshapen helper can be detected rather than silently
  producing nonsense.
- `!input` is not visible to templates. Every input a template uses must be
  mapped in `variables:` first. An unmapped one renders empty with only a
  `'x' is undefined` warning.
- `variables:` render in order; each is available to the next, with
  `literal_eval` applied — a template rendering `[1, 2]` yields a real list.
- Top-level `variables:` render before any action. That is fine here because
  nothing is written before they are read; if that changes, move the affected
  ones into an action-level `variables:` step.
- `selectattr` / `rejectattr` / `sort(attribute=...)` work on dicts, because
  Jinja falls back to `getitem`. Avoid keys that collide with dict methods
  (`items`, `values`, `keys`, `get`, `pop`, `copy`), which would resolve to the
  method instead.
- `map(attribute='x', default='')` is available and is the clean way to read a
  field that may be missing from an object-selector entry.
- `trigger` is always defined. A manual "Run actions" supplies
  `{'platform': None}` — defined, but with no `id`.
- Manual runs skip top-level `conditions:` entirely, which is why the
  configuration guard is the first *action*.
- `default()` replaces *undefined*, not `None`. `state_attr()` returns `None`
  for a missing attribute, so use `or []` / an explicit comparison.
- Jinja supports `**` unpacking, so `timedelta(**duration_input)` is fine.

## Don't invent fallbacks

A default that lets the automation keep acting on invented data is worse than
stopping. `current_power` reading `0` from an unavailable meter looks exactly
like an empty house and triggers a full restoration, which is why the guard
checks `has_value()` before anything else. `| float(0)` on a per-load power
sensor is fine: the worst case is one redundant shed.

Likewise, don't add defensive handling for a failure mode you have only
imagined. Fix what is observed.

## Comments

Comments explain **why**. If a comment restates the line below it, delete it.
Don't repeat the same comment on two similar branches, and don't let a rationale
grow into an essay — the long form belongs in `README.md`.

```yaml
# bad
# Sort candidates by index descending
shed_plan: >-

# good
# Idle loads are shed too: one left free to start would blow the budget the
# moment it does.
shed_plan: >-
```

## Verify before committing

Templates are not type-checked and a broken blueprint fails at runtime,
unattended, on the coldest evening of the year. Check what can be checked:

```bash
# YAML parses, with the Home Assistant tags registered
python3 -c "
import yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!input', lambda l,n: {'!input': l.construct_scalar(n)})
yaml.load(open('load_shedding.yaml'), L); print('OK')"
```

For anything non-trivial, render the templates locally: load the YAML, walk
`variables:` in order applying `literal_eval` as Home Assistant does, and mock
`states`, `state_attr`, `is_state`, `has_value` and `now` over a dict of fake
entity states. That catches Jinja errors and, more usefully, lets you assert the
arithmetic against a table of scenarios.

Check boundaries, not just the middle: one watt either side of both thresholds,
a budget that fits exactly, a load rated above the shedding threshold, an
unavailable meter, a tracker helper that is missing, date-only and time-only.

State clearly what was verified and what was not. `has_value()`, `is_state()`
and friends only exist inside Home Assistant; local tests mock them, and that is
not the same as the code working.

## Debugging: traces are the source of truth

Ask for a trace export (Settings → Automations → Traces → download) rather than
reasoning from the config. The trace carries **Changed Variables**, with every
rendered value and type — `load_status`, `shed_plan` and `restore_plan` are the
three worth reading first — plus the result of every condition and which branch
ran.

Read the numbers before proposing a fix, and check where a log line comes from
before attributing it to the blueprint.

When a claim about Home Assistant's own behaviour decides the design, check the
source rather than guessing:
`raw.githubusercontent.com/home-assistant/core/dev/homeassistant/...`, and
compare release tags when a feature's minimum version is in question.

## Docs must match the code

Three surfaces drift, in rising order of how often they are missed:

1. `README.md`, including the decision tables
2. `CHANGELOG.md`
3. the blueprint's own `description:` and each input's `description:` — what
   users read in the UI, and the ones that go stale

When behaviour changes, update all three in the same commit, including the
tradeoffs. Write the README as a description of what the blueprint does, never
as a diff against what it used to do.

## Versioning and releases

Semantic versioning. A change requiring users to touch their existing
automation is a major bump, whatever its size. Record breaking changes under a
`### Breaking` heading at the top of the release, with the remedy and the exact
error.

Version lives in `README.md` and `CHANGELOG.md` only — blueprints have no
version field. Tag annotated, as `vX.Y.Z`. Current release: `v1.1.0`.

## Commits

Commit messages explain the reasoning, not the diff: what was wrong, why the
chosen fix, and what tradeoff it accepts. Commits are gpg-signed; if signing
times out, retry rather than disabling it.
