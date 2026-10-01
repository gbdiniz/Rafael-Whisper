# 1. Deployment runbook (aaPanel · Debian 13)

Deploy Rafael-Whisper on a **Debian 13 server managed with [aaPanel](https://www.aapanel.com/)** that also hosts other low-traffic Laravel sites. Rafael itself runs on a **different** machine and calls this service over HTTPS.

This guide assumes you manage Nginx, SSL, and firewall through aaPanel’s web UI, and run Docker commands over SSH or aaPanel **Terminal**.

**See also:** [Documentation index](README.md)

## Architecture

```text
Rafael server                         Whisper server (aaPanel + Docker)
┌─────────────────┐                  ┌──────────────────────────┐
│ TranscribeJob   │ ─── HTTPS ────▶  │ aaPanel Nginx (TLS)      │
│ HttpTranscriber │                  │   └─▶ Docker 127.0.0.1:9000 │
└─────────────────┘                  │         faster-whisper   │
                                     └──────────────────────────┘
```



## Decisions baked into this repo


| Topic        | Choice                                            |
| ------------ | ------------------------------------------------- |
| Panel        | aaPanel on Debian 13                              |
| Engine       | `faster_whisper`                                  |
| Model        | `base`                                            |
| Quantization | `int8` on CPU                                     |
| API          | OpenAI-compatible `POST /v1/audio/transcriptions` |
| Exposure     | Public HTTPS, bearer token, optional IP allowlist |
| Concurrency  | One transcription at a time (single-user Rafael)  |
| Bind         | Docker on `127.0.0.1:9000` only — not public      |




## Prerequisites

Before you start, confirm:


| Item             | Action                                                                                       |
| ---------------- | -------------------------------------------------------------------------------------------- |
| aaPanel          | Installed and reachable (HTTPS on your panel port, e.g. `7800`)                              |
| Web stack        | **Nginx** selected in aaPanel (not OpenLiteSpeed for this guide)                             |
| DNS              | A record `whisper.example.com` → this server’s public IP                                     |
| Bearer token     | Generate once: `openssl rand -hex 32` — same value in Nginx and Rafael `TRANSCRIBER_API_KEY` |
| Rafael server IP | Optional — for IP allowlist in Nginx or aaPanel firewall                                     |
| SSH or Terminal  | aaPanel → **Terminal** (or SSH) for `git` and `docker compose`                               |


You do **not** need prior Docker experience — sections 1–2 cover install and daily commands.

---



## 1. Install Docker on aaPanel (Debian 13)

> **Do not use App Store → Docker Manager.** That plugin was delisted (Dec 2023). Use the left sidebar **Docker** menu instead.

Two options: try **A** first; use **B** if the panel installer fails (common on Debian when aaPanel’s mirror returns 404).

### Option A — aaPanel **Docker** menu (try first)

1. Log in to aaPanel.
2. Click **Docker** in the **left sidebar** (not App Store).
3. If Docker is not installed, the page shows something like *“Currently not installed docker or docker-compose, click install”* — click **Install** and wait until it finishes.
4. When successful, the Docker page shows tabs such as **Overview**, **Container**, **Compose**, **Settings**.
5. Open **Terminal** (or SSH) and verify:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

You should see version strings and `Hello from Docker!`.

**If install fails** (errors about `mirrors.aliyun.com`, `Release file`, or `no installation candidate`), skip to Option B — do not retry App Store plugins.

**If Docker works in Terminal but aaPanel still says “not installed”:** refresh the browser (hard refresh). If it persists, run Option B (official packages); aaPanel usually detects an existing Docker install. As a last resort, aaPanel support suggests:

```bash
ln -sf /usr/libexec/docker/cli-plugins/docker-compose /usr/bin/docker-compose
```



### Option B — Official Docker packages (fallback / recommended on Debian 13)

Run on the server as root or with `sudo` (aaPanel **Terminal** is fine):

**1.1 Remove old packages** (safe if Docker was never installed)

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  apt-get remove -y "$pkg" 2>/dev/null || true
done
```

**1.2 Add Docker’s apt repository**

```bash
apt-get update
apt-get install -y ca-certificates curl

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
```

**1.3 Install Docker Engine + Compose plugin**

```bash
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
```

**1.4 Allow your SSH user to run Docker** (optional; root in aaPanel Terminal can skip)

```bash
usermod -aG docker "$USER"
```

Log out and back in (or `newgrp docker`), then:

```bash
docker run --rm hello-world
```



### aaPanel note

- **Docker menu** is for installing Docker and optional UI management (Overview, Container, **Compose**).
- This guide runs Whisper via `docker compose` **in Terminal** from `/opt/Rafael-Whisper` — simple and matches this repo’s files.
- Optional: after clone, aaPanel → **Docker** → **Compose** → **Add** → point at `/opt/Rafael-Whisper/docker-compose.yml` if you prefer starting/stopping from the panel.
- Public HTTPS still goes through **Website** → Nginx reverse proxy (section 4), not aaPanel’s “Allow external access” on a one-click app.

---



## 2. Docker basics (for this repo)



### What Docker does here


| Term                          | Meaning                                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------------------- |
| **Image**                     | Template (`onerahmet/openai-whisper-asr-webservice`) — downloaded with `docker compose pull` |
| **Container**                 | Running Whisper process — started with `docker compose up -d`                                |
| **Volume** (`whisper-models`) | Stores the downloaded **base** model so restarts do not re-download                          |




### Commands you will use

Run these inside the cloned `Rafael-Whisper` directory (where `docker-compose.yml` lives). aaPanel → **Terminal** is fine.


| Command                                       | What it does                          |
| --------------------------------------------- | ------------------------------------- |
| `docker compose up -d`                        | Start Whisper in the background       |
| `docker compose ps`                           | Show running / stopped state          |
| `docker compose logs -f whisper`              | Live logs (`Ctrl+C` to stop watching) |
| `docker compose logs --tail=50 whisper`       | Last 50 lines                         |
| `docker compose restart whisper`              | Restart after editing `.env`          |
| `docker compose down`                         | Stop container (model volume is kept) |
| `docker compose pull && docker compose up -d` | Upgrade image                         |


**First start** may take several minutes (image pull + model download). **Later starts** are much faster.

### Check Whisper on [localhost](http://localhost)

```bash
curl -sS http://127.0.0.1:9000/docs
```

Expect HTML (API docs). Port **9000** must stay on `127.0.0.1` only — aaPanel Nginx exposes HTTPS publicly.

### If something fails

```bash
docker compose ps
docker compose logs whisper
systemctl status docker
```

---



## 3. Deploy the container

Use a path outside aaPanel website roots, e.g. `/opt/Rafael-Whisper` (keeps Docker data separate from `/www/wwwroot` Laravel sites).

In aaPanel **Terminal** (or SSH):

```bash
cd /opt
git clone https://github.com/gbdiniz/Rafael-Whisper.git
cd Rafael-Whisper
cp .env.example .env
```

Edit `.env` (aaPanel **Files** → `/opt/Rafael-Whisper/.env`, or `nano .env`):

```env
ASR_ENGINE=faster_whisper
ASR_MODEL=base
ASR_DEVICE=cpu
ASR_COMPUTE_TYPE=int8
WHISPER_PORT=9000
```

Start:

```bash
cd /opt/Rafael-Whisper
docker compose up -d
docker compose logs -f whisper
```

Wait until logs show the service is ready, then confirm [localhost check](#check-whisper-on-localhost).

---



## 4. Nginx reverse proxy + TLS (aaPanel)

Whisper needs HTTPS, a **12 MB** upload limit, **bearer token** auth, and optional **IP allowlist** for the Rafael server. aaPanel handles the site and Let’s Encrypt; you add custom Nginx snippets from `[nginx/whisper.conf.example](../nginx/whisper.conf.example)`.

Replace `whisper.example.com` and `YOUR_BEARER_TOKEN` everywhere below.

### 4.1 Add the website

1. aaPanel → **Website** → **Add site**.
2. **Domain:** `whisper.example.com`
3. **Root directory:** default (`/www/wwwroot/whisper.example.com`) is fine — no PHP app runs here; Nginx will proxy everything.
4. **PHP version:** Pure static / None if offered; otherwise any — unused.
5. Create the site.



### 4.2 Issue SSL (Let’s Encrypt)

1. **Website** → click `whisper.example.com` → **SSL**.
2. Choose **Let’s Encrypt**.
3. Tick the domain → **Apply** / **Save**.
4. Enable **Force HTTPS** if the toggle is shown.

DNS must already point to this server or issuance fails.

### 4.3 Bearer token map (Nginx global config)

The `map` directive must live in the `http { }` block, not inside one site file.

1. aaPanel → **App Store** → **Nginx** → **Settings** → **Configuration file** (or **Service** → Nginx → config).
2. aaPanel usually opens `/www/server/nginx/conf/nginx.conf`. **Back up** if prompted.
3. Inside `http {`, **before** the line that includes vhosts (often `include /www/server/panel/vhost/nginx/*.conf;`), add:

```nginx
map $http_authorization $whisper_auth_ok {
    default 0;
    "Bearer YOUR_BEARER_TOKEN" 1;
}
```

1. Save and **reload Nginx** from aaPanel (Nginx → Service → Reload).

Use the **exact** same token string in Rafael’s `TRANSCRIBER_API_KEY`.

### 4.4 Site config — reverse proxy + auth

1. **Website** → `whisper.example.com` → **Conf** / **Configuration file**
  (aaPanel path is usually `/www/server/panel/vhost/nginx/whisper.example.com.conf`).
2. Find the `location /` block inside the **443 SSL** `server { }` section.
3. Replace that `location /` (and remove conflicting PHP/static handlers for `/` if aaPanel added them) with:

```nginx
client_max_body_size 12M;

# Optional: only allow the Rafael server (replace IP)
# allow RAFAEL_SERVER_IP;
# deny all;

location / {
    if ($whisper_auth_ok = 0) {
        return 401;
    }

    proxy_pass http://127.0.0.1:9000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 120s;
    proxy_send_timeout 120s;
}
```

1. Save → aaPanel **Nginx test** (if offered) → **Reload Nginx**.

**Alternative:** aaPanel **Reverse proxy** UI can set `http://127.0.0.1:9000`, but it does **not** add bearer auth or `client_max_body_size 12M`. Prefer the manual **Conf** edit above.

### 4.5 Confirm Nginx

```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://whisper.example.com/v1/audio/transcriptions
```

Expect **401** without a token. Full test: [verification.md](verification.md).

Site logs in aaPanel: **Website** → **Logs** (paths like `/www/wwwlogs/whisper.example.com.log`).

---



## 5. Firewall (aaPanel)

Do **not** open port **9000** in the firewall. Only **80** and **443** need to be public.

1. aaPanel → **Security** → **Firewall** (or **System Firewall**).
2. Ensure **80/tcp** and **443/tcp** are allowed (usually already open for other sites).
3. Do **not** add a rule for port 9000.

Optional — restrict HTTPS to the Rafael server only:

- aaPanel firewall: allow **443** from `RAFAEL_SERVER_IP` only, **or**
- Uncomment `allow` / `deny` in the site Nginx config (section 4.4).

If you use **Cloudflare** or another proxy, allowlist the Rafael server’s **outbound** IP, not the browser IP.

---



## 6. Resource limits

`[docker-compose.yml](../docker-compose.yml)` limits the container to **6 CPUs** and **8 GB RAM** so other aaPanel Laravel sites keep headroom. Edit the file under `/opt/Rafael-Whisper` if the box is quieter or busier, then:

```bash
cd /opt/Rafael-Whisper
docker compose up -d
```

Monitor load in aaPanel **Dashboard** or **Monitor** during a test transcription.

---



## 7. Upgrades

In Terminal:

```bash
cd /opt/Rafael-Whisper
git pull
docker compose pull
docker compose up -d
docker compose ps
```

Reload Nginx only if you changed site config.

---



## 8. Operations on a shared aaPanel server


| Topic             | Guidance                                                                                             |
| ----------------- | ---------------------------------------------------------------------------------------------------- |
| **CPU spikes**    | Normal during transcription; Rafael queues one job at a time in v1                                   |
| **Disk**          | aaPanel **Files** or `df -h` — Docker data under `/var/lib/docker`; model in volume `whisper-models` |
| **Logs**          | `docker compose logs -f whisper`; Nginx via aaPanel **Website → Logs**                               |
| **Backups**       | No audio stored here; model cache is re-downloadable                                                 |
| **Panel updates** | aaPanel Nginx reloads are safe; avoid overwriting custom `map` block when updating Nginx templates   |


---



## 9. Failure modes


| Symptom                      | Likely cause                                                                        |
| ---------------------------- | ----------------------------------------------------------------------------------- |
| 401 from HTTPS URL           | Bearer token mismatch — check Nginx `map` and Rafael `TRANSCRIBER_API_KEY`          |
| 502 Bad Gateway              | Container down — `docker compose ps`; Whisper not on `127.0.0.1:9000`               |
| 413 Request Entity Too Large | Missing `client_max_body_size 12M` in site config                                   |
| SSL errors in browser        | Let’s Encrypt not applied or DNS wrong — aaPanel **SSL** tab                        |
| `docker: command not found`  | Docker not installed — use left menu **Docker** → Install, or Option B in section 1 |
| aaPanel Docker install 404   | Broken mirror — use Option B (official apt repo), then refresh panel                |
| Slow first request           | Cold model load after container restart                                             |
| OOM / container restart      | Lower memory limit in `docker-compose.yml` or use model `tiny`                      |


**Next steps:** [2. Rafael integration](rafael-integration.md) · [3. Verification checklist](verification.md) · [Documentation index](README.md)