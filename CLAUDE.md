# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

A minimal containerized HTTP redirector: every request on port 80 is 301-redirected to `https://www.bigconfig.ai{uri}`, with a `/up` health endpoint returning `200 OK`. There is no application code — the entire behavior lives in `Caddyfile`.

## Architecture

- **`Caddyfile`** — Caddy config. `auto_https off` and `protocols h1` because the container only serves plain HTTP/1.1 on `:80` (TLS is expected to terminate upstream, e.g. at a load balancer/CDN). Two handlers: `/up` for health checks, catch-all `handle` for the permanent redirect preserving path + query via `{uri}`.
- **`Procfile`** — single `caddy:` process line consumed by Hivemind. Kept as a Procfile (rather than running Caddy directly) so additional sidecars can be added later without changing the container entrypoint.
- **`Dockerfile`** — two-stage build. Stage 1 downloads the Hivemind binary matching `TARGETARCH` (multi-arch aware). Stage 2 is `caddy:2-alpine` with Hivemind copied in; `CMD` runs `hivemind Procfile`.
- **`.github/workflows/docker-publish.yml`** — on push to `main`, builds on `ubuntu-24.04-arm` and pushes to `ghcr.io/<repo>` tagged `latest` and `sha-<short>`. Single-arch (arm64) image; no `platforms:` matrix.

## Commands

Local Caddy run (no container):
```
caddy run --config Caddyfile --adapter caddyfile
```

Validate Caddyfile syntax:
```
caddy validate --config Caddyfile --adapter caddyfile
```

Build and run the container locally (arm64 host):
```
docker build -t once-root .
docker run --rm -p 8080:80 once-root
curl -I http://localhost:8080/        # expect 301 to www.bigconfig.ai/
curl    http://localhost:8080/up      # expect "OK"
```

For an x86_64 build, pass `--platform linux/amd64` (the CI workflow only builds arm64).
