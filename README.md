# ShinyProxy Usage Stats Pipeline
---
## Overview

This README documents in detail how to set up and maintain the usage‑statistics pipeline comprised of:

1. **ShinyProxy** (native JAR)\
   ShinyProxy is an application that acts as a homepage to run Docker containers from.
   In this project, ShinyProxy is set up to launch two Docker-containerized apps:  
   - Nginx  
   - Redmine
3. **InfluxDB** (v1.8.10, installed via .deb)
   - Database to store ShinyProxy useage statistics
4. **Grafana** services for Grafana and demo apps (nginx & Redmine)
   Application to visualize dashboards of data pulled from InfluxDB

Each component’s configuration and dependencies are described, along with how they interconnect and where to find key files.


## Architecture Diagram  
### Along with commands to start ShinyProxy and Grafana, and the URLs to see them running.

![alt text](/excalidraw-diagram/diagram.png)



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
[Link to official ShinyProxy usage statistics](https://shinyproxy.io/documentation/usage-statistics/#influxdb)

### 1. Configure `influxdb.conf`

- We need to modify two lines in the default config at `/etc/influxdb/influxdb.conf`. By default, InfluxDB does not allow us to use the Flux query language (not to be confused with InfluxQL). The queries provided in `/flux-queries` use the Flux query language.
  - Commented lines indicate that those settings are used by default. To explicitly make InfluxDB do something else, we can uncomment a line to overide default settings.
  - Under `[http]`:
     - Find the commented settings `flux-enabled` and `flux-log-enabled`, uncomment them, and change their bool values to `true`
    
### 2. Create the database and table

```bash
# From project root:
sudo service influxdb start
# Make sure its running
sudo service influxdb status
```
In order to store the usage statistics, a specific database needs to be created on the InfluxDB instance. In this example we use the name `shinyproxy_usagestats`.
```bash
influx
# Then, once in the database prompt, enter:
CREATE DATABASE shinyproxy_usagestats
quit
```


### 3. To view the InfluxDB data

```bash
# This shows you all the tables in InfluxDB. ShinyProxy usage statistics are stored in the table `event`.
influx -database shinyproxy_usagestats -execute "SHOW MEASUREMENTS"

# This shows you all the entries that ShinyProxy has logged in InfluxDB.
influx -database shinyproxy_usagestats        -execute "SELECT * FROM \"event\" LIMIT 300"
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

