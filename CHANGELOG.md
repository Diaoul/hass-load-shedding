# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
