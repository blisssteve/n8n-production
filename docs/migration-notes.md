# Migration Notes

Source: `vultr1:/opt/n8n`
Target: Dokploy cluster (`dokploy-cloud`)

## Source State (as of 2026-03-10)

- **n8n version**: 2.12.0 (running behind `:next` tag)
- **Database**: SQLite (~821 MB + WAL)
- **Binary storage**: filesystem (`/opt/n8n/binaryData`)
- **Workflows**: 480 total, 160 active
- **Credentials**: ~200
- **Community nodes**: 15+ packages (see README.md)
- **Custom nodes**: `/opt/n8n/custom`

## Key Differences

| Aspect | Source | Target |
|--------|--------|--------|
| Database | SQLite | Postgres 16 |
| Binary storage | filesystem | filesystem (R2 later) |
| Workers | None | None (Phase 2) |
| Queue mode | No | No (Phase 2) |
| Hostname | n8n.blisshome.xyz | n8n.emerald.vet |
| Proxy | Traefik + Cloudflare | Traefik + Cloudflare |

## Encryption Key

**CRITICAL**: The encryption key must be identical for credentials to decrypt.

Source location: `/opt/n8n/config`

Do **not** store the real key in git. Read it from the source host at migration time and set `N8N_ENCRYPTION_KEY` in Dokploy as a secret.

## Community Nodes Strategy

Two options:

### Option A: Install via n8n UI (Recommended for Phase 1)
1. Deploy stack
2. Go to Settings → Community Nodes
3. Install each package manually
4. Restart n8n

### Option B: Custom Docker Image (Phase 2+)
1. Create Dockerfile extending `n8nio/n8n`
2. Pre-install community nodes
3. Build and push to GHCR
4. Update compose to use custom image

For Phase 1, use Option A for simplicity.

## Sidecar Services (Not Migrating Yet)

Source has additional services that are NOT in Phase 1:
- `gotenberg` - PDF generation
- `ghostscript` - PDF processing
- `qdrant` - Vector DB
- `ydl_api_ng` - YouTube download API

These can be added later if needed by specific workflows.

## Validation Checklist

After migration, validate:

- [ ] Can log in to n8n UI
- [ ] Workflows imported successfully
- [ ] Credentials decrypted (test a workflow)
- [ ] Webhooks working (test a webhook URL)
- [ ] Production workflows executing
- [ ] Binary data accessible
- [ ] Community nodes working

## Rollback Plan

If migration fails:

1. Keep source instance running
2. Revert DNS to `n8n.blisshome.xyz`
3. Investigate issue
4. Re-attempt migration

Source will remain available until cutover is validated.
