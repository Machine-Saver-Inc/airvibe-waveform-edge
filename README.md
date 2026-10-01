# AirVibe_Waveform_Edge

<!-- machine-saver-scope:start -->
## Scope

Existing AirVibe waveform edge-service implementation: API, MQTT broker and deployment configuration for its application.

**Owner:** Machine-Saver-Inc. **Development area:** Applications and documentation.

## Ownership boundaries

This application-specific runtime is distinct from the platform fleet stack. Any consolidation requires an explicit migration plan and compatibility evidence.

## Development tracking

Track work in this repository's issues and pull requests. Cross-repository work is coordinated through the [Machine Saver development Projects](https://github.com/orgs/Machine-Saver-Inc/projects).

Follow this repository's contribution instructions and preserve links to related product issues. Scope describes responsibility; release and deployment readiness require the repository's own evidence.
<!-- machine-saver-scope:end -->

**Edge runtime for AirVibe time-waveform ingestion**: MQTT broker (Mosquitto), TWAB API (segment assembly + downlinks), TLS/mTLS, and CI/CD. Pairs with the UI repo: [`AirVibe_Waveform_React`](https://github.com/Machine-Saver-Inc/AirVibe_Waveform_React).

* **UI →** GitHub Pages at `ui.<domain>` (protected by Cloudflare Access)
* **Edge →** VPS at `edge.<domain>` running **API** (HTTPS) + **Broker** (MQTTS)
* **Data (Phase 1)** → SQLite + local disk
* **TLS** → Let’s Encrypt (server TLS) + Private CA (client mTLS for Actility)

---

## TL;DR

1. Run Actions: **`dns-apply`**, **`dns-check`**, **`access-apply`** to set up `ui.` and `edge.` and gate UI with Cloudflare Access.
2. Bootstrap VPS and bring up **Caddy + Mosquitto + API** via `docker compose`.
3. Run Action **`issue-connector-cert`** to generate a **private** client cert/key bundle for Actility.
4. Create the Actility MQTT connector with the provided files and fields.
5. Send uplinks → watch segments assemble, trigger downlinks, download assembled waveforms.

---

## What this repo contains

* `docker-compose.yml` — runs **caddy**, **mosquitto**, **twab-api**
* `Caddyfile` — HTTPS termination + API reverse proxy
* `mosquitto.conf`, `aclfile` — broker TLS/mTLS + topic ACLs
* `scripts/mk-ca.sh`, `scripts/mk-client.sh` — private CA + per-connector client cert issuance
* `.github/workflows/` — Actions for **DNS**, **Access**, **Deploy**, **Issue Connector Cert**
* `Makefile` — `make up|down|logs|restart`

---

## Prerequisites

* **Domain** managed in **Cloudflare** (API token with DNS)
* **VPS** (Ubuntu 22.04/24.04) with public IPv4
* **GitHub** repo secrets:

  * `CF_API_TOKEN`, `CF_ZONE_ID`, `CF_ACCOUNT_ID`
  * `VPS_EDGE_IPV4`
  * `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY` (for deploy & cert workflows)
  * (optional) `GHCR_TOKEN` if pushing to GHCR

---

## Milestone 0 — DNS & Access (via Actions)

1. **Run `dns-apply`** (workflow dispatch):

   * `UI_SUBDOMAIN=ui`, `EDGE_SUBDOMAIN=edge`, `PAGES_TARGET=<org>.github.io`
2. **Run `dns-check`** to verify A/CNAME + basic HTTP reachability.
3. **Configure GitHub Pages** in the UI repo with custom domain `ui.<domain>`.
4. **Run `access-apply`** to create Cloudflare Access app + allow policy (OTP or your email domain).

> Result: `ui.<domain>` is gated by Cloudflare Access; `edge.<domain>` points to your VPS.

---

## Milestone 1 — Edge stack on the VPS

SSH once, then:

```bash
sudo apt-get update && sudo apt-get install -y docker.io docker-compose-plugin git certbot
mkdir -p ~/twab && cd ~/twab
# Add: docker-compose.yml, Caddyfile, mosquitto.conf, aclfile, Makefile, .env, scripts/
sudo mkdir -p /var/www/certbot && sudo chown $USER:$USER /var/www/certbot

# Get a server cert for edge.<domain> (webroot for Caddy/HTTP-01)
sudo certbot certonly --webroot -w /var/www/certbot -d edge.<domain> --agree-tos -m you@domain --non-interactive

make up   # starts caddy, mosquitto, api
```

**.env (sample)**

```env
DOMAIN=example.com
EDGE_HOST=edge.example.com
MQTT_TLS_PORT=8883

# API config
API_PORT=8080
STORAGE_DIR=/app/data
SQLITE_PATH=/app/data/twab.sqlite
RETENTION_DAYS=7
```

---

## Certificates & Actility connector

**Server TLS:**

* Caddy terminates HTTPS on `edge.<domain>` using Let’s Encrypt.
* Mosquitto uses LE certs for `:8883` (server TLS).

**Client mTLS (Actility):**

* Run once: `scripts/mk-ca.sh` to create a **private CA**.
* Then use the **`issue-connector-cert`** Action (input `cn`, e.g., `actility-prod-YYYY-MM-DD`) which:

  * SSHes to the VPS and runs `mk-client.sh <cn>`
  * Pulls back **`client.crt`**, **`client.key.pk8`** (PKCS#8), and **`ca.crt`**
  * Publishes a **private artifact** for download (expires)

**Actility connector fields**

* **Hostname:** `edge.<domain>:8883`
* **Protocol:** SSL/TLS
* **CA Certificate:** `ca.crt` (your private CA)
* **Client Certificate:** `client.crt`
* **Private Key:** `client.key.pk8` (PKCS#8, unencrypted)
* **Username/Password:** blank
* **Published Topic:** `mqtt/things/{DevEUI}/uplink`
* **Subscribed Topic:** `mqtt/things/{DevEUI}/downlink`
* **QoS:** 1 (everywhere)

> **Do not** publish private keys on the public UI; downloads come from the workflow artifact or an authenticated API route.

---

## API contract (Phase 1)

* `GET /api/healthz` → `{status:"ok"}`
* `GET /api/devices` → `[ { dev_eui, last_seen, last_txn_id } ]`
* `GET /api/transactions/:id` → `{ id, dev_eui, expected_segments, received_segments, missing:[...], status }`
* `GET /api/events` (SSE) → `segment_received | missing_updated | transaction_done`
* `GET /api/waveforms/:id/download` → assembled file
* `POST /api/transactions/:id/resend-missing` → publishes missing list (downlink)
* `POST /api/downlink` → `{ dev_eui, fport, payload_hex }`

**Topics**

* **Uplink (Actility → broker):** `mqtt/things/{DevEUI}/uplink`
* **Downlink (API → broker):** `mqtt/things/{DevEUI}/downlink`
* **QoS:** 1; **retain:** off for commands; optional retain for `/status`.

---

## Data & retention (Phase 1)

* **SQLite** file: `data/twab.sqlite`
* **Segments:** `data/segments/<DevEUI>/<txnId>/<idx>.bin`
* **Waveforms:** `data/waveforms/<DevEUI>/<txnId>.<ext>`
* **Janitor:** daily cleanup for DONE/EXPIRED beyond `RETENTION_DAYS` (default 7)

---

## Make targets

```bash
make up        # start caddy + mosquitto + api
make down      # stop stack
make logs      # follow logs
make restart   # redeploy
```

---

## Security notes

* Public UI is gated by **Cloudflare Access** (OTP or SSO).
* Broker accepts **only** clients signed by your **private CA** (Actility + optionally TWAB).
* API is the only public write path for downlinks.
* **Never** expose private keys on GitHub Pages.

---

## Roadmap

* Phase 1: working ingestion, retries, assembly, download (SQLite + disk)
* Phase 2: Postgres + object storage (S3/MinIO), metrics, per-connector cert rotation
* Phase 3: Multi-tenant, RBAC, audit trails, dashboards

---

## License

MIT

---

## Related

* UI: [`AirVibe_Waveform_React`](https://github.com/Machine-Saver-Inc/AirVibe_Waveform_React)

---
