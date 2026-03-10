# n8n Production Stack

Production n8n deployment for Emerald Vet, designed for Dokploy.

## Architecture

### Phase 1 (Current)
- **Postgres 16** - Primary database (migrating from SQLite)
- **n8n 2.12.0** - Single main instance (no workers yet)
- **Task Runners** - External mode for Python support
- **Filesystem binary storage** - Local volumes (R2 comes later)

### Phase 2 (Future)
- Add **Redis** for queue mode
- Add **n8n workers** for horizontal scaling
- Migrate binary storage to **Cloudflare R2**
- Enable multi-main setup for HA

## Quick Start

### 1. Generate Secrets

```bash
# Postgres password
openssl rand -base64 32

# Task runner auth token
openssl rand -hex 32
```

### 2. Get Source Encryption Key

The encryption key MUST match the source instance for credential migration.

Retrieve it from the source instance at deploy time and set it in Dokploy as a secret. Do not commit the actual key to git.

```bash
# On vultr1
cat /opt/n8n/config | jq -r '.encryptionKey'
```

### 3. Configure Environment

Copy `.env.example` to `.env` and fill in:
- `POSTGRES_PASSWORD` - Generated in step 1
- `N8N_ENCRYPTION_KEY` - From source instance
- `N8N_RUNNERS_AUTH_TOKEN` - Generated in step 1

### 4. Deploy to Dokploy

1. Create new **Compose** application in Dokploy
2. Point to this Git repository
3. Set environment variables in Dokploy Environment tab
4. Add domain: `n8n.emerald.vet`
5. Deploy

## Migration from Source

### Pre-Migration Checklist

- [ ] Source n8n is running version 2.12.0
- [ ] Encryption key copied from source
- [ ] Backup of source SQLite database (`/opt/n8n/database.sqlite`)
- [ ] List of community nodes to install
- [ ] Custom nodes copied from `/opt/n8n/custom`

### Migration Steps

1. **Deploy new stack** (it will start empty)
2. **Install community nodes** - Either:
   - Via n8n UI (Settings → Community Nodes)
   - Or pre-bake into custom Docker image
3. **Export workflows from source**:
   ```bash
   # On source instance
   n8n export:workflow --all --output=/tmp/workflows/
   ```
4. **Import to new instance**:
   ```bash
   # On new instance
   n8n import:workflow --input=/tmp/workflows/
   ```
5. **Migrate credentials** (requires same encryption key):
   ```bash
   # On source
   n8n export:credentials --all --output=/tmp/creds/
   # On new instance
   n8n import:credentials --input=/tmp/creds/
   ```
6. **Test production workflows**
7. **Update DNS** to point to new instance
8. **Keep source running** as fallback during validation

### Community Nodes (from source)

```json
{
  "@marcuson/n8n-nodes-ics-utils": "^0.1.3",
  "n8n-nodes-cloudconvert": "^0.1.7",
  "n8n-nodes-deepseek-1clickai": "^1.0.20",
  "n8n-nodes-deepseek-llm": "^1.0.7",
  "n8n-nodes-firecrawl-v1": "^0.4.21",
  "n8n-nodes-mcp": "0.1.37",
  "n8n-nodes-notion-markdown": "^1.1.0",
  "n8n-nodes-notionmd": "^0.1.0",
  "n8n-nodes-skyvern": "0.0.11",
  "@rxap/n8n-nodes-yaml": "0.1.7",
  "n8n-nodes-enhanced-chat": "2.0.0",
  "n8n-nodes-trino": "1.0.3",
  "n8n-nodes-firecrawl": "0.3.0",
  "n8n-nodes-yaml": "0.1.5",
  "n8n-nodes-notion-any-converter-to-markdown": "1.0.1"
}
```

Install via n8n UI or add to a custom image.

## Volume Structure

| Volume | Purpose | Backup |
|--------|---------|--------|
| `n8n_postgres_data` | Postgres data | Yes (Dokploy Volume Backups) |
| `n8n_data` | n8n config, workflows | Yes (Dokploy Volume Backups) |
| `n8n_binary_data` | Binary file storage | Yes (Dokploy Volume Backups) |
| `n8n_custom_nodes` | Custom community nodes | Yes (Dokploy Volume Backups) |

## Networking

- Connects to Dokploy's `dokploy-network` for Traefik routing
- Internal `n8n-network` for service-to-service communication
- Exposed via Traefik at `n8n.emerald.vet`

## Monitoring

- **Health checks**: Postgres has built-in healthcheck
- **Logs**: View via Dokploy Logs tab
- **Metrics**: n8n exposes `/metrics` endpoint (Prometheus format)

## Troubleshooting

### n8n won't start
1. Check Postgres is healthy: `docker compose logs postgres`
2. Verify encryption key matches source
3. Check n8n logs: `docker compose logs n8n`

### Can't connect to Postgres
1. Verify Postgres is running: `docker compose ps`
2. Check credentials match in .env
3. Test connection: `docker compose exec postgres psql -U n8n -d n8n`

### Workflows missing after migration
1. Check import succeeded: `docker compose logs n8n | grep import`
2. Verify encryption key is correct
3. Re-import from backup

## References

- [n8n Queue Mode Docs](https://docs.n8n.io/hosting/scaling/queue-mode/)
- [n8n External Storage Docs](https://docs.n8n.io/hosting/scaling/external-storage/)
- [Dokploy Compose Docs](https://docs.dokploy.com/docs/core/docker-compose)
- [Migration Tracker (Notion)](https://www.notion.so/n8n-Production-Migration-31f7534076ee8189b112faaed6a66f5a)
