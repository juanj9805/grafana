# How the Monitoring Stack Works

This document explains the concepts behind the stack — not just how to run it, but why it's built this way and what each piece is actually doing.

---

## The Problem This Solves

When you run a server, things happen that you can't see in real time:
- CPU spikes when traffic increases
- Memory fills up slowly until the app crashes
- Disk runs out at 3am
- A service goes down and nobody notices for hours

You need a system that **continuously watches your server**, stores that data over time, and lets you **visualize and alert** on it. That's exactly what this stack does.

---

## What is Docker (and why use it here)

### The core idea

Normally, to run Prometheus on a server you would:
1. Download the binary
2. Install its dependencies
3. Configure it to start on boot
4. Hope it doesn't conflict with something else already installed

Docker solves this differently. A **container** is an isolated process that packages the application and everything it needs to run — dependencies, config, runtime — into a single unit. It runs on your server but is isolated from everything else.

Think of it like this: your server is a building, and each container is an apartment. Each apartment has its own kitchen, bathroom, and utilities. They share the same building infrastructure (the Linux kernel) but don't interfere with each other.

### Docker image vs container

- **Image**: the blueprint (read-only). Like a class in programming.
- **Container**: a running instance of that image. Like an object instantiated from a class.

When you run `docker compose up`, Docker pulls the images (if not cached) and starts containers from them.

### Docker Compose

Running containers one by one with `docker run` gets tedious when you have multiple services. **Docker Compose** lets you define all your services in a single `docker-compose.yml` file and start/stop them together with one command.

Your `docker-compose.yml` defines three services: Prometheus, Grafana, and Node Exporter. Each one becomes a container when you run `docker compose up`.

### Docker networks

By default, containers are isolated — they can't talk to each other. When you define a network in `docker-compose.yml`, Docker creates a private virtual network and connects the specified containers to it.

In your setup, all three containers are connected to a network called `monitoring`. Inside this network, containers can reach each other using their **container name** as a hostname. That's why in `prometheus.yml` you write `node-exporter:9100` instead of an IP — Docker resolves that name to the right container automatically.

This is the same concept as DNS, but scoped to your private Docker network.

### Docker volumes

Containers are ephemeral by default — if you stop and remove a container, all data inside it is lost. A **volume** is a persistent storage area that lives outside the container and survives restarts.

In your setup, Grafana uses a volume called `grafana-storage` to persist your dashboards and configuration. Prometheus doesn't have a volume configured, which means if you restart it, historical metrics are lost — something to improve later.

---

## The Three Services

### Node Exporter — the sensor

Node Exporter is a program that reads system-level metrics from the Linux kernel and exposes them over HTTP.

Your Linux kernel tracks everything happening on the machine: how much CPU each process uses, how much memory is free, how many bytes have been written to disk, how many packets went through the network interface. This data lives in special virtual files under `/proc` and `/sys`.

Node Exporter reads those files and serves the data at:
```
http://node-exporter:9100/metrics
```

If you open that URL you'll see thousands of raw lines like:
```
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_memory_MemAvailable_bytes 2147483648
node_filesystem_avail_bytes{mountpoint="/"} 53687091200
```

This format is called **Prometheus exposition format** — a plain text format that Prometheus knows how to read.

Node Exporter doesn't store anything. It just reads and exposes. It's a sensor, not a database.

Notice in `docker-compose.yml` that Node Exporter mounts `/proc`, `/sys`, and `/` from the host into the container:

```yaml
volumes:
  - /proc:/host/proc:ro
  - /sys:/host/sys:ro
  - /:/rootfs:ro
```

This is necessary because Node Exporter runs inside a container, which is isolated from the host. Without these mounts, it would only see the container's own (empty) metrics, not your actual server's. The `:ro` means read-only — the container can read these paths but cannot modify them. This is a security boundary.

---

### Prometheus — the database

Prometheus is a **time-series database**. A time-series database stores values indexed by time — every data point has a value and a timestamp.

This is different from a relational database like PostgreSQL. Instead of rows and columns representing entities, you have **metrics** that change over time. For example:

```
node_memory_MemAvailable_bytes at 12:00:00 → 2.1 GB
node_memory_MemAvailable_bytes at 12:00:05 → 2.0 GB
node_memory_MemAvailable_bytes at 12:00:10 → 1.9 GB
```

That's a time series — the same metric tracked across time.

#### How Prometheus collects data — the pull model

Most systems push data to a central collector. Prometheus does the opposite: it **pulls** (scrapes) data from targets on a schedule.

