# netbird-grafana

A Grafana dashboard for self-hosted [NetBird](https://netbird.io) community edition.

Built against the metrics actually exposed by the OSS `netbird-server` binary on
`:9090` — no enterprise-only series, no metrics that don't exist in current
releases. Includes panels for the NetBird app, Traefik edge routing (optional),
and the underlying host.

Tested against **NetBird 0.71.2**. See [Compatibility](#compatibility) for notes
on earlier versions.

## What's in the dashboard

| Section | Panels |
|---|---|
| **Header stats** | Connected peers, relay active/idle, proxy connections, uptime, scrape health |
| **Peers** | Connected peers over time, active vs idle relay peers |
| **Management API** | Request rate by endpoint, response codes, error rate (5xx + auth failures), p50/p90/p99 latency, sortable per-endpoint p99 table |
| **Store (SQLite)** | Transaction p50/p90/p99 latency, transaction rate |
| **Account updates** | Peer-notification rate broken down by resource × operation |
| **Activity** | IdP `get_accounts` rate, PAT usage rate |
| **Reverse proxy** | Proxy connections over time, heartbeats per second |
| **Traefik (edge)** | Request rate / p99 latency / 4xx-5xx rate by router, open connections by entrypoint |
| **Host (EC2 or other)** | CPU %, memory used %, 1/5/15m load average, network throughput |
| **Process** *(collapsed)* | CPU, RSS, file descriptors, network I/O of `netbird-server` |
| **Go runtime** *(collapsed)* | Goroutines, GC pause p99, heap in use |

An annotation overlay marks every `netbird-server` restart so metric jumps stay interpretable.

## Prerequisites

- A NetBird OSS server reachable by your metrics agent. The server exposes
  Prometheus metrics on `:9090` by default — usually only on the internal Docker
  network, not the host.
- A Prometheus-compatible datasource in Grafana. Grafana Cloud's free tier is
  enough for a homelab.
- A metrics agent — [Grafana Alloy](https://grafana.com/docs/alloy/) or vanilla
  Prometheus in agent mode — running somewhere with network access to the
  NetBird server.

## Setup

### 1. Ship metrics to your Prometheus

Pick one of the example configs and adapt the placeholder hostnames:

- [`examples/alloy.config.alloy`](examples/alloy.config.alloy) — recommended;
  also collects host metrics via Alloy's built-in `node_exporter` equivalent
- [`examples/prometheus.yml`](examples/prometheus.yml) — vanilla Prometheus agent
  mode equivalent

Both ship metrics to a remote-write endpoint (Grafana Cloud or self-hosted) using
basic auth from environment variables. Don't bake credentials into the file —
keep them in a `.env` (gitignored).

### 2. Import the dashboard

In Grafana:

1. **Dashboards → New → Import**
2. Upload `netbird-oss.json` (or paste the contents)
3. When prompted, pick your Prometheus datasource

The `instance` dropdown at the top of the dashboard is data-driven: it auto-populates from
whatever `instance` labels Prometheus has seen, so adding a second NetBird deployment
just makes another option appear.

## Compatibility

- Built and tested against **NetBird 0.71.2**.
- Older releases will have most panels working but may show "No data" for any
  panel relying on metrics added in later versions (e.g. the **Store** row
  depends on `management_store_transaction_duration_ms_milliseconds`).
- Conversely, the `management_updatechannel_*` family that earlier dashboards
  used is **not present in 0.71.x**, so don't expect to find fan-out latency
  panels here.

The dashboard hardcodes exactly one job label — `up{job="netbird-server"}` in
the Scrape OK indicator. Everything else uses metric-name selectors, so it
composes with whatever scrape topology you have.

## License

[MIT](LICENSE).
