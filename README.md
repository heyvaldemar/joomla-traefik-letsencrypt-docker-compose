# Joomla + Traefik + Let's Encrypt — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/joomla-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/joomla-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys **Joomla** behind **Traefik** with automatic **Let's Encrypt TLS**, backed by **PostgreSQL 16**, with an unattended first-start installation (site and admin account from `.env`), scheduled **backups** (database + site files) and companion **restore scripts**.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-joomla-using-docker-compose/](https://www.heyvaldemar.com/install-joomla-using-docker-compose/).

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/joomla-traefik-letsencrypt-docker-compose
cd joomla-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create joomla-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: JOOMLA_DB_PASSWORD, JOOMLA_ADMIN_PASSWORD (12+ chars),
#   JOOMLA_ADMIN_EMAIL, JOOMLA_HOSTNAME, TRAEFIK_HOSTNAME,
#   TRAEFIK_ACME_EMAIL, TRAEFIK_BASIC_AUTH.

# 4. Deploy
docker compose -f joomla-traefik-letsencrypt-docker-compose.yml -p joomla up -d
```

The image installs Joomla unattended on first start — within a couple of minutes `https://${JOOMLA_HOSTNAME}` serves the site and `/administrator` accepts the admin credentials from `.env`.

### What success looks like

```bash
docker compose -f joomla-traefik-letsencrypt-docker-compose.yml -p joomla ps
curl -fskL -o /dev/null -w "%{http_code}\n" "https://${JOOMLA_HOSTNAME}/"
docker compose -p joomla logs traefik | grep -i "adding certificate"
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **Installer page instead of the site.** The unattended install needs all JOOMLA_* variables — an empty admin password (or one under 12 characters) leaves the wizard waiting.
- **`docker compose up` fails with `set in .env`.** A required variable is empty; the error names it.
- **Networks not found.** Step 2 was skipped.

## Supply chain trust

Three images — [`traefik`](https://hub.docker.com/_/traefik), [`joomla`](https://hub.docker.com/_/joomla), [`postgres`](https://hub.docker.com/_/postgres), all Docker Hub official — pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. Earlier revisions deployed from the floating `joomla` tag, which meant every fresh deploy could get a different major version; the pin ends that. An `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The daily `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned Joomla and Traefik versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Strong secrets** — both passwords at 24+ random characters; regenerate the Traefik dashboard hash.
- [ ] **Host-mount the backup volumes** for disaster recovery.
- [ ] **Verify Let's Encrypt cert issuance** in the Traefik logs on first start.
- [ ] **Keep Joomla and extensions updated** — the image pins the core; extensions update from the admin UI.
- [ ] **Back up before core upgrades** — restore is the rollback.

## Backups and restore

The `backups` container runs a `pg_dump | gzip` + `tar.gz`-of-site-files → prune → sleep loop (defaults: 30-minute warm-up, 24-hour interval, 7-day retention). Restore with the interactive scripts (`chmod +x *.sh` once): `./joomla-restore-database.sh`, then `./joomla-restore-application-data.sh`.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults — the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> --format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. Infrastructure containers (the reverse proxy, databases, caches, backups) run with `cap_drop: [ALL]` and add back only what their entrypoints need: `NET_BIND_SERVICE` for Traefik to bind :80/:443, `CHOWN`/`SETUID`/`SETGID` (and friends) for database images to own their data directory and drop to their service user. Application containers keep the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/joomla-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: shellcheck + actionlint, Trivy scans of all three pinned images, the weekly freshness check, and a deploy-and-test job that boots the stack with ephemeral credentials, lets the unattended installer run, and requires the site to answer through Traefik.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the HTTPS smoke. The scenario that matters most is the restore roundtrip: insert a marker row, restore the earliest backup, assert the marker is gone — a backup that cannot be restored fails the build. Run it yourself against a running deployment with short intervals in `.env` (`BACKUP_INIT_SLEEP=15s`, `BACKUP_INTERVAL=60s`):

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

It stops the database container briefly to prove failure detection — run it on a staging copy, not on production.

## Security Notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-09-01) shipped a tracked `.env` with generated-looking database and admin passwords. Rotate them if your deployment reused them.
- PostgreSQL listens only on the internal network.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
