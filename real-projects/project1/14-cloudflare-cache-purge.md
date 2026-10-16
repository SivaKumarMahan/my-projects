# 14 — Cloudflare Cache Purge in the Release Pipeline

The frontend sits behind Cloudflare. After a deployment, users could still get cached assets, so a reusable **Azure DevOps task group** was built to purge specific URLs as part of the release.

## Configuration

### Variable group — `CloudFlare Purge Cache`

| Variable | Notes |
|---|---|
| `CLOUDFLARE_API_TOKEN` | Secret; API token scoped to cache purge on the zone |
| `CLOUDFLARE_PURGE_URL` | URL to purge, defaults to the app's root URL; overridable per release |

### Task group — `CloudFlare Cache Purge`

One Bash task running the purge script, either inline or from source control (versioned). Inputs: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_PURGE_URL`.

### Release integration

- Added as a stage after the frontend `deploy` stage in the frontend release pipeline; it runs automatically once deploy succeeds.
- Requires **manual approval** when triggered on demand.
- To purge on demand: update `CLOUDFLARE_PURGE_URL` if needed → create a release → run the purge stage → approve.

## What the script does

1. **Validate input** — exit with an error if `CLOUDFLARE_PURGE_URL` is empty.
2. **Parse the domain** — full host (e.g. `app.example.com`) and root domain (`example.com`).
3. **Look up the Zone ID** — Cloudflare API call with the token; fail if not found.
4. **Purge** — `POST` to the zone's `purge_cache` endpoint with only that URL (not the whole zone).
5. **Check the response** — `"success": true` → *Cache purge successful*, else *Cache purge failed*.
6. **Prefetch** — request the URL again so Cloudflare pulls fresh content from the origin straight away.

```bash
#!/bin/bash
set -euo pipefail

: "${CLOUDFLARE_PURGE_URL:?CLOUDFLARE_PURGE_URL is not set}"
HOST=$(echo "$CLOUDFLARE_PURGE_URL" | awk -F/ '{print $3}')
ROOT_DOMAIN=$(echo "$HOST" | awk -F. '{print $(NF-1)"."$NF}')

ZONE_ID=$(curl -s "https://api.cloudflare.com/client/v4/zones?name=${ROOT_DOMAIN}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" | jq -r '.result[0].id')
[ -z "$ZONE_ID" ] || [ "$ZONE_ID" = "null" ] && { echo "Zone ID not found"; exit 1; }

RESPONSE=$(curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "{\"files\":[\"${CLOUDFLARE_PURGE_URL}\"]}")

if echo "$RESPONSE" | jq -e '.success == true' >/dev/null; then
  echo "Cache purge successful"
else
  echo "Cache purge failed: $RESPONSE"; exit 1
fi

curl -s -o /dev/null "$CLOUDFLARE_PURGE_URL" && echo "Prefetched fresh content"
```

## Proving the behaviour — cache test lab

A small Django app (Docker, Python 3.11-slim, OpenTelemetry) was deployed on an Azure VM behind NGINX, with a Cloudflare-proxied DNS record, SSL mode *Full (strict)* and *Always Use HTTPS*.

| Path | Purpose |
|---|---|
| `/` | Returns 404 (test target) |
| `/health/` | 200 OK |
| `/home/` | Main page |

Cloudflare page rules for each path: **Cache Everything**, Edge TTL 2 h, Browser TTL 2 min. Headers observed: `CF-Cache-Status` (HIT / MISS / DYNAMIC), `Cache-Control: public, max-age=300`, `Vary: Accept-Encoding`.

```bash
docker build -t django-website:2 .
az acr login --name <acr>
docker tag django-website:2 <acr>.azurecr.io/django-website:2
docker push <acr>.azurecr.io/django-website:2
docker run -d -p 8000:8000 --name django-app <acr>.azurecr.io/django-website:2
```

```nginx
server {
    listen 80;
    server_name <domain>;
    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Test method (`finalCachePurgeTest.sh`)

1. **Before purge** — hit each URL 3 times to warm the cache.
2. **Purge** — only the root URL, via API.
3. **After purge** — hit each URL 3 times again.
4. **Analyse** — compare cache status and response time.

```bash
get_cache_status() {
  curl -s -I "$1" | grep -i "cf-cache-status" | head -1 | awk '{print $2}' | tr -d '\r'
}
```

### Results

```
ROOT URL   before: HIT HIT HIT   after: MISS HIT HIT   → purge worked
HEALTH URL before: HIT HIT HIT   after: HIT HIT HIT    → unaffected
HOME URL   before: HIT HIT HIT   after: HIT HIT HIT    → unaffected
```

- Purge works; the first request after purge is a MISS that rebuilds the cache, then HITs.
- **Each URL has its own cache key** — purging one does not touch others.
- Response time: HIT ~0.2–0.3 s vs MISS ~0.4–0.5 s, so cache is **~40–50% faster**.

### Troubleshooting notes

| Problem | Fix |
|---|---|
| Status always `DYNAMIC` | Page rule *Cache Everything* + explicit Edge TTL (not "respect existing headers") |
| Purge has no effect | Check token permissions, Zone ID, exact URL match (scheme, trailing slash); wait 10–15 s |
| Script errors | Run with `bash`, not `sh`; use `awk` instead of `bc` |
| 400 Bad Request from Django | Add the domain to `ALLOWED_HOSTS`; `DEBUG=False` |
