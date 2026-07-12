# Eirdom — First-Deploy Order & Validation Checklist

Greenfield bring-up of the Docker host (EIRDOM-DOCKER-01, 10.1.50.10).
The AI stack (`eirdom-intelligence`) runs on the laptop and is **out of scope** here.

**The rule:** every phase ends in a **GATE**. Do not start the next phase until the
gate passes. Two layers carry everything else — Traefik+certs (Phase 1) and Authentik
(Phase 2) — so a failure there is not worth deploying past.

Dependency shape:
```
proxy network ─► Traefik (TLS) ─► Authentik (SSO) ─► app stacks ─► arr-stack ─► cloudflared
                                   │
        chain-public services can run without Authentik;
        chain-standard / chain-admin services return 502 until it's configured.
```

---

## Phase 0 — Host prerequisites (no containers yet)

- [x] Docker Engine + Compose v2 installed on EIRDOM-DOCKER-01.
- [x] **Create the shared proxy network** (every web stack declares it `external: true`):
      ```bash
      docker network create proxy
      ```
- [x] **Storage tiers mounted** (`/etc/fstab`) and base dirs created with `1000:1000`:
      ```bash
      # Tier 1 configs (OS SSD), Tier 2 download cache (250GB SSD), Tier 3 = /media/arr (3TB, already mounted)
      sudo mkdir -p /opt/eirdom/appdata /mnt/downloads-cache
      sudo mkdir -p /media/arr/downloads/complete/{radarr,radarr-4k,sonarr,sonarr-4k,lidarr,manual}
      sudo chown -R 1000:1000 /opt/eirdom/appdata /mnt/downloads-cache /media/arr
      ```
      Pre-create each stack's config dir under `/opt/eirdom/appdata/<svc>` (or run `provision.sh`).
      > WARNING: Docker auto-creates missing bind-mount paths as **root**, which breaks
      > linuxserver.io images expecting UID 1000. Pre-create + `chown 1000:1000`.
- [x] **Cloudflare**: zone `eirdom.homes` exists; API token with `Zone → DNS → Edit`; note the Zone ID.
- [x] **Internal DNS** (AD on EIRDOM-DC-01): the service hostnames (and a wildcard if you
      prefer) resolve to **10.1.50.10**. Without this, hostnames are unreachable on the LAN
      even with valid certs. (Cert issuance itself uses DNS-01, so it does *not* need public
      reachability — which is why internal-only services still get real certs.)
- [x] **Fill the root `.env`** from `.env.example`: `TZ`, `PUID/PGID`, `ROOT_DOMAIN=eirdom.homes`,
      `DOCKER_DATA_PATH=/opt/eirdom/appdata`, `MEDIA_PATH=/media/arr`, `CF_API_TOKEN`, `CF_ZONE_ID`,
      `TRAEFIK_ACME_EMAIL`, SMTP_*, backup vars.
- [x] **Generate secrets** — one per DB / app key:
      ```bash
      openssl rand -base64 32     # repeat for each *_DB_PASSWORD, AUTHENTIK_SECRET_KEY, JWT, etc.
      ```
- [x] **Fill each `docker/<svc>/.env`** from its `.env.example`.
- [x] **Apply the audit fixes before first boot** (they only bite on deploy):
      - `traefik/traefik.yml` — hardcoded domain + **your real ACME email** (not `CHANGEME@…`)
      - `traefik/dynamic/middlewares.yml` — `ipAllowList`, `sslRedirect` removed
      - `traefik/dynamic/routers.yml` — hardcoded `eirdom.homes`
      - `ntfy` → `chain-public`; `webserver` → apache image + `adminer-ipallowlist`

**GATE 0:** `docker network ls | grep proxy` returns a row; tier dirs exist and are `1000:1000`;
no required value in any `.env` is blank.

---

## Phase 1 — Traefik (TLS + routing backbone)

```bash
cd docker/traefik
docker compose up -d
docker compose logs -f
```

**GATE 1 — certificate issuance is make-or-break:**
- [x] `docker compose ps` → `traefik` healthy.
- [x] Logs show the wildcard cert obtained, no repeating ACME errors:
      ```bash
      docker logs traefik 2>&1 | grep -iE "certificate|acme|error" | tail -20
      ```
