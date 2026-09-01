# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.0.0] - 2026-09-01

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose)
v1.2.0.

### Changed

- **Joomla was deployed from the floating `joomla` (latest) tag — now
  pinned to 6.1.3** by `tag@sha256:digest`, alongside PostgreSQL 16 and
  Traefik 3.7 (3.2's Docker client cannot talk to Docker Engine 29), all
  as `x-images` interpolation defaults. `git pull` delivers the tested
  combination; `.env` carries only secrets and deliberate overrides.
  ❗ If your deployment installed as Joomla 4/5 from the floating tag,
  update through the Joomla admin UI first and only then adopt the pin —
  see the release notes.

### Security

- **Credentials untracked from git.** The tracked `.env` carried
  generated-looking database and admin passwords — rotate them if
  reused.

### Fixed

- Backup-loop variables are `$$`-escaped so the container shell resolves
  them at runtime; shellcheck findings in both restore scripts.

### Added

- **Deployment Verification workflow**: shellcheck + actionlint; Trivy
  scans of all three pinned images; weekly `check-pin-freshness` (digest
  drift + Joomla Docker Hub tag lag + Traefik release lag); and a
  deploy-and-test job that boots the stack, lets the unattended installer
  run, and requires the site to answer through Traefik.

[Unreleased]: https://github.com/heyvaldemar/joomla-traefik-letsencrypt-docker-compose/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/heyvaldemar/joomla-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
