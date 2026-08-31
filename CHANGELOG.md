# Changelog

All notable changes to this dataset are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versions here track
the *dataset publication*, not the day-to-day follower readings — those land
continuously and are recorded in the git history of
`datasets/party-channels/observations/`.

Cut a new version (and a matching GitHub Release / Zenodo version) when the
schema changes, coverage expands meaningfully, or you want a fresh citeable
snapshot. See the "Releasing a new version" section of the README.

## [Unreleased]

## [0.1.0] - 2026-08-31

### Added

- First public release of the dataset: the party registry
  (`datasets/party-channels/parties/*.json`, including supranational
  `_europe.json`), observed channel profiles (`profiles.json`), mirrored channel
  avatars (`avatars/`), the append-only follower-reading log
  (`observations/<YYYY-MM>.csv`), and the ISO 3166 geography tables
  (`datasets/places/`).
- `datapackage.json` (Frictionless) describing the observation columns and the
  JSON resources, `LICENSE` (CC BY 4.0), `CITATION.cff` and `.zenodo.json`.