- [x] `acme.json` is populated and `600`:
      ```bash
      ls -l /opt/eirdom/appdata/traefik/certs/acme.json   # non-zero size
      ```
- [x] A **real** cert is served (not Traefik's self-signed default):
      ```bash
      echo | openssl s_client -connect 127.0.0.1:443 -servername jellyfin.eirdom.homes 2>/dev/null \
        | openssl x509 -noout -issuer -dates
      # issuer = Let's Encrypt   (NOT "CN = TRAEFIK DEFAULT CERT")
      ```

> If the cert fails: this is audit finding **C1**. Confirm `traefik.yml` has the literal
> `eirdom.homes` and a real email (no `${...}`), and that `CF_API_TOKEN` has DNS-edit rights.
> The Traefik dashboard (`traefik.eirdom.homes`) uses `chain-admin` and will **not** load until
> Phase 2 — validate via certs/logs here, not the dashboard.

---

## Phase 2 — Authentik (SSO; gates every chain-standard / chain-admin service)

```bash
cd docker/authentik
docker compose up -d
docker compose logs -f
```

- [x] `authentik-postgres`, `authentik-server`, `authentik-worker` all healthy.
- [ ] `https://auth.eirdom.homes` loads with a valid cert.
- [ ] Set the admin password via the first-run flow: `https://auth.eirdom.homes/if/flow/initial-setup/`.

**Configure forward-auth** (so the middleware chains actually authenticate):
- [x] Create a **Proxy Provider** → *Forward auth (domain level)*: auth domain `auth.eirdom.homes`,
      cookie domain `eirdom.homes`, external host `https://auth.eirdom.homes`.
- [x] Create an **Application** bound to that provider.
- [x] Add it to the **embedded outpost** (Applications → Outposts → *authentik Embedded Outpost*).
- [x] (Optional now or later) configure the **LDAP source** to AD (`10.1.10.10`) for domain logins.

> Use Authentik's current "Traefik forward auth" docs for exact field names — the UI shifts
> between releases, but the three objects (provider → application → outpost) are the constant.

**GATE 2 — prove SSO end-to-end with one service:**
- [x] Bring up `uptime-kuma` (chain-standard), hit `https://status.eirdom.homes`, and confirm it
      **redirects to Authentik → login → bounces back**. That validates the outpost + chain wiring.
      If you instead get **502/Bad Gateway**, the outpost/provider isn't bound yet — fix before Phase 3.

---

## Phase 3 — Application stacks

Pattern for each: `cd docker/<svc> && docker compose up -d && docker compose ps` (all healthy) → load its URL.
Order among these doesn't matter — **except deploy `uptime-kuma` last** so it can monitor the rest.

Per-stack notes:
- **netbox** — build the custom image first (UniFi-sync plugin): `docker compose build` then `up -d`.
  LDAP to `10.1.10.10`. `chain-public`.
- **immich** — set `IMMICH_VERSION` in `immich/.env` and confirm the DB image tag matches it.
  `chain-public`. First start initialises VectorChord and creates the upload subdirs.
- **jellyfin** — Jellyfin + Jellyseerr are `chain-public`; **Jellystat is `chain-admin`** (needs Phase 2).
  Configure the Jellyfin LDAP plugin per the arr/jellyfin README.
- **paperless** — `chain-standard`. Add `PAPERLESS_ENABLE_HTTP_REMOTE_USER=true` (finding **M4**) or the
  Authentik header won't auto-log-in users.
- **webserver** — WordPress (apache, fixed) + **Adminer is `chain-admin`, VLAN-10 only**.
- **ntfy** — `chain-public`. After boot, create the publish user/token:
  `docker exec ntfy ntfy user add --role=admin tyler` then `ntfy token add tyler` → put the token in root `.env` `NTFY_TOKEN`.
- **actual / grocy / homebox / mealie / stirling-pdf** — straightforward; `chain-standard` (except Actual `chain-public`).

**GATE 3 — per stack:**
- [ ] `docker compose ps` → all containers healthy (a stuck DB blocks its app via `depends_on: healthy`).
- [ ] `chain-public` URLs load directly; `chain-standard`/`chain-admin` URLs redirect through Authentik.
- [ ] No DB connection errors in app logs.

---

## Phase 4 — arr-stack (VPN-gated — validate the tunnel before it touches the internet)

Prereqs:
- [ ] 250GB SSD mounted at `DOWNLOADS_CACHE_PATH`; incomplete dir exists, `1000:1000`.
- [ ] AirVPN: WireGuard config generated, reserved port set; fill `arr-stack/.env`
      (`WIREGUARD_PRIVATE_KEY`, `WIREGUARD_PRESHARED_KEY`, `WIREGUARD_ADDRESSES`,
      `VPN_SERVER_COUNTRIES`, `VPN_FORWARDED_PORT`).

```bash
cd docker/arr-stack
docker compose up -d
docker compose logs -f gluetun
```

**GATE 4 — VPN + kill switch, before downloading or seeding:**
- [ ] `gluetun` healthy; logs show the WireGuard tunnel up.
- [ ] qBittorrent exits via the VPN, not your home IP:
      ```bash
      docker exec qbittorrent curl -s https://ipinfo.io     # AirVPN IP/country, NOT yours
      ```
- [ ] Kill switch holds:
      ```bash
      docker stop gluetun
      docker exec qbittorrent curl -s --max-time 8 https://ipinfo.io || echo "BLOCKED (correct)"
      docker start gluetun
      ```
- [ ] Set qBittorrent's listening port = `VPN_FORWARDED_PORT`; confirm the *arr apps reach qBit at
      `http://gluetun:8080` (Settings → Download Clients).

Then finish per the arr-stack README: categories, **Use Hardlinks**, Prowlarr indexers, `recyclarr sync`.

---

## Phase 5 — External access (cloudflared)

- [ ] Fill `CLOUDFLARE_TUNNEL_TOKEN`; `cd docker/cloudflared && docker compose up -d`.
- [ ] In the Cloudflare dashboard, map **only** the hostnames you want public (e.g. `photos`,
      `requests`, `jellyfin`) → tunnel → Traefik.

> WARNING (finding L4/H1): do **not** publish internal-only admin UIs through the tunnel
> (`adminer`, `traefik`, `proxmox`, `netbox`, `wazuh`). Their VLAN-10 IP allowlist sees the
> tunnel's source IP, not the visitor's, so it can't protect them from tunnel traffic.

**GATE 5:**
- [ ] Each intended public hostname loads from **off-network** (mobile data).
- [ ] An internal-only hostname is **not** reachable externally.

---

## Phase 6 — Whole-system validation

- [ ] **Uptime-Kuma**: add a monitor per service URL + the Docker socket; route alerts to **ntfy**.
- [ ] **ntfy**: subscribe on mobile; send a test publish.
- [ ] **backup.sh**: dry-run; confirm it covers `/opt/eirdom/appdata` (all configs now live there),
      the NAS target is reachable, and the ntfy completion notification fires.
- [ ] **Hardlinks**: after the first arr import, run the README's `stat … | grep Links` check (expect 2).
- [ ] **Certs**: `acme.json` contains the `*.eirdom.homes` wildcard — Traefik auto-renews, nothing to schedule.

---

## Quick troubleshooting

| Symptom | Most likely cause |
|---|---|
| Self-signed / cert warning on every site | ACME failed — **C1** (literal `${}` in `traefik.yml`) or CF token lacks DNS-edit |
| 404 on a hostname | Internal DNS not pointing the host to 10.1.50.10, or the router/label is missing |
| 502 Bad Gateway on a `chain-standard`/`chain-admin` site | Authentik outpost/provider not configured (Phase 2) |
| qBittorrent has no connectivity | `gluetun` unhealthy (VPN creds/port) — that's the kill switch doing its job |
| Container stuck "unhealthy", dependents won't start | `depends_on: service_healthy` — read that container's logs first |
| Admin UI reachable from the internet | It's mapped in the Cloudflare tunnel — remove it (Phase 5) |
