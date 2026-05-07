# once-caddy-redirect

Minimal HTTP redirector that 301s every request to `https://www.bigconfig.ai{uri}`, preserving path and query string. Path `/up` returns `200 OK` for health checks.

Packaged as a small Caddy + Hivemind container, published to GHCR on every push to `main`.

## Behavior

| Request                          | Response                                              |
| -------------------------------- | ----------------------------------------------------- |
| `GET /up`                        | `200 OK`, `Content-Type: text/plain; charset=utf-8`   |
| Anything else (any host, any path) | `301` → `https://www.bigconfig.ai{uri}`              |

The container listens on port `80` only, HTTP/1.1, with Caddy's automatic HTTPS disabled — TLS is expected to terminate upstream (load balancer / CDN).

## Run locally

With Caddy installed:

```sh
caddy run --config Caddyfile --adapter caddyfile
```

With Docker:

```sh
docker build -t once-caddy-redirect .
docker run --rm -p 8080:80 once-caddy-redirect

curl -I http://localhost:8080/      # 301 → https://www.bigconfig.ai/
curl    http://localhost:8080/up    # OK
```

CI publishes a multi-arch image covering both `linux/arm64` and `linux/amd64`. For a cross-arch local build pass `--platform linux/amd64` or `--platform linux/arm64` to `docker build`.

## Files

- `Caddyfile` — redirect + health endpoint
- `Procfile` — Hivemind process definitions (currently just `caddy`)
- `Dockerfile` — two-stage build: fetch Hivemind, layer it onto `caddy:2-alpine`
- `.github/workflows/cicd.yml` — builds arm64 and amd64 in parallel, publishes multi-arch manifests `ghcr.io/<repo>:latest` and `:sha-<short>`, then SSH-deploys to the server with `sudo once update bigconfig.ai`

## Image

```
ghcr.io/amiorin/once-caddy-redirect:latest
```
