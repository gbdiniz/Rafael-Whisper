# Documentation index

All docs for Rafael-Whisper, in recommended reading order.

| # | Document | When to read |
|---|----------|--------------|
| 1 | [deployment.md](deployment.md) | First deploy on **aaPanel + Debian 13** — Docker, container, site + SSL, Nginx proxy, firewall |
| 2 | [rafael-integration.md](rafael-integration.md) | Connect Rafael to this service — API contract, `.env`, curl tests, troubleshooting |
| 3 | [verification.md](verification.md) | After deploy — health checks, port binding, HTTPS auth, end-to-end voice turn |

**Related (not in `docs/`):**

| File | Purpose |
|------|---------|
| [../README.md](../README.md) | Repo overview and quick start |
| [../nginx/whisper.conf.example](../nginx/whisper.conf.example) | Nginx snippets for aaPanel (bearer token + reverse proxy) |
| [../docker-compose.yml](../docker-compose.yml) | Container definition (faster-whisper, resource limits) |
| [../.env.example](../.env.example) | Environment variables for Docker Compose |
