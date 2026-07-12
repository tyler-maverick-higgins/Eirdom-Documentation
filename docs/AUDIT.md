# Eirdom Docker Environment — Audit

**Scope:** 17 service stacks under `docker/`, the shared root `.env.example`, and the Traefik static + dynamic configuration.
**Date:** 2026-06-28

## How to read this

Findings are ordered by severity. Each has a **location**, the **problem**, and the **fix**. Where the conclusion depends on a version-specific fact, the supporting source is named so you can verify it rather than take my word.

Two caveats on what I *can* and *can't* see:

- I can't test the running system. Several findings are "this config, as written, won't do X." If X currently works for you, you have a mechanism not in the archive (an `envsubst` wrapper, hardcoded values, etc.) — I call those out explicitly.
- The Cloudflare Tunnel's public-hostname mappings live in the Cloudflare dashboard, not in the archive, so I can't see which services (if any) you expose to the internet. One finding depends on that.

The headline: **this is a well-built environment.** Network segmentation, per-service internal networks, `no-new-privileges`, read-only Docker sockets, healthcheck-gated `depends_on`, and consistent SSO chaining are all done properly, and your env-var references are fully consistent (every `${VAR}` is defined). The issues below are concentrated in Traefik (version-3 breakage + a substitution assumption) and a coherence gap that opened when the arr-stack moved to ADR-043. Fix those and the rest is solid.

---

## CRITICAL

### C1 — Traefik does not substitute `${VAR}` in `traefik.yml` or `routers.yml`
**Location:** `traefik/traefik.yml` (ACME email, TLS domains), `traefik/dynamic/routers.yml` (all `Host()` rules)

`traefik.yml` sets the ACME email to `${TRAEFIK_ACME_EMAIL}` and the wildcard cert domains to `${ROOT_DOMAIN}` / `*.${ROOT_DOMAIN}`. `routers.yml` builds `Host(\`proxmox.${ROOT_DOMAIN}\`)` and similar for the dashboard, Wazuh, Security Onion, and Home Assistant.

Traefik does **not** expand shell-style `${VAR}` placeholders in either its static config file or its file-provider dynamic configs. Setting `ROOT_DOMAIN`/`TRAEFIK_ACME_EMAIL` in the compose `environment:` does not help — Traefik doesn't read its own environment to rewrite file contents. A user who put `email: ${ACME_EMAIL}` in `traefik.yml` got back, verbatim from Let's Encrypt, `"${ACME_EMAIL}" is not a valid e-mail address` and had to hardcode it (Traefik community forum, "Environment variable syntax in traefik.yml"; confirmed still true in v3, forum thread "Environment variables in static config dynamic config not working, traefik v3").

Consequence as written: ACME registration uses a literal invalid email and literal domains → **wildcard cert issuance fails → no service gets a valid certificate**. The static routers (Proxmox/Wazuh/SO/HA/dashboard) also never match, because their `Host()` rule is the literal string `proxmox.${ROOT_DOMAIN}`.

Why your **Docker-label** routers (Radarr, Jellyfin, Authentik, etc.) are unaffected: Docker Compose substitutes `${ROOT_DOMAIN}` in the compose file *before* Traefik ever sees the label. That substitution is Compose's, not Traefik's. So labels work; mounted YAML files don't.

**If your certificates currently work**, you already hardcode these or run an `envsubst` step that isn't in this archive — in which case this applies only to anyone deploying the archive as-is. If you're not certain, check `docker logs traefik` for ACME errors and `acme.json` for a real cert.

