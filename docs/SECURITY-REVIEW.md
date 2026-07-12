# Eirdom — Docker Security-by-Design Review

Scope: `docker/` across all 19 stacks, plus repo-level secret hygiene.
Method: extracted the repo, checked the **git index** (what's actually pushed, not just
what's on disk), inspected the one committed database, swept every compose file for
port exposure / auth chains / privilege, and ran a masked secret scan over tracked files.

**Bottom line:** the foundation is sound. The single most important property —
*nothing reaches the network except through Traefik* — is correctly implemented. The
findings below are refinements plus **one real design pitfall** (private media in
WordPress) that directly affects your stated goal.

---

## What's already right (the hard parts)

- **Single ingress.** Only Traefik publishes host ports (`80`, `443`). Every other
  service, databases included, is reachable *only* through the proxy. No exposed DB
  ports, nothing bound to `0.0.0.0` behind Traefik's back.
- **Network segmentation.** Per-stack `internal: true` networks isolate databases
  (wordpress-internal, jellyfin-internal, etc.) from the proxy plane.
- **Secret hygiene.** `.gitignore` excludes all `.env`, `secrets.yml`, cloudflared
  json/pem, and Traefik certs — and the git index confirms **no `.env` is tracked** and
  **no live credential appears in any tracked text/config file.**
- **Auth model (ADR-044).** ForwardAuth (`chain-standard`) on most apps; ForwardAuth +
  VLAN-10 (`chain-admin`) on admin tools; `chain-public` reserved for apps that self-auth.
- **Torrent isolation.** arr traffic routes through gluetun (AirVPN) with `NET_ADMIN`
  scoped — **not** `privileged`. Correct.
- **Selective pinning.** Infra-critical images are pinned (authentik `2026.2`, immich via
  `${IMMICH_VERSION}`, all databases, netbox `v4.5`).

---

## Findings (priority order)

### CRITICAL / HIGH

**H1 — WordPress cannot gate private media the way intended.**
WordPress serves uploaded files directly from `/wp-content/uploads/…` with **no auth
check**, at a guessable URL, regardless of whether the containing post is private /
password-protected / members-only. Membership plugins gate the *page*, not the *file*.
Result: "approved-accounts-only" family photos are retrievable by direct URL. This is a
real disclosure path, not theoretical.
- **Fix (recommended):** don't host private media in WordPress. Use **Immich**
  (photos.eirdom.homes — real per-album/user ACLs, already OIDC-gated) for photos, and a
  purpose-built calendar (**Radicale** or **Nextcloud**) for calendars. Keep WordPress as
  the public face and link out to the gated apps.
- **If insisting on in-WP gating:** move uploads outside the webroot or PHP-proxy them
  through the auth check, add a membership plugin, and put Cloudflare edge WAF in front.
  Fiddly and easy to get wrong — not recommended for anything sensitive.

**H2 — WordPress router has no protective middleware.**
The `wordpress` router carries only `www-redirect` — no security-headers, no rate-limit,
no login-path protection — on the most-attacked app in the stack, about to go public.
- **Fix:** create `chain-wordpress` = `security-headers` + `rate-limit` + a strict
  limiter scoped to `/wp-login.php` and `/xmlrpc.php` (often `/xmlrpc.php` is blocked
  outright). Add Cloudflare edge WAF + edge rate-limit on the login path — the one
  service where orange-cloud proxy earns its keep (WAF, caching, edge throttling).

**H3 — `docker.sock` mounted into three containers; `:ro` is false comfort.**
`traefik`, `authentik`, and `uptime-kuma` each bind `/var/run/docker.sock`. A read-only
*mount* does **not** make the Docker *API* read-only — a compromised container with
socket access can still `POST` a privileged container and escape to host root.
`uptime-kuma` is the weakest link (least-hardened, monitoring-only).
- **Fix:** front all three with `tecnativa/docker-socket-proxy`, exposing only the API
  endpoints each needs (Traefik: containers read; Authentik: containers/images as its
  outpost needs; Kuma: containers read). Removes three host-root escape paths.

**H4 — `webui.db` committed to git history.**
Tracked in commit *"Created local AI platform for eirdom."* Inspected: **1** account
(email + bcrypt hash), **4** chat messages, **1** knowledge-file reference; `api_key` and
`oauth_session` tables empty; no secret-named fields in `config`. So: a credential + PII
in the repo, but **not** a key trove — high-hygiene, not catastrophic.
- **Fix:** add `*.db` / `*.sqlite*` to `.gitignore`; purge from history with
  `git filter-repo` (or BFG) and force-push; rotate that Open WebUI password (bcrypt is
  slow to crack, but treat any committed credential as burned).

**H5 — NetBox on `chain-public`.**
NetBox is your network source-of-truth (IPAM/DCIM: full topology, device inventory, IPs,
possibly the secrets plugin), gated only by its own login. Inconsistent with how Proxmox /
Wazuh / Security Onion / the Traefik dashboard are (correctly) locked behind `chain-admin`.
- **Fix:** move NetBox to `chain-admin` (ForwardAuth + VLAN-10). Confirm it is **not**
  in the Cloudflare tunnel ingress.

### MEDIUM

**M1 — Authentik login not rate-limited at Traefik.**
The `authentik` router applies `security-headers@file` only — `rate-limit-auth` (your
10/min brute-force limiter) is defined but attached to nothing, right as auth goes public.
Authentik's internal flow policies are a backstop, but wire the layer you built.
- **Fix:** add `rate-limit-auth` to the authentik router's middleware list.

**M2 — Actual Budget on `chain-public`.**
Actual holds financial data and uses a single shared instance password (no per-user).
`chain-public` means only that one password stands in front of it.
- **Fix:** put ForwardAuth in front (`chain-standard`), or confirm it is strictly
  internal / VPN-only and never tunneled.

**M3 — Adminer gated by IP only, no SSO.**
`adminer-ipallowlist` (VLAN-10) with no `authentik` — a single VLAN-10 foothold equals
direct database access, while your other admin UIs require SSO too.
- **Fix:** switch Adminer to `chain-admin` (adds ForwardAuth on top of the IP gate), or
  drop Adminer and use a DB client over SSH when needed.

**M4 — ~27 images on `:latest`.**
Supply-chain and reproducibility risk: a compromised or breaking upstream tag
auto-deploys on next pull, and you can't pin-and-roll-back. You already pin the
infra-critical tier — extend that to the app tier.
- **Fix:** pin to explicit version tags (ideally digests) and adopt Renovate or
  Dependabot for controlled bumps. Prioritize internet-facing / socket-holding images
  first (traefik, cloudflared, open-webui).

### LOW / hygiene

- **L1 — Tracked artifacts to scrub:** `attempts.log`, `scripts/__pycache__/gate.cpython-314.pyc`.
  Add `__pycache__/` and `*.pyc` to `.gitignore` and `git rm --cached` both.
- **L2 — MCP git server mounts `/repo-rw` (read-write).** If Open WebUI is ever exposed or
  compromised, the model can write to a repo. Currently dormant (its Traefik router is
  commented out and it publishes no ports), so low priority — but scope the mount to
  read-only or a throwaway repo before you expose the AI stack (intended `chain-admin`).
- **L3 — `insecureSkipVerify` on `proxmox-transport`** (Proxmox / Wazuh / Security Onion /
  Home Assistant): acceptable for LAN self-signed certs, but move the **security tools**
  (Wazuh, Security Onion) onto internal-PKI certs (`internal-transport`, already reserved)
  first — a LAN-resident attacker is exactly their threat model.
- **L4 — `www-redirect` duplicated:** live as `@docker` (webserver labels), orphaned as
  `@file` (middlewares.yml). Delete the file-provider copy.
- **L5 — `proxmox-transport` comment** lists only "Proxmox, Wazuh, Security Onion" but
  `homeassistant-svc` uses it too. Stale comment.

---

## Suggested order of operations

1. **Before tunneling anything public:** H2 (WordPress chain) + M1 (Authentik rate-limit)
   + H5 (NetBox → chain-admin, and keep it off the tunnel).
2. **The photo/calendar decision (H1):** stand up Immich albums + a real calendar app;
   point the family site's links there. This is the actual answer to "approved accounts
   gating pictures and calendars."
3. **Host-escape surface (H3):** socket-proxy in front of the three docker.sock consumers.
4. **Repo cleanup (H4, L1):** history purge + gitignore, password rotation.
5. **Ongoing (M4):** pin images + Renovate/Dependabot.

Nothing here undermines the core design — the ingress model, segmentation, and secret
hygiene are right. These close the gaps around a solid base.
