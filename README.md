# Nextcloud

Docker Compose stack for a self-hosted Nextcloud instance behind Traefik, with
PostgreSQL, Redis and a dedicated cron container. User data lives on an
SMB/NFS share.

## Architecture

| Service     | Image                 | Role                                            |
|-------------|-----------------------|-------------------------------------------------|
| `nextcloud` | `nextcloud:<version>` | Apache + PHP app server, exposed via Traefik     |
| `cron`      | `nextcloud:<version>` | Runs Nextcloud background jobs (`/cron.sh`)     |
| `db`        | `postgres:<tag>`      | Database                                        |
| `redis`     | `redis:<tag>`         | Memory cache and transactional file locking     |

Networks:

Both networks are dual-stack (IPv4 + IPv6) with subnets set in `.env`.

- `nextcloud` — frontend bridge. Traefik (running on the host network) reaches
  Nextcloud through this network's gateways, which is why both of its subnets
  are set as Nextcloud's trusted proxies. Only Nextcloud is attached to it.
- `nextcloud-backend` — internal network (no outbound access) shared by
  Nextcloud, cron, Postgres and Redis.

Volumes:

| Volume            | Contents                                         |
|-------------------|--------------------------------------------------|
| `nextcloud-html`  | Nextcloud code, config and apps (`/var/www/html`) |
| `nextcloud-data`  | User data, mounted from the SMB/NFS share        |
| `nextcloud-db`    | PostgreSQL data                                  |
| `nextcloud-redis` | Redis persistence                                |

## Prerequisites

- Docker Engine with Compose v2. IPv6 networks need Docker 27+ (or
  `"ipv6": true` / `"ip6tables": true` in `daemon.json` on older versions).
- Traefik running on the host network, with the Docker provider enabled and a
  `websecure` entrypoint that serves a (wildcard) default TLS certificate
  covering `DOMAIN`. This stack does not define a certificate resolver.
