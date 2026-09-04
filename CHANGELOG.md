# Changelog

Notable changes to bunnyarchives, in the format of
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/). This project
follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html); while
the major version is `0`, breaking changes arrive in minor bumps.

Entries land under `[Unreleased]` in the same PR as the change they
record; the release PR renames that section to the version it ships and
starts a fresh one.

## [Unreleased]

### Added

- `docs/requirements/incoming-requirements.md`: the running requirements
  doc, imported as captured and then updated with the first round of
  clarifications (tags-first access model, capacity/waitlists, refunds,
  character approval and advancement, data deletion, offline scope).

### Changed

- `docs/requirements/incoming-requirements.md`: now holds the content of
  the former `incoming-requirements.final.md` — the last checkpoint
  (`20260901T045556`), which resolved the registry defaults, settled P22
  as item print formats, and made class and tier optional attributes. Its
  disaster-recovery header, which asked to be deleted once merged back
  into the running doc, has been removed.

### Removed

- The ten timestamped `incoming-requirements` snapshots. They were
  disaster-recovery checkpoints taken while the doc was being elicited;
  git history preserves them, and leaving them in the tree made a public
  reader's first encounter with `docs/requirements/` a directory of
  near-duplicates.
