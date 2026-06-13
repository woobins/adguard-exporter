# Fork changes (`woobins/adguard-exporter@homelab-patches`)

Fork of [henrywhitaker3/adguard-exporter](https://github.com/henrywhitaker3/adguard-exporter),
branched from upstream `v1.2.1`. Deployed in the homelab via the `adguard-exporter`
Ansible role (`homelab-infra`) as a pinned build at `/opt/adguard-exporter/` on the
prometheus host. **Update this file when adding/removing patches.**

## Patches

### 1. De-duplicate DHCP lease label sets (fixes hard 500 on duplicate leases)

`internal/metrics/metrics.go` — `DhcpLeasesServer.Collect`

AdGuard Home can return the *same* DHCP lease row more than once:

- An in-app **"Update"** (in-place re-exec) rebuilds the lease table and can
  materialize duplicate rows.
- Clients behind a **MAC-NAT'ing wifi repeater** collapse onto a single bridge
  MAC, so multiple `ip/hostname` rows share one MAC and AdGuard records
  conflicting/duplicate leases.

The collector emitted one `prometheus.MustNewConstMetric` per lease with no
dedup, so a duplicate label tuple made the registry fail the entire `/metrics`
scrape with:

```
collected metric "adguard_dhcp_leases" { ... } was collected before with the same name and label values
```

Prometheus reads that 500 as the exporter being **down** (`InstanceDown`), even
though DNS/DHCP are healthy — a metrics-only fault escalating to a critical alert.

Fix: dedup on the full label tuple (`server,type,ip,mac,hostname,expires_at`) and
skip rows already emitted this scrape. Build the metric set under `d.mu` (the
worker writes `d.leases` under the same lock — `Collect` previously read it
unlocked, a data race), then release the lock before sending to the channel so a
slow scrape can't stall `Record`. The dedup key is an inline struct (allocation-
free, collision-proof). Regression test in `internal/metrics/metrics_test.go`
(uses a pedantic registry; verified it fails on the pre-fix collector).

Candidate for upstreaming.
