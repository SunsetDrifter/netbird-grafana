# netbird-grafana

A Grafana dashboard for self-hosted [NetBird](https://netbird.io) community edition.

Where NetBird's [official dashboards](https://docs.netbird.io/selfhosted/observability/dashboards)
are deep, per-component views (separate Management, Signal, and Relay
dashboards), this is **one at-a-glance dashboard for IT admins** — the most
important health signals up top, then logically ordered rows covering every
NetBird component, the optional Traefik edge, and the underlying host.

Tested against **NetBird 0.71.2** (combined and individual-component
deployments). See [Compatibility](#compatibility).

## What's in the dashboard

The top **at-a-glance strip** answers "is it healthy right now?" first:

- **Health:** Services up (UP only when *every* scraped component — Management,
  Signal, Relay — is healthy), uptime, restarts in the selected window.
- **Peers:** Connected peers (Management), Relay active/idle. Signal's peer count
  isn't a separate tile — every online peer holds both a Management and a Signal
  stream, so the two counts track each other; the **Peers** row plots "Signal
  active peers over time" next to "Connected peers over time" so a *divergence*
  (peers on Management but not Signal → Signal unreachable) stands out.
- **Host pressure:** CPU %, memory %, 1-minute load.

Then, top to bottom by admin priority:

| Section | Panels |
|---|---|
| **Peers** | Connected peers over time, Signal active peers over time, Relay peers (total/active/idle) |
| **Management API** | Request rate by endpoint, response codes, error rate (5xx + auth failures), p50/p90/p99 latency, sortable per-endpoint p99 table |
| **gRPC (peer sync)** | Sync / Login / GetServerKey request rate + p50/p90/p99 latency — the core peer heartbeat |
| **Signal** | Message-forward rate / failures / latency, registration & deregistration rate / failures / latency, `get_registration` latency, peer connection duration, gRPC RPC rate & p99 latency by method |
| **Relay** | Traffic bandwidth (sent/received), peer authentication latency, peer store latency |
| **Store** | Transaction p50/p90/p99 latency, transaction rate, global-lock acquisition latency, persistence latency (works for both SQLite and PostgreSQL backends) |
| **Account updates / network map** | Peer-notification rate by resource × operation, network-map build latency, peer-update fan-out latency, update-channel queue length, network-map size (object count) |
| **Activity** | IdP request rates by operation (authenticate / get_account / get_accounts / update_user_meta), PAT usage rate |
| **Reverse proxy** | Proxy connections over time, heartbeats per second |
| **Traefik (edge)** | Request rate / p99 latency / 4xx-5xx rate by router, open connections by entrypoint |
| **Host (EC2 or other)** | CPU %, memory used %, 1/5/15m load average, network throughput |
| **Process** *(collapsed)* | CPU, RSS, file descriptors, network I/O of `netbird-server` |
| **Go runtime** *(collapsed)* | Goroutines, GC pause p99, heap in use |

An annotation overlay marks every `netbird-server` restart so metric jumps stay interpretable.

## Prerequisites

- A NetBird OSS server reachable by your metrics agent, exposing Prometheus
  metrics on `:9090` (usually only on the internal Docker network, not the host).
  On the **combined** single-container image, Management + Signal + Relay all
  share that one `:9090`, so a single scrape job lights up every NetBird row. On
  a **separated** deployment, scrape each component's `:9090` as its own job to
  cover the Signal and Relay rows (see [`EXAMPLES.md`](EXAMPLES.md)).
- A Prometheus-compatible datasource in Grafana. Grafana Cloud's free tier is
  enough for a homelab.
- A metrics agent — [Grafana Alloy](https://grafana.com/docs/alloy/) or vanilla
  Prometheus in agent mode — running somewhere with network access to the
  NetBird server.

## Setup

### 1. Ship metrics to your Prometheus

See [`EXAMPLES.md`](EXAMPLES.md) for full agent configs — a Grafana Alloy
example (recommended; also collects host metrics via its built-in
`node_exporter` equivalent) and a vanilla Prometheus agent-mode equivalent.

Adapt the placeholder hostnames, and keep credentials in a `.env`
(gitignored) rather than the config itself.

### 2. Import the dashboard

In Grafana:

1. **Dashboards → New → Import**
2. Upload `netbird-oss.json` (or paste the contents)
3. When prompted, pick your Prometheus datasource

The `instance` dropdown at the top of the dashboard is data-driven: it auto-populates from
whatever `instance` labels Prometheus has seen, so adding a second NetBird deployment
just makes another option appear. Give the Signal and Relay scrape jobs the
**same `instance` label** as Management so a single `instance` selection covers
all three components (the `EXAMPLES.md` configs do this).

## Compatibility

- Built and tested against **NetBird 0.71.2**; metric names verified against the
  NetBird source instruments rendered through its OTel Prometheus exporter.
- Metric suffixes are set by NetBird's OTel exporter version, so a **much older or
  newer** build may rename a series and show "No data". To check, list your
  build's series via Grafana's metrics browser or `/api/v1/label/__name__/values`.
- **Combined vs. individual components.** Run separately, Signal emits unprefixed
  names (`active_peers`, …) under `job="netbird-signal"`. Run combined (the default
  `netbird-server` container), Signal instruments carry a `signal_` prefix under
  `job="netbird-server"`. Signal panels match both.
- **Scrape-job naming.** The Signal panels filter on `job=~"netbird.*"`, so any
  job name starting with `netbird` works — `netbird-server`, `netbird-signal`,
  `netbird-management`, whatever you already call it. Name the job something
  without that prefix and the Signal rows render "No data" while every other row
  keeps working, since the rest of the dashboard filters on `instance` alone.
- A counter/histogram only appears after its first observation, so some panels read
  "No data" on an idle server until the activity occurs. The "gRPC by method" panels
  need `rpc_server_*` (otelgrpc), which 0.71.2's combined server doesn't emit.

## License

[MIT](LICENSE).
