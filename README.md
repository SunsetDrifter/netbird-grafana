# netbird-grafana

A Grafana dashboard for self-hosted [NetBird](https://netbird.io) community edition.

Where NetBird's [official dashboards](https://docs.netbird.io/selfhosted/observability/dashboards)
are deep, per-component views (separate Management, Signal, and Relay
dashboards), this is **one at-a-glance dashboard for IT admins** — the most
important health signals up top, then logically ordered rows covering every
NetBird component, the optional Traefik edge, and the underlying host.

Tested against **NetBird 0.71.2**. The Signal, gRPC, and Relay-detail panels are
built from NetBird's current observability docs and may need a newer release.
See [Compatibility](#compatibility).

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

- A NetBird OSS server reachable by your metrics agent. Each component
  (Management, Signal, Relay) exposes Prometheus metrics on `:9090` by default —
  usually only on the internal Docker network, not the host. Scrape **all three**
  to light up the Signal and Relay rows (see [`EXAMPLES.md`](EXAMPLES.md)); the
  dashboard still works if you only scrape Management — Signal/Relay panels just
  show "No data".
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

- Built and tested against **NetBird 0.71.2**.
- Metric names were verified against the NetBird source instrument definitions
  (`management/server/telemetry`, `signal/metrics`, `relay/metrics`) rendered
  through NetBird's exact OpenTelemetry Prometheus exporter (`v0.64.0`). The
  rendering rules that matter: monotonic counters get `_total`; unit-`1`
  observable gauges get `_ratio` (e.g. `…connected_streams_ratio`); millisecond
  histograms render as `…_ms_milliseconds_bucket`; the network-map size metric
  carries its `objects` unit (`…object_count_objects_bucket`).
- These suffixes are produced by the OTel exporter version, which has changed
  across releases — so on a **much older or newer** NetBird, a panel may show
  "No data" if the suffix differs. To resolve: list your build's series with
  Grafana's metrics browser or `/api/v1/label/__name__/values` and adjust.
- Note: NetBird's own published per-component dashboards currently carry some
  stale suffixes (e.g. `…_counter_ratio_total`, `…_ms_bucket` for the gRPC
  metrics) that today's binary does not emit — this dashboard uses the
  source-verified names instead.

The dashboard scopes Signal panels by `job="netbird-signal"` (Signal's metric
names — `active_peers`, `registrations_total`, … — are unprefixed and would
otherwise collide with other exporters). The only other hardcoded job label is
`up{job="netbird-server"}` in the Scrape OK indicator. Everything else uses
metric-name selectors, so it composes with whatever scrape topology you have.

## License

[MIT](LICENSE).
