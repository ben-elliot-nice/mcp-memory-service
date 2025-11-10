# Railway Deployment Guide

This guide covers deploying the MCP Memory Service to [Railway.app](https://railway.app), a modern platform for deploying applications with zero configuration.

## Prerequisites

- Railway account ([sign up](https://railway.app))
- GitHub repository (fork of mcp-memory-service)
- Basic understanding of environment variables

## Architecture Overview

**Recommended Configuration for Railway:**
- **Storage Backend**: SQLite-vec
- **Deployment Method**: Dockerfile (existing at `tools/docker/Dockerfile`)
- **Persistence**: Railway Volume mounted at `/app/data`
- **Port**: Railway's dynamic `$PORT` environment variable (automatically configured)

## Quick Start

### 1. Create Railway Project

```bash
# Option A: Railway CLI (recommended)
railway login
railway init
railway up

# Option B: Railway Dashboard
# 1. Go to railway.app/new
# 2. Connect your GitHub repository
# 3. Select mcp-memory-service fork
```

### 2. Configure Environment Variables

In the Railway dashboard, add these environment variables:

**Required:**
```bash
MCP_MEMORY_STORAGE_BACKEND=sqlite_vec
MCP_HTTP_ENABLED=true
MCP_API_KEY=<generate-with-openssl-rand-base64-32>
ALLOW_ANONYMOUS_ACCESS=true  # or false for API key authentication
```

**Optional:**
```bash
LOG_LEVEL=INFO
PYTHONUNBUFFERED=1
MCP_MEMORY_SQLITE_PATH=/app/data/sqlite_vec.db
MCP_MEMORY_BACKUPS_PATH=/app/data/backups
```

**Generate secure API key:**
```bash
openssl rand -base64 32
```

### 3. Create Railway Volume

**Critical:** SQLite requires persistent storage

1. In Railway dashboard → your service → Settings → Volumes
2. Click "New Volume"
3. **Mount Path**: `/app/data`
4. **Volume name**: `mcp-memory-data`

This ensures your database persists across deploys.

### 4. Deploy

Railway will auto-detect the Dockerfile at `tools/docker/Dockerfile` and deploy automatically.

**Deployment will:**
- Build Docker image with Python 3.12 + SQLite-vec
- Start FastAPI server on Railway's assigned port
- Enable health checks at `/api/health`
- Mount volume for database persistence

### 5. Access Your Service

Once deployed, Railway provides a URL:
```
https://your-app-name.railway.app
```

**Test deployment:**
```bash
# Health check
curl https://your-app-name.railway.app/api/health

# Dashboard (if anonymous access enabled)
open https://your-app-name.railway.app
```

## Environment Variable Reference

### Storage Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_MEMORY_STORAGE_BACKEND` | `sqlite_vec` | Storage backend: `sqlite_vec`, `cloudflare`, or `hybrid` |
| `MCP_MEMORY_SQLITE_PATH` | `/app/data/sqlite_vec.db` | SQLite database path (inside volume) |
| `MCP_MEMORY_BACKUPS_PATH` | `/app/data/backups` | Backup directory path |

### HTTP Server Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | (Railway-assigned) | **Auto-set by Railway** - Don't override |
| `MCP_HTTP_ENABLED` | `false` | Enable HTTP server (set to `true`) |
| `MCP_HTTP_HOST` | `0.0.0.0` | Bind host (leave as default) |
| `MCP_API_KEY` | None | API key for authentication |
| `ALLOW_ANONYMOUS_ACCESS` | `false` | Allow unauthenticated access |

### Performance & Logging

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_LEVEL` | `INFO` | Logging level: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `PYTHONUNBUFFERED` | `0` | Set to `1` for real-time logs in Railway |

## Health Checks

Railway automatically monitors your service health.

**Health endpoint:**
```
GET /api/health
```

**Response (healthy):**
```json
{
  "status": "healthy",
  "service": "mcp-memory-service",
  "version": "8.22.2",
  "backend": "sqlite_vec"
}
```

## Volume Management

### Backup Database

```bash
# Using Railway CLI
railway run bash

# Inside container
cp /app/data/sqlite_vec.db /tmp/backup.db
# Download via Railway dashboard → Files
```

### Restore Database

```bash
# Upload backup.db via Railway dashboard → Files to /app/data/
railway run bash

# Inside container
cp /tmp/backup.db /app/data/sqlite_vec.db
```

### Monitor Volume Usage

```bash
railway run df -h /app/data
```

## Troubleshooting

### Common Issues

**1. Service won't start**
- Check Railway logs: `railway logs`
- Verify `MCP_HTTP_ENABLED=true` is set
- Ensure no `PORT` variable override (Railway sets this automatically)

**2. Database not persisting**
- Verify Railway Volume is mounted at `/app/data`
- Check volume mount in Railway dashboard → Settings → Volumes
- Ensure `MCP_MEMORY_SQLITE_PATH=/app/data/sqlite_vec.db`

**3. 502 Bad Gateway**
- Service may be starting (takes 30-60 seconds first time for model download)
- Check health endpoint: `/api/health`
- Review logs: `railway logs --tail 100`

**4. Authentication errors**
- If using API key: Verify `MCP_API_KEY` is set
- If using anonymous: Set `ALLOW_ANONYMOUS_ACCESS=true`
- Dashboard requires anonymous access OR valid API key

### View Logs

```bash
# Railway CLI
railway logs
railway logs --tail 100

# Dashboard
# Project → Service → Deployments → View Logs
```

## Advanced Configuration

### Using Cloudflare Backend

If you prefer cloud storage over SQLite:

```bash
# Environment variables
MCP_MEMORY_STORAGE_BACKEND=cloudflare
CLOUDFLARE_API_TOKEN=<your-token>
CLOUDFLARE_ACCOUNT_ID=<your-account>
CLOUDFLARE_D1_DATABASE_ID=<your-d1-id>
CLOUDFLARE_VECTORIZE_INDEX=mcp-memory-index
```

**No Railway Volume needed** - all data stored in Cloudflare.

### Using Hybrid Backend

Best of both worlds (local speed + cloud backup):

```bash
MCP_MEMORY_STORAGE_BACKEND=hybrid
# Add both SQLite path + Cloudflare credentials
```

**Requires:** Railway Volume + Cloudflare credentials

### Custom Domain

1. Railway dashboard → your service → Settings → Domains
2. Add custom domain
3. Configure DNS CNAME record
4. Update OAuth issuer if using OAuth:
   ```bash
   MCP_OAUTH_ISSUER=https://your-domain.com
   ```

## Performance Optimization

### Reduce Build Time

Create `.railwayignore` in project root:
```
.git
.github
*.md
docs/
tests/
*.pyc
__pycache__/
.pytest_cache/
```

### Enable Caching

Railway caches Docker layers automatically. No configuration needed.

### Monitor Resource Usage

```bash
# Check memory/CPU
railway run top

# Check disk usage
railway run df -h
```

## Cost Estimation

Railway pricing (as of 2024):
- **Free tier**: $5 credits/month (good for testing)
- **Pro plan**: $20/month + usage
- **Volume storage**: ~$0.25/GB/month

**Estimated monthly cost for production:**
- Small deployment (512MB RAM, 1GB storage): ~$10-15/month
- Medium deployment (1GB RAM, 5GB storage): ~$20-30/month

## Security Best Practices

1. **Always set a strong API key**
   ```bash
   MCP_API_KEY=$(openssl rand -base64 32)
   ```

2. **Disable anonymous access in production**
   ```bash
   ALLOW_ANONYMOUS_ACCESS=false
   ```

3. **Use custom domain with HTTPS**
   - Railway provides free SSL certificates
   - Configure via Dashboard → Settings → Domains

4. **Regular backups**
   - Download database weekly via Railway dashboard
   - Consider hybrid backend for automatic cloud backup

## Next Steps

- [Configure OAuth 2.1](../oauth-setup.md) for team collaboration
- [Set up monitoring](../monitoring.md) with Railway metrics
- [Enable CORS](../cors-configuration.md) for frontend integration
- [Scale your deployment](../scaling.md) with Railway replicas

## Support

- **Railway Issues**: https://help.railway.app
- **MCP Memory Service Issues**: https://github.com/doobidoo/mcp-memory-service/issues
- **Railway Community**: https://discord.gg/railway

## References

- [Railway Documentation](https://docs.railway.app)
- [FastAPI Deployment Best Practices](https://fastapi.tiangolo.com/deployment/)
- [SQLite-vec Documentation](https://github.com/asg017/sqlite-vec)