**Fix:** Hardcode the stable values in the two files. `ROOT_DOMAIN` is `eirdom.homes` and rarely changes; the ACME email never changes. (Static config has no templating at all; the dynamic file provider does support Go templating — `{{ env "ROOT_DOMAIN" }}` — if you'd rather keep it variable, but hardcoding is simpler for values this stable.) Corrected `traefik.yml` and `routers.yml` are included with this audit.

### C2 — Root `.env` still encodes the pre-ADR-043 storage model and contradicts the arr-stack you just updated
**Location:** `root .env.example`

The arr-stack now implements the three-tier layout (ADR-042/043): configs on the OS SSD (`DOCKER_DATA_PATH`), in-progress downloads on the 250 GB SSD (`DOWNLOADS_CACHE_PATH`), completed + libraries on the 3 TB Red (`MEDIA_PATH`). The root `.env.example` was not updated to match. It still says:

```
DOCKER_DATA_PATH=/media/arr/config
# CRITICAL: Both paths point to /media/arr ... They must be on the same
# filesystem for hardlinks to work. DO NOT split these across different drives.
```

That comment is now the exact opposite of the decision you adopted, and `DOCKER_DATA_PATH=/media/arr/config` keeps every config on the 3 TB — so Tier 1 is never actually implemented. Deploy as-is and ADR-043 is a no-op.

**Important blast radius:** `DOCKER_DATA_PATH` is shared by *every* stack (Authentik, NetBox, Jellyfin, Paperless, Immich, …), not just the arr-stack. Repointing it to the OS SSD migrates **all** config directories. That is the correct Tier-1 outcome, but it's a coordinated migration (stop stacks → `rsync /media/arr/config/ → /opt/eirdom/appdata/` → update the var → restart), not a per-stack change.

**Fix:** Update the root `.env.example` to point `DOCKER_DATA_PATH` at the OS SSD, delete the "same filesystem / DO NOT split" warning, and note that `DOWNLOADS_CACHE_PATH` lives in `arr-stack/.env`. Corrected file included. `MEDIA_PATH=/media/arr` stays — it already *is* the 3 TB Red.

### C3 — `security-headers` uses an option Traefik v3 removed, and that middleware is in every chain
**Location:** `traefik/dynamic/middlewares.yml` line 38 (`sslRedirect: true`)

Traefik v3 removed `sslRedirect`, `sslTemporaryRedirect`, `sslHost`, `sslForceHost`, and `featurePolicy` from the Headers middleware (Traefik "v2-to-v3 migration details"). Your `security-headers` middleware still sets `sslRedirect: true`, and `security-headers` is a member of **all three** chains (`chain-standard`, `chain-public`, `chain-admin`). If v3 rejects the middleware over the unknown field, every router that references any chain loses its middleware — i.e. effectively the whole proxy.

`sslRedirect` is also redundant here: your `web` entrypoint already does a global permanent HTTP→HTTPS redirect.

**Fix:** Delete the `sslRedirect: true` line. Corrected `middlewares.yml` included. (At minimum, watch `docker logs traefik` for "field not found" / middleware errors after any Traefik restart.)

---

## HIGH

### H1 — `ipWhiteList` was renamed to `ipAllowList` in v3 (two places)
**Location:** `traefik/dynamic/middlewares.yml` (`ipwhitelist-corporate` → `ipWhiteList:`), and `webserver/docker-compose.yml` line 108 (Adminer label `...ipwhitelist.sourcerange`)

Traefik v3 renamed the middleware; the old `ipWhiteList` key is not the v3 spelling and fails to load (Traefik v3 migration notes; Helm-chart issue #911: "When using old type ipWhitelist controller fails"). The dynamic one feeds `chain-admin` (Traefik dashboard, Proxmox, Wazuh, Security Onion, Jellystat). The Adminer label is your **database** admin UI's only network restriction.

Failure mode matters: if the middleware doesn't load, the IP restriction may simply not apply (fail-open) — meaning Adminer and the admin UIs lose their VLAN-10 gate. Combine that with C1/the tunnel question (T-low) and an internal-only DB console could be reachable more widely than intended.

**Fix:** Rename both to `ipAllowList` / `ipallowlist` (config keys are otherwise identical). Corrected `middlewares.yml` included; the Adminer label change is a one-liner noted in the checklist.

Side note on IP allowlists behind a tunnel: Traefik sees the *source* IP. LAN clients hitting Traefik directly present their real `10.1.10.x` (allowlist works). Anything arriving via `cloudflared` presents the Docker gateway IP, so the `10.1.10.0/24` rule will never match tunnel traffic — fine as a fail-closed default, but it means these UIs are LAN-only by design. Good, as long as the middleware actually loads.

### H2 — Immich runs the deprecated pgvecto.rs extension on an auto-updating `:release` tag
**Location:** `immich/docker-compose.yml`

`immich-db` is `tensorchord/pgvecto-rs:pg16-v0.2.0`, while `immich-server` and `immich-machine-learning` are `:release` (rolling latest). Immich migrated off pgvecto.rs to **VectorChord**; pgvecto.rs is deprecated and the server logs `DEPRECATION WARNING: The pgvecto.rs extension is deprecated and support for it will be removed very soon` (Immich upgrade docs; live reports Feb 2026). Immich also explicitly recommends **pinning** a version because they ship breaking DB migrations.

The danger is the combination: `:release` means the next image pull can be the version that drops pgvecto.rs, at which point migrations fail and the server won't start (`Error: Invalid upgrade path`). You're one `docker compose pull` away from a broken Immich, with no version pin to roll back to.

**Fix (a guided DB migration — back up first):**
1. Pin all three Immich images to a specific current version instead of `:release`.
2. Switch the DB image to `ghcr.io/immich-app/postgres:16-vectorchord0.3.0-pgvectors0.3.0` (it bundles VectorChord *and* pgvecto.rs/pgvector so it can read your existing data).
3. Change the DB `command` `shared_preload_libraries` from `vectors.so` to `vchord.so, vectors.so`.
4. Start Immich and let it reindex (can take minutes).
This one needs your hands on a DB backup, so I've left it as a recommendation rather than editing the file. Happy to produce the exact compose diff when you're ready.

Also non-blocking but worth folding in: your Immich uses the **old** split-volume layout (`library/upload/thumbs/profile/video` as five separate host mounts, with `UPLOAD_LOCATION` set to a container path). Current Immich uses a single `${UPLOAD_LOCATION}:/usr/src/app/upload` host mount. Consolidating avoids surprises with Immich's storage-migration features.

### H3 — ntfy is behind Authentik ForwardAuth, which blocks the clients ntfy exists to serve
**Location:** `ntfy/docker-compose.yml` (`middlewares=chain-standard`)

`chain-standard` applies Authentik ForwardAuth to every request to `ntfy.eirdom.homes`. But ntfy's whole job is to receive token-authenticated publishes (Uptime Kuma, Wazuh, `backup.sh`) and serve the mobile app — none of which can complete Authentik's browser SSO flow. The mobile app and any `https://ntfy.eirdom.homes` publisher will get bounced to the Authentik login page and fail. This is the same reason Jellyfin/Jellyseerr/Immich correctly use `chain-public`; ntfy was missed. Your own `backup.sh` notifications (`NTFY_TOKEN`) would fail if they post through Traefik.

**Fix:** Switch ntfy to `chain-public` (ntfy enforces its own `deny-all` + token auth, so it's not unprotected). Corrected `ntfy/docker-compose.yml` included.

### H4 — WordPress uses the FPM image but Traefik routes to it as HTTP
**Location:** `webserver/docker-compose.yml` (`image: wordpress:php8.3-fpm-alpine`, `loadbalancer.server.port=80`)

The `-fpm` images run PHP-FPM speaking FastCGI on port 9000 and contain **no HTTP server**. Traefik speaks HTTP to the backend, so routing to port 80 on this container yields a connection refused / 502 — unless there's an nginx/Apache front-end not present in this stack.

**Fix:** Use `wordpress:php8.3-apache` (serves HTTP on 80, drop-in for your label), or add an nginx sidecar that proxies FastCGI to the FPM container. The apache variant is the low-friction choice. Left for you to pick — tell me which and I'll wire it up.

---

## MEDIUM

### M1 — `env_file: ../../.env` injects the entire secret set into every container
**Location:** all stacks

Every service loads the full root `.env`, so `grocy`, `stirling-pdf`, `wordpress`, etc. each get `CF_API_TOKEN`, `AUTHENTIK_SECRET_KEY`, every DB password, and SMTP creds in their environment — readable via `/proc/self/environ` inside that container. A compromise of any one service (Stirling-PDF and WordPress process untrusted input) exposes infrastructure-wide secrets. This undercuts the segmentation you've otherwise built carefully.

**Fix (hardening, not a bug):** Pass only the vars each service needs via explicit `environment:` entries and drop the blanket `env_file: ../../.env` where it isn't required. This is a larger refactor and a judgment call on effort-vs-benefit — flagging it because it's the one place the design fights your security posture. The root `.env` is still useful for genuinely shared values (TZ, PUID/PGID, paths, ROOT_DOMAIN).

### M2 — A populated `.env` is sitting in `eirdom-intelligence/`
**Location:** `eirdom-intelligence/.env`

Unlike every other stack (which ships only `.env.example`), this directory contains a real `.env` with `WEBUI_SECRET_KEY`, `MCPO_API_KEY`, `QDRANT_API_KEY`, etc. The file's own header says "NEVER commit .env." Confirm it's gitignored and was never committed; if it ever landed in git history, rotate those three secrets. (It also has CRLF line endings — Windows-edited — which is cosmetic but worth normalizing.)

### M3 — eirdom-intelligence uses named volumes that your backup script likely doesn't cover
**Location:** `eirdom-intelligence/docker-compose.yml` (`qdrant_storage`, `openwebui_data`)

Every other stack bind-mounts under `${DOCKER_DATA_PATH}`, which is presumably what `backup.sh` walks. These two are Docker named volumes living in `/var/lib/docker/volumes`, so the Qdrant vector store and Open WebUI data (chats, RAG config) are probably outside the backup. Either convert them to bind mounts under the data path, or add the volumes explicitly to the backup job.

### M4 — Paperless remote-user SSO may not be enabled
**Location:** `paperless/docker-compose.yml`

You set `PAPERLESS_HTTP_REMOTE_USER_HEADER_NAME` but I don't see `PAPERLESS_ENABLE_HTTP_REMOTE_USER=true`. Without the enable flag, Paperless typically ignores the header and falls back to its own login, so the Authentik auto-login won't actually happen. Worth verifying against the current Paperless env reference. Security note that comes with it: remote-user auth is only safe because Paperless is reachable solely via Traefik — keep it off any directly-reachable port, or the username header could be spoofed.

---

## LOW

- **L1 — Postgres version drift:** `jellystat-db` is `postgres:15-alpine`; every other Postgres is `:16-alpine`. Intentional if Jellystat pins 15, otherwise align.
- **L2 — `delayBeforeCheck: "30"`** in `traefik.yml` ACME: durations want a unit (`"30s"`). Minor; verify cert renewals aren't affected.
- **L3 — Stale comment in root `.env`:** the storage block describes container paths as `/data/movies`, `/data/tv`, etc., but the arr-stack uses `/data/radarr`, `/data/sonarr`. Cosmetic, but it's the kind of drift worth killing while you're in there (handled in the corrected file).
- **L4 — Cloudflare Tunnel mappings (can't see them):** confirm which hostnames the tunnel publishes. Anything internal-only (Adminer, Traefik dashboard, Proxmox, NetBox) should **not** be in the tunnel's public hostname list — its IP-allowlist gate (H1) won't protect it from tunnel traffic, and the tunnel bypasses your LAN firewall entirely.

---

## What's done well (so you don't change it)

- **Network segmentation is correct:** each DB/Redis sits on an `internal: true` bridge unreachable from Traefik or other stacks; only the app tier joins `proxy`. This is the right pattern and it's applied consistently.
- **Authentik-without-Redis is correct.** Your comment ("PostgreSQL-based since 2025.10") matches reality — Authentik removed the Redis requirement in 2025.10. Don't add Redis back.
- **arr-stack hardlinks are correct:** every *arr mounts `${MEDIA_PATH}:/data` at the root (not per-app subpaths), so completed-downloads and libraries share a filesystem.
- **Socket hygiene:** Traefik and Uptime-Kuma mount the Docker socket read-only; Traefik runs `no-new-privileges`. The Authentik worker needs the socket for outpost management — acceptable.
- **Dependency ordering:** healthcheck-gated `depends_on` on the DB tiers is exactly right.
- **eirdom-intelligence is bound to `127.0.0.1`** for all three ports — good defense for the AI stack; just note it's host-local only (reach it via tunnel/SSH, not the LAN).
- **Env-var consistency is clean:** every `${VAR}` referenced across all compose files is defined in the root or local `.env.example`. No empty-variable surprises.

---

## Managing this environment — and the Portainer question

Short version: **for your specific philosophy, Portainer is not the best fit, and I'd skip it.** Here's the reasoning rather than just the verdict.

Your environment is already managed the "right" way for someone who values documentation-first, deliberate, version-controlled infrastructure: per-service compose files in git, a shared `.env`, ADRs recording decisions, provisioning scripts. That *is* a management system — a declarative, auditable one. The question is what gap a tool would fill.

**Where Portainer fights your model:**
- **Source-of-truth drift.** Portainer's common usage (paste/edit stacks in the web UI) puts state in Portainer's own database, divorced from git. The moment you fix something in the GUI at 11pm, your repo is wrong and you won't know it. That's the precise failure your documentation-first approach exists to prevent. (Portainer *can* do GitOps against a repo, but at that point it's a heavier wrapper around `git pull && docker compose up` than you need.)
- **A new privileged attack surface.** Portainer wants the Docker socket with read/write. Given your security posture — segmented VLANs, read-only sockets elsewhere, Authentik in front of everything — adding a full-control socket consumer is a meaningful target you'd then have to gate behind Authentik + IP allowlist and patch forever.
- **Redundancy.** Its monitoring/log/health views overlap what you already have in Uptime-Kuma (status), Wazuh (logs/security), and the Traefik dashboard (routing). It duplicates more than it adds.

This is your own ADR-003 principle applied: don't add a stateful, privileged component until there's a demonstrated need, and the need here ("see and restart containers") is real but small.

**What actually fills the gaps, in order of leverage:**
1. **A top-level orchestration Makefile** (or thin shell script). The real friction in a 17-stack split layout is running things across all of them. A `make up`, `make down SERVICE=immich`, `make pull`, `make logs SERVICE=ntfy` target set gives you fleet control while git stays the source of truth. Highest value, near-zero cost, zero new attack surface.
2. **Deliberate version pinning + a notify-only update checker** (`diun`), *not* Watchtower. H2 is exactly why: auto-updates break things that ship breaking migrations. Pin versions, get notified when updates exist, update on purpose. This matches how you already think.
3. **If you genuinely want a GUI**, look at **Dockge** rather than Portainer. Dockge keeps your compose files on disk as the source of truth and is a thin visual layer over them — it respects the git-first model instead of competing with it. It's the compose-native answer to "I want buttons." Still adds a socket consumer, so weigh it against #1.
4. **Read-only log tailing**: if Wazuh feels heavy for "what is ntfy logging right now," **Dozzle** is a tiny read-only log viewer (no socket *write*). Optional.

So: keep git + per-service compose as the spine, add a Makefile for ergonomics, pin versions and use `diun` for updates, and reach for Dockge only if you decide you want a GUI — but not Portainer, which would quietly undermine the documentation-first discipline that's the whole point of Eirdom.