Every 5 seconds (configured in `prometheus.yml` with `scrape_interval: 5s`), Prometheus makes an HTTP GET request to each target's `/metrics` endpoint, parses the response, and stores the values with a timestamp.

This pull model has advantages:
- You can tell exactly when a target went down (it stopped responding)
- Targets don't need to know where Prometheus is — they just expose an endpoint
- Easy to test manually (you can curl any target directly)

#### prometheus.yml

This file tells Prometheus what to scrape:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

`node-exporter:9100` resolves to the Node Exporter container on the Docker network. Prometheus hits `http://node-exporter:9100/metrics` every 5 seconds.

#### PromQL

Prometheus comes with its own query language called **PromQL**. You use it to ask questions about your metrics. For example:

```promql
node_memory_MemAvailable_bytes
```
Returns the current available memory.

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```
Returns the per-second rate of CPU idle time over the last 5 minutes — from this you can calculate CPU usage percentage.

PromQL is designed for time-series data. It's not SQL, but the mental model is similar: you're filtering and transforming data, just across time instead of rows.

---

### Grafana — the visualization layer

Grafana is a dashboarding tool. It doesn't collect or store any metrics itself — it connects to data sources (like your Prometheus) and queries them to display graphs, gauges, tables, and alerts.

When you added Prometheus as a data source with URL `http://prometheus:9090`, you told Grafana: "when you need data, ask Prometheus at this address." Again, container name resolution via the Docker network.

When you load a dashboard, each panel runs a PromQL query against Prometheus and renders the result as a chart. The data is always live — Grafana queries Prometheus on every refresh.

#### Why import dashboard ID 1860

Dashboard 1860 is **Node Exporter Full**, maintained by the community. Someone already wrote all the PromQL queries for common system metrics and organized them into a clean layout. You get a production-quality dashboard in seconds instead of building it from scratch.

This is a pattern you'll see everywhere in the Prometheus ecosystem — a standard metrics format means dashboards are reusable across any infrastructure.

---

## How the Three Services Connect

```
┌─────────────────────────────────────────────────┐
│              Docker "monitoring" network         │
│                                                 │
│  ┌──────────────┐    scrapes     ┌────────────┐ │
│  │ Node Exporter│ ◄────────────  │ Prometheus │ │
│  │   :9100      │  every 5s      │   :9090    │ │
│  └──────────────┘                └────────────┘ │
│                                        ▲        │
│                                        │ queries │
│                                   ┌────────┐    │
│                                   │Grafana │    │
│                                   │  :3000 │    │
│                                   └────────┘    │
└─────────────────────────────────────────────────┘
          ▲                              ▲
          │                              │
     (optional)                    you, in browser
   raw metrics
```

The flow for every panel you see in Grafana:

1. You open a dashboard in your browser
2. Grafana sends a PromQL query to Prometheus (`http://prometheus:9090`)
3. Prometheus looks up the stored time-series data
4. Grafana receives the data points and renders the chart
5. Every 30 seconds (by default), Grafana refreshes and repeats

Meanwhile, in the background:

1. Every 5 seconds, Prometheus scrapes `http://node-exporter:9100/metrics`
2. Node Exporter reads `/proc` and `/sys` from the host and returns the metrics
3. Prometheus stores the values with timestamps

---

## Why This Architecture

You might wonder: why three separate services? Why not one program that does everything?

This is the **separation of concerns** principle applied to infrastructure:

- Node Exporter only knows how to read system metrics. It does one thing well.
- Prometheus only knows how to scrape, store, and query time-series data. It does one thing well.
- Grafana only knows how to visualize data from various sources. It does one thing well.

Because they're decoupled:
- You can replace Grafana with another visualization tool without touching Prometheus
- You can add other exporters (databases, web servers, your own app) without changing anything else
- You can run multiple Prometheus instances scraping the same exporters
- Each service can be scaled, updated, or restarted independently

This is the same principle behind microservices — small, focused components connected by well-defined interfaces (in this case, HTTP endpoints).

---

## What You Should Explore Next

Now that you understand how it works, here are the natural next steps:

1. **Write your own PromQL queries** — go to `http://<your-ip>:9090` and explore the metrics Node Exporter exposes. Try to answer: what is my current CPU usage? How much disk space is left?

2. **Build a panel from scratch in Grafana** — add a new panel to a dashboard, write a PromQL query, and choose a visualization type. This is where you'll really learn PromQL.

3. **Add alerting** — configure Prometheus alerting rules that fire when CPU goes above 80% or disk is below 10%. This is how on-call systems work in production.

4. **Instrument your own application** — use a Prometheus client library (available for Node.js, Python, Go, etc.) to expose custom metrics from an app you build. This is the real production use case.
