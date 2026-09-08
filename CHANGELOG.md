# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [v1.0.0] - 2026-09-08 - Initial release

### Added

- Logbook entry under Tips, plus an optional ticker line, the first time a black marketeer is seen on
  a station the player already knows. The entry opens the map on that station.
- Lead planting: while the player is at such a station and its marketeer is still locked, the mod
  asks the vanilla signal leak manager to run a pass on it, so the unlock lead is actually placed.
  Per-station cooldown, attempt limit and leak cap.
- Black Market Leads menu through SirNukes Simple Menu API, listing the marketeers on stations the
  player knows with sector, owner and status, reachable from Extension Options, the `/leads` chat
  command, the station right-click menu and a bindable hotkey.
- Eight settings in Extension Options, and the same values overridable at runtime through globals.

### Notes

- Nothing is ever reported, listed or acted on for a station the game does not consider known to the
  player, so exploration is unchanged.
- Stations without docking permission are never given a lead, because the vanilla mission setup
  rejects them and destroys the leak it just created.

[v1.0.0]: https://github.com/drjele/x4-black-market-leads/releases/tag/v1.0.0
