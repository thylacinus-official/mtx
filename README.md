# Self-Hosted Matrix Stack (Tuwunel & LiveKit)

A self-hosted Docker Compose stack for deploying a Matrix server with support for group video calls (Matrix 2.0 / Element Call) via LiveKit.

---

## Stack Components

* **Caddy** — Reverse proxy, automatic SSL/TLS certificate provisioning, and HTTP/WebSocket routing.
* **Tuwunel** — Matrix homeserver.
* **Element Web** — Matrix web client (optional).
* **LiveKit Server** — SFU server for WebRTC media streams. Uses SSL/TLS certificates automatically generated and managed by Caddy for its built-in TURN server.
* **LiveKit JWT Service** — Authentication bridge service for Matrix users connecting to LiveKit.

---

## Requirements

1. **Docker** and **Docker Compose v2** (`docker compose version`).
2. DNS `A` record(s) pointing to your server (a single domain name is sufficient for the entire stack).
3. Open firewall ports:
    * **TCP 80, 443** — HTTP/HTTPS
    * **TCP 8448** — Matrix Federation
    * **UDP 3478** — LiveKit TURN
    * **TCP 7881** — LiveKit RTC TCP
    * **UDP 50000-60000** — LiveKit RTC UDP

---

## Installation and Launch

1. Copy the configuration template:
   ```bash
   cp .env.example .env
   ```

2. Generate secret keys:
   ```bash
   openssl rand -hex 32  # For MTX_LIVEKIT_SECRET
   openssl rand -hex 16  # For TUWUNEL_REGISTRATION_TOKEN
   ```

3. Fill in the required parameters in `.env`:
    * `LIVEKIT_RTC__NODE_IP` — Public IPv4 address of your server.
    * `MTX_DOMAIN` — Your base domain name (e.g., `example.com`).
    * `MTX_LIVEKIT_SECRET` — Generated 32-byte secret key.
    * `TUWUNEL_REGISTRATION_TOKEN` — Generated 16-byte token.

4. Launch the stack:

    * **Standard Setup** (without Element Web):
      ```bash
      docker compose up -d
      ```

    * **Setup with Element Web**:
      ```bash
      docker compose --profile web up -d
      ```

> **Customization Note:**  
> All additional parameters (port customization, separate domains per service, Docker image versions) are fully documented and commented inside `.env.example`.

---

## Advanced Configuration

### 1. Account Domain Different from Server Domain (Delegation)

If user accounts are registered on a root domain (e.g., `@user:example.com`), while the Matrix server operates on a dedicated subdomain (e.g., `matrix.example.com`):

1. Set the server domain and account domain name in `.env`:
   ```env
   MTX_DOMAIN=matrix.example.com
   MTX_TUWUNEL_SERVER_NAME=example.com
   ```

2. Configure delegation responses on the account domain's web server (`example.com`). Example Caddy configuration:

```caddyfile
    example.com {
        # Client delegation (Includes MSC4143 for LiveKit Foci)
        handle /.well-known/matrix/client {
            header Access-Control-Allow-Origin "*"
            header Content-Type "application/json"
            respond `{
                "m.homeserver": {
                    "base_url": "https://matrix.example.com/"
                },
                "org.matrix.msc4143.rtc_foci": [
                    {
                        "type": "livekit",
                        "livekit_service_url": "https://matrix.example.com/livekit/jwt"
                    }
                ]
            }`
        }
    
        # Server delegation (Federation)
        handle /.well-known/matrix/server {
            header Access-Control-Allow-Origin "*"
            header Content-Type "application/json"
            respond `{"m.server":"matrix.example.com:443"}`
        }
    }
   ```

### 2. Configuring Element Web

The stack automatically serves default `/config.json` for Element Web. To apply custom JSON configurations per domain, place configuration files in `conf/element/` following this pattern:

```text
conf/element/config.<your-domain>.json
```

Caddy will serve it at `https://<your-domain>/config.<your-domain>.json`.


### 3. Modular Caddy Extensions

You can add custom Caddy snippets or separate site blocks by placing them into ./conf/caddy/includes/ or ./conf/caddy/sites/.

---

## Maintenance

```bash
# View logs
docker compose logs -f

# Restart services
docker compose restart

# Recreate containers (apply .env or compose changes)
docker compose up -d --force-recreate
```