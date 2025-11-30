# ERPNext mit Traefik v3 und CrowdSec Installation

## 1. Grundvoraussetzung

- Docker & Docker Compose v2
  - [Debian / Bullseye 11](https://goneuland.de/docker-docker-compose-v2-auf-debian-bullseye-11-installieren/)
  - [Ubuntu 22.04 LTS](https://goneuland.de/docker-docker-compose-v2-auf-ubuntu-22-04-lts-installieren/)
  - [Traefik V3 Installation, Konfiguration und CrowdSec-Security](https://goneuland.de/traefik-v3-installation-konfiguration-und-crowdsec-security/)
- Root oder Sudo-Zugriff

## 2. Verzeichnisse anlegen

```bash
mkdir -p /opt/containers/erpnext/{data,compose}
cd /opt/containers/erpnext/
```

## 3. Dateien erstellen

```bash
touch .env docker-compose.yml compose/erpnext.yml
touch /opt/containers/traefik-crowdsec-stack/data/traefik/dynamic_conf/http.middlewares.erpnext-headers.yml
```

## 4. Inhalt der Dateien

### 4.1 `.env` Datei

```env
# filepath: /opt/containers/erpnext/.env

# Absolute Path
ABSOLUTE_PATH=/opt/containers/erpnext

# ERPNext Configuration
ERPNEXT_VERSION=v15.43.3
FRAPPE_VERSION=v15.43.4

# Timezone
TZ=Europe/Berlin

# Database Credentials - ÄNDERE DIESE ZU SICHEREN PASSWÖRTERN!
DB_PASSWORD=ErpN3xt_DB_S3cur3_P@ssw0rd_2024!
DB_ROOT_PASSWORD=ErpN3xt_R00t_Sup3r_S3cur3_P@ss_2024!

# ERPNext Site Configuration - ÄNDERE DIESE ZU DEINER DOMAIN!
SITE_NAME=erp.domain.de
ADMIN_PASSWORD=ErpN3xt_Admin_Str0ng_P@ssw0rd!

# Traefik Network
TRAEFIK_NETWORK=proxy
```

**⚠️ Wichtig:** Ersetze folgende Werte:
- `erp.domain.de` → Deine tatsächliche Domain
- `DB_PASSWORD` → Starkes Passwort
- `DB_ROOT_PASSWORD` → Starkes Passwort
- `ADMIN_PASSWORD` → Starkes Passwort

---

### 4.2 `docker-compose.yml` Datei

```yaml
# filepath: /opt/containers/erpnext/docker-compose.yml

include:
  - compose/erpnext.yml
```

---

### 4.3 `compose/erpnext.yml` Datei

```yaml
# filepath: /opt/containers/erpnext/compose/erpnext.yml

services:
  mariadb:
    image: mariadb:10.6
    container_name: erpnext-db
    hostname: erpnext-db
    restart: unless-stopped
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed
      - --bind-address=0.0.0.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/mariadb:/var/lib/mysql
    networks:
      - erpnext-internal
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis-cache:
    image: redis:7-alpine
    container_name: erpnext-redis-cache
    hostname: erpnext-redis-cache
    restart: unless-stopped
    environment:
      TZ: ${TZ}
    networks:
      - erpnext-internal
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  redis-queue:
    image: redis:7-alpine
    container_name: erpnext-redis-queue
    hostname: erpnext-redis-queue
    restart: unless-stopped
    environment:
      TZ: ${TZ}
    networks:
      - erpnext-internal
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  redis-socketio:
    image: redis:7-alpine
    container_name: erpnext-redis-socketio
    hostname: erpnext-redis-socketio
    restart: unless-stopped
    environment:
      TZ: ${TZ}
    networks:
      - erpnext-internal
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  configurator:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-configurator
    hostname: erpnext-configurator
    restart: "no"
    user: root
    entrypoint:
      - bash
      - -c
    command:
      - |
        chown -R frappe:frappe /home/frappe/frappe-bench/sites
        su frappe -c '
        cd /home/frappe/frappe-bench
        ls -1 apps > sites/apps.txt
        echo "{}" > sites/common_site_config.json
        bench set-config -g db_host mariadb
        bench set-config -gp db_port 3306
        bench set-config -g redis_cache "redis://redis-cache:6379"
        bench set-config -g redis_queue "redis://redis-queue:6379"
        bench set-config -g redis_socketio "redis://redis-socketio:6379"
        bench set-config -gp socketio_port 9000
        echo "✅ Configuration completed successfully"
        '
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      mariadb:
        condition: service_healthy
      redis-cache:
        condition: service_healthy
      redis-queue:
        condition: service_healthy
      redis-socketio:
        condition: service_healthy

  create-site:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-create-site
    hostname: erpnext-create-site
    restart: "no"
    user: frappe
    entrypoint:
      - bash
      - -c
    command:
      - |
        cd /home/frappe/frappe-bench
        export start=$(date +%s)
        echo "⏳ Waiting for common_site_config.json..."
        until [ -f sites/common_site_config.json ] && 
              grep -q "db_host" sites/common_site_config.json &&
              grep -q "redis_cache" sites/common_site_config.json &&
              grep -q "redis_queue" sites/common_site_config.json; do
          sleep 5
          if [ $(($(date +%s)-start)) -gt 120 ]; then
            echo "❌ Timeout: common_site_config.json not found"
            exit 1
          fi
        done
        echo "✅ common_site_config.json found"
        echo "🚀 Creating site ${SITE_NAME}..."
        bench new-site ${SITE_NAME} --no-mariadb-socket --admin-password="${ADMIN_PASSWORD}" --db-root-password="${DB_ROOT_PASSWORD}" --install-app erpnext --set-default
        echo "⚙️  Enabling scheduler..."
        bench --site ${SITE_NAME} scheduler enable
        echo "🔓 Disabling maintenance mode..."
        bench --site ${SITE_NAME} set-maintenance-mode off
        echo "✅ Site creation completed successfully"
    environment:
      SITE_NAME: ${SITE_NAME}
      DB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      configurator:
        condition: service_completed_successfully

  backend:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-backend
    hostname: erpnext-backend
    restart: unless-stopped
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
      - proxy
    depends_on:
      create-site:
        condition: service_completed_successfully

  websocket:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-websocket
    hostname: erpnext-websocket
    restart: unless-stopped
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      create-site:
        condition: service_completed_successfully

  frontend:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-frontend
    hostname: erpnext-frontend
    restart: unless-stopped
    command:
      - nginx-entrypoint.sh
    environment:
      BACKEND: backend:8000
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 172.20.0.0/16
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "on"
      FRAPPE_SITE_NAME_HEADER: ${SITE_NAME}
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
      - proxy
    depends_on:
      - backend
      - websocket
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.erpnext.rule=Host(`${SITE_NAME}`)"
      - "traefik.http.routers.erpnext.entrypoints=websecure"
      - "traefik.http.routers.erpnext.tls.certresolver=http_resolver"
      - "traefik.http.routers.erpnext.middlewares=default@file,erpnext-headers@file"
      - "traefik.http.routers.erpnext.service=erpnext"
      - "traefik.http.services.erpnext.loadbalancer.server.port=8080"
      - "traefik.http.services.erpnext.loadbalancer.server.scheme=http"

  queue-short:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-queue-short
    hostname: erpnext-queue-short
    restart: unless-stopped
    command:
      - bench
      - worker
      - --queue
      - short
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      create-site:
        condition: service_completed_successfully

  queue-default:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-queue-default
    hostname: erpnext-queue-default
    restart: unless-stopped
    command:
      - bench
      - worker
      - --queue
      - default
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      create-site:
        condition: service_completed_successfully

  queue-long:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-queue-long
    hostname: erpnext-queue-long
    restart: unless-stopped
    command:
      - bench
      - worker
      - --queue
      - long
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      create-site:
        condition: service_completed_successfully

  scheduler:
    image: frappe/erpnext:${ERPNEXT_VERSION}
    container_name: erpnext-scheduler
    hostname: erpnext-scheduler
    restart: unless-stopped
    command:
      - bench
      - schedule
    environment:
      TZ: ${TZ}
    volumes:
      - ${ABSOLUTE_PATH}/data/sites:/home/frappe/frappe-bench/sites
    networks:
      - erpnext-internal
    depends_on:
      create-site:
        condition: service_completed_successfully

networks:
  erpnext-internal:
    name: erpnext-internal
    driver: bridge
  proxy:
    external: true
```

---

### 4.4 Traefik Middleware Datei

```yaml
# filepath: /opt/containers/traefik-crowdsec-stack/data/traefik/dynamic_conf/http.middlewares.erpnext-headers.yml

http:
  middlewares:
    erpnext-headers:
      headers:
        customRequestHeaders:
          X-Frappe-Site-Name: "erp.domain.de"
          X-Forwarded-Proto: "https"
        customResponseHeaders:
          X-Frame-Options: "SAMEORIGIN"
          Referrer-Policy: "strict-origin-when-cross-origin"
```

**⚠️ Wichtig:** Ersetze `erp.domain.de` durch deine tatsächliche Domain!

---

## 5. Traefik-CrowdSec-Stack neu starten

Nach dem Anlegen und Anpassen der Middleware-Datei den Traefik-Stack neu starten:

```bash
docker compose -f /opt/containers/traefik-crowdsec-stack/docker-compose.yml up -d --force-recreate
```

---

## 6. ERPNext starten

```bash
cd /opt/containers/erpnext
docker compose up -d
```

**⏱️ Warte 5-10 Minuten** – Der erste Start dauert länger!

Nach dem Start sollte ERPNext unter deiner Domain erreichbar sein: `https://erp.domain.de`

---

## 🎯 Zusammenfassung

| Schritt | Befehl |
|---------|--------|
| Verzeichnisse anlegen | `mkdir -p /opt/containers/erpnext/{data,compose} && cd /opt/containers/erpnext/` |
| Dateien erstellen | `touch .env docker-compose.yml compose/erpnext.yml` |
| Traefik neu starten | `docker compose -f /opt/containers/traefik-crowdsec-stack/docker-compose.yml up -d --force-recreate` |
| ERPNext starten | `docker compose up -d` |