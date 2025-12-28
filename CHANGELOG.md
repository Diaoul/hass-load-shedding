# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial release of Load Shedding Blueprint
- Real-time power monitoring from total consumption sensor (e.g., Linky meter)
- Proactive load shedding with configurable safety margins (default 90%)
- Order-based priority system (top = highest priority, bottom = lowest)
- Priority-based restoration (highest priority first)
- Anti-flapping protection with hysteresis and time delays
- Structured load configuration using object selector (HA 2025.7+)
- Manual override to globally disable load shedding
- State persistence across HA restarts
- Configuration validation (duplicate names/switches)
- Automatic cleanup of orphaned loads from tracker
- MIT License
- CHANGELOG.md for tracking version history

### Requirements
- Home Assistant 2025.7.0 or later (for object selector support)
