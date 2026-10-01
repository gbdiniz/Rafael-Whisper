# 3. Verification checklist

Run these after deploying Rafael-Whisper and configuring Rafael production `.env`.

**See also:** [1. Deployment](deployment.md) · [2. Rafael integration](rafael-integration.md) · [Documentation index](README.md)

## On the Whisper server (aaPanel)

Use aaPanel **Terminal** or SSH. Nginx logs: **Website** → your domain → **Logs**.

### Container health

```bash
docker compose ps
docker compose logs --tail=50 whisper
curl -sS -o /dev/null -w "%{http_code}" http://127.0.0.1:9000/docs
```

Expect container `running` and HTTP 200 from `/docs`.

### Port binding

```bash
ss -tlnp | grep 9000
```

Expect `127.0.0.1:9000` only — not `0.0.0.0:9000`.

### Public HTTPS + auth

```bash
curl -sS -o /dev/null -w "%{http_code}" \
  https://whisper.example.com/v1/audio/transcriptions
```

Expect **401** without a token.

```bash
curl -sS https://whisper.example.com/v1/audio/transcriptions \
  -H "Authorization: Bearer YOUR_SECRET" \
  -F file=@sample.webm \
  -F model=base \
  -F language=pt
```

Expect **200** and `"text": "..."`.

## On the Rafael server

Repeat the authenticated `curl` above from the Rafael host to confirm network path and firewall rules.

### End-to-end voice turn

1. Log in to Rafael.
2. Record a short phrase.
3. Confirm in database:

```sql
SELECT status, transcript, error_message FROM voice_turns ORDER BY id DESC LIMIT 1;
```

Expect `status = completed` and a non-empty `transcript`.

### Queue worker

Confirm `TranscribeVoiceTurn` jobs are consumed (Rafael `queue:work` or supervisor logs).

## Shared server sanity

During one transcription, run `htop` on the Whisper box. Other Laravel sites should remain responsive. If not, lower Docker CPU/memory limits or use model `tiny`.
