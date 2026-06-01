# Agent configuration examples

Two equivalent ways to feed the dashboard: Grafana Alloy (recommended) or
vanilla Prometheus in agent mode. Both ship metrics to a Prometheus-compatible
`remote_write` endpoint (Grafana Cloud or self-hosted) using basic auth.

In both examples, set the credentials in the environment, not in the file:

```sh
export GC_PROM_URL=https://prometheus-prod-XX-prod-REGION.grafana.net/api/prom/push
export GC_PROM_USER=...    # your stack's Prometheus username / instance id
export GC_PROM_TOKEN=...   # an access policy token with metrics:write scope
```

Replace `netbird.example.com` with a stable identifier for the NetBird
deployment — it becomes the `instance` label that the dashboard's `$instance`
dropdown filters on.

## Grafana Alloy

Scrape jobs in one agent:
- `netbird-server` — Management API / process / Go series from NetBird itself
- `netbird-signal` — Signal service (peer registration, message forwarding)
- `netbird-relay` — Relay service (relayed-traffic peers, bandwidth, latency)
- `traefik` — router / entrypoint / service metrics (optional; omit if you don't
  front NetBird with Traefik)
- `host` (unix exporter) — CPU / memory / disk / network on the host running Alloy

Give every job the **same `instance` label** so one `$instance` selection covers
the whole deployment.

There are two topologies, and the dashboard supports both:

