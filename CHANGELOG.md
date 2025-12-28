# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial release of Load Shedding Blueprint
- Real-time power monitoring from total consumption sensor (e.g., Linky meter)
- Proactive load shedding with configurable safety margins (default 90%)
- 5-level priority system (Critical, High, Medium, Low, Very Low)
- Priority-based restoration (highest priority first)
- Anti-flapping protection with hysteresis and time delays
- Flexible configuration via input helpers
- Manual overrides (global disable + per-load exemptions)
- State persistence across HA restarts
- MIT License
- CHANGELOG.md for tracking version history