- A reachable SMB or NFS share for user data. For CIFS, the host needs
  `cifs-utils`; for NFS, `nfs-common` (or your distro's equivalent).

## Setup

1. Create the environment file and restrict its permissions:

   ```sh
   cp .env.example .env
   chmod 600 .env
   ```

2. Edit `.env`. At minimum set:

   - `DOMAIN` — public hostname, e.g. `cloud.example.com`
   - `POSTGRES_PASSWORD`, `REDIS_PASSWORD` — strong random values
     (`openssl rand -base64 32`)
   - `DATA_MOUNT_TYPE`, `DATA_MOUNT_DEVICE`, `DATA_MOUNT_O` — the data share.
     Files must be owned by `www-data` (`uid=33,gid=33`).

3. Check that the `*_NETWORK_SUBNET_*` ranges don't overlap your LAN or any
   existing Docker network:

   ```sh
   docker network inspect -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}' $(docker network ls -q)
   ```

4. Validate and start:

   ```sh
   docker compose config --quiet
   docker compose up -d
   docker compose ps
   ```

   The first start can take a few minutes while Nextcloud installs/upgrades;
   the healthcheck allows up to 180 s.

5. Open `https://$DOMAIN` and finish the installer (or log in, if migrating an
   existing instance).

## Configuration

All settings are in `.env`; see `.env.example` for the full list with comments.

### Required

| Variable            | Description                                        |
|---------------------|----------------------------------------------------|
| `DOMAIN`            | Public hostname; used for routing and trusted domains |
| `POSTGRES_PASSWORD` | Database password (must match an existing DB user) |
| `REDIS_PASSWORD`    | Redis `requirepass`                                |
| `DATA_MOUNT_TYPE`   | `cifs` or `nfs`                                    |
| `DATA_MOUNT_DEVICE` | Share path, e.g. `//host/share` or `host:/export`  |
| `DATA_MOUNT_O`      | Mount options                                      |

### Common

| Variable                                | Default            | Description                                   |
|-----------------------------------------|--------------------|-----------------------------------------------|
| `NEXTCLOUD_VERSION`                     | `35.0.1-apache`    | Image tag for both `nextcloud` and `cron`     |
| `POSTGRES_TAG`                          | `18.6-alpine`      | Postgres image tag                            |
| `REDIS_TAG`                             | `8.10-alpine3.23`  | Redis image tag                               |
| `POSTGRES_DB` / `POSTGRES_USER`         | `nextcloud`        | Database name and user                        |
| `NEXTCLOUD_NETWORK_SUBNET_V4` / `NEXTCLOUD_NETWORK_SUBNET_V6` | `10.100.0.0/24` / `fd00:100:0::/64` | Frontend network; both are trusted proxies |
| `BACKEND_NETWORK_SUBNET_V4` / `BACKEND_NETWORK_SUBNET_V6` | `10.100.1.0/24` / `fd00:100:1::/64` | Internal backend network |
| `IPV4_ALLOWLIST` / `IPV6_ALLOWLIST`     | `0.0.0.0/0` / `::/0` | Source ranges allowed by Traefik            |
| `PHP_MEMORY_LIMIT`                      | `1G`               | PHP memory limit per request                  |
| `PHP_UPLOAD_LIMIT`                      | `16G`              | Max upload size                               |
| `NEXTCLOUD_MAX_WORKERS`                 | `16`               | Apache `MaxRequestWorkers`                    |
| `REDIS_MAXMEMORY`                       | `200mb`            | Redis `maxmemory`                             |
| `TZ`                                    | `Europe/Athens`    | Container time zone                           |

### Resource limits

Each service has `*_CPU_LIMIT`, `*_MEMORY_LIMIT`, `*_MEMORY_RESERVATION` and
`*_PIDS_LIMIT` (prefixes `NEXTCLOUD_`, `CRON_`, `DB_`, `REDIS_`). Keep these in
balance:

- `NEXTCLOUD_MAX_WORKERS` × typical request memory (~100 MB) should fit within
  `NEXTCLOUD_MEMORY_LIMIT`.
- `REDIS_MAXMEMORY` should stay below `REDIS_MEMORY_LIMIT`, so Redis evicts keys
  instead of being OOM-killed (which would drop all file locks at once).

## Security

- Secrets are passed to containers as Docker secrets (`/run/secrets/*`), not as
  plain environment variables.
- All containers run with `no-new-privileges`.
- Postgres and Redis are only on the internal backend network.
- Traefik applies an IP allowlist, HSTS headers and the CalDAV/CardDAV
  `.well-known` redirects.
- Encoded `/ % # ? ;` are allowed in request paths so WebDAV can handle file
  names containing those characters.

## Operations

Run `occ` commands:

```sh
docker compose exec -u www-data nextcloud php occ status
```

Logs (rotated at 10 MB × 5 files per container):

```sh
docker compose logs -f nextcloud
```

### Upgrading

1. Back up (see below).
2. Bump `NEXTCLOUD_VERSION` by **one major version at a time** — `nextcloud`
   and `cron` always use the same tag.
3. Apply and check:

   ```sh
   docker compose pull
   docker compose up -d
   docker compose exec -u www-data nextcloud php occ status
   docker compose exec -u www-data nextcloud php occ db:add-missing-indices
   ```

Postgres major upgrades require a dump/restore (or `pg_upgrade`); changing
`POSTGRES_TAG` across a major version alone will not start.

### Backup

Enable maintenance mode, dump the database, then back up the volumes and the
data share:

```sh
docker compose exec -u www-data nextcloud php occ maintenance:mode --on
docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > nextcloud-$(date +%F).sql
# back up the nextcloud-html volume (config/, apps/) and the data share
docker compose exec -u www-data nextcloud php occ maintenance:mode --off
```

## Notes

- Postgres 15+ no longer lets non-owners create tables in `public`. An init
  script grants `CREATE ON SCHEMA public` so the role Nextcloud's installer
  creates can set up its tables. It only runs on a fresh database volume.
