# Active Directory — Groups & Service Accounts
> Eirdom Infrastructure Reference
> Domain: ad.eirdom.homes · DC: EIRDOM-DC-01 (10.1.10.10)
> Last Updated: April 2026

---

## Overview

This document is the single reference for every AD security group
and service account in the Eirdom domain. Each entry lists the group's
purpose, which services depend on it, and where it lives in the
directory.

**Before deploying any service that uses AD authentication, verify
the required groups and service accounts in this document exist.**

---

## OU Structure

```
DC=ad,DC=eirdom,DC=homes
└── OU=Eirdom
    ├── OU=Users
    │   ├── OU=Family             ← Family member accounts
    │   └── OU=Service Accounts   ← Service binding accounts (read-only)
    ├── OU=Computers
    │   ├── OU=Workstations
    │   └── OU=Servers
    └── OU=Groups
        └── OU=Security Groups    ← All groups documented below
```

---

## Security Groups

All groups live in:
`OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes`

---

### Jellyfin-Users

| Field | Value |
|-------|-------|
| DN | `CN=Jellyfin-Users,OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | Grants access to log into Jellyfin |
| Required by | Jellyfin LDAP plugin (`docker/jellyfin/`) |
| Members | All family members who will use Jellyfin |
| Effect | Without membership in this group, AD login to Jellyfin is denied |

---

### Jellyfin-Admins

| Field | Value |
|-------|-------|
| DN | `CN=Jellyfin-Admins,OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | Grants Jellyfin administrator access |
| Required by | Jellyfin LDAP plugin (`docker/jellyfin/`) |
| Members | Tyler (admin account) |
| Effect | Members get full admin access to Jellyfin dashboard |

---

### NetBox-Users

| Field | Value |
|-------|-------|
| DN | `CN=NetBox-Users,OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | Required group for any AD user to log into NetBox |
| Required by | NetBox `configuration.py` (`AUTH_LDAP_REQUIRE_GROUP`) |
| Members | Tyler (admin) — add others as needed |
| Effect | Without membership, AD login to NetBox is denied entirely |

---

### NetBox-Staff

| Field | Value |
|-------|-------|
| DN | `CN=NetBox-Staff,OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | Grants NetBox staff-level access (can manage objects) |
| Required by | NetBox `configuration.py` (`AUTH_LDAP_USER_FLAGS_BY_GROUP`) |
| Members | Tyler (admin) |
| Effect | Members get `is_staff=True` in NetBox — access to admin UI |

---

### NetBox-Admins

| Field | Value |
|-------|-------|
| DN | `CN=NetBox-Admins,OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | Grants NetBox superuser access |
| Required by | NetBox `configuration.py` (`AUTH_LDAP_USER_FLAGS_BY_GROUP`) |
| Members | Tyler (admin) |
| Effect | Members get `is_superuser=True` — full NetBox admin rights |

---

## Deployment Checklist

Create all groups before deploying the services that depend on them.
In Active Directory Users and Computers on EIRDOM-DC-01:

```
New Group settings:
  Group scope:  Global
  Group type:   Security
  Location:     OU=Security Groups,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes
```

| Group | Create Before | Members to Add |
|-------|--------------|----------------|
| `Jellyfin-Users` | Deploying Jellyfin | All family members |
| `Jellyfin-Admins` | Deploying Jellyfin | Tyler |
| `NetBox-Users` | Deploying NetBox | Tyler |
| `NetBox-Staff` | Deploying NetBox | Tyler |
| `NetBox-Admins` | Deploying NetBox | Tyler |

---

## Service Accounts

Service accounts live in:
`OU=Service Accounts,OU=Eirdom,DC=ad,DC=eirdom,DC=homes`

All service accounts share these properties:
- Password never expires
- User cannot change password
- Account is enabled
- No logon rights (cannot interactively log in)
- Member of Domain Users only — no elevated privileges needed

---

### authentik-svc

| Field | Value |
|-------|-------|
| CN | `authentik-svc` |
| DN | `CN=authentik-svc,OU=Service Accounts,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | LDAP bind account for Authentik directory sync |
| Used by | Authentik LDAP Source (`docker/authentik/`) |
| Credential storage | Password manager → `docker/authentik/.env` → `AUTHENTIK_LDAP_BIND_PASSWORD` |
| Permissions needed | Read-only LDAP search across `OU=Eirdom` |

---

### netbox-svc

| Field | Value |
|-------|-------|
| CN | `netbox-svc` |
| DN | `CN=netbox-svc,OU=Service Accounts,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | LDAP bind account for NetBox authentication |
| Used by | NetBox `configuration.py` (`docker/netbox/`) |
| Credential storage | Password manager → `docker/netbox/.env` → `NETBOX_LDAP_BIND_PASSWORD` |
| Permissions needed | Read-only LDAP search across `OU=Eirdom` |

---

### jellyfin-svc

| Field | Value |
|-------|-------|
| CN | `jellyfin-svc` |
| DN | `CN=jellyfin-svc,OU=Service Accounts,OU=Eirdom,DC=ad,DC=eirdom,DC=homes` |
| Purpose | LDAP bind account for Jellyfin LDAP Authentication plugin |
| Used by | Jellyfin LDAP plugin (`docker/jellyfin/`) |
| Credential storage | Password manager → `docker/jellyfin/.env` → `JELLYFIN_LDAP_BIND_PASSWORD` |
| Permissions needed | Read-only LDAP search across `OU=Eirdom` |

---

### netbox-sync (UniFi local account)

> **Note:** This is a local UniFi account on the UDM-Pro-Max, not an
> AD account. Documented here for completeness.

| Field | Value |
|-------|-------|
| Username | `netbox-sync` |
| Role | Read Only |
| Created in | UniFi Network → Settings → Admins & Users |
| Purpose | NetBox `netbox_unifi_sync` plugin polls UniFi API with this account |
| Credential storage | Password manager → NetBox UI → Plugins → UniFi Sync → Controllers |

---

## LDAP Firewall Requirement

All services that perform LDAP queries against EIRDOM-DC-01 require
the following firewall rule to be in place:

**DOCKER (VLAN 50) → EIRDOM-DC-01 (10.1.10.10) on TCP 389**

This is `docker_corp_ldap` in `unifi/firewall/lan-rules.md`. Without
it, AD authentication fails silently for Authentik, NetBox, and
Jellyfin.

---

## Related Documentation

- [`docker/authentik/README.md`](../docker/authentik/README.md) — Authentik LDAP source setup
- [`docker/netbox/README.md`](../docker/netbox/README.md) — NetBox LDAP configuration
- [`docker/jellyfin/README.md`](../docker/jellyfin/README.md) — Jellyfin LDAP plugin setup
- [`unifi/firewall/lan-rules.md`](../unifi/firewall/lan-rules.md) — LDAP firewall rule
- [`docs/decisions.md`](decisions.md) — ADR-003, ADR-029