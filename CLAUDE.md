# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

A minimal containerized HTTP redirector: every request on port 80 is 301-redirected to `https://www.bigconfig.ai{uri}`, with a `/up` health endpoint returning `200 OK`. There is no application code — the entire behavior lives in `Caddyfile`.

## Architecture

- **`Caddyfile`** — Caddy config. `auto_https off` and `protocols h1` because the container only serves plain HTTP/1.1 on `:80` (TLS is expected to terminate upstream, e.g. at a load balancer/CDN). Two handlers: `/up` for health checks, catch-all `handle` for the permanent redirect preserving path + query via `{uri}`.
- **`Procfile`** — single `caddy:` process line consumed by Hivemind. Kept as a Procfile (rather than running Caddy directly) so additional sidecars can be added later without changing the container entrypoint.
- **`Dockerfile`** — two-stage build. Stage 1 downloads the Hivemind binary matching `TARGETARCH` (multi-arch aware). Stage 2 is `caddy:2-alpine` with Hivemind copied in; `CMD` runs `hivemind Procfile`.
- **`.github/workflows/cicd.yml`** — on push to `main`, builds arm64 and amd64 in parallel jobs (`build-arm` on `ubuntu-24.04-arm`, `build-amd` on `ubuntu-24.04`), each pushing per-arch tags (`:arm`, `:amd`, plus `sha-<short>-<arch>`) to `ghcr.io/<repo>`. A `manifest` job then stitches them into multi-arch manifests `:latest` and `:sha-<short>`. Finally, a `deploy` job SSHes to `$SERVER_IP` as `ubuntu` and runs `sudo once update bigconfig.ai`.

## Commands

Local Caddy run (no container):
```
caddy run --config Caddyfile --adapter caddyfile
```

Validate Caddyfile syntax:
```
caddy validate --config Caddyfile --adapter caddyfile
```

Build and run the container locally:
```
docker build -t once-caddy-redirect .
docker run --rm -p 8080:80 once-caddy-redirect
curl -I http://localhost:8080/        # expect 301 to www.bigconfig.ai/
curl    http://localhost:8080/up      # expect "OK"
```

CI publishes both arm64 and amd64; for a cross-arch local build, pass `--platform linux/amd64` or `--platform linux/arm64`.
