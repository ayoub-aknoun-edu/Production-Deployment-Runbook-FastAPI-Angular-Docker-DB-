# Production Deployment Runbook (FastAPI + Angular + Docker DB)

**Target stack:** Contabo VPS • CloudPanel • Cloudflare • FastAPI (Gunicorn/Uvicorn) • Angular (SPA) • Dockerized Postgres & Redis

---

## Table of Contents

1. [What You’ll Deploy](#what-youll-deploy)
2. [Prerequisites](#prerequisites)
3. [Variables to Replace](#variables-to-replace)
4. [DNS (Cloudflare)](#dns-cloudflare)
5. [Server Prep (SSH + Docker)](#server-prep-ssh--docker)
6. [Database Stack (Docker Compose)](#database-stack-docker-compose)
7. [Create Sites in CloudPanel](#create-sites-in-cloudpanel)
8. [Backend Deploy (FastAPI)](#backend-deploy-fastapi)
9. [Systemd Service (Gunicorn/Uvicorn)](#systemd-service-gunicornuvicorn)
10. [Nginx Custom (API site)](#nginx-custom-api-site)
11. [Frontend Deploy (Angular SPA)](#frontend-deploy-angular-spa)
12. [Cloudflare: Switch to Proxied](#cloudflare-switch-to-proxied)
13. [Health Checks](#health-checks)
14. [Updating Later (No Git)](#updating-later-no-git)
15. [.env Template](#env-template)
16. [Backups & Restore](#backups--restore)
17. [Firewall & Hardening](#firewall--hardening)
18. [Troubleshooting](#troubleshooting)
19. [Directory Layout](#directory-layout)

---

## What You’ll Deploy

* **Frontend (Angular SPA)** → CloudPanel **Static HTML** site on `FRONT_DOMAIN`.
* **Backend (FastAPI)** → CloudPanel **Python** site proxying to **Gunicorn** on `127.0.0.1:APP_PORT`.
* **Postgres & Redis** → **Docker Compose**, bound to `127.0.0.1`.
* **DNS/SSL** → Cloudflare DNS; Let’s Encrypt in CloudPanel (use **DNS only** while issuing SSL, then switch to **Proxied**).

---

## Prerequisites

* Contabo (or any VPS) with **CloudPanel** installed.
* Cloudflare access to your domain.
* Your app structured roughly like:

  * Backend: `app/`, `alembic.ini`, `migrations/`, `requirements.txt`, etc.
  * Frontend: Angular project with `ng build --configuration production` producing `dist/<project>/browser/`.

---

## Variables to Replace

```
VPS_IP                = <YOUR_SERVER_IP>
FRONT_DOMAIN          = <frontend.example.com>
API_DOMAIN            = <api.example.com>
SITE_USER_FRONT       = <cloudpanel-frontend-user>
SITE_USER_BACK        = <cloudpanel-backend-user>
APP_PORT              = <port from CloudPanel Python Settings, e.g. 8020>

DB_NAME               = <db_name>
DB_USER               = <db_user>
DB_PASS               = <STRONG_DB_PASSWORD>

JWT_SECRET            = <LONG_RANDOM>
JWT_REFRESH_SECRET    = <LONG_RANDOM>
OTHER_API_KEYS        = <e.g. THIRD_PARTY_API_KEY>
WEBHOOK_SECRET        = <LONG_RANDOM>
```

---

## DNS (Cloudflare)

1. Create **A** records:

   * `FRONT_DOMAIN` → `VPS_IP` (**DNS only** while issuing SSL)
   * `API_DOMAIN` → `VPS_IP` (**DNS only** while issuing SSL)
2. **SSL/TLS**: set **Mode = Full**.

---

## Server Prep (SSH + Docker)

```bash
ssh root@VPS_IP
apt update && apt -y upgrade
curl -fsSL https://get.docker.com | sh
apt -y install docker-compose-plugin
```

---

## Database Stack (Docker Compose)

```bash
mkdir -p /opt/appstack && cd /opt/appstack
cat > compose.yaml <<'YAML'
version: "3.9"
services:
  postgres:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_DB: ${DB_NAME:-appdb}
      POSTGRES_USER: ${DB_USER:-appuser}
      POSTGRES_PASSWORD: ${DB_PASS:-change_me}
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    restart: always
    volumes:
      - ./redisdata:/data
    ports:
      - "127.0.0.1:6379:6379"
YAML

DB_NAME="<db_name>" DB_USER="<db_user>" DB_PASS="<STRONG_DB_PASSWORD>" docker compose up -d
```

(Optional) Auto-start on boot:

```bash
cat >/etc/systemd/system/appstack-compose.service <<'EOF'
[Unit]
Description=App Docker Compose Stack
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/appstack
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now appstack-compose
```

---

## Create Sites in CloudPanel

1. **Static HTML** site → domain = `FRONT_DOMAIN` → issue Let’s Encrypt (works best when DNS is **DNS only**).
2. **Python** site → domain = `API_DOMAIN` → issue Let’s Encrypt.

   * CloudPanel shows a **Python Settings → Port**. Note this as `APP_PORT`.
   * CloudPanel simply reverse-proxies requests to `127.0.0.1:APP_PORT`.

---

## Backend Deploy (FastAPI)

```bash
ssh SITE_USER_BACK@VPS_IP
cd ~/htdocs/API_DOMAIN
mkdir -p backend && cd backend

# Python venv
python3 -m venv .venv
source .venv/bin/activate

# Upload your backend (SCP/SFTP) so this folder contains: app/, alembic.ini, migrations/, requirements.txt, etc.
pip install -U pip wheel
pip install -r requirements.txt
```

Create `.env` (see template below) in **this** backend folder:

```bash
nano .env
```

Run DB migrations and create an assets dir:

```bash
alembic upgrade head
mkdir -p assets
```

---

## Systemd Service (Gunicorn/Uvicorn)

Create a service that binds to the `APP_PORT` shown by CloudPanel:

```bash
sudo tee /etc/systemd/system/app-backend.service > /dev/null <<EOF
[Unit]
Description=App FastAPI (Gunicorn/Uvicorn)
After=network.target

[Service]
User=SITE_USER_BACK
Group=SITE_USER_BACK
WorkingDirectory=/home/SITE_USER_BACK/htdocs/API_DOMAIN/backend
Environment="PYTHONUNBUFFERED=1"
ExecStart=/home/SITE_USER_BACK/htdocs/API_DOMAIN/backend/.venv/bin/gunicorn \
  -k uvicorn.workers.UvicornWorker \
  -w 2 \
  -b 127.0.0.1:APP_PORT \
  --timeout 120 --graceful-timeout 120 \
  app.main:app
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now app-backend
sudo journalctl -u app-backend -f
```

> Prefer Gunicorn+Uvicorn workers for production. You can swap to plain Uvicorn if you want by adjusting `ExecStart`.

---

## Nginx Custom (API site)

CloudPanel → **Sites → API\_DOMAIN → Vhost → Custom Config**:

```nginx
server {
  listen 80;
  listen [::]:80;
  listen 443 quic;
  listen 443 ssl;
  listen [::]:443 quic;
  listen [::]:443 ssl;
  http2 on;
  http3 off;
  {{ssl_certificate_key}}
  {{ssl_certificate}}

  server_name DOMAIN_NAME;
    {{root}}

  {{nginx_access_log}}
  {{nginx_error_log}}

 {{nginx_access_log}}
  {{nginx_error_log}}

  if ($scheme != "https") {
    rewrite ^ https://$host$request_uri permanent;
  }

  location ~ /.well-known {
    auth_basic off;
    allow all;
  }

{{settings}}

  include /etc/nginx/global_settings;

  index index.html;


  location /uwsgi {
    include uwsgi_params;
    uwsgi_read_timeout 3600;
    #uwsgi_pass unix:///run/uwsgi/app/weblate/socket;
    uwsgi_pass 127.0.0.1:{{app_port}};
  }

  location / {
    proxy_pass http://127.0.0.1:{{app_port}}/;
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $host;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_pass_request_headers on;
    proxy_max_temp_file_size 0;
    proxy_connect_timeout 900;
    proxy_send_timeout 900;
    proxy_read_timeout 900;
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    proxy_temp_file_write_size 256k;
  }

  location ~* ^.+\.(css|js|jpg|jpeg|gif|png|ico|gz|svg|svgz|ttf|otf|woff|woff2|eot|mp4|ogg|ogv|webm|webp|zip|swf)$ {
    add_header Access-Control-Allow-Origin "*";
    add_header alt-svc 'h3=":443"; ma=86400';
    expires max;
    access_log on;
  }

  if (-f $request_filename) {
    break;
  }


}
```

Save → CloudPanel reloads Nginx automatically.

---

## Frontend Deploy (Angular SPA)

**Local build:**

```bash
# in environment.prod.ts ensure the API points to your backend:
# apiBaseUrl: 'https://API_DOMAIN/api/v1'
ng build --configuration production
```

**Upload to the frontend site root (no nested browser/ folder left behind):**

```bash
scp -r dist/<your-project>/browser/* \
  SITE_USER_FRONT@VPS_IP:/home/SITE_USER_FRONT/htdocs/FRONT_DOMAIN/
```

**SPA fallback & asset caching** (CloudPanel → **FRONT\_DOMAIN → Vhost → Custom Config**):

```nginx
# SPA routing
location / { try_files $uri $uri/ /index.html; }

# cache hashed assets hard, but not index.html
location ~* \.(?:js|css|png|jpg|jpeg|gif|ico|svg|webp|woff2?|ttf|otf|eot)$ {
  expires 30d;
  add_header Cache-Control "public, max-age=31536000, immutable";
  access_log off;
}
location = /index.html {
  expires -1;
  add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

---

## Cloudflare: Switch to Proxied

Once both domains serve HTTPS correctly, switch the Cloudflare A records for `FRONT_DOMAIN` and `API_DOMAIN` from **DNS only** to **Proxied** (orange cloud). Keep **SSL/TLS: Full**.

---

## Health Checks

* API health: `https://API_DOMAIN/healthz` should return 200.
* API docs: `https://API_DOMAIN/docs`
* SPA loads at: `https://FRONT_DOMAIN`
* CORS: your backend must allow the frontend origin (see `.env`).

---

## Updating Later (No Git)

**Backend (safe update):**

```bash
ssh SITE_USER_BACK@VPS_IP
cd ~/htdocs/API_DOMAIN/backend
cp .env /tmp/env.backup
tar czf /tmp/assets-$(date +%F).tgz assets

# remove old code (keep .venv/.env/assets)
rm -rf app migrations alembic.ini requirements.txt pyproject.toml

# upload new code here…

source .venv/bin/activate
pip install -r requirements.txt
alembic upgrade head
sudo systemctl restart app-backend
```

**Frontend:**

```bash
ng build --configuration production
ssh SITE_USER_FRONT@VPS_IP "rm -rf ~/htdocs/FRONT_DOMAIN/*"
scp -r dist/<your-project>/browser/* SITE_USER_FRONT@VPS_IP:/home/SITE_USER_FRONT/htdocs/FRONT_DOMAIN/
```

**Breaking schema changes (wipe DB):**

```bash
docker exec -it $(docker ps -qf name=postgres) psql -U DB_USER -d postgres -c "DROP DATABASE DB_NAME;"
docker exec -it $(docker ps -qf name=postgres) psql -U DB_USER -d postgres -c "CREATE DATABASE DB_NAME OWNER DB_USER;"
# then
alembic upgrade head
```

---

## .env Template

Create this file in your **backend working directory** (the folder with `app/`):

```env
ENV=production
APP_NAME=MyApp

APP_PUBLIC_BASE_URL=https://API_DOMAIN
FRONTEND_PUBLIC_URL=https://FRONT_DOMAIN
CORS_ORIGINS=https://FRONT_DOMAIN

DATABASE_URL=postgresql+asyncpg://DB_USER:DB_PASS@127.0.0.1:5432/DB_NAME

JWT_SECRET=JWT_SECRET
JWT_REFRESH_SECRET=JWT_REFRESH_SECRET

# Optional third parties
OTHER_API_KEYS=...

# Webhooks (example)
WEBHOOK_SECRET=WEBHOOK_SECRET

# App storage for generated files
ASSETS_DIR=/home/SITE_USER_BACK/htdocs/API_DOMAIN/backend/assets
```

> For multiple frontend origins, comma-separate `CORS_ORIGINS` (no spaces).

---

## Backups & Restore

**Nightly dump (cron):**

```bash
mkdir -p /opt/appstack/backups
cat >/etc/cron.d/appstack-pg-backup <<'EOF'
30 2 * * * root /usr/bin/docker exec -t $(/usr/bin/docker ps -qf name=postgres) \
  pg_dump -U DB_USER -d DB_NAME -Fc > /opt/appstack/backups/db_$(date +\%F).dump
EOF
```

**Restore:**

```bash
docker exec -i $(docker ps -qf name=postgres) pg_restore -U DB_USER -d DB_NAME < /opt/appstack/backups/db_YYYY-MM-DD.dump
```

---

## Firewall & Hardening

```bash
ufw allow 22 && ufw allow 80 && ufw allow 443
ufw --force enable
```

* Use SSH keys, disable root SSH, consider changing SSH port.
* Keep packages updated, consider fail2ban, rotate JWT secrets periodically.

---

## Troubleshooting

| Symptom                      | Likely Cause                        | Fix                                                                         |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------- |
| 502 on API domain            | App not running or wrong `APP_PORT` | `systemctl status app-backend`; confirm the CloudPanel Python Settings port |
| CORS errors                  | Origin not allowed                  | Add frontend origin to `CORS_ORIGINS`, restart app                          |
| Angular deep-link 404        | No SPA fallback                     | Add `try_files $uri $uri/ /index.html;` in frontend vhost                   |
| Let’s Encrypt fails          | Cloudflare proxy on                 | Temporarily set **DNS only**, issue SSL, then switch back to **Proxied**    |
| Webhook 403                  | Wrong secret or URL                 | Ensure `WEBHOOK_SECRET` matches; base URL correct and HTTPS                 |
| 413 Request Entity Too Large | Nginx limit                         | Add `client_max_body_size` on API site                                      |
| Slow uploads / timeouts      | Proxy timeouts                      | Increase `proxy_read_timeout` / `proxy_send_timeout` in API vhost           |

**Logs & checks**

```bash
# App logs
journalctl -u app-backend -n 100 --no-pager
# Check port
ss -ltnp | grep APP_PORT
# Nginx logs (CloudPanel path may vary)
ls -l /var/log/nginx/domains/
```

---

## Directory Layout

```
/home/SITE_USER_BACK/htdocs/API_DOMAIN/backend
  ├─ app/
  ├─ migrations/
  ├─ assets/
  ├─ .env
  ├─ .venv/
  └─ requirements.txt

/home/SITE_USER_FRONT/htdocs/FRONT_DOMAIN
  ├─ index.html
  ├─ main-*.js, chunk-*.js, styles-*.css
  └─ media/
```
