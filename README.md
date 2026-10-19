# blackbox-exporter

This Helm chart deploys Prometheus Blackbox Exporter in a per-namespace pattern for RKE2 clusters running RHEL 9 with SELinux enabled. It includes SELinux-friendly pod security settings and a CiliumNetworkPolicy to allow metrics scraping and outbound probe traffic.

## Features

- Per-namespace deployment using a normal Helm release
- Prometheus Operator `Probe` resources for monitoring configured URLs
- `ServiceMonitor` for exporter metrics scraping
- SELinux-aware container defaults (`runAsNonRoot`, read-only root filesystem, seccomp)
- Cilium network policy for metrics ingress and egress to DNS and HTTP/HTTPS

## Install

```bash
helm install blackbox-exporter ./blackbox-exporter \
  --namespace app-namespace \
  --create-namespace \
  --values ./blackbox-exporter/values.yaml
```

## Configure URLs

Edit the `targets` list in `values.yaml` to add or remove URLs that should be probed:

```yaml
targets:
  - name: example
    url: https://example.com
    module: http_2xx_ssl_expiry
```

The chart creates a `Probe` resource for each entry, which allows the Prometheus stack in the `monitoring` namespace (for example `kube-prometheus-stack-1`) to scrape it.

## SSL certificate days remaining

For HTTPS targets, Blackbox Exporter exposes the metric `probe_ssl_earliest_cert_expiry`. To see how many days remain before expiry, use:

```promql
((probe_ssl_earliest_cert_expiry - time()) / 86400)
```

This returns the remaining certificate lifetime in days. A common alert threshold is:

```promql
probe_ssl_earliest_cert_expiry - time() < 30 * 86400
```

## TCP probe examples

The chart includes example TCP modules for a TN3270 service on port 1022 and PostgreSQL on port 5432.

```yaml
targets:
  - name: tn3270-1022
    url: 10.0.0.10:1022
    module: tcp_tn3270_1022

  - name: postgres-5432
    url: 10.0.0.10:5432
    module: tcp_postgres_5432
```

For a TN3270 listener, the safest first check is usually a basic TCP socket test because the exact banner or negotiation differs by vendor and environment. If you know the service emits a recognizable banner, you can add a `query_response` check.

```yaml
blackbox:
  config:
    modules:
      tcp_tn3270_1022:
        prober: tcp
        timeout: 5s
        tcp:
          preferred_ip_protocol: ip4

      tcp_postgres_5432:
        prober: tcp
        timeout: 5s
        tcp:
          preferred_ip_protocol: ip4
          query_response:
            - expect: "^PostgreSQL"
              send: ""
```

