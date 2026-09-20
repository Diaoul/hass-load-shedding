# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-09-20

Re-import the blueprint to pick this up; existing automations keep their
settings and need no edits. One thing is worth checking: your tracking helper
must have **both date and time** enabled, or restoration will refuse to run (see
below).

### Changed

- **The restoration budget now runs up to the shedding threshold instead of the
  restoration threshold.** The restoration margin decides when it is calm enough
  to start giving power back; it should not also decide how much comes back.
  Using it for both left any load rated between the two margins permanently
  shed, however empty the house was. A load rated above the shedding threshold
  still stays shed, because releasing it would only shed it again on the next
  run.
- Restoration releases a shed load that is still drawing power, at no cost to
  the budget. Its consumption is already in the meter reading, so charging it
  its rated power only stranded the override for as long as the load ran.
- Restoration skips a load that does not fit and keeps looking at the smaller
  ones behind it, instead of stopping at the first one that does not fit.
- Shedding only counts loads that are actually drawing power towards the
  deficit. Idle loads are still shed preventively, but no longer pretend to free
  up power that was never being used, which made the automation under-shed.
- Restoration refuses to run while the tracking helper has only a date or only a
  time. Such a helper reports `timestamp` as seconds since midnight rather than
  an epoch, which reads as "ages ago" on every run and silently disabled the
  minimum shed duration. Shedding still runs: a tripped breaker is worse than a
  load staying off.
- The capacity sensor is now a trigger, so a dynamic limit that drops is acted
  on immediately rather than at the next meter update.
- Both state triggers carry `not_to: [unavailable, unknown]`, which is what
  makes Home Assistant ignore attribute-only updates. A state trigger with no
  `to`/`from`/`not_to`/`not_from` matches every change, so each attribute
  refresh on the meter re-ran the automation.
- An unavailable meter or capacity sensor, a capacity of 0, or a load missing a
  required field now stops the run outright. A broken entry makes the shed
  arithmetic wrong for every load, so acting on part of the list is worse than
  not acting.
- Updated to current Home Assistant syntax: `triggers:`/`actions:`, `action:`
  instead of `service:`, `trigger: <platform>`, shorthand template conditions,
  and `has_value()` in place of comparisons against `unavailable`/`unknown`.
- Declared `min_version: "2025.7.0"` and restored `blueprint.source_url`, so the
  blueprint can be updated in place after import, and an install that is too old
  fails at import with a clear message instead of a confusing one.

### Fixed

- Shed and restore plans were accumulated with `variables:` steps inside a
  `repeat:`, which Home Assistant scopes to a single iteration. The running
  deficit reset on every load and the "did we act" flag never escaped the loop,
  so the wrong loads were shed and the tracking helper was stamped
  inconsistently. Both plans are now computed in one template before anything is
  sent.
- An unavailable meter read as 0 W, which looks like an empty house and
  triggered a full restoration.
- The minimum shed duration is read from the helper's `timestamp` attribute
  instead of parsing its state, removing a timezone assumption. A missing helper
  now reads as "long ago" instead of raising.
- The configuration check emitted several values and only returned the right
  answer by accident. It is a single boolean expression now.
- A load configured without an off override, or with an invalid maximum power,
  was skipped while still counting towards the shed arithmetic.

### Removed

- The 1-minute `time_pattern` failsafe. Triggers are Home Assistant start, the
  meter and the capacity sensor. An off override flipped by hand is picked up on
  the next meter update.
- Dead code: shedding worked out whether each load was on, then ignored the
  answer.

## [1.0.0] - 2025-12-30

### Added

- Initial release of Load Shedding Blueprint
- Real-time power monitoring from meter sensors (Linky, Shelly, etc.)
- Order-based priority system (top = highest priority, bottom = lowest)
- Configurable safety and restoration margins with hysteresis
- Anti-flapping protection with minimum shed duration (default: 5 minutes)
- Structured load configuration using blueprint object arrays
- Support for climate entities with multiple detection methods:
  - hvac_action detection (accurate, recommended)
  - sensor_based detection (fallback for thermostats without hvac_action)
- Support for switch entities
- Detection entity field for monitoring load power consumption state
- Off override switches for stateful load control (ON = disabled, OFF = enabled)
- Optional power sensor field for real-time consumption monitoring
- Maximum power field for restoration budget calculations and sensor fallback
- Dynamic capacity support via template sensors (solar + grid, time-of-use, battery state)
- Preventive shedding (sheds loads even when OFF to prevent turn-on during constraints)
- Priority-based shedding: lowest priority, highest power loads shed first
- Priority-based restoration: highest priority, lowest power loads restored first
- Budget-aware restoration ensuring loads fit within available power capacity
- DateTime helper for tracking last action (minimum shed duration enforcement)
- Instant decision-making based on power sensor updates
- Periodic failsafe check (1-minute intervals)
- Comprehensive validation (duplicate detection entities, margin validation, mandatory off override)

### Requirements
- Home Assistant 2025.7.0 or later (for object selector support)