- **Separated components** (this section's layout): Management, Signal, and Relay
  run as distinct services, each exposing its own `/metrics` on `:9090`. Scrape
  them as three jobs — `netbird-server`, `netbird-signal`, `netbird-relay`. Here
  the Signal binary emits its metrics **unprefixed** (`active_peers`,
  `registrations_total`, …), so the `netbird-signal` job label is what
  distinguishes them.
- **Combined single container** (the default `netbirdio/netbird-server` image):
  Management, Signal, and Relay share **one** meter on **one** `:9090`, so a
  single `netbird-server` scrape job captures everything — there is no `signal:9090`
  or `relay:9090` to scrape. In this mode the combined binary registers Signal
  instruments with a **`signal_` prefix** (`signal_active_peers`,
  `signal_registrations_total`, …); Relay metrics keep their `relay_` prefix.

To work across both, the dashboard's Signal panels match either name form and
either job, e.g.
`{__name__=~"(signal_)?registrations_total", job=~"netbird-(server|signal)"}`.
So for a combined deployment you only need the `netbird-server` job below; the
`netbird-signal` / `netbird-relay` jobs apply only to a separated deployment.

The unix exporter needs three read-only bind mounts on the agent container:
- `/proc:/host/proc:ro,rslave`
- `/sys:/host/sys:ro,rslave`
- `/:/host/rootfs:ro,rslave`

Without them, the exporter reads its own container's namespace instead of the
host's.

```hcl
// ---- NetBird application metrics ----
prometheus.scrape "netbird" {
  targets = [
    {
      __address__ = "netbird-server:9090",
      instance    = "netbird.example.com",
      job         = "netbird-server",
    },
  ]
  forward_to      = [prometheus.relabel.netbird.receiver]
  scrape_interval = "60s"
}

prometheus.relabel "netbird" {
  // NetBird emits otel_scope_* labels with empty values on every series —
  // drop them at the agent to save cardinality.
  rule {
    action = "labeldrop"
    regex  = "otel_scope_.*"
  }
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}

// ---- Signal service metrics (separated deployments only) ----
// Skip this block on a combined single-container deployment — Signal metrics
// already arrive on the netbird-server job above (prefixed "signal_").
// In a separated deployment, use job "netbird-signal" (unprefixed names).
prometheus.scrape "netbird_signal" {
  targets = [
    { __address__ = "signal:9090", instance = "netbird.example.com", job = "netbird-signal" },
  ]
  forward_to      = [prometheus.relabel.netbird.receiver]
  scrape_interval = "60s"
}

// ---- Relay service metrics ----
prometheus.scrape "netbird_relay" {
  targets = [
    { __address__ = "relay:9090", instance = "netbird.example.com", job = "netbird-relay" },
  ]
  forward_to      = [prometheus.relabel.netbird.receiver]
  scrape_interval = "60s"
}

// ---- Traefik metrics (optional) ----
// Enable on Traefik with:
//   --entrypoints.metrics.address=:8082
//   --metrics.prometheus=true
//   --metrics.prometheus.entrypoint=metrics
//   --metrics.prometheus.addrouterslabels=true
//   --metrics.prometheus.addserviceslabels=true
prometheus.scrape "traefik" {
  targets = [
    { __address__ = "traefik:8082", instance = "netbird.example.com", job = "traefik" },
  ]
  forward_to      = [prometheus.remote_write.grafana_cloud.receiver]
  scrape_interval = "60s"
}

// ---- Host metrics via Alloy's built-in unix (node_exporter) integration ----
prometheus.exporter.unix "host" {
  rootfs_path = "/host/rootfs"
  procfs_path = "/host/proc"
  sysfs_path  = "/host/sys"

  set_collectors = ["cpu", "loadavg", "meminfo", "filesystem", "netdev", "diskstats", "uname", "time"]

  filesystem {
    fs_types_exclude     = "^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$"
    mount_points_exclude = "^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)"
  }
}

prometheus.scrape "host" {
  targets         = prometheus.exporter.unix.host.targets
  forward_to      = [prometheus.relabel.host.receiver]
  scrape_interval = "60s"
}

prometheus.relabel "host" {
  // Force the same instance label as the other jobs.
  rule {
    action       = "replace"
    target_label = "instance"
    replacement  = "netbird.example.com"
  }
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}

// ---- Remote write to Grafana Cloud (or any Prometheus remote_write endpoint) ----
prometheus.remote_write "grafana_cloud" {
  endpoint {
    name = "grafana-cloud"
    url  = sys.env("GC_PROM_URL")

    basic_auth {
      username = sys.env("GC_PROM_USER")
      password = sys.env("GC_PROM_TOKEN")
    }
  }
}
```

## Vanilla Prometheus (agent mode)

Run with: `prometheus --enable-feature=agent --config.file=prometheus.yml`

The host metrics scrape assumes you're running a separate `node_exporter`
sidecar (Prometheus doesn't bundle one the way Alloy does).

```yaml
global:
  scrape_interval: 60s
  external_labels:
    # Reused across jobs so the dashboard's $instance filter sees everything together.
    instance: netbird.example.com

scrape_configs:
  - job_name: netbird-server
    static_configs:
      - targets: ["netbird-server:9090"]
    metric_relabel_configs:
      # NetBird emits empty otel_scope_* labels; drop them to save cardinality.
      - regex: "otel_scope_.*"
        action: labeldrop

  # Signal service (separated deployments only). Skip on a combined
  # single-container deployment — Signal metrics arrive on netbird-server
  # (prefixed "signal_"). In a separated deployment, job netbird-signal
  # carries the unprefixed names.
  - job_name: netbird-signal
    static_configs:
      - targets: ["signal:9090"]
    metric_relabel_configs:
      - regex: "otel_scope_.*"
        action: labeldrop

  # Relay service.
  - job_name: netbird-relay
    static_configs:
      - targets: ["relay:9090"]
    metric_relabel_configs:
      - regex: "otel_scope_.*"
        action: labeldrop

  # Optional — omit if you don't run Traefik in front of NetBird.
  - job_name: traefik
    static_configs:
      - targets: ["traefik:8082"]

  # Host metrics — run a node_exporter sidecar and point this at it.
  - job_name: integrations/unix
    static_configs:
      - targets: ["node-exporter:9100"]

remote_write:
  - url: ${GC_PROM_URL}
    basic_auth:
      username: ${GC_PROM_USER}
      password: ${GC_PROM_TOKEN}
```
