# 2. Rafael integration

How the [Rafael](https://github.com/gbdiniz/Rafael) Laravel app connects to this service.

**See also:** [1. Deployment](deployment.md) · [3. Verification](verification.md) · [Documentation index](README.md)

## Flow

1. User records audio in Rafael (TalkControl).
2. Rafael stores the file and dispatches `TranscribeVoiceTurn` on Redis.
3. `HttpTranscriber` POSTs the file to this service.
4. Rafael stores `transcript`, deletes the audio file, sets status `completed`.

Rafael never runs Whisper code — only HTTP.

## API contract

Rafael’s `HttpTranscriber` expects an **OpenAI-compatible** transcription endpoint.

**Request**

```http
POST /v1/audio/transcriptions
Authorization: Bearer <TRANSCRIBER_API_KEY>
Content-Type: multipart/form-data

file=<audio.webm>
model=base
language=pt
```

**Success response**

```json
{
  "text": "transcrição em português"
}
```

**Field notes**

| Field | Rafael sends | Rafael-Whisper expects |
|-------|----------------|------------------------|
| `file` | WebM from browser | Supported via FFmpeg in container |
| `model` | `TRANSCRIBER_MODEL` (use `base`) | Must match `ASR_MODEL` in Docker |
| `language` | `pt` from turn locale `pt-BR` | Optional hint for Whisper |

## Rafael environment

On the **Rafael server** (not this box):

```env
TRANSCRIBER_URL=https://whisper.example.com/v1/audio/transcriptions
TRANSCRIBER_API_KEY=<same secret as nginx YOUR_BEARER_TOKEN>
TRANSCRIBER_MODEL=base
```

Leave `OPENAI_API_KEY` empty when using self-hosted Whisper.

After changing `.env`:

```bash
php artisan config:clear
```

Ensure the queue worker is running (`php artisan queue:work` or your process manager).

## Test from the Rafael server

```bash
curl -sS https://whisper.example.com/v1/audio/transcriptions \
  -H "Authorization: Bearer YOUR_SECRET" \
  -F file=@sample.webm \
  -F model=base \
  -F language=pt
```

Expect HTTP 200 and JSON with a `text` field.

## Test in Rafael UI

1. Log in to Rafael.
2. Click **Falar**, speak a short pt-BR phrase, click **Parar**.
3. Watch the voice turn: `uploaded` → `transcribing` → `completed`.
4. Check `voice_turns.transcript` in the database.

## Troubleshooting

| Rafael shows | Check |
|--------------|-------|
| `Não foi possível transcrever.` | Rafael logs (`storage/logs`), Whisper `docker compose logs`, Nginx error log |
| `Créditos da API...` | Rafael still pointing at OpenAI — fix `TRANSCRIBER_URL` |
| `Chave da API...` | Bearer token mismatch |
| `O áudio gravado está vazio...` | Browser/MediaRecorder issue — record longer clip |
| Hangs on “Transcrevendo...” | Queue worker not running on Rafael, or Whisper timeout |

## Phase reference

Rafael tracks this setup as **Phase 5.7** in `docs/project-phases.md`. Application phases 5.1–5.6 are code in Rafael; 5.7 is this external service.
