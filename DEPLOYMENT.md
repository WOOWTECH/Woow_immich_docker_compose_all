# Immich Deployment — Quick Reference Skill Card

> Deployment operations reference for the Immich Docker Compose stack with pgvector.

## Stack Summary

| Component | Details |
|-----------|---------|
| **Application** | Immich (self-hosted photo & video management) |
| **Containers** | immich-server, immich-machine-learning, redis (Valkey), database (PostgreSQL) |
| **Database** | PostgreSQL 14 + pgvector + VectorChord |
| **Default Port** | 2283 (Web UI) |
| **Config Files** | `.env`, `docker-compose.yml` |
| **Data Dirs** | `./library/` (photos), `./postgres/` (database) |

---

## Pre-Flight Checklist

```
[ ] Docker Engine 20.10+ installed          → docker --version
[ ] Docker Compose v2 plugin installed      → docker compose version
[ ] Minimum 4GB RAM available               → free -h
[ ] Port 2283 is available                  → ss -tlnp | grep 2283
[ ] .env file created from .env.example     → cp .env.example .env
[ ] DB_PASSWORD changed to strong password  → edit .env
[ ] DB_DATA_LOCATION is on local disk       → NOT on NFS/NAS
[ ] TZ set to correct timezone              → edit .env
```

---

## Deploy

```bash
# 1. Clone
git clone https://github.com/WOOWTECH/Woow_immich_docker_compose_all.git
cd Woow_immich_docker_compose_all

# 2. Configure
cp .env.example .env
nano .env   # Set DB_PASSWORD, TZ, storage paths

# 3. Launch
docker compose up -d

# 4. Verify (all 4 healthy)
docker compose ps

# 5. Access
#    http://<server-ip>:2283
#    First registered user = admin
```

---

## Day-2 Operations

### Update

```bash
docker compose pull
docker compose up -d
docker compose ps
```

### Backup — Database

```bash
# Compressed full backup
docker compose exec -T database pg_dumpall -U postgres \
  | gzip > "backup_immich_$(date +%Y%m%d_%H%M%S).sql.gz"
```

### Backup — Photos

```bash
rsync -av --progress ./library/ /backup/immich-library/
```

### Restore — Database

```bash
docker compose stop immich-server immich-machine-learning
gunzip < backup_immich_YYYYMMDD.sql.gz \
  | docker compose exec -T database psql -U postgres
docker compose up -d
```

### Logs

```bash
docker compose logs -f                      # All services
docker compose logs -f immich-server        # Server only
docker compose logs --tail=50 database      # Last 50 DB lines
```

### Health Check

```bash
docker compose ps                                                    # Container status
docker compose exec database pg_isready -U postgres                  # DB ready?
docker compose exec database psql -U postgres -d immich \
  -c "SELECT extname, extversion FROM pg_extension \
      WHERE extname IN ('vector','vchord');"                         # pgvector OK?
docker compose exec redis redis-cli ping                             # Redis OK?
```

### Database Info

```bash
# Database size
docker compose exec database psql -U postgres -d immich \
  -c "SELECT pg_size_pretty(pg_database_size('immich'));"

# Table sizes
docker compose exec database psql -U postgres -d immich \
  -c "SELECT tablename, pg_size_pretty(pg_total_relation_size('public.'||tablename)) AS size FROM pg_tables WHERE schemaname='public' ORDER BY pg_total_relation_size('public.'||tablename) DESC LIMIT 10;"
```

### Reset (Destructive)

```bash
docker compose down -v
rm -rf ./library ./postgres
docker compose up -d
```

---

## Environment Variables

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `UPLOAD_LOCATION` | `./library` | No | Photo/video storage path |
| `DB_DATA_LOCATION` | `./postgres` | No | PostgreSQL data path (**local disk only**) |
| `IMMICH_VERSION` | `v2` | No | Image tag (pin for stability) |
| `TZ` | — | No | Timezone (e.g., `Asia/Taipei`) |
| `IMMICH_PORT` | `2283` | No | Web UI host port |
| `DB_PASSWORD` | — | **Yes** | PostgreSQL password (A-Za-z0-9) |
| `DB_USERNAME` | `postgres` | No | PostgreSQL user |
| `DB_DATABASE_NAME` | `immich` | No | PostgreSQL database name |

---

## Network Topology

```
Internet/LAN
      │
      ▼
Host:2283 ──► immich-server ──┬──► database (PostgreSQL:5432)
                              │      pgvector + VectorChord
                              ├──► redis (Valkey:6379)
                              └──► immich-machine-learning:3003
```

---

## Security Notes

- **Change `DB_PASSWORD`** — never use the default
- **Never expose port 5432** — PostgreSQL is internal only
- **Use HTTPS** — put behind Nginx/Caddy reverse proxy for production
- **Set `IMMICH_TRUSTED_PROXIES`** — when using reverse proxy
- **Pin `IMMICH_VERSION`** — avoid unexpected breaking changes
- **Backup regularly** — both database and photo library
- **DB on local disk** — NFS causes PostgreSQL corruption

---

## File Map

```
.
├── docker-compose.yml      # 4-container service stack
├── .env.example            # Configuration template → copy to .env
├── .env                    # Active configuration (git-ignored)
├── .gitignore              # Exclusion rules
├── LICENSE                 # MIT License
├── README.md               # English documentation
├── README_zh-TW.md         # 繁體中文 documentation
├── DEPLOYMENT.md           # This file — quick reference
├── library/                # Photo uploads (git-ignored)
└── postgres/               # Database files (git-ignored)
```
