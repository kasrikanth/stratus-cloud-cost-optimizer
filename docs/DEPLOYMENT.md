# Production Deployment Guide

This guide shows one concrete way to deploy Stratus behind an HTTPS reverse proxy on Linux. The example uses **Caddy** as the reverse proxy. Nginx is an equivalent alternative; adapt the configuration accordingly.

**Important:** Do not use the Vite development server or `vite preview` as a production server. Build the frontend and serve the static files through the reverse proxy. Run the FastAPI backend as a supervised process on loopback only.

## Architecture

- Frontend static files are served by the reverse proxy.
- `/api` and `/health` are proxied to the FastAPI backend listening on `127.0.0.1:8000`.
- HTTPS is terminated at the reverse proxy.
- The backend binds only to loopback.
- SQLite is suitable for a single backend process. For shared or multi-process deployments, use PostgreSQL.

## Prerequisites

- A Linux server.
- Caddy (or Nginx) installed.
- Python 3.12 and a process supervisor such as systemd.
- A domain name pointing to the server (use placeholders below).
- The project cloned and the backend dependencies installed in a virtual environment.

## 1. Build the frontend

From the project root:

```bash
cd frontend
npm ci
npm run build
```

This produces static files in `frontend/dist/`.

## 2. Configure the backend

Create a `.env` in the project root (or use the setup script). For production:

```dotenv
DATABASE_URL=postgresql+psycopg://USERNAME:PASSWORD@HOST:5432/stratus?sslmode=verify-full
SECRET_KEY=your-long-random-secret
# Other settings as needed
```

Use a strong secret. Restrict file permissions on `.env` (e.g., `chmod 600 .env`).

Run migrations:

```bash
cd backend
source .venv/bin/activate
alembic upgrade head
```

## 3. Run the backend under supervision

Example systemd unit (`/etc/systemd/system/stratus-backend.service`):

```ini
[Unit]
Description=Stratus backend
After=network.target

[Service]
User=stratus
WorkingDirectory=/path/to/stratus/backend
EnvironmentFile=/path/to/stratus/.env
ExecStart=/path/to/stratus/backend/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start it. The backend must listen only on loopback.

## 4. Configure the reverse proxy (Caddy example)

Caddyfile:

```caddy
your-domain.example {
    # Serve frontend static files
    root * /path/to/stratus/frontend/dist
    file_server

    # Proxy API and health
    handle /api/* {
        reverse_proxy 127.0.0.1:8000
    }
    handle /health {
        reverse_proxy 127.0.0.1:8000
    }

    # Forwarded headers for HTTPS
    header {
        X-Forwarded-Proto {scheme}
        X-Forwarded-For {remote_host}
    }
}
```

Caddy obtains and renews certificates automatically. For Nginx, use a similar `proxy_pass` with `proxy_set_header` for `X-Forwarded-*` and configure TLS certificates.

## 5. CORS and forwarded headers

Configure the FastAPI app (if not already) to trust the reverse proxy's forwarded headers and to allow the public origin in CORS settings. The backend should see the correct scheme and host.

## 6. Database notes

- **SQLite:** Suitable only for a single backend process. Do not share the database file across processes.
- **PostgreSQL:** Recommended for any shared or multi-instance deployment. Update `DATABASE_URL` accordingly and ensure the database user has the necessary privileges.

## 7. Backups and secrets

- Back up the database regularly (PostgreSQL `pg_dump` or SQLite file copy when the process is stopped).
- Protect `.env` and any credential files with restrictive permissions (`chmod 600`).
- Never commit secrets. Use placeholders in documentation.

## 8. Verification

After deployment:

```bash
# Health endpoint
curl -f https://your-domain.example/health

# Frontend route (should return HTML)
curl -I https://your-domain.example/

# Authenticated API proxy (example; adjust token)
curl -H "Authorization: Bearer YOUR_TOKEN" https://your-domain.example/api/v1/organizations
```

Confirm the responses are correct and that HTTPS is enforced.

## Limitations

This is one example configuration. Adapt it to your environment, add monitoring, logging, and rate limiting as needed. Review `docs/END_TO_END.md` for product limitations before production use.
