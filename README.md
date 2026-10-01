# Rafael-Whisper

Self-hosted speech-to-text for [Rafael](https://github.com/gbdiniz/Rafael). Runs **faster-whisper** behind an OpenAI-compatible HTTP API so the Laravel app can transcribe voice turns without calling OpenAI.

This repository contains **only** Docker and server ops. It does not include the Rafael application.

## Quick start

Install Docker via aaPanel **left menu → Docker** (not App Store — the old Docker Manager plugin is delisted). See [docs/deployment.md §1](docs/deployment.md#1-install-docker-on-aapanel-debian-13).

```bash
git clone https://github.com/gbdiniz/Rafael-Whisper.git
cd Rafael-Whisper
cp .env.example .env
# Edit .env — set API_BEARER_TOKEN (openssl rand -hex 32)

docker compose up -d
docker compose logs -f whisper
```

Then configure Nginx + TLS in **aaPanel** using [`nginx/whisper.conf.example`](nginx/whisper.conf.example). Full runbook: [`docs/deployment.md`](docs/deployment.md) (Debian 13 + aaPanel).

## Rafael connection

Point Rafael production `.env` at this service:

```env
TRANSCRIBER_URL=https://whisper.example.com/v1/audio/transcriptions
TRANSCRIBER_API_KEY=<same value as API_BEARER_TOKEN in nginx>
TRANSCRIBER_MODEL=base
```

Details: [`docs/rafael-integration.md`](docs/rafael-integration.md).

## Documentation

Full index: [docs/README.md](docs/README.md).

| # | Doc | Purpose |
|---|-----|---------|
| 1 | [docs/deployment.md](docs/deployment.md) | **aaPanel + Debian 13** — Docker install & usage, site, SSL, Nginx proxy, firewall |
| 2 | [docs/rafael-integration.md](docs/rafael-integration.md) | API contract and troubleshooting with Rafael |
| 3 | [docs/verification.md](docs/verification.md) | Post-deploy checks |

## Hardware notes

Tested target: **aaPanel on Debian 13**, CPU inference, **base** model with **int8**. A typical i7-class box with 16 GB RAM is sufficient for single-user Rafael (one transcription at a time).
