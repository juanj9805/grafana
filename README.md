# Grafana Monitoring Stack

Docker-based monitoring stack using Prometheus, Grafana, and Node Exporter.

## Architecture

```
Node Exporter → Prometheus → Grafana
  (collects)     (stores)    (displays)
```

| Service       | Port | Role                              |
|---------------|------|-----------------------------------|
| Grafana       | 3000 | Visualization and dashboards      |
| Prometheus    | 9090 | Metrics storage and querying      |
| Node Exporter | 9100 | Exposes host system metrics       |

---

## Prerequisites

- Docker and Docker Compose installed on your server
- Ports 3000, 9090, and 9100 open on your firewall

---

## Step 1 — Start the stack

```bash
cd monitoring
docker compose up -d
```

Verify all containers are running:

```bash
docker ps
```

You should see `grafana`, `prometheus`, and `node-exporter` all with status `Up`.

---

## Step 2 — Verify Prometheus targets

Open in your browser:

```
http://<your-server-ip>:9090/targets
```

Both `prometheus` and `node-exporter` should show state **UP**.

The `docker` job may show as DOWN — this is expected unless Docker metrics are explicitly enabled on the host.

---

## Step 3 — Connect Grafana to Prometheus

1. Open Grafana: `http://<your-server-ip>:3000`
2. Login with `admin` / `admin` — you will be prompted to change the password
3. Go to **Connections → Data sources → Add data source**
4. Select **Prometheus**
5. Set the URL to:
   ```
   http://prometheus:9090
   ```
6. Click **Save & test**

Expected result: `Successfully queried the Prometheus API.`

> Grafana and Prometheus communicate using the container name `prometheus` because both are on the same Docker network (`monitoring`).

---

## Step 4 — Import a pre-built dashboard

1. Go to **Dashboards → New → Import**
2. Enter dashboard ID:
   ```
   1860
   ```
3. Click **Load**
4. Under the Prometheus field, select your Prometheus data source
5. Click **Import**

This imports the **Node Exporter Full** dashboard — a community dashboard with panels for CPU, memory, disk I/O, network, and more.

---

## Step 5 — Explore your metrics

Once the dashboard is loaded you can see:

- CPU usage per core
- Memory and swap usage
- Disk read/write throughput
- Network traffic in/out
- System load average

You can also query metrics manually at:

```
http://<your-server-ip>:9090
```

Example PromQL queries:

```promql
# CPU usage percentage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Available memory in GB
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# Disk usage percentage
100 - ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes)
```

---

## Stopping the stack

```bash
cd monitoring
docker compose down
```

To also remove stored Grafana data:

```bash
docker compose down -v
```
