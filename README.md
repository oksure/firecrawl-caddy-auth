# firecrawl-caddy-auth

A minimal Caddy reverse proxy that gates your self-hosted [Firecrawl](https://github.com/mendableai/firecrawl) instance behind Bearer token authentication. Designed to be deployed as a separate service on [Railway](https://railway.com) in front of the Firecrawl API service.

## Architecture

```
Internet
   │
   ▼  HTTPS (public)
┌─────────────────────┐
│  Caddy Service      │  ← validates "Authorization: Bearer <key>"
│  (Railway public)   │
└─────────────────────┘
         │
         │  HTTP (private, internal Railway network)
         ▼
┌─────────────────────┐
│  Firecrawl API      │  ← no public domain, port 3002
│  (Railway private)  │
└─────────────────────┘
```

## Deployment on Railway

### 1. Remove the public domain from your Firecrawl API service

In Railway: **Firecrawl API service → Settings → Networking → Public Networking → Remove domain.**

### 2. Add this repo as a new service in your Railway project

- **New Service → GitHub Repo** → select `firecrawl-caddy-auth`
- Railway will detect the `Dockerfile` and build automatically.

### 3. Set the following environment variables on the Caddy service

| Variable | Example Value | Description |
|---|---|---|
| `FIRECRAWL_API_KEY` | `my-secret-token` | The Bearer token clients must send. Choose any strong, random string. |
| `FIRECRAWL_UPSTREAM` | `api.railway.internal:3002` | Internal hostname and port of your Firecrawl API service. Replace `api` with your actual Railway service name. |

> `PORT` is injected automatically by Railway — do not set it manually.

Alternatively, use Railway reference variables for `FIRECRAWL_UPSTREAM`:
```
${{api.RAILWAY_PRIVATE_DOMAIN}}:${{api.PORT}}
```

### 4. Add a public domain to the Caddy service

In Railway: **Caddy service → Settings → Networking → Public Networking → Generate Domain.**

### 5. Test

```bash
# Should return 401
curl -i https://your-caddy-domain.up.railway.app/v1/scrape

# Should succeed
curl -i https://your-caddy-domain.up.railway.app/v1/scrape \
  -H "Authorization: Bearer my-secret-token" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'
```

## Files

- **`Caddyfile`** — Caddy configuration. Listens on `$PORT`, validates the `Authorization` header, proxies authorized requests to `$FIRECRAWL_UPSTREAM`, and returns `401` for everything else.
- **`Dockerfile`** — Extends `caddy:2-alpine` and copies the `Caddyfile` into place.

## Notes

- **Legacy Railway environments** (created before October 16, 2025) use IPv6-only private networking. If you are on a legacy environment, add `HOSTNAME=::` to your Firecrawl API service's environment variables.
- Caddy and Firecrawl must reside in the **same Railway environment** (e.g., both in `production`) for private networking to function.
- Railway handles TLS termination at its edge; Caddy communicates with Firecrawl over HTTP internally.
