# 🗄️ Mainframe

**A self-hosted photo and video library for the home network.**

Mainframe is a Docker Compose stack that runs [Immich](https://immich.app) — an open-source, self-hosted alternative to Google Photos — on a Windows host. Your media stays on your own drive, indexed by a local database and searchable through on-device machine learning. Nothing leaves the network.

![Immich](https://img.shields.io/badge/Immich-v3.0.3-4250af)
![Postgres](https://img.shields.io/badge/PostgreSQL-16_+_VectorChord-336791)
![Valkey](https://img.shields.io/badge/Valkey-8--alpine-c93b3b)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-project%3A_mainframe-2496ed)
![Platform](https://img.shields.io/badge/Host-Windows_11_%2F_Docker_Desktop-0078d4)

---

## Contents

- [What's in the box](#whats-in-the-box)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [Where your data lives](#where-your-data-lives)
- [Quick start](#quick-start)
- [Accessing Mainframe from other devices](#accessing-mainframe-from-other-devices)
- [Everyday operations](#everyday-operations)
- [Backups](#backups)
- [Troubleshooting](#troubleshooting)
- [Upgrading](#upgrading)
- [Hardening notes](#hardening-notes)

---

## What's in the box

Four containers on one private bridge network. Only one of them is reachable from outside.

| Service | Container | Image | Exposed | Job |
|---|---|---|---|---|
| `immich-server` | `mainframe_immich_server` | `ghcr.io/immich-app/immich-server:release` | **`2283`** | Web UI, REST API, and the background worker that ingests and transcodes media |
| `immich-machine-learning` | `mainframe_immich_ml` | `ghcr.io/immich-app/immich-machine-learning:release` | internal only | CLIP embeddings for smart search, plus face detection and recognition |
| `postgres` | `mainframe_postgres` | `ghcr.io/immich-app/postgres:16-vectorchord0.4.3-pgvector0.8.0` | internal only | All metadata, accounts, and the vector index that powers search |
| `redis` | `mainframe_redis` | `valkey/valkey:8-alpine` | internal only | Job queue between the API and the background workers |

The database image is Immich's own Postgres build, not stock Postgres. It ships the **VectorChord** (`vchord`) and **pgvector** (`vector`) extensions, and Immich refuses to start without one of them. See [No vector extension found](#-no-vector-extension-found) for why this matters more than it sounds.

---

## Architecture

```mermaid
flowchart TB
    subgraph lan["🏠 Home network"]
        phone["📱 Phone / laptop / TV"]
    end

    subgraph host["💻 Windows host — 192.168.0.5"]
        port["Published port 2283"]

        subgraph net["mainframe_network (bridge)"]
            server["immich-server<br/><i>API + workers</i>"]
            ml["immich-machine-learning<br/><i>CLIP + faces</i>"]
            pg[("postgres<br/><i>metadata + vectors</i>")]
            redis[("redis<br/><i>job queue</i>")]
        end

        media[/"D:/general/mainframe<br/><i>your photos & videos</i>"/]
    end

    phone -->|"http://192.168.0.5:2283"| port
    port --> server
    server -->|":3003"| ml
    server -->|":5432"| pg
    server -->|":6379"| redis
    server -.->|"bind mount"| media

    style server fill:#4250af,color:#fff
    style pg fill:#336791,color:#fff
    style redis fill:#c93b3b,color:#fff
    style ml fill:#6b46c1,color:#fff
```

**Service names are load-bearing.** Containers find each other by Docker's internal DNS, which resolves the *service* name and the *container_name* — nothing else. Three names must stay in agreement:

| Consumer | Setting | Must match |
|---|---|---|
| `immich-server` | `DB_HOSTNAME` in `.env` | the `postgres:` service key in `docker-compose.yml` |
| `immich-server` | `REDIS_HOSTNAME` in `.env` | the `redis:` service key |
| `immich-server` | *(default)* `http://immich-machine-learning:3003` | the `immich-machine-learning:` service key |

Rename a service and you must update its partner, or you get the failure in [ENOTFOUND](#-getaddrinfo-enotfound-postgres). The ML service has no matching `.env` variable because Immich hardcodes a default — if you ever rename it, set `IMMICH_MACHINE_LEARNING_URL` explicitly.

---

## Configuration

Everything tunable lives in `.env`, which is loaded two ways: `env_file` hands it to the Immich containers as environment variables, and Compose interpolates `${...}` references inside `docker-compose.yml`.

| Variable | Current value | What it controls |
|---|---|---|
| `UPLOAD_LOCATION` | `D:/general/mainframe` | Host folder bind-mounted to `/usr/src/app/upload`. **This is your media library.** |
| `DB_HOSTNAME` | `postgres` | Hostname the server dials for Postgres |
| `DB_USERNAME` | `postgres` | Postgres role — also becomes `POSTGRES_USER` at cluster creation |
| `DB_PASSWORD` | `password` | Postgres password — see [Hardening notes](#hardening-notes) |
| `DB_DATABASE_NAME` | `immich` | Database name — also becomes `POSTGRES_DB` |
| `REDIS_HOSTNAME` | `redis` | Hostname the server dials for the job queue |

Optional variables worth knowing about, none of which are set today:

| Variable | Effect |
|---|---|
| `DB_VECTOR_EXTENSION` | Forces `vchord` or `vector` instead of letting Immich autodetect. Set this only to pin behaviour deliberately. |
| `IMMICH_MACHINE_LEARNING_URL` | Overrides the ML service address. Needed if you rename or externalize that container. |
| `TZ` | Container timezone. Currently inherited from the host via the `/etc/localtime` mount. |

> ⚠️ **The `POSTGRES_*` variables only take effect on an empty data volume.** They are read by `initdb` the first time the cluster is created and ignored on every start after that. Changing `DB_PASSWORD` in `.env` does **not** change the password of an existing database — you have to `ALTER USER` it in SQL, or delete the volume and start over.

---

## Where your data lives

Four distinct places, with very different consequences if lost.

| What | Where | Backed by | If you delete it |
|---|---|---|---|
| **Photos & videos** | `D:/general/mainframe` on the host | bind mount | 💀 Irreplaceable. This is the actual library. |
| **Accounts & metadata** — users, password hashes, sessions, API keys, albums, faces, search vectors | `mainframe_pgdata` volume | named volume | 😖 Media survives, but every account, album, and share link is gone and the library must be re-imported |
| **ML models** | `mainframe_model-cache` volume | named volume | 😌 Harmless — re-downloaded on demand |
| **Server settings** | inside Postgres | *(see above)* | Covered by the database |

**Authentication data is entirely in Postgres**, not in a config file and not on the media drive. User records, hashed passwords, active sessions, and API keys all live in the `immich` database inside the `mainframe_pgdata` volume. Two things follow from that:

- Wiping the database volume logs everyone out permanently and forgets all accounts, even though every photo is still sitting safely on `D:`.
- A backup that copies only `D:/general/mainframe` is **not** a full backup. See [Backups](#backups).

Inspect the volumes anytime:

```bash
docker volume ls --filter name=mainframe
docker volume inspect mainframe_pgdata
```

---

## Quick start

**Prerequisites** — Docker Desktop for Windows (WSL2 backend), and enough free space on `D:` for the library plus roughly 2 GB for models and database.

```bash
cd D:/code/projects/ongoing/mainframe

# Start everything, detached
docker compose up -d

# Watch it come up
docker compose logs -f immich-server
```

First boot takes a few minutes: Postgres runs `initdb`, Immich installs the vector extension and applies its schema migrations, and the ML container fetches models on first use.

**Expect some noise on startup.** `depends_on` here waits only for containers to *start*, not for Postgres to be *ready to accept connections*, so `immich-server` will typically fail and restart once or twice before the database is listening. `restart: always` recovers it automatically. You'll know it's healthy when the log settles and this reports `healthy`:

```bash
docker compose ps
```

Then open **<http://localhost:2283>** and create the admin account. The first account registered becomes the administrator.

<details>
<summary><b>Optional:</b> silence the startup crash loop with a health gate</summary>

Add a healthcheck to the `postgres` service and make the server wait for it:

```yaml
  postgres:
    # ...existing config...
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME} -d ${DB_DATABASE_NAME}"]
      interval: 5s
      timeout: 5s
      retries: 10

  immich-server:
    depends_on:
      redis:
        condition: service_started
      postgres:
        condition: service_healthy
```

Purely cosmetic — the stack works without it — but it makes real failures easier to spot, because a restart loop stops being normal background noise.
</details>

---

## Accessing Mainframe from other devices

Any phone, laptop, or TV on the same network reaches Mainframe at:

```
http://192.168.0.5:2283
```

For the **Immich mobile app**, put that URL in the *Server Endpoint* field — the app appends `/api` itself. If it rejects the address, enter `http://192.168.0.5:2283/api` explicitly.

### Why this works without a firewall rule

Verified state of the host networking:

| Check | Value |
|---|---|
| Host LAN address | `192.168.0.5` (Ethernet, /24) |
| Listener | `0.0.0.0:2283` — all interfaces, not just loopback |
| Firewall rule | `Docker Desktop Backend` → **Allow**, profile **Public** |
| Ethernet network category | **Public** — matches the rule |

Docker Desktop's firewall rule is *program-based*, granted to `com.docker.backend` rather than to port 2283, which is why searching the firewall for a 2283 rule turns up nothing while inbound traffic still works.

### Two ways this quietly breaks

> ⚠️ **Do not set this network to "Private."** The Docker rule covers the **Public** profile only. Flipping the category — normally the safer choice on a home LAN — stops matching that rule and cuts off LAN access until you add an equivalent Private rule. This is the counterintuitive one.

> ⚠️ **`192.168.0.5` is DHCP-assigned** and can move after a lease renewal or reboot, breaking every saved URL. Pin it with a DHCP reservation on the router, or give the host a static address.

### Beyond the LAN

Nothing here is exposed to the internet, and port 2283 is plain HTTP with no TLS — fine inside a trusted network, unsuitable for direct port-forwarding. For remote access, prefer a VPN or overlay network such as Tailscale or WireGuard, or a reverse proxy that terminates HTTPS. Forwarding 2283 straight through the router publishes an unencrypted login form to the open internet.

---

## Everyday operations

```bash
# Status of all four containers
docker compose ps

# Follow logs — all services, or just one
docker compose logs -f
docker compose logs -f immich-server

# Restart a single service
docker compose restart immich-server

# Stop everything, keeping all data
docker compose down

# Pick up an .env change (env vars are baked in at container creation)
docker compose up -d --force-recreate immich-server

# Open a SQL shell
docker exec -it mainframe_postgres psql -U postgres -d immich

# Shell inside the server container
docker exec -it mainframe_immich_server sh

# Disk usage of the stack
docker system df -v | grep mainframe
```

`docker compose down` is safe: it removes containers but leaves named volumes and your media alone. The destructive variant is `down -v`, which deletes `mainframe_pgdata` and `mainframe_model-cache` — see the warning in [Where your data lives](#where-your-data-lives).

---

## Backups

A complete backup is **two** things. Missing either one makes recovery painful or impossible.

**1. The database** — accounts, albums, and all metadata:

```bash
docker exec -t mainframe_postgres pg_dumpall --clean --if-exists -U postgres \
  > "D:/backups/mainframe-db-$(date +%Y%m%d).sql"
```

**2. The media** — everything under `UPLOAD_LOCATION`. Use whatever file-level tool you already trust (robocopy, Restic, a NAS sync). The important part is that it covers `D:/general/mainframe` in full.

Restore is the reverse: bring up a fresh stack, restore the media folder, then pipe the dump back in:

```bash
cat mainframe-db-YYYYMMDD.sql | docker exec -i mainframe_postgres psql -U postgres -d postgres
```

> ℹ️ Back up the SQL **dump**, not the `pgdata` volume directory. A file-level copy of a running Postgres data directory is not crash-consistent and may restore to a corrupt cluster.

---

## Troubleshooting

### ❌ `getaddrinfo ENOTFOUND postgres`

```
Error: getaddrinfo ENOTFOUND postgres
    at GetAddrInfoReqWrap.onlookupall [as oncomplete] (node:dns:122:26) {
  errno: -3008, code: 'ENOTFOUND', syscall: 'getaddrinfo', hostname: 'postgres'
}
```

**Cause.** `DB_HOSTNAME` names a host that Docker's DNS can't resolve. A container answers to its **service name** and its **container_name** — nothing else. If the service key is `database` while `.env` says `postgres`, the lookup returns NXDOMAIN and Node dies on the unhandled rejection, producing an endless restart loop.

**Diagnose.** Ask Docker what names the database actually answers to, then compare against what the server is asking for:

```bash
docker inspect mainframe_postgres \
  --format '{{range $k,$v := .NetworkSettings.Networks}}{{$v.DNSNames}}{{end}}'

docker exec mainframe_immich_server printenv | grep -E "^DB_|^REDIS_"
```

**Fix.** Make the two agree — either rename the service or point `DB_HOSTNAME` at the existing name — then **recreate** the container. A plain `restart` is not enough, because environment variables are fixed at creation time:

```bash
docker compose up -d --force-recreate immich-server
```

The same logic applies to `REDIS_HOSTNAME` and to `IMMICH_MACHINE_LEARNING_URL`.

### ❌ `No vector extension found`

```
Error: No vector extension found. Available extensions: vchord, vector
```

**This message is misleading.** `vchord, vector` is not a list of what your database has — it's Immich's list of what it *supports*, printed straight from its own constants:

```js
throw new Error(`No vector extension found. Available extensions: ${VECTOR_EXTENSIONS.join(', ')}`)
```

**Cause.** The Postgres image doesn't provide either supported extension. The classic case is a pre-v2-era compose file pinning `tensorchord/pgvecto-rs`, which supplies `vectors` (pgvecto-rs) — a backend Immich has since dropped — while `immich-server:release` keeps upgrading itself. The server outgrows the database.

**Diagnose.** Ask Postgres directly what it can offer:

```bash
docker exec mainframe_postgres psql -U postgres -d immich \
  -c "select name, default_version, installed_version
      from pg_available_extensions where name in ('vchord','vector','vectors');"
```

Seeing only `vectors` confirms it.

**Fix.** Use Immich's own Postgres image, as this stack now does:

```yaml
image: ghcr.io/immich-app/postgres:16-vectorchord0.4.3-pgvector0.8.0
```

⚠️ **Swapping the image is not sufficient on an existing cluster.** VectorChord must be listed in `shared_preload_libraries`, and the image writes that configuration only during first-time cluster initialization. Point it at a volume that some *other* image already ran `initdb` on and the setup step is skipped, so the extension still fails to load. On a fresh install with nothing to lose, recreate the volume:

```bash
docker compose down
docker volume rm mainframe_pgdata   # ⚠️ destroys all accounts and metadata
docker compose up -d
```

Confirm the database is genuinely empty first — if this returns `0`, there is nothing to lose but do check:

```bash
docker exec mainframe_postgres psql -U postgres -d immich \
  -tAc "select count(*) from information_schema.tables where table_schema='public';"
```

If the database *does* hold a real library, do not delete the volume — follow Immich's documented pgvecto-rs → VectorChord migration instead.

### ❌ Reachable from the host but not from other devices

Port 2283 accepts the connection and then drops it, which looks like a hang rather than a clean refusal. Almost always one of:

1. **The server is crash-looping.** Docker's port proxy holds 2283 open even with nothing behind it. Check `docker compose ps` and the logs before suspecting the network.
2. **Network profile flipped to Private.** See [the warning above](#two-ways-this-quietly-breaks).
3. **The host's IP changed.** Re-check with `Get-NetIPAddress -AddressFamily IPv4`.

### 🔍 General triage order

```bash
docker compose ps                          # is it even running?
docker compose logs --tail 50 immich-server # what did it say on the way down?
docker exec mainframe_postgres pg_isready -U postgres   # is the DB accepting connections?
docker exec mainframe_immich_server printenv | grep ^DB_ # is it configured as intended?
```

---

## Upgrading

All three Immich images track `:release`, so an upgrade is a pull away:

```bash
docker compose pull
docker compose up -d
```

Schema migrations run automatically on first boot of the new version.

> ⚠️ **Take a database dump before upgrading.** Migrations are one-way; there is no downgrade path once the schema has moved forward.
>
> ⚠️ **`:release` is a moving target.** The `ENOTFOUND` and vector-extension failures above were both caused by the server drifting ahead of a pinned dependency. Read Immich's release notes before major version jumps — especially anything touching the database image — or pin the server to an explicit version tag so upgrades happen when *you* choose.

---

## Hardening notes

Known-weak spots in the current setup, roughly worst first:

| ⚠️ | Issue | Why it matters | Fix |
|---|---|---|---|
| 🔴 | `DB_PASSWORD=password` | Trivially guessable. Postgres isn't published to the LAN, so the exposure is limited to anything already running on this host or inside the Docker network — but it's still a plaintext credential in a file. | Set a strong value **before** the first `up`, while `initdb` can still apply it. On an existing cluster, `ALTER USER postgres WITH PASSWORD '…'` and update `.env` to match. |
| 🟠 | `.env` holds a plaintext secret | Easy to leak by copying the folder or committing it. | Keep it out of version control — add `.env` to `.gitignore` if this ever becomes a git repository. |
| 🟠 | No TLS on port 2283 | Logins and photos cross the network unencrypted. Acceptable on a trusted LAN, unsafe anywhere else. | Reverse proxy with HTTPS, or reach it over a VPN. |
| 🟡 | Network category is Public | Windows applies stricter defaults, which is fine — but the Docker rule is Public-only, so the safer-looking change is the one that breaks access. | Leave as-is unless you add a matching Private rule. |
| 🟡 | Volume-only database durability | A corrupt volume takes every account and album with it. | Scheduled `pg_dumpall`, per [Backups](#backups). |

---

## Reference

- [Immich documentation](https://immich.app/docs) — features, mobile apps, external libraries
- [Immich Docker Compose reference](https://immich.app/docs/install/docker-compose) — upstream compose file to diff against
- [Immich backup and restore](https://immich.app/docs/administration/backup-and-restore)
- [VectorChord](https://github.com/tensorchord/VectorChord) — the vector index behind smart search
- [Valkey](https://valkey.io) — the Redis fork used for the job queue

---

<div align="center">
<sub>Mainframe · self-hosted, local-first, no cloud in the loop</sub>
</div>
