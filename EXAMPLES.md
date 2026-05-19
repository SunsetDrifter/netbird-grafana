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

Three scrape jobs in one agent:
- `netbird-server` — management / relay / process / Go series from NetBird itself
- `traefik` — router / entrypoint / service metrics (optional; omit if you don't
  front NetBird with Traefik)
- `host` (unix exporter) — CPU / memory / disk / network on the host running Alloy

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
