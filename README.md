

```markdown
# Local Grafana Enterprise & Image Renderer Setup with Docker

This guide details how to run **Grafana Enterprise** locally with **Docker Compose**, alongside the **Grafana Image Renderer**, configured for an HTTPS domain (`https://grafana.local:3000`) matching an Enterprise license with PDF and image export enabled.

---

## Architecture Highlights

- **Native TLS Termination**: Grafana terminates HTTPS directly on port 3000 (no NGINX required).
- **Network Aliasing**: The `renderer` container resolves `grafana.local` back to the Grafana container inside Docker's internal bridge network.
- **Renderer Security & TLS Bypass**: 
  - A shared secret authentication token (`AUTH_TOKEN` / `GF_RENDERING_RENDERER_TOKEN`) authenticates render requests.
  - Chromium is instructed via `BROWSER_FLAG` to bypass self-signed certificate warnings.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running.
- macOS terminal (or Linux equivalent).
- OpenSSL (pre-installed on macOS).
- Your Grafana Enterprise license file (`license.jwt`).

---

## Step 1: Update `/etc/hosts`

Map `grafana.local` to `127.0.0.1` on your host machine:

```bash
sudo nano /etc/hosts
```

Add the following line:

```text
127.0.0.1   grafana.local
```

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

---

## Step 2: Create Project Structure

Create the project directory and the required subdirectories:

```bash
mkdir -p grafana-enterprise/{certs,license,data}
cd grafana-enterprise
```

Your project directory will look like this:

```text
grafana-enterprise/
├── certs/
│   ├── grafana.crt
│   └── grafana.key
├── license/
│   └── license.jwt
├── data/
└── docker-compose.yml
```

---

## Step 3: Generate Self-Signed TLS Certificates

Modern browsers and Chromium require the **Subject Alternative Name (SAN)** extension. Generate a 2048-bit RSA key and a self-signed certificate valid for `grafana.local`:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/grafana.key \
  -out certs/grafana.crt \
  -subj "/CN=grafana.local" \
  -addext "subjectAltName=DNS:grafana.local,IP:127.0.0.1"
```

Set permissions so the non-root container user (`472:472`) can read them:

```bash
chmod 644 certs/grafana.crt certs/grafana.key
```

*(Optional macOS Step)*: To prevent the "Your connection is not private" warning in your host browser:
1. Double-click `certs/grafana.crt` to open it in **Keychain Access**.
2. Find `grafana.local` under **Certificates**.
3. Double-click it, expand **Trust**, and change **When using this certificate** to **Always Trust**.

---

## Step 4: Add Your Enterprise License

Copy your `license.jwt` file into the `license/` folder:

```bash
cp /path/to/your/license.jwt license/license.jwt
chmod 644 license/license.jwt
```

---

## Step 5: Create `docker-compose.yml`

Create `docker-compose.yml` in the `grafana-enterprise` root directory:

```yaml
services:
  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana-enterprise
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      # --- Server & HTTPS Configuration ---
      GF_SERVER_PROTOCOL: https
      GF_SERVER_HTTP_PORT: 3000
      GF_SERVER_DOMAIN: grafana.local
      GF_SERVER_ROOT_URL: https://grafana.local:3000/
      GF_SERVER_CERT_FILE: /etc/grafana/certs/grafana.crt
      GF_SERVER_CERT_KEY: /etc/grafana/certs/grafana.key

      # --- Enterprise License Configuration ---
      GF_ENTERPRISE_LICENSE_PATH: /etc/grafana/license/license.jwt

      # --- Image Renderer Configuration ---
      GF_RENDERING_SERVER_URL: http://renderer:8081/render
      GF_RENDERING_CALLBACK_URL: https://grafana.local:3000/
      GF_RENDERING_RENDERER_TOKEN: "secret-token-grafana-renderer-123"
      GF_LOG_FILTERS: "rendering:debug"
    volumes:
      - ./certs:/etc/grafana/certs:ro
      - ./license:/etc/grafana/license:ro
      - ./data:/var/lib/grafana
    networks:
      grafana-net:
        aliases:
          # Allows renderer to resolve 'grafana.local' over Docker bridge network
          - grafana.local
    depends_on:
      - renderer

  renderer:
    image: grafana/grafana-image-renderer:latest
    container_name: grafana-renderer
    restart: unless-stopped
    environment:
      # Shared secret with Grafana
      AUTH_TOKEN: "secret-token-grafana-renderer-123"
      # Tells Chromium in Go-based renderer to bypass self-signed cert validation
      BROWSER_FLAG: "--ignore-certificate-errors"
      BROWSER_FLAGS: "--ignore-certificate-errors"
    networks:
      - grafana-net

networks:
  grafana-net:
    driver: bridge
```

---

## Step 6: Start the Services

Start the containers in detached mode:

```bash
docker compose up -d
```

Verify that both containers started cleanly:

```bash
docker compose logs -f
```

---

## Step 7: Verify & Test

1. **Access Grafana**: Open [https://grafana.local:3000](https://grafana.local:3000) in your browser.
2. **Login**: 
   - **Username**: `admin`
   - **Password**: `admin` (you will be prompted to change it).
3. **Verify License**:
   - Navigate to **Administration** > **General** > **Stats and license**.
   - Verify that your Enterprise license is active and recognized.
4. **Test Image Export**:
   - Open any dashboard with a panel.
   - Click the panel title menu (`...`) > **Share** > **Direct link rendered image**.
5. **Test PDF Export**:
   - From any dashboard, click the dashboard share/report icon in the top bar.
   - Select **Export to PDF** (or **Generate Report**).

---

## Stopping or Restarting

To stop the services:
```bash
docker compose down
```

To stop and remove data volumes:
```bash
docker compose down -v
```
```
