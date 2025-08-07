Native ShinyProxy Usage Stats Pipeline

This README documents in detail how to set up and maintain the usage‑statistics pipeline comprised of:

1. **ShinyProxy** (native JAR)  
   Inside ShinyProxy, you can launch two Docker-containerized apps:  
   - **Nginx**  
   - **Redmine**
2. **InfluxDB** (v1.8.10, installed via .deb)
3. **Grafana** services for Grafana and demo apps (nginx & Redmine)

Each component’s configuration and dependencies are described, along with how they interconnect and where to find key files.


## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Prerequisites](#prerequisites)
4. [InfluxDB Installation & Configuration](#influxdb-installation--configuration)
   - Install .deb
   - Configure `influxdb.conf`
   - Create the database
5. [ShinyProxy Configuration](#shinyproxy-configuration)
   - `application.yml` snippet for usage stats
   - Running the JAR
6. [Docker‑Compose Services](#docker-compose-services)
   - Grafana
   - Demo Apps (nginx, Redmine)
   - Networking details
7. [Grafana Setup](#grafana-setup)
   - Data source connection
   - Dashboards (TODO)
8. [Usage & Queries](#usage--queries)
9. [Troubleshooting & Maintenance](#troubleshooting--maintenance)

---

## Overview

This pipeline captures each ShinyProxy user event—login, logout, app start, and app stop—and writes it directly into InfluxDB. Grafana then queries InfluxDB every 10 seconds to visualize usage metrics and session durations (calculated via start/stop timestamps) via custom InfluxQL queries.

- **ShinyProxy** pushes usage entries to InfluxDB using the `usage-stats` block in `application.yml`.
- **InfluxDB** stores events in the `shinyproxy_usagestats` database, specifically within the `event` table.
- **Grafana** runs in Docker and connects InfluxDB as a data source, with dashboards built manually in the UI.

## Architecture Diagram


![alt text][/excalidraw-diagram/diagram.png]

---

## Prerequisites

- **Java 11+** (to run ShinyProxy JAR)
- **Debian/Ubuntu** host for InfluxDB
- **Docker & Docker‑Compose** (v3.7+) for Grafana and demo containers
- **Ports**:
  - `8081` (ShinyProxy)
  - `8086` (InfluxDB HTTP API)
  - `3001` (Grafana UI)

---

## InfluxDB Installation & Configuration

### 1. Install via `.deb`

```bash
# From project root:
sudo dpkg -i influxdb_1.8.10_amd64.deb
sudo apt-get install -f    # install any missing dependencies
```

### 2. Configure `influxdb.conf`

- Replace the default config at `/etc/influxdb/influxdb.conf` with the provided `influxdb.conf`.
- Key settings:
  - `bind-address = ":8086"` (HTTP API)
  - Retention, monitoring, and write settings (see the file for full defaults).

```bash
sudo cp path/to/influxdb.conf /etc/influxdb/influxdb.conf
sudo systemctl restart influxdb
```

### 3. Create the database

```bash
# Ensure InfluxDB is running
influx -execute "CREATE DATABASE \"shinyproxy_usagestats\""
```

- Database: `shinyproxy_usagestats`
- Measurement: `event` (fields: `data`, `type`, `username`)

---

## ShinyProxy Configuration

### `application.yml` Usage‑Stats Snippet

```yaml
proxy:
  title: Native ShinyProxy with Metrics
  port: 8081
  usage-stats:
    - url: http://localhost:8086/write?db=shinyproxy_usagestats
      attributes:
        - name: username
          expression: "#{authentication.name}"
  authentication: simple
  admin-groups: admins
  container-warmup-time: 100000
  container-wait-retries: 30
  container-wait-time: 900000
  users:
    - name: jack
      password: password
      groups: admins
    - name: user
      password: password
    - name: jonah
      password: password

  specs:
    - id: redmine-app
      display-name: Redmine Test App (SQLite)
      description: A self contained web app that will start without a database
      container-image: redmine
      port: 3000
      container-env:
        REDMINE_SECRET_KEY_BASE: "this-is-a-long-and-super-secret-key-for-testing"

    - id: nginx-demo
      display-name: Nginx Demo
      description: A simple Nginx welcome page
      container-image: nginxdemos/hello:latest
      port: 80
```

### Running the JAR

```bash
java -jar shinyproxy-3.1.1.jar --spring.config.location=application.yml
```

- ShinyProxy will automatically push login, logout, app start, and stop events to InfluxDB.

---

## Docker‑Compose Services

The `docker-compose.yml` in the `shinyproxy-native-demo` directory defines Grafana and the two demo containers.

```yaml
version: '3.7'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - shiny-monitoring
    extra_hosts:
      - "host.docker.internal:host-gateway"

networks:
  shiny-monitoring:

volumes:
  grafana-data:
```

- **Grafana** UI available at `http://localhost:3001`
- **nginx-demo** and **redmine-app** launched via ShinyProxy but connected to the same Docker network (`shiny-monitoring`) for inter-container networking.

---

## Grafana Setup

1. **Add Data Source**

   - Type: InfluxDB
   - URL: `http://host.docker.internal:8086`
   - Database: `shinyproxy_usagestats`
   - (Auth disabled by default)

2. **Dashboards**

   - *(Dashboards are configured manually in the Grafana UI.)*
   - **TODO:** Document each dashboard’s panels, panels’ InfluxQL queries, and folder structure.

---

## Usage & Queries

### Sample InfluxQL

```sql
-- Show raw events (limit 100):
SELECT * FROM "event" LIMIT 100;

-- Count ProxyStart events per app in the last 1h:
SELECT COUNT("type")
  FROM "event"
  WHERE "type" = 'ProxyStart' AND time > now() - 1h
  GROUP BY "data";

-- Session duration (ProxyStop - ProxyStart) calculated in Grafana
-- via two queries and transform in the dashboard.
```

---

## Troubleshooting & Maintenance

- **InfluxDB not receiving writes?**

  1. Verify ShinyProxy logs: look for HTTP 204 from `localhost:8086/write?db=...`
  2. Confirm InfluxDB is listening on port 8086 (`netstat -tlnp`).

- **Grafana data source errors?**

  1. Ensure Docker network and `host.docker.internal` resolution.
  2. Check Grafana logs (container logs via `docker logs grafana`).

- **Upgrading InfluxDB**:

  - Install new `.deb`, merge or review `influxdb.conf`, restart service.

---

*End of README draft. Please review and let me know if any details are missing or need adjustment.*

