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
- Measurement: `event` (fields: `data`, `type`, `username`, `time`)

---

## ShinyProxy Configuration

### `application.yml` is the config file for ShinyProxy.
   - `proxy.specs` defines what images are to be pulled from Docker Hub and made available inside ShinyProxy.
   - `proxy.users` defines username and password credentials for logging in.
      - For instance, 'jonah' and 'password' is a valid login.
   - `proxy.usage-stats` defines how usage stats are exposed for InfluxDB to record them.
      - `proxy.usage-stats.url : http://localhost:8086/write?db=shinyproxy_usagestats`. This config line tells ShinyProxy to write useage stats to the local host port 8086, and specifically to the InfluxDB database `shinyproxy_usagestats`.
   - `proxy.port' defines what port on the local machine ShinyProxy can be accessed on once running. Set to `8081`
      - Go to `http://localhost:8081/`

### Running the JAR

```bash
java -jar shinyproxy-3.1.1.jar --spring.config.location=application.yml
```

- ShinyProxy will automatically push login, logout, app start, and stop events to InfluxDB.

---


## Grafana Setup
1. **Get it running**
   The `docker-compose.yml` in the `shinyproxy-native-demo` directory defines Grafana. In the CLI, run `docker-compose up -d`.\UI available at `http://localhost:3001`.
   
2. **Add Data Source**
   - Type: InfluxDB
   - URL: `http://host.docker.internal:8086`
   - Database: `shinyproxy_usagestats`
   - Query Language: `Flux`
   - (Auth disabled by default)

3. **Dashboards**
   - To add a dashboard, start a new project in Grafana and select the above data source. Add a Flux query, and select premade dashboards to visualize them.
   - Flux commands can be found in `/flux-queries`
      - `/flux-queries/aggregate_time_by_user.flux`
         -
         ```bash
           from(bucket: "shinyproxy_usagestats")
             |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
             |> filter(fn: (r) =>
                 r._measurement == "event" and
                 (r.type == "ProxyStart" or r.type == "ProxyStop")
               )
             |> group(columns: ["username"])
             |> sort(columns: ["_time"])
             |> elapsed(unit: 1s)
             |> filter(fn: (r) => r.type == "ProxyStop")
             |> sum(column: "elapsed")
           
             // ← add this to merge all your tables into one
             |> group()
           
             |> yield(name: "usage_by_user_and_app")
           ```


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

